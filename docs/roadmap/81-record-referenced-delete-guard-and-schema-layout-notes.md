## Phase 81: 2 phát hiện ghi lại theo yêu cầu chủ dự án — **chỉ note, chưa fix/chưa làm**
(2026-09-08)

Cả 2 mục dưới đây là quyết định sản phẩm/kiến trúc chủ dự án muốn tự quyết sau, không phải bug code
cần fix ngay — ghi lại đúng như yêu cầu ("note lại trước" / "note chưa làm").

### 1. `DELETE /api/waf.zones/{id}` trả `record_referenced` — điều tra xong: đúng thiết kế, không
phải bug dữ liệu

Chủ dự án báo lỗi khi xoá 1 `waf.zones` thật trên portal (`d9eca273-8760-4fc3-b693-dc989e80a27c`,
tenant `2e87cf98-46a7-473f-bebc-20307e17d3b3`):

```json
{"error":{"code":"record_referenced","message":"This record is referenced by \"zoneId\" on \"waf.ddos_policies\" and cannot be deleted."}}
```

Trace trực tiếp qua Postgres dev thật (`docker exec metap-postgres-1 psql`, không suy luận từ
code): zone này `deleted=false` thật, và có đúng 1 dòng `entities.waf_ddos_policies` sống
(`deleted=false`) trỏ `zoneId` vào nó — **không phải dữ liệu rác/stale** từ sự cố khôi phục dữ liệu
ở Phase 79. `find_referencing_record` (`metap-crud/src/crud_service/helpers.rs`) đã lọc
`deleted = false` đúng ở cả 2 nhánh (`records` lẫn bảng riêng) — khác hẳn class bug unique-constraint
ở Phase 79/80, guard này **không có bug soft-delete-unaware**. Message trả về cũng đã đủ ngữ cảnh
(tên field + entity vi phạm), không lặp lại gap "lỗi thiếu context" của Phase 80.

Vậy đây không phải lỗi code — là câu hỏi thiết kế sản phẩm chưa quyết: `Zone` 1—0..1
`DdosPolicy` hiện là quan hệ "block xoá cha nếu con còn sống" (đúng behavior chung của
`find_referencing_record` cho mọi `Reference` field, không riêng cặp này). Với 1 policy sở hữu
gần như 1-1 bởi zone (không phải tham chiếu độc lập kiểu nhiều-nhiều), có 2 hướng khả dĩ, **chưa
chọn**:
- Cascade soft-delete `DdosPolicy`/`FirewallRule` khi xoá `Zone` sở hữu chúng, hoặc
- Giữ nguyên block, nhưng portal hướng dẫn rõ xoá policy/rule trước (hiện chỉ có lỗi JSON thô, chưa
  có UX riêng).

Không tự ý chọn hướng — để chủ dự án quyết định.

### 2. Schema layout trong Postgres — xác nhận đúng như chủ dự án nghi ngờ, qua truy vấn trực tiếp

Truy vấn `\dn` + `information_schema.tables` trên DB dev thật:

```
control    2 bảng   — control.tenants, control.tenant_hostnames
entities  23 bảng   — mọi entity table-per-entity của CẢ crm, jira(*), waf gộp chung 1 schema
public    28 bảng   — mọi bảng khung metap (users, policies, records, outbox_events, cron_jobs,
                       low_code_entity_versions, reconciler_*, ...) nằm chung schema mặc định
                       Postgres tạo sẵn, không có schema riêng cho "cấu hình platform"
```

(*) `jira` không lộ ra ở đây vì 2 tenant jira dùng `strategy=dedicated_db` (DB riêng qua
`dsn_secret_ref`), nên các bảng `jira_*` nằm ở 1 database Postgres khác hẳn — không xung đột.
Nhưng `control.tenants` cho thấy **mọi tenant `crm`/`waf` hiện tại đều `strategy=schema,
schema_name=public`** — tức đúng như chủ dự án nghi ngờ, `entities` schema đang bị chia sẻ thật
giữa 2 sản phẩm khác nhau (không chỉ giữa các tenant cùng 1 sản phẩm), chỉ chưa va chạm vì tên
bảng (`crm_customers` vs `waf_zones`, ...) tình cờ không trùng — không phải vì có cơ chế phân
tách nào.

Đây là gap đã ghi nhận trước ở dạng tổng quát hơn (`metap/CLAUDE.md`'s `metap-reconciler` bullet:
"`qualified_table_name_for` chưa tenant-scoped"; Phase 79/80's "Còn lại" section) nhưng chưa từng
được xác nhận sống là **cũng xuyên luôn cả ranh giới sản phẩm** (crm/waf), không chỉ ranh giới
tenant cùng sản phẩm. Ghi lại rõ thêm 1 lần cho đúng mức độ nghiêm trọng thật.

**Hướng chủ dự án muốn (chưa làm, đang note)**:
- Bảng khung `metap` (config/platform: `users`/`policies`/`records`/`outbox_events`/`cron_jobs`/
  `low_code_entity_*`/`reconciler_*`/...) → nên nằm ở **1 schema riêng tên `metadata`**, không
  phải `public` (schema mặc định Postgres tạo, không có ý nghĩa gì đặc biệt — hiện đang dùng nó vì
  chưa ai đổi).
- Schema `entities` → nên đổi thành tên **theo tenant/db** (ví dụ `<tenant_db_name>`) thay vì 1
  schema `entities` cố định dùng chung cho mọi tenant/mọi sản phẩm — đúng tinh thần thiết kế
  multi-tenant per-schema-hoặc-per-db đã có sẵn ở `TenantStrategy`, nhưng `qualified_table_name_for`
  (`metap-reconciler/src/compile.rs`) hiện hardcode `ENTITY_SCHEMA = "entities"` bất kể tenant.
- Schema `control` → thuộc về `metap-lowcode`. Chủ dự án giữ quyền quyết định tenant DB nằm ở đâu
  (DB mới hay schema khác) — không đề xuất hướng cụ thể ở đây.

**Chưa làm gì trong phiên này** — đây thuần là ghi nhận đã trace/verify sống, không phải kế hoạch
đã duyệt. Cả 2 mục cần chủ dự án quyết định hướng trước khi lên kế hoạch implement (đổi schema
layout là thay đổi có thể breaking cho mọi tenant hiện có, cần chiến lược migrate riêng, không làm
tuỳ tiện).
