## Phase 89: `BackfillScope` — đóng root-cause item 3 của `metap-demo-waf` finding thứ 9 (2026-09-17)

Trigger: chủ dự án hỏi "roadmap với audit thì cần làm gì k" ngay sau khi audit staleness được phát
hiện và sửa; được đề xuất WAF finding #9 là việc thật duy nhất còn "nợ" (kế hoạch 3 phần đã viết ở
`metap-demo-waf/CLAUDE.md`, item 1 đã đóng ở Phase 84, item 2 chỉ là câu hỏi cần chủ dự án chốt
hướng chứ không phải code, item 3 chưa làm) — chủ dự án xác nhận tiếp tục.

### Bối cảnh (đã ghi ở `metap-demo-waf/CLAUDE.md`'s 9th finding, không nhắc lại chi tiết ở đây)

Một service `Schema`-strategy reconcile bảng của chính nó lúc boot luôn truyền
`metap_control::PLATFORM_TENANT_ID` (`Uuid::nil()`) — sentinel không sở hữu dòng thật nào.
`backfill::run_batched_update`'s batch `SELECT` lại lọc cứng `WHERE t.tenant_id = $2` bằng đúng
sentinel đó — trên bảng dùng chung thật sự (nhiều tenant thật cùng ở 1 bảng vật lý), câu query này
khớp **0 dòng vĩnh viễn**, nhưng `mark_completed` vẫn chạy sau đó, không phân biệt được "xong thật"
với "chưa bao giờ có cơ hội tìm thấy gì".

### Hướng chọn: bỏ hẳn filter, không loop theo tenant

Kế hoạch gốc nêu 2 hướng. Chọn **"bỏ hẳn filter `tenant_id` cho caller đã biết trước bảng là dùng
chung"** thay vì "resolve tập tenant thật rồi loop từng tenant" — lý do:
- Bảng `reconciler_backfill_progress` đã khoá theo `(tenant_id, entity_name, op_id)`; giữ nguyên
  `tenant_id` truyền vào (sentinel) làm khoá ledger biểu diễn đúng nghĩa "backfill của cả bảng dùng
  chung này", không cần đổi schema hay thêm bảng theo dõi "đã xong tenant nào chưa" cho vòng lặp.
- Vòng lặp qua `SELECT DISTINCT tenant_id` cần tự xây thêm cơ chế resume-giữa-chừng-vòng-lặp (crash
  giữa tenant thứ 5/10 thì sao) — máy trạng thái mới, trong khi bỏ filter tái dùng nguyên vẹn
  keyset-pagination-theo-`id`-đã-đúng sẵn có, chỉ đổi đúng 1 mệnh đề `WHERE`.

`BackfillScope` (enum `SingleTenant`/`AllTenants`) thêm vào `metap-reconciler`, xuyên qua
`backfill::run_batched_update`/`run_heavy_backfill` → `executor::execute_with_scope` →
`reconcile::reconcile_with_scope` — **hàm cũ `reconcile()`/`execute()` giữ nguyên chữ ký, mặc định
`SingleTenant`**, không phải thêm tham số bắt buộc: 2 sibling repo (`metap-demo-jira`/`metap-demo-crm`)
gọi `reconcile()` trực tiếp trong `main.rs` của chính họ và nằm ngoài phạm vi truy cập của phiên
này — đổi chữ ký cũ sẽ làm 2 repo đó gãy build mà không ai sửa được ngay. Cùng cách tiếp cận
`mint_jwt`/`mint_oauth_access_token` (Phase 88) và `find_oidc_user`/`find_external_user` (Phase 88)
đã dùng trong phiên trước.

### Phát hiện thêm khi rà: bug không chỉ ở WAF

`metap-app::MetapApp::with_entities` (crate `metap-app`, dùng chung bởi `templates/metap-app`,
`../metap-lowcode`'s `lowcode-admin-api`/`control-api`, và chính 3 service WAF) **cũng luôn
reconcile bằng `PLATFORM_TENANT_ID`** cho mọi entity nó đăng ký — không phải điểm riêng của WAF, mà
là hành vi mặc định của builder dùng chung. Bảng `self.pool` trỏ tới luôn là pool platform dùng
chung (từ `bootstrap_platform`), nên mọi entity qua builder này **luôn** nằm trên 1 bảng vật lý dùng
chung giữa mọi tenant thật — `BackfillScope::AllTenants` là lựa chọn đúng, không có nhánh nào khác
hợp lý ở đây, nên hardcode thẳng trong `with_entities`, không cần tham số cấu hình thêm. Sửa 1 chỗ
này tự động đóng cùng gap cho mọi service dùng `MetapApp` builder, không chỉ WAF.

`reconciler-orchestrator` (`../metap-lowcode`) đã được kiểm tra riêng — **không dính bug này**:
`reconcile_one` resolve `tenant_id` thật của từng `(tenant, entity)` deployment qua
`router.pool_for(entity.tenant_id.into())`, không bao giờ dùng sentinel. Không cần sửa.

### Sự cố thật gặp lúc verify (ngoài phạm vi fix chính, sửa luôn vì đang chặn build)

`scanning-service`/`alerting-service`/`zones-service` (`metap-demo-waf`) đều build lỗi
`missing field 'metadata' in initializer of OptionalServeConfig` — lỗi có sẵn từ audit 04 A#10
(field `metadata` thêm vào struct này nhưng 3 service chưa cập nhật theo, đã ghi nhận trước đó
trong phiên OAuth2 nhưng chưa sửa vì ngoài phạm vi lúc đó). Sửa cả 3 (`metadata: state.metadata.clone()`)
vì đang chặn thẳng việc build/verify fix chính của phiên này — không phải mở rộng phạm vi tuỳ tiện.

### Verify

`cargo build/clippy(-D warnings)/fmt --check/test --workspace` sạch trên cả `metap` và
`metap-demo-waf/data-plane`. E2e mới (`metap-reconciler/tests/reconcile_postgres.rs`,
`all_tenants_scope_backfills_a_shared_table_reconciled_with_a_sentinel_tenant_id`) tái hiện đúng
bug sống: `reconcile()` (mặc định `SingleTenant`) với sentinel tenant trên bảng có 2 tenant thật →
cả 2 dòng vẫn `NULL` sau backfill (bug xác nhận còn nguyên trước fix); `reconcile_with_scope(...,
AllTenants)` → cả 2 dòng được backfill đúng giá trị, hội tụ `ops_applied: 0` ở lần chạy kế tiếp.
Toàn bộ 9 test `reconcile_postgres.rs` + 6 `migration_postgres.rs` + 9 `orchestrator_postgres.rs`
pass trên Postgres 16 native.

**2 fail môi trường có sẵn, không liên quan, xác nhận bằng `git stash` trên chính commit này**:
`migrate_postgres.rs`'s 2 test (`migrates_existing_records_rows_onto_a_dedicated_table_without_loss`,
`resumes_from_a_saved_checkpoint_after_a_simulated_crash`) — cả 2 thao tác trực tiếp bảng
`records`, đã bị xoá vật lý từ Phase 86 (`0033_drop_records_table.sql`). Đây là tàn dư test chưa
được dọn theo — cùng loại gap Phase 87 đã dọn cho 7 crate khác nhưng bỏ sót `migrate_postgres.rs`.
**Chưa sửa trong phiên này** (ngoài phạm vi finding #9) — ghi lại làm việc còn nợ.

### Còn nợ

- `migrate_postgres.rs`'s 2 test cần trỏ lại fixture khỏi bảng `records` đã xoá (như trên).
- Item 2 của kế hoạch gốc (`metap-reconciler`'s `introspect()` nên re-derive mọi tín hiệu hội tụ từ
  `pg_catalog` trực tiếp, hay giữ ledger + cross-check định kỳ) — câu hỏi kiến trúc, chưa chốt,
  không tự quyết trong phiên này.
- `migrate.rs::copy_generic_records`'s `mark_completed` bắn vô điều kiện bất kể copy được bao nhiêu
  dòng — gap dạng phòng thủ-chiều-sâu đã ghi nhận từ trước (`metap-demo-waf/CLAUDE.md`), hiện chưa
  có caller thật nào dùng sentinel nên chưa phải bug sống — không sửa trong phiên này.
