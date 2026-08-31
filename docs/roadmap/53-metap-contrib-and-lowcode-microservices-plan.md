## Phase 53: Đổi tên `metap-lowcode-platform` → `metap-lowcode`, thêm crate dùng chung `metap-contrib`, rà soát mono-repo microservices cho `metap-lowcode` (2026-08-31, in-progress)

Tiếp nối Phase 52 (tách tầng SaaS low-code control-plane). 3 phần việc:

**1. Đổi tên repo (xong).** `../metap-lowcode-platform` → `../metap-lowcode` theo đúng tên chủ dự
án chọn. Đổi tên cơ học: `mv` thư mục, sửa lại mọi path dependency trỏ vào nó
(`../metap-demo-crm/Cargo.toml`), sửa tham chiếu trong docs sống của `metap`
(`README.md`/`CLAUDE.md`/`docs/roadmap.md`) — docs lịch sử (`docs/roadmap/52-*.md`,
`docs/features/07-*.md`, đã `Trạng thái: done`) giữ nguyên tên cũ, đúng tiền lệ không sửa lịch sử.
Verify: `cargo build --workspace` sạch ở cả `metap-lowcode` và `metap-demo-crm` sau khi đổi.

**2. Crate `metap-contrib` (xong, chi tiết ở `docs/features/08-metap-contrib-common-crate.md`).**
Rà soát dependency graph thật tìm boilerplate lặp giữa `metap` và `metap-lowcode`, xác nhận 3 ứng
viên thật (HTTP client thiếu timeout — rủi ro hang thật, không chỉ trùng lặp; bearer-token parse
lặp 3 lần 3 error type; env-var-with-default lặp ~35 lần tổng cộng), tạo `crates/metap-contrib`
làm nơi chung — build/test/clippy sạch. **Chưa di trú call site cũ nào** sang dùng crate mới (theo
đúng yêu cầu "rà soát trước" của chủ dự án) — đó là follow-up riêng.

**3. Mono-repo microservices cho `metap-lowcode` — hướng đã chốt, T1 xong, T2-T6 chưa code.** Yêu
cầu ban đầu: tách `metap-lowcode-http`/`metap-control-http`/`reconciler-orchestrator` thành các
service độc lập trong cùng 1 repo (`metap-lowcode`). Phát hiện khi rà soát trước khi viết plan:

- **Mâu thuẫn với ADR đã ghi** (`docs/architectures/09-adr.md` dòng ~55): "Không tách microservice
  cho hướng SaaS multi-tenant" — quyết định trước đó chọn modular monolith vì `CrudService` (Dispatch
  contract sạch) đã "distributed-ready" mà chưa trả giá phân tán (mất ACID xuyên audit/outbox/lock),
  chỉ tách khi có tín hiệu cụ thể, không phải quyết định trả trước.
- **Blocker kỹ thuật thật, không chỉ về nguyên tắc**: `metap-lowcode-http::apply_registry` là nơi
  duy nhất gọi `.store()` lên `AppState.metadata` (`Arc<ArcSwap<MetadataRegistry>>`) — thiết kế này
  giả định `metap-lowcode-http` và tiến trình phục vụ `/api/:entity*` chạy **chung 1 process**. Tách
  `metap-lowcode-http` ra process riêng thì publish 1 entity sẽ không còn tự động có hiệu lực ở
  tiến trình phục vụ — cần một cơ chế phân phối registry mới (event qua `EventBus`, hoặc poll định
  kỳ giống `reconciler-orchestrator` đã làm) mà hiện chưa tồn tại.
- Ngược lại, `reconciler-orchestrator` (đã là process riêng từ đầu, chỉ đọc `pg_catalog`/ghi bảng
  reconciled) và `metap-control-http` (chỉ ghi `control.tenants`, đã có `RegistryCache` TTL 30s
  eventual-consistency sẵn) không có blocker tương tự.

**Quyết định chủ dự án** (sau khi trình 3 phương án): ADR "không tách microservice" chỉ áp cho
`metap`-core (business CRUD hot path cần ACID) — `metap-lowcode` là repo private SaaS riêng, không
chạm `CrudService`, chọn có chủ đích đi hướng ngược lại: microservice ngay từ đầu, để giao việc
song song rõ ràng cho nhiều agent code. Ghi lại phạm vi quyết định gốc bằng 1 bullet mới nối vào
`docs/architectures/09-adr.md` (không sửa bullet cũ — chỉ làm rõ phạm vi). Thiết kế đầy đủ +
task breakdown (T1-T6) ở `../metap-lowcode/docs/architecture.md`:
- **T1 (xong)**: tách vật lý `services/` (service thật) khỏi `crates/` (thư viện) trong
  `metap-lowcode`, move `reconciler-orchestrator` sang `services/reconciler-orchestrator` — build
  sạch, không đổi logic.
- **T2-T6 (chưa code)**: `metap_lowcode::build_live_registry` mới + outbox event
  `lowcode.registry.changed` (giải blocker registry distribution — chọn event-driven qua
  outbox/EventBus có sẵn, không phải poll mới, cùng pattern `notification-worker` đã dùng); 2
  binary mới `services/lowcode-admin-api`/`services/control-api`; subscriber trong
  `metap-demo-crm/src/main.rs` (rủi ro cao nhất — sửa repo khác, cần e2e Postgres+RabbitMQ thật);
  `metap_contrib::bootstrap` (tách sau khi có caller thứ 2/3 thật, không tách trước).
