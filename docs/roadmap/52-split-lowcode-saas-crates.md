## Phase 52: Tách tầng SaaS low-code control-plane ra repo riêng `metap-lowcode-platform` (2026-08-31)

Đúng việc `docs/features/07-split-lowcode-saas-crates.md` phân tích: `metap` (facade crate)
re-export không điều kiện cả tầng "core, entity-agnostic" lẫn tầng "SaaS low-code control-plane"
(định nghĩa entity qua UI/API, tự provision tenant qua HTTP). Verify bằng dependency graph thật +
đọc `main.rs` của cả 2 demo-app trước khi làm (không phải đoán): không crate core nào phụ thuộc
ngược lại `metap-lowcode`/`metap-lowcode-http`/`metap-control-http`/`reconciler-orchestrator`;
`../metap-demo-jira` là bằng chứng sống — app multi-tenant thật chạy hoàn toàn không đụng tầng
này; `../metap-demo-crm` là consumer duy nhất (mount cả `lowcode_http`/`control_http` router).

**1 điều chỉnh so với phân tích ban đầu, phát hiện lúc lên plan (`EnterPlanMode`)**: KHÔNG tách
`metap-control::provisioning.rs` (`provision_schema_tenant`/`provision_dedicated_db_tenant`) ra
khỏi `metap-control` như dự kiến ban đầu. Bằng chứng: `jira-server` — app không có tí low-code
nào — vẫn dùng đúng 2 hàm này (qua `dev-tools provision-tenant`) để provision tenant dedicated-DB
của chính nó. Provisioning tenant là năng lực multi-tenancy core, không phải năng lực low-code —
gộp nhầm bucket ở bước phân tích trước. Tách ra sẽ tạo dependency ngược xấu: `dev-tools` (ở lại
core) gọi thẳng `metap_control::provision_schema_tenant`, tách hàm này ra repo mới nghĩa là
`dev-tools` phải phụ thuộc ngược vào repo SaaS-extension.

**4 crate di chuyển nguyên trạng** (copy qua `git ls-files`, chỉ sửa path dependency, không đổi
logic) sang `../metap-lowcode-platform` (workspace Cargo mới, `crates/` chứa cả 4):
- `metap-lowcode` (1628 dòng) — chỉ phụ thuộc `metap-metadata` (core).
- `metap-lowcode-http` (997 dòng) — phụ thuộc `metap-http`/`metap-metadata`/`metap-peripherals`/
  `metap-permission` (core) + `metap-lowcode` (sibling cùng repo mới).
- `metap-control-http` (474 dòng) — phụ thuộc `metap-control`/`metap-http`/`metap-reconciler`
  (core).
- `reconciler-orchestrator` (743 dòng, crate + binary `metap-reconciler-orchestrator`) — phụ
  thuộc `metap-control`/`metap-infra`/`metap-metadata`/`metap-reconciler` (core) + `metap-lowcode`
  (sibling).

Path dependency vào core: `../../../metap/crates/X` (3 cấp — `metap-lowcode-platform/crates/X/`
nằm sâu hơn 2 cấp/root repo, khác với `metap-demo-crm`'s `../metap/crates/X` 1 cấp vì đó không có
tầng `crates/` con). `edition.workspace = true` mỗi crate con trỏ vào
`metap-lowcode-platform/Cargo.toml`'s `[workspace.package] edition = "2021"` mới.

Seam đã đúng sẵn trong core, không cần sửa gì: `metap-http`'s `AppState.metadata` là
`Arc<ArcSwap<MetadataRegistry>>` generic — `metap-http` không hề biết `metap-lowcode` tồn tại (0
dependency), chỉ expose slot `.store()`-được; `metap-lowcode-http::apply_registry` là nơi duy
nhất gọi `.store()`. `dev-tools`/`../metap-demo-jira` không cần sửa gì cả (verify bằng grep:
không có `use metap_lowcode`/`metap_control_http`/`reconciler_orchestrator` thật nào, chỉ
doc-comment nhắc).

**Thứ tự an toàn** (giống Phase 51 — tạo + verify build trước, xoá bản gốc sau): tạo
`metap-lowcode-platform/` + sửa path dep + `cargo build --workspace` sạch (trong khi `metap` vẫn
còn nguyên 4 crate) → xoá 4 crate khỏi `metap` (`git rm -r`) + bỏ 4 dòng workspace member + bỏ 3
dòng re-export trong facade `crates/metap/src/lib.rs` (`control_http`/`lowcode`/`lowcode_http` —
`reconciler-orchestrator` chưa từng được facade re-export nên không cần sửa gì thêm ở đó) + bỏ 3
dòng dependency trong `crates/metap/Cargo.toml` → `cargo build --workspace` lại, sạch → sửa
`../metap-demo-crm` (`Cargo.toml` thêm 3 path dep trỏ `../metap-lowcode-platform/crates/X`,
`main.rs` đổi `metap::lowcode::`/`metap::lowcode_http::`/`metap::control_http::` thành
`metap_lowcode::`/`metap_lowcode_http::`/`metap_control_http::`) → `cargo build` sạch cả
`metap-demo-crm` lẫn `metap-demo-jira` (không sửa gì, chỉ verify không vỡ) → `cargo test
--workspace` ở `metap` pass (unit test, không cần DB).

`CLAUDE.md` cập nhật: bullet `reconciler-orchestrator` cũ (mô tả chi tiết) rút gọn thành 1 câu
trỏ sang repo mới, giữ lại phần mô tả `metap-reconciler::orchestrator` module (vẫn ở core); bullet
`metap-control` sửa 2 chỗ nhắc `reconciler-orchestrator`/`metap-control-http` trỏ sang repo mới,
làm rõ provisioning tenant không phải low-code-specific; thêm note "No SaaS low-code
control-plane crates in this repo" cạnh note "No example apps" đã có từ Phase 51.

Chưa làm (ngoài phạm vi lượt này, giống các gap có sẵn ở `metap-demo-crm`/`jira`/`metap-demo-waf`):
CI cho `metap-lowcode-platform`, Docker build (path dependency chưa build được trong container),
e2e Postgres thật cho cả `metap`/`metap-lowcode-platform` sau khi tách (chỉ verify build + unit
test lượt này), git history thật cho repo mới (init mới, không giữ history — theo đúng tiền lệ
Phase 51 đã chọn).
