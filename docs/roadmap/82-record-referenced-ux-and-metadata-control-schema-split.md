## Phase 82: `record_referenced` delete UX + tách `public`/`entities` thành `metadata`/`control`/
`waf`/`crm`/`jira` thật (2026-09-08/09)

Bối cảnh: Phase 81 ghi lại 2 phát hiện chờ quyết định — chủ dự án trả lời qua AskUserQuestion và
yêu cầu làm luôn cả hai trong phiên này.

### 1. `record_referenced`: giữ block, cải thiện UX (không cascade-delete)

Xác nhận lại (Phase 81): guard chặn xoá đúng thiết kế, không phải bug. Chủ dự án chọn "giữ block,
thêm UX rõ ràng" thay vì cascade-delete hay endpoint force-delete riêng.

- `metap-crud/src/crud_service/helpers.rs`: `find_referencing_record` (trả về đúng 1 kết quả) đổi
  thành `find_referencing_records` (trả về **toàn bộ** bản ghi đang chặn, tối đa 50) —
  `ReferencingRecordHit { entity, field, id }`. Vẫn giữ tối ưu "1 query gộp mỗi bảng vật lý" từ
  code review 2026-08-22 (không quay lại N query/field).
- `metap-crud/src/crud_service/delete.rs`: build lỗi qua constructor mới
  `ServiceResult::err_with_message_and_field_errors` — key `"<entity>.<field>"`, value = danh sách
  id bản ghi chặn. **Tái dùng nguyên `field_errors`** (đã có sẵn, đã chạy xuyên suốt cả 35 điểm
  destructure `ServiceResult::Err` trong `metap-http`/`metap-graphql`/`metap-grpc`) thay vì thêm
  field mới vào enum — tránh lặp lại kiểu cascade ~50 điểm sửa như đợt `unique_constraints` (Phase
  79/80). Không đổi chữ ký `service_error_response` (47 điểm gọi), không đụng gRPC/GraphQL.
- `platform-ui`: component mới `ReferencedByErrorMessage.tsx` (cạnh `ApiErrorMessage.tsx`) — parse
  `fieldErrors` key `"<entity>.<field>"`, render list link tới từng bản ghi chặn qua
  `adapter.toRecordDetail(entity, id)` có sẵn. Áp dụng ở cả `RecordDetail.tsx` (trang chi tiết) và
  `GeneratedList.tsx` (xoá trực tiếp từ danh sách) — entity-agnostic, không riêng WAF, mọi app
  dùng `platform-ui` đều được lợi. Không tự browser-test (đúng quy ước repo), chỉ
  `tsc`/`oxlint`/`prettier` sạch.
- Test mới: `crud_service_postgres.rs`'s `delete_rejected_lists_every_blocking_record_not_just_the_first`
  — 2 bản ghi cùng field + 1 bản ghi field khác cùng chặn 1 target, xác nhận cả 3 đều lên danh
  sách, không chỉ bản ghi đầu tiên.

### 2. Schema layout: `public` → tách `metadata`/`control`/`entities`/`public` thật, apply live

Trace lại Phase 81 xác nhận: `public` giữ mọi bảng khung `metap`, `entities` dùng chung giữa
`crm`/`waf`. Chủ dự án chốt qua AskUserQuestion: bảng `low_code_*` gộp vào `control` (không phải
`metadata`), và **apply live luôn** (không chỉ xây cơ chế).

- Migration `0028_metadata_schema.sql`: tạo schema `metadata`, chuyển 17 bảng khung
  (`users`/`policies`/`outbox_events`/`cron_jobs`/`reconciler_*`/`tenant_configs`/...) từ `public`
  sang `metadata`. `records`/`attachments` **ở lại `public`** (dữ liệu tenant thật, không phải
  config platform — cùng loại với `entities.*`, khác với bảng khung). `_sqlx_migrations` không
  đụng tới.
- Migration đặt DB-level default `ALTER DATABASE %I SET search_path TO public, metadata, control`
  (dùng `current_database()`, không hardcode tên DB — chạy đúng trên cả DB chung lẫn từng DB riêng
  của tenant `dedicated_db`) — mọi kết nối không tự set search_path (dev-tools, cron-scheduler,
  outbox-publisher, `Router::pool_for`'s bare pool) tự động thấy được bảng khung mà không cần sửa
  code ở từng nơi. `Router::begin`'s `SET LOCAL search_path` (tenant `Schema`-strategy) đổi thành
  `TO {schema_name}, metadata, control` — `SET LOCAL` thay hẳn search_path cho transaction đó nên
  vẫn phải liệt kê tường minh, default cấp DB không tự cộng vào.

**Bug thật tự phát hiện qua e2e test, không phải chủ dự án báo**: gộp `low_code_*` vào `control`
theo đúng lựa chọn ban đầu làm hỏng `run_tick_reaches_a_dedicated_db_tenant_own_database`
(`reconciler-orchestrator`'s test) — lý do: `metap_control::provision_dedicated_db_tenant` xoá
hẳn schema `control` khỏi MỌI database riêng của tenant `dedicated_db` (`DROP SCHEMA control
CASCADE`), vì `control.tenants`/`tenant_hostnames` là dữ liệu platform toàn cục thật, không thuộc
về DB riêng của 1 tenant. Nhưng `low_code_entity_versions`/... lại là dữ liệu **theo từng tenant**
(1 tenant `dedicated_db` cần bản riêng của chính nó, không phải bản chung) — gộp chung schema với
`control.tenants` khiến nó bị xoá theo mỗi khi provision 1 tenant `dedicated_db` mới. Migration
sửa `0029_lowcode_tables_belong_in_metadata_not_control.sql`: chuyển 4 bảng `low_code_*`/
`lowcode_impersonation_sessions` từ `control` sang `metadata` (không bị `DROP SCHEMA` đụng tới).
**Bài học**: `control` không đơn thuần là "thuộc về metap-lowcode" — nó mang nghĩa "chỉ tồn tại ở
DB chung, bị xoá khỏi mọi DB riêng tenant", một ràng buộc không rõ ràng khi chỉ đọc tên schema.

Kèm 1 fix nhỏ liên quan: `metap_control::provision_dedicated_db_tenant` chạy migration xong dùng
lại đúng 1 connection cũ (`max_connections(1)`, mở trước khi `ALTER DATABASE ... SET search_path`
chạy) để query tiếp — connection đó không tự thấy default mới (chỉ áp dụng cho connection *mới*).
Thêm `SET search_path TO public, metadata, control` tường minh ngay sau `sqlx::migrate!` trên
chính pool đó.

`metap-reconciler`: thêm `qualified_table_name_in(entity_name, schema)` cạnh
`qualified_table_name_for` (giờ chỉ là wrapper gọi `qualified_table_name_in(name, ENTITY_SCHEMA)`)
— chuẩn bị sẵn cho việc `entities` tách theo app.

**Sự cố môi trường không liên quan tới thay đổi trên**: đang chạy e2e test thì database
`postgres` (mặc định, dùng làm maintenance DB để tạo/xoá database tạm trong test) biến mất khỏi
Postgres dev — không tìm được nguyên nhân từ code trong repo (không có chỗ nào `DROP DATABASE`
nhắm đúng tên `"postgres"`), nghi do tác động ngoài phiên này lên DB dev dùng chung. Tạo lại bằng
`CREATE DATABASE postgres;` (DB rỗng, an toàn), test pass lại bình thường sau đó.

### 3. `entities` → tách thật theo từng app (`waf`/`crm`/`jira`) — mở rộng scope giữa phiên, theo
yêu cầu trực tiếp của chủ dự án ("chưa tách schema mỗi application ra khỏi entities à?")

Ban đầu Phase 82 chỉ định làm B1/B2 (mục 2 ở trên) và để nguyên `entities` (per-tenant thật là việc
lớn, xếp vào tương lai). Chủ dự án ngắt giữa chừng, chỉ rõ: mức tối thiểu hợp lý là tách theo
**từng sản phẩm** (waf/crm/jira hiểu được ngay từ tên schema), không cần đợi tới per-tenant thật.
Đây là trục khác — và nhỏ hơn hẳn — so với per-tenant: mỗi bảng `entities.*` vốn đã chỉ thuộc về
đúng 1 app (tên bảng không đụng nhau giữa các app), nên tách theo app chỉ là đổi chỗ vật lý, không
phải chia lại dữ liệu theo `tenant_id` như per-tenant thật sẽ cần.

Thực hiện:
- 9 entity file `metap-demo-waf` + 4 entity file code-authored của `metap-demo-crm` (không đụng 10
  entity low-code — chúng đi qua `metap-lowcode`'s `resolve_table_name`, một tầng generic không
  biết "crm" là gì, hardcode "crm" vào đó sẽ vi phạm layering; để lại `entities` cho tới khi
  per-tenant thật giải quyết luôn) + 8 entity file `metap-demo-jira` (đã tách DB riêng nhưng vẫn
  đổi tên schema nội bộ cho nhất quán) — `table_name` đổi từ
  `qualified_table_name_for(name)` sang `qualified_table_name_in(name, "waf"|"crm"|"jira")`.
- **Bug thật tìm ra ngay khi build**: `compile()` (`metap-reconciler/src/compile.rs`) từ trước tới
  giờ **không hề đọc `entity.table_name`** — nó tự tính lại bằng
  `qualified_table_name_for(&entity.name)` (luôn ra `entities.*`), chỉ *tình cờ* khớp với
  `entity.table_name` vì mọi entity trước giờ đều set `table_name` bằng đúng hàm đó. Đổi
  `table_name` sang `qualified_table_name_in` phá vỡ sự trùng khớp ngầm này — reconciler sẽ vẫn
  quản lý DDL trên bảng `entities.*` cũ trong khi `CrudService` đọc/ghi bảng mới, 2 tiến trình
  âm thầm bất đồng. Fix gốc: `compile()` giờ dùng thẳng `entity.table_name`, và FK target
  (`Reference` field) cũng đổi từ luôn-`qualified_table_name_for` sang suy ra schema từ chính
  entity đang compile (`schema_of(&entity.table_name)`) rồi gọi `qualified_table_name_in(ref_entity,
  đó)` — giả định bảng được reference nằm cùng schema với bảng đang compile, đúng cho mọi
  `Reference` hiện có trong toàn bộ codebase (chưa entity nào reference chéo app). 2 test fixture
  (`metap-reconciler`'s `compile.rs`/`migrate_postgres.rs`) sửa theo, cả 2 trước giờ cũng dựa vào
  đúng hành vi ngầm này.
- **Dữ liệu thật được di chuyển bằng `ALTER TABLE ... SET SCHEMA`** (giữ nguyên toàn bộ dữ liệu,
  index, FK — không phải copy rồi bỏ bảng cũ như sự cố Phase 79), không đưa vào migration chung của
  `metap` (khác `metadata`/`control` — bảng `waf_zones`/`crm_customers` chỉ tồn tại ở DB đã từng
  reconcile chúng, chạy migration này trên 1 DB chưa từng có sẽ lỗi "relation does not exist"),
  chạy tay qua `docker exec psql` trên: DB dev chung (9 bảng WAF + 4 bảng CRM) và DB riêng
  `metap_jira_demo` (8 bảng jira).
- **Bug thứ 2 tự phát hiện lúc verify sống**: `jira.projects`'s field `key` (`unique: true`) có
  index unique kiểu blanket (tạo từ trước Phase 80's partial-unique-index fix — đây là lần đầu
  `jira.projects` được reconcile lại từ sau phase đó) — tái hiện đúng gap "transition-state" đã ghi
  ở Phase 80 (index cũ không tự chuyển sang partial qua reconcile bình thường), lần này ở jira chứ
  không phải WAF. **Sửa lại nhận định sai ở Phase 79/80**: "chỉ WAF có field `unique: true`" — thực
  ra jira cũng có (`project.key`), chỉ chưa lộ ra vì chưa reconcile lại kể từ khi field đó tồn tại.
  Workaround giống Phase 80: drop tay index cũ (qua block one-shot env-gated tạm thời trong
  `main.rs`, kèm thêm `sqlx` tạm vào `Cargo.toml` vì `jira-server` chưa từng cần gọi SQL thô — gỡ
  cả 2 ngay sau khi chạy xong 1 lần) rồi để reconcile tự xây lại đúng.
- 1 lần sed hàng loạt bị chặn giữa chừng bởi permission classifier khiến `sprint_entity.rs` (1/8
  file jira) sót lại chưa sửa — đây chính là nguyên nhân trực tiếp của bug FK-target ở trên lộ ra
  rõ nhất (build cache tưởng chừng cũ, hoá ra do thiếu đúng 1 file).

Verify sống: build/clippy(`-D warnings`)/test + e2e `--ignored` sạch lại lần cuối trên cả 5 repo;
boot tay từng service (`zones-service`/`scanning-service`/`alerting-service`/`crm-server`/
`jira-server`) 2 lần liên tiếp, xác nhận `ops_applied: 0` cho mọi entity ở lần boot thứ 2; đối
chiếu số dòng dữ liệu trước/sau di chuyển (`crm.crm_customers` 1684 dòng, `waf.waf_zones` 7 dòng —
khớp, không mất dữ liệu).

### Verify tổng thể

Build/clippy(`-D warnings`)/test + e2e `--ignored` sạch trên cả 5 repo Rust (`metap`,
`metap-lowcode`, `metap-demo-crm`, `metap-demo-jira`, `metap-demo-waf/data-plane`) sau khi áp toàn
bộ migration live — xác nhận qua truy vấn `\dn`/`information_schema` thật trên DB dev chung:
`metadata` 21 bảng, `control` 2 bảng (chỉ `tenants`/`tenant_hostnames`), `waf` 9 bảng, `crm` 4
bảng, `entities` còn lại 10 bảng (CRM low-code, chưa tách), `public` 7 (gồm `records`/
`attachments`/`_sqlx_migrations` + 2 bảng test debris + `pg_stat_statements`'s views). DB riêng
`metap_jira_demo`: schema `jira` 8 bảng, `entities` rỗng. `platform-ui` sạch `tsc`/`oxlint`/
`prettier`.

### Còn lại / hướng tương lai

- `metap-lowcode`'s entity low-code (10 bảng CRM hiện tại) vẫn ở `entities` — tầng generic của nó
  không biết "crm" là gì, cần một cơ chế khác (ví dụ suy ra schema từ tenant, không hardcode tên
  app) trước khi tách được mà không vi phạm layering.
- Per-tenant thật (nhiều tenant *cùng 1 app* tách riêng nhau, không chỉ tách theo app) vẫn chưa làm
  — cần: (a) gán `schema_name` riêng cho từng tenant `Schema`-strategy lúc provision (hiện luôn
  `"public"`), (b) tách bảng đang share theo cột `tenant_id` thành bảng riêng theo schema mới — lớn
  hơn hẳn việc đổi tên đã làm ở phase này, chưa lên lịch.
- Vẫn chưa xoá code path `records` khỏi `metap` core (hướng tương lai đã ghi từ Phase 79).
- Root cause thật của gap "transition-state" (index blanket không tự chuyển sang partial qua
  reconcile bình thường) vẫn chưa tìm — giờ đã tái hiện 2 lần độc lập (WAF Phase 80, jira Phase 82),
  càng đáng điều tra riêng ở tầng `executor.rs`.

Chi tiết `metap` core: `metap/CLAUDE.md`'s `metap-control`/`metap-crud`/`metap-reconciler` bullet.
Chi tiết WAF: `metap-demo-waf/CLAUDE.md`'s domain-model open-questions section.
