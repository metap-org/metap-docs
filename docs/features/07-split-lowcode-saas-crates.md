# Tách phần SaaS low-code control-plane ra khỏi `metap` core

- **Trạng thái:** done
- **Người đề xuất:** chủ dự án, 2026-08-31 (sau khi tách `apps/crm-server`/`jira-server` ra 2 repo
  riêng — Phase 51, thấy rõ `metap` đang mix "core, entity-agnostic" với "SaaS low-code
  control-plane" trong cùng 1 facade crate)
- **Track sở hữu:** Backend Core
- **Phase roadmap liên quan:** Phase 52 (`docs/roadmap/52-split-lowcode-saas-crates.md`)

## Vấn đề / động lực

`metap` (facade crate, `crates/metap/src/lib.rs`) re-export không điều kiện toàn bộ 19 sub-crate,
gộp chung 2 tầng khác hẳn nhau:

1. **Core, entity-agnostic** — bất kỳ downstream app nào cũng cần: CRUD/workflow/permission/query
   cho entity code-authored, tenant routing cơ bản, gRPC/GraphQL transport.
2. **SaaS low-code control-plane** — chỉ cần khi xây 1 sản phẩm SaaS đa-tenant thật, cho phép
   khách hàng **tự định nghĩa entity qua UI/API** (không phải Rust code) và tự provision tenant.

Không có ranh giới compile-time nào giữa 2 tầng — mọi downstream app depend `metap` đều kéo theo
cả `metap-lowcode`/`metap-lowcode-http`/`metap-control-http` dù không dùng, không có Cargo feature
nào để tắt. Verify bằng dependency graph thật (`Cargo.toml` mọi crate) + đọc `main.rs` của cả 2
demo-app:

- **`metap-demo-jira` là bằng chứng sống**: 1 app multi-tenant thật (dedicated-DB tenant,
  `Router`/`SecretStore` đầy đủ) chạy hoàn toàn không đụng `metap::lowcode`/`lowcode_http`/
  `control_http`.
- **`metap-demo-crm`** mount cả `metap::lowcode_http::router()` lẫn `metap::control_http::router()`
  — là app duy nhất hiện có thật sự dùng tầng SaaS low-code.
- Chiều phụ thuộc chỉ 1 hướng: `grep` toàn bộ `crates/` xác nhận **không có crate core nào** (
  `metap-crud`/`metap-http`/`metap-metadata`/...) import bất kỳ thứ gì từ `metap-lowcode`/
  `metap-lowcode-http`/`metap-control-http` — điều kiện cần để tách sạch đã có sẵn trong code, chỉ
  chưa được phản ánh vào cấu trúc crate/repo.
- Trong `metap-control` (2626 dòng) còn 1 ranh giới nhỏ hơn: `router.rs` (Router thật,
  233 dòng) không gọi `provisioning.rs` (`provision_schema_tenant`/`provision_dedicated_db_tenant`,
  103 dòng) — 2 hàm provisioning chỉ được gọi bởi `dev-tools provision-tenant` và
  `metap-control-http`'s `POST /platform/tenants`, không phải bởi Router hay bất kỳ core crate nào.

## Phạm vi

**2 nhóm crate xác định được (bằng chứng dependency graph + demo-app usage, không phải đoán):**

Core — giữ trong `metap`/`metap` facade, không đổi:
`metap-infra` `metap-metadata` `metap-permission` `metap-query` `metap-workflow` `metap-crud`
`metap-http` `metap-control` (trừ `provisioning.rs`) `metap-cache` `metap-reconciler`
`metap-storage` `metap-attachments` `metap-dashboards` `metap-peripherals` `metap-auth`
`metap-grpc` `metap-graphql`/`graphql-http` `metap-jwks`/`jwks-http` + ops binary
(`outbox-publisher`/`notification-worker`/`cron-scheduler`/`db-migrate`) + phần lớn `dev-tools`
(`mint-token`/`seed-admin`/`create-user`/`gen-keys`).

SaaS low-code control-plane — đã tách (~3842 dòng Rust, không phải 3945 như ước tính ban đầu —
xem điều chỉnh dưới):
- `metap-lowcode` (1628 dòng) — entity định nghĩa qua API, draft/publish/version.
- `metap-lowcode-http` (997 dòng) — admin API cho low-code builder (11 route,
  `/admin/lowcode/entities/...`).
- `metap-control-http` (474 dòng) — admin API tenant lifecycle (`/platform/tenants`,
  wave-rollout, 4 route).
- `reconciler-orchestrator` (743 dòng, crate + binary) — fleet-wide reconcile cho entity
  **low-code đã publish**, không liên quan entity code-authored (mỗi app code-authored tự
  `reconcile()` ở boot, không qua orchestrator này).

**Điều chỉnh phát hiện lúc lên plan (`EnterPlanMode`, không phải ở bước phân tích này) — KHÔNG
tách**: `metap-control::provisioning` (103 dòng) và `dev-tools`'s 2 subcommand
`provision-tenant`/`enqueue-reconcile`. Bằng chứng: `jira-server` (0% low-code) vẫn dùng
`provision_schema_tenant`/`provision_dedicated_db_tenant` qua `dev-tools provision-tenant` để
provision tenant dedicated-DB của chính nó — đây là năng lực multi-tenancy core, không phải
low-code-specific. Tách sẽ tạo dependency ngược xấu: `dev-tools` (core) gọi thẳng
`metap_control::provision_schema_tenant`, tách hàm ra repo mới nghĩa là `dev-tools` phải phụ
thuộc lùi vào repo SaaS-extension. Xem `docs/roadmap/52-split-lowcode-saas-crates.md`.

**Phương án đã chọn:** repo riêng, `../metap-lowcode-platform`, path dependency vào
`../metap/crates/*` — đúng pattern `../platform-ui`/`../design-system` và
`../metap-demo-crm`/`../metap-demo-jira` (Phase 51) đã dùng.

## Tiêu chí chấp nhận

- `cargo build --workspace` sạch ở cả `metap` (sau khi xoá 4 crate) và `metap-lowcode-platform`
  (repo mới) — đạt.
- `cargo build` sạch ở `metap-demo-crm` (đổi sang path dependency trực tiếp vào
  `metap-lowcode-platform`, bỏ qua facade `metap`) và `metap-demo-jira` (không sửa gì, verify
  không vỡ) — đạt.
- `cargo test --workspace` ở `metap` pass (unit test, không cần DB) — đạt.
- `CLAUDE.md`/`README.md` phản ánh đúng cấu trúc mới — đạt.

## Ranh giới kiến trúc bị đụng tới

`docs/architectures/05-building-blocks.md`'s layering không đổi (đây là tách theo chiều ngang —
core vs SaaS-control-plane — không phải đổi layer routes→service→core). Nếu chọn tách repo, cần
ADR mới trong `docs/architectures/09-adr.md` (tiền lệ: ADR đã ghi cho lần tách frontend library
2026-08-28 và lần tách demo app 2026-08-31, Phase 51).

## Rủi ro / phụ thuộc

- `metap-demo-crm` là consumer thật duy nhất của tầng SaaS low-code — đã verify build sạch sau
  khi tách (giống cách Phase 51 verify build thật trước khi xoá bản gốc).
- `reconciler-orchestrator` phụ thuộc `metap-lowcode` (`get_published`) — cả 2 chuyển cùng lúc
  sang `metap-lowcode-platform`, `metap-lowcode` là sibling-crate path dependency của
  `reconciler-orchestrator` trong cùng repo mới.
- Repo `metap-demo-waf` (WAAP, đang ở giai đoạn chỉ có doc, chưa code) dự kiến sẽ cần multi-tenant
  SaaS thật — khi tới lúc code, nó sẽ depend `metap-lowcode-platform` giống `metap-demo-crm` đang
  làm, chưa cần việc gì thêm ở đây.
