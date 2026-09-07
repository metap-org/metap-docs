## Phase 79: table-per-entity động cho entity low-code, migrate hết `metap-demo-crm`/
`metap-demo-waf` khỏi `records` chung, và 2 bugfix thật trong `metap-reconciler` (2026-09-07)

Bối cảnh: `metap-demo-crm`'s 4 entity code-authored và toàn bộ `metap-demo-jira` đã ở bảng riêng từ
lâu (Phase 36/45/21). Còn lại 2 nhóm vẫn dùng `records` chung: entity low-code của `metap-demo-crm`
(định nghĩa qua API `metap-lowcode`, không phải Rust) và toàn bộ 9 entity code-authored của
`metap-demo-waf`. Chủ dự án yêu cầu migrate hết + định hướng tương lai loại bỏ hẳn `records` khỏi
`metap` core.

### 1. `metap-lowcode`: dynamic table-per-entity (trước đây không tồn tại, không phải chỉ "chưa bật")

Phát hiện quan trọng đầu phiên: cơ chế DDL cho entity low-code **đã tồn tại một nửa nhưng không nối
dây** — `reconciler-orchestrator`'s `resolve_definition` đã ép entity thành
`qualified_table_name_for(...)` khi reconcile, nhưng override đó không bao giờ chạm tới
`MetadataRegistry` đang serve — `CrudService` vẫn đọc/ghi `records` mãi mãi, và không có API nào
enqueue job vào `reconciler_entity_deployments` cho entity low-code cả.

Đóng nốt phần dây nối (`metap-lowcode`):
- `LowCodeEntityDefinition` giờ có field `table_name` thật (JSON, nằm sẵn trong blob
  `low_code_entity_versions.definition` — không cần migration SQL riêng). `to_entity_definition()`
  hết hardcode `"records"`.
- `store::publish` (`resolve_table_name`): entity xuất bản lần đầu tiên được gán
  `qualified_table_name_for(name)` ngay, không bao giờ chạm `records`; version sau của cùng entity
  giữ nguyên `table_name` cũ (bảng vật lý không đổi theo version nội dung).
- Entity mới xuất bản lần đầu được enqueue qua `orchestrator::enqueue_deployment` (không reconcile
  inline) — tái dùng cơ chế lease/topo-sort có sẵn, quan trọng vì entity low-code có thể tham chiếu
  chéo nhau theo thứ tự tuỳ ý qua API.
- Route mới `POST /admin/lowcode/entities/{name}/migrate-to-dedicated-table` cho entity xuất bản
  trước khi tính năng này tồn tại (còn `"records"` với dữ liệu thật) — gọi
  `metap_reconciler::migrate_generic_to_dedicated` (đã có sẵn từ trước, reconcile + copy
  checkpointed trong 1 lệnh), rồi flip `table_name` + `apply_registry` trong cùng process (bắt buộc
  — 1 CLI process riêng sửa DB trực tiếp sẽ không đẩy được vào registry in-memory của server đang
  chạy).
- Guardrail thêm vào `metap-reconciler`: `check_table_name_length` — tên entity mangled quá 63 byte
  giờ bị từ chối rõ ràng thay vì Postgres âm thầm cắt ngắn (rủi ro đụng độ tên bảng thật khi tên
  entity đến từ operator qua API, khác trường hợp cũ luôn là literal Rust ngắn do dev tự đặt).

### 2. Migrate thật `metap-demo-crm` (9/10 entity low-code)

Chạy qua `crm-server` thật + admin token thật (không chỉ gọi thư viện): `hr.departments`
(1 row), `hr.employees` (2 row, tham chiếu `hr.departments`/`hr.locations`), `hr.locations`,
`hr.positions`, `helpdesk.tickets`, `demo.projects`, `ops.demo_a_fcbfb890`, `ops.demo_b_3ad50e6c`,
`bench.issues` — tất cả 0 dòng ở DB dev hiện tại ngoại trừ 2 entity đầu (khác tài liệu cũ ghi
`bench.issues` có ~300K dòng — không đúng cho DB dev thực tế lúc chạy phase này). Verify sống qua
`GET /api/hr.departments`/`GET /api/hr.employees`: `relatedDisplay` hydrate đúng xuyên bảng riêng.

**`demo.tickets` chưa migrate được** — tham chiếu `demo.projects`, nhưng `demo.projects` đang
disabled nên `validate_references()` từ chối coi nó tồn tại. Phụ thuộc có từ trước, không phải bug
phase này; cần quyết định bật `demo.projects` trước (ngoài phạm vi thuần "migrate bảng").

**Phát hiện thứ tự FK khi migrate qua route HTTP thủ công**: migrate `hr.employees` trước
`hr.locations` lỗi thật `relation "entities.hr_locations" does not exist` — đúng thiết kế đã biết
(FK build nhắm bảng đích bất kể đã reconcile hay chưa), route này không có topo-sort tự động (khác
đường `enqueue_deployment` dùng cho entity mới). Fix bằng cách thử lại theo đúng thứ tự.

**Chưa dọn**: 3 dòng cũ của `hr.departments`/`hr.employees` vẫn còn trong `records` — migrate chỉ
copy, không xoá nguồn (đúng thiết kế `migrate_generic_to_dedicated`). Cần lệnh `DELETE` riêng, có
chủ đích — chưa chạy trong phiên này (bị chặn bởi safety classifier khi thao tác trực tiếp trên DB
dev dùng chung, cần xác nhận rõ ràng từ operator).

### 3. Migrate thật `metap-demo-waf` (9/9 entity) — kèm sự cố mất dữ liệu tạm thời, đã khôi phục

`zones-service` (`waf.zones` → `waf.ddos_policies`/`waf.firewall_rules`), `scanning-service`
(`waf.scan_jobs` → `waf.scan_findings`), `alerting-service` (`waf.alert_policies` →
`waf.alert_notifications`; `waf.security_events`/`waf.incidents` độc lập, không có `Reference`).
Verify sống: boot 2-3 lần liên tiếp mỗi service, `ops_applied` hội tụ về 0 cho 9/9 entity sau khi
bugfix #4 dưới đây được áp dụng; e2e test `http_server.rs` của cả 3 service pass trên Postgres dev
thật.

**Sự cố thật, tự gây ra rồi tự khôi phục cùng phiên**: lượt đầu tin nhầm cả 9 entity đều 0 dòng ở
DB dev — suy từ "không có seed script" + 1 dòng cũ trong
`data-plane/docs/05-metap-technical-mapping.md`, **không trực tiếp query `records`**. Thực tế 2
tenant (`2e87cf98-46a7-473f-bebc-20307e17d3b3`, `9de4259e-dd15-44e2-a0ff-70d323ad0ae9`) có 25 dòng
dữ liệu thật từ lúc test portal trước đó. Flip `table_name` + reconcile bảng riêng rỗng mà không
copy dữ liệu khiến toàn bộ dữ liệu "biến mất" qua API (login vẫn được — auth không liên quan đến
dữ liệu entity — nhưng mọi list đều rỗng). Chủ dự án phát hiện và báo trực tiếp ("login vẫn được
nhưng dữ liệu mất, phần waf ý"). Khôi phục ngay trong phiên bằng `metap_reconciler::migrate_generic_to_dedicated`
theo từng (tenant, entity) — số dòng khớp `records` 100% cho 8/9 entity. `waf.ddos_policies` lộ
thêm 1 gap thật: `UNIQUE` constraint mới trên `zoneId` (bảng `records` cũ chưa từng enforce) đụng
2 dòng lịch sử đã soft-delete (`deleted=true`) trùng `zoneId` với 1 dòng còn sống — xử lý bằng cách
chỉ copy dòng `deleted=false` (đúng những gì API từng hiển thị, vì dòng soft-delete chưa từng thấy
được qua API). Gap tổng quát hơn còn lại, chưa fix: field `unique: true` lẽ ra nên là partial
unique index (`WHERE deleted = false`) thay vì blanket constraint, để 1 giá trị unique có thể tái
sử dụng sau khi record cũ bị soft-delete — ghi lại như follow-up, không tự ý mở rộng fix ngoài nhu
cầu khôi phục dữ liệu ngay lúc đó.

**Bài học rút ra, ghi lại rõ để không lặp lại**: bất kỳ lần migrate table-per-entity nào sau này
trên 1 bảng đã có traffic thật **bắt buộc** phải tự `SELECT entity, tenant_id, count(*) FROM
records WHERE entity = ... GROUP BY 1,2` trực tiếp trước khi flip `table_name` — không được suy
luận từ tài liệu hay "không thấy seed script nào".

### 4. Bốn bug thật, tìm thấy và fix ngay trong `metap` core (không phải workaround ở
`metap-demo-waf`) — 2 phát hiện ngay, 2 phát hiện sau khi user báo trực tiếp lỗi thật khi dùng
portal

`waf.ddos_policies.zoneId`/`waf.zones.hostname` là các field `unique: true` **đầu tiên trong toàn
bộ codebase** đi qua table-per-entity — phơi ra 4 bug tiềm ẩn từ trước, chưa test case nào chạm
tới:

1. `compile.rs` phát ra **2 cấu trúc unique riêng biệt** cho cùng 1 field (`IndexSpec` **và**
   `UniqueSpec`) — dư thừa, vì `ADD CONSTRAINT UNIQUE` đã tự có index backing riêng.
2. `diff.rs`'s bước dọn index thừa không biết index backing của 1 `UNIQUE`/`PRIMARY KEY` constraint
   không phải object độc lập — cứ đề xuất `DropIndexConcurrently` nhắm vào nó mỗi lần reconcile,
   Postgres âm thầm từ chối (không lỗi, không crash boot), nên `ops_applied` không bao giờ về 0.
3. **Phát hiện sau khi user báo lỗi trực tiếp trên portal** (`unique_violation` khi tạo lại
   `DdosPolicy` cho 1 zone đã từng xoá policy cũ): constraint `UNIQUE` không có khái niệm
   soft-delete — 1 dòng đã `deleted=true` vẫn giữ chỗ giá trị unique mãi mãi, chặn dòng mới hợp lệ.
   Fix gốc (không phải xoá dòng chặn cho qua): `unique: true` giờ là **partial unique index**
   (`WHERE deleted = false`), không còn blanket constraint — `schema.uniques`/`UniqueSpec` không
   còn được `compile()` dùng nữa.
4. **Phát hiện khi rà soát toàn bộ field `unique: true`** (theo yêu cầu user "rà lại toàn bộ waf,
   jira, crm"): `compile()`'s nhánh `searchable` `continue` vô điều kiện, bỏ qua hẳn logic xử lý
   `unique` cho field vừa `searchable` vừa `unique` — `waf.zones.hostname` mất hẳn ràng buộc unique
   ngay khi chuyển sang table-per-entity (trước đó vẫn được enforce trên `records` cũ). Chỉ có
   WAF có field `unique: true` (2 field: `ddos_policies.zoneId`, `zones.hostname`) — jira/crm's
   entity code-authored không có field nào.

**Bug thứ 5, cũng phát hiện khi user báo "lỗi trả ra cho trình duyệt rất thiếu context"**:
`metap-crud`'s `unique_violation` (map lỗi Postgres 409 kèm tên field) chỉ thử ĐÚNG 1 tiền tố
`uniq_records_<entity>_` (quy ước đặt tên của bảng `records` chung) — bảng riêng đặt tên
`uniq_<table>_<field>` (không có `records_`), nên MỌI entity đã ở table-per-entity (không chỉ WAF)
khi bị unique_violation đều chỉ trả về `{"code":"unique_violation"}` trống trơn, không tên field/
bảng — lỗi có từ trước, chỉ chưa ai chạm entity table-per-entity nào có field `unique: true` để lộ
ra. Fix: nhận `EntityDefinition` đầy đủ thay vì chỉ tên, chọn đúng tiền tố qua `is_dedicated`.

Cả 5 fix có unit test + e2e test mới chống regression (`compile.rs`, `reconcile_postgres.rs`,
`crud_service_postgres.rs`). Verify sống: `zones-service` hội tụ `ops_applied: 0` cho toàn bộ
entity qua nhiều lần boot liên tiếp, `waf.zones.hostname` unique thật lại, tạo trùng
`waf.ddos_policies` giờ trả về `field_errors: {"zoneId": [...]}` thay vì code trống. Build/clippy/
test sạch lại trên `metap`/`metap-lowcode`/`metap-demo-crm`/`metap-demo-jira`/`metap-demo-waf`.

### Còn lại / hướng tương lai (không làm trong phiên này)

- Xoá hẳn code path `records` chung khỏi `metap` core — đây là thay đổi breaking cho bất kỳ
  downstream nào còn muốn shared-schema tenancy, cần review riêng từng consumer trước, chưa lên
  lịch.
- `qualified_table_name_for` chưa tenant-scoped — 2 tenant `Schema`-strategy khác nhau dùng chung 1
  pool mà có entity low-code trùng tên sẽ tranh chấp cùng 1 bảng vật lý. Chưa gặp thật (DB dev hiện
  tại chỉ 1 tenant có dữ liệu low-code), nhưng là giới hạn thật cần biết trước khi nhân rộng.
- `demo.tickets` (CRM) chờ quyết định bật `demo.projects`.
- Dọn 3 dòng cũ trong `records` của `hr.departments`/`hr.employees` — chờ operator xác nhận.

Chi tiết CRM: `metap-demo-crm/docs/roadmap/46-lowcode-entities-table-per-entity.md`. Chi tiết WAF:
`metap-demo-waf/CLAUDE.md`'s domain-model section. Root `metap-org/CLAUDE.md`'s cross-repo
conventions bullet cập nhật cùng phiên.
