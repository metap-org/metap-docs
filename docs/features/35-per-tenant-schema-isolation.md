# Real per-tenant schema isolation cho `TenantStrategy::Schema`

- **Trạng thái:** done (2026-09-11) — `metap` commit chứa `crates/metap-control/src/tenant_schema.rs`
  (mới) + `crates/metap-control/src/{provisioning,lib}.rs` +
  `crates/metap-control/tests/provisioning_postgres.rs`.
- **Người đề xuất:** chủ dự án, nợ kỹ thuật #2 flagged từ Phase 82
  (`../roadmap/82-record-referenced-ux-and-metadata-control-schema-split.md`'s "Còn lại") — khác
  với Phase 82's tách theo **product** (`waf`/`crm`/`jira`), đây là tách theo **từng tenant** thật.
- **Track sở hữu:** Backend Core
- **Phase roadmap liên quan:** không thuộc phase nào

## Vấn đề / động lực

`Router::begin()`'s cơ chế `SET LOCAL search_path TO {schema_name}, metadata, control` đã tổng
quát sẵn cho per-tenant schema thật — nhưng `provision_schema_tenant` (`metap-control`) từ trước
tới giờ luôn hardcode `schema_name = "public"` cho mọi tenant `Schema`-strategy. Kết quả: mọi
tenant hiện có (ở mọi môi trường từng thấy) đều nằm chung `public`, chỉ cô lập bằng cột
`tenant_id`, không phải bằng schema vật lý riêng.

**Phát hiện khi làm, không chỉ lý thuyết**: đây không đơn thuần là cải thiện cô lập. Với 1 tenant
có `schema_name` thật sự khác `"public"`, search path của `Router::begin()`
(`"{schema_name}, metadata, control"`) **không hề có `public` trong đó** — nghĩa là 1 query không
schema-qualify tới `records`/`policies`/`users`/... trong transaction của tenant đó sẽ không tìm
thấy gì và lỗi, trừ khi các bảng đó thật sự tồn tại vật lý trong `{schema_name}`. Cô lập schema
thật là điều kiện cần để 1 tenant `Schema` không phải `"public"` hoạt động được, không phải chỉ là
polish thêm.

Chủ dự án chốt phạm vi (qua AskUserQuestion trong phiên): tách **toàn bộ** bảng có cột
`tenant_id` (không chỉ `records`/`attachments`), và xây đầy đủ cơ chế ngay dù chưa có tenant thật
nào cần gấp — chưa cần migrate dữ liệu thật (mọi tenant cũ vẫn ở `public`, không đụng).

## Phạm vi

**Trong phạm vi — 16 bảng tìm được qua introspect DB thật** (không grep lịch sử migration, vì
migration trộn lẫn `CREATE TABLE` với `ALTER TABLE ADD COLUMN` qua nhiều file):
```sql
SELECT table_schema, table_name FROM information_schema.columns WHERE column_name = 'tenant_id';
```
`metadata.{cron_job_runs, cron_jobs, dashboard_configs, policies, reconciler_backfill_progress,
reconciler_entity_deployments, reconciler_entity_status, tenant_auth_configs, tenant_configs,
user_preferences, user_roles, users, workflow_events, workflow_runs}` + `public.{attachments,
records}`.

**Loại trừ có chủ đích**:
- `control.tenant_hostnames` — có cột `tenant_id` nhưng bản thân nó là config platform toàn cục
  (hostname nào map tới tenant nào), cùng loại với `control.tenants` — không copy theo tenant.
- `metadata.outbox_events` — **không có cột `tenant_id`** (xác nhận qua introspect) — 1 bảng
  dùng chung, 1 `outbox-publisher` drain bất kể tenant, đúng như thiết kế.

**Cơ chế** (`crates/metap-control/src/tenant_schema.rs`, module mới):
- `schema_name = "t_" + tenant_id.simple()` (32 hex thường, khớp sẵn whitelist
  `Router::validate_schema_name`'s `^t_[a-z0-9]+$`, tất định, không đụng nhau vì 1-1 với UUID).
- `create_tenant_schema(pool, schema_name)`: `CREATE SCHEMA IF NOT EXISTS`, rồi
  `CREATE TABLE IF NOT EXISTS {schema}.{table} (LIKE {source} INCLUDING DEFAULTS INCLUDING
  CONSTRAINTS INCLUDING INDEXES)` cho từng bảng trong danh sách 16 (danh sách **hardcode, review
  được** trong Rust — không tự suy từ `information_schema` lúc chạy, để 1 bảng mới có `tenant_id`
  bắt buộc phải sửa file này mới được cô lập, không âm thầm thiếu). `LIKE` không bao giờ copy
  foreign key dù `INCLUDING` gì — 3 FK thật giữa các bảng này (`cron_job_runs.job_id` →
  `cron_jobs.id`, `workflow_runs.cron_job_run_id` → `cron_job_runs.id`, `workflow_runs.job_id` →
  `cron_jobs.id`, cả 3 đều `ON DELETE CASCADE` ở bảng gốc) được `ADD CONSTRAINT` lại tay, đúng
  `ON DELETE CASCADE` khớp bảng gốc.
- **Idempotent có chủ đích** (`IF NOT EXISTS` xuyên suốt, `ADD CONSTRAINT` dung thứ lỗi `42710`
  giống hệt `metap-reconciler::executor::ensure_schema_exists` đã làm cho race tạo schema) — không
  phải để chống race (provision 1 tenant không phải hot path đáng lo), mà để gọi
  `provision_schema_tenant` 2 lần cho cùng 1 `tenant_id` vẫn lỗi đúng tại bước INSERT
  `control.tenants` (unique_violation) như trước khi có tính năng này — `metap-control-http`'s
  `duplicate_tenant_id_response` (ở `metap-lowcode`) downcast đúng lỗi đó để trả `409` sạch;
  nếu DDL tạo schema/bảng thất bại trước vì đã tồn tại, downcast đó sẽ không nhận ra, rơi về `500`
  chung chung — bug thật tìm ra qua test hồi quy sẵn có (`provisioning_a_duplicate_tenant_id_fails_
  with_a_downcastable_unique_violation`), không phải suy đoán.
- `provision_schema_tenant`: gọi `create_tenant_schema` **trước** khi ghi `control.tenants`
  (đúng thứ tự `provision_dedicated_db_tenant` đã dùng — làm xong mọi DDL rồi mới ghi 1 dòng
  registry duy nhất, không có tenant "tồn tại" trước khi schema của nó tồn tại). Seed admin
  user/auth-config giờ chạy qua 1 connection có `SET search_path TO {schema}, metadata, control`
  tường minh (trước đây chạy thẳng qua `shared_pool`, đúng chỉ vì `schema_name` luôn là `"public"`
  — mặc định của chính pool đó).

**Ngoài phạm vi, cố ý chưa làm**:
- Migrate dữ liệu tenant cũ (đang ở `public`) sang schema riêng — không có tenant thật nào cần,
  theo đúng quyết định của chủ dự án.
- `Router::pool_for()`'s gap có sẵn (trả pool thô, không `SET search_path`, cho strategy `Schema`)
  — giờ mới thật sự "lộ ra" (trước không sao vì mọi tenant đều `public`, khớp default của pool).
  Chỉ 3 chỗ gọi `pool_for()` (`dev-tools enqueue-reconcile`/`migrate-to-dedicated-table`/
  `reconcile-plan`), luôn thao tác bảng entity (đã schema-qualify sẵn như `waf.zones`) trừ default
  `records` — đường HTTP/`CrudService` thật không bị ảnh hưởng (luôn qua `Router::begin()`'s
  transaction, set đúng search_path). Ghi nhận, không sửa trong đợt này — sửa đúng cần quyết định
  lại shape trả về của `pool_for()` (`PgPool` hiện tại không mang được session state).
- Schema drift theo thời gian: 1 migration mới thêm cột vào `policies` chỉ cập nhật bản gốc
  `metadata.policies` — mọi schema tenant đã tạo cần `ALTER` lại tay, chưa có cơ chế tự động
  (tương đương việc `metap-reconciler`'s orchestrator đã giải cho bảng entity, chưa có bản tương
  đương cho các bảng platform viết tay này). Chưa lên lịch.
- `PLATFORM_TENANT_ID` (`Uuid::nil()`) tiếp tục resolve về `public` mãi mãi qua fallback có sẵn
  của `Router::resolve()` ("không có dòng `control.tenants`" → `Schema{"public"}"`) — sentinel,
  không bao giờ được provision như tenant thật, không bao giờ có schema riêng.

## Tiêu chí chấp nhận

- `cargo build/clippy --all-targets -- -D warnings` sạch toàn workspace `metap`.
- Test e2e mới (`provisioning_postgres.rs`'s
  `provision_schema_tenant_creates_isolated_real_tables_with_working_fks`): 2 tenant provision
  qua `provision_schema_tenant`, ghi 1 dòng `policies` vào transaction của tenant A qua
  `Router::begin()` thật (đường HTTP thật dùng, không tắt qua schema-qualify tay), xác nhận tenant
  B không thấy dòng đó; xác nhận FK `cron_job_runs.job_id` được enforce thật trong schema mới
  (insert tham chiếu id có thật → thành công, tham chiếu id không tồn tại → bị từ chối).
- Test cũ (`provision_schema_tenant_writes_registry_row_and_admin_user`,
  `provisioning_a_duplicate_tenant_id_fails_with_a_downcastable_unique_violation`,
  `list_returns_every_provisioned_tenant`, `set_status_to_suspended_is_immediately_enforced_by_
  router`, `deprovisioning_is_immediately_enforced_by_router_with_a_404_not_a_403`) cập nhật đúng
  hành vi mới (`schema_name` giờ `t_<uuid>`, không còn `"public"`) và vẫn pass — kể cả ca
  duplicate-tenant-id, xác nhận idempotency fix ở trên hoạt động đúng.
- `cargo test -p metap-control -- --ignored` sạch (trừ `vault_store.rs`'s 5 test — cần Vault
  server thật, không chạy được trong môi trường phiên này, không liên quan thay đổi).

## Ranh giới kiến trúc bị đụng tới

`crates/metap-control` (`provisioning.rs`, `tenant_schema.rs` mới, `lib.rs`'s export list) — không
đổi public API của `Router`/`CrudService` phía đọc, chỉ đổi hành vi ghi lúc provision. Không cần
ADR — không đổi mô hình routing đã có (`TenantStrategy::Schema { schema_name }` đã tồn tại từ đầu,
đây chỉ là lần đầu thật sự gán giá trị khác `"public"`).

## Rủi ro / phụ thuộc

- Danh sách 16 bảng là hardcode có chủ đích (xem trên) — rủi ro thật là 1 bảng mới có `tenant_id`
  trong tương lai bị quên thêm vào `TENANT_SCOPED_TABLES`, âm thầm không được cô lập. Không có
  cảnh báo tự động cho việc này trong đợt này (không phải mục tiêu, xem "Ngoài phạm vi" ở trên).
- Không phụ thuộc feature khác. Không ảnh hưởng tenant cũ (`dedicated_db` không đụng gì; `schema`
  cũ vẫn `"public"`, hành vi y hệt trước) — chỉ tenant `Schema`-strategy provision **mới** từ giờ
  trở đi mới nhận schema riêng.
