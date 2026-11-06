## Phase 103: index theo tenant, unique theo tenant, field hệ thống, partition audit (2026-11-06)

Trigger: thiết kế database cho Management plane của sản phẩm WAAP (`metap-demo-waf`,
`docs/architecture/05-building-blocks/08-components/14-management-database.md`, ADR-0046). Khi rà
`metap-go` cho quy mô thiết kế của WAAP (khoảng 400.000 org, 1 triệu zone trong bảng dùng chung) thấy
5 thiếu sót, đặt mã MG1–MG5. Chủ dự án chốt: "làm theo cách tốt nhất, metap vẫn đang trong phase xây
app để thử nghiệm, WAAP chính là sản phẩm để cung cấp thêm tính năng đó", tức là sửa ở `metap-go`
thay vì vá bằng migration riêng trong sản phẩm.

### Đã làm

| # | Thiếu sót | Sửa |
|---|---|---|
| MG1 | Reconciler không tạo index nào bắt đầu bằng `tenant_id`, trong khi mọi truy vấn của `CrudService`/`QueryPlanner` đều lọc `tenant_id = $n` | Mọi bảng entity có `idx_<table>_tenant_id`. Index của field `indexed`/`sortable` (không unique) thành `(tenant_id, field)`. Cột FK không unique giữ index một cột để kiểm FK của chính Postgres (`ON DELETE RESTRICT`) vẫn dùng được index |
| MG2 | `unique` có hiệu lực trên toàn bảng: hai tenant dùng chung bảng không thể có cùng giá trị | `EntityField.UniqueScope` và `EntityUniqueConstraint.Scope`: `tenant` (**mặc định mới**, index unique dẫn đầu bằng `tenant_id`) hoặc `global` (giữ hành vi cũ, cho giá trị định danh ngoài tenant như hostname). `metadata.Validate` từ chối giá trị lạ và `uniqueScope` khi không có `unique: true` |
| MG4 | `metadata.audit_trail_entries` một bảng, không partition, không bao giờ xoá | Migration `0038`: partition theo tháng (`occurred_at`), PK thành `(id, occurred_at)`, partition `default` bắt mọi dòng ngoài tháng đã tạo nên insert không bao giờ lỗi. Hàm `metadata.ensure_audit_partitions(from_month, months_ahead)` tạo partition và chuyển dòng từ `default` vào đúng tháng. `cron-scheduler` gọi hằng ngày (`AUDIT_PARTITION_INTERVAL_SECONDS`, mặc định 86400; `AUDIT_PARTITION_MONTHS_AHEAD`, mặc định 3). **Không xoá dòng audit nào**, giữ đúng quyết định "audit không bao giờ prune"; partition chỉ để người vận hành có thể detach và archive cả tháng thay vì `DELETE` hàng giờ |
| MG5 | Field hệ thống (ví dụ `lastUsedAt` của API key) cập nhật qua CRUD sẽ tăng `version` và ghi audit, gây `version_conflict` cho người đang sửa | `EntityField.System`: `Create`/`Update` từ chối field này trong payload (`system_field`), `Update` giữ giá trị đang lưu trong DB (SQL đọc lại key hệ thống từ dòng hiện tại, không lấy bản đọc lúc precheck). `CrudService.SetSystemFields` ghi field hệ thống cho caller nội bộ: không kiểm quyền, không tăng `version`, không đổi `updated_at`, không outbox, không audit |

MG3 (loại entity "append-only có partition" trong reconciler) **không làm**: WAAP để các bảng lớn chỉ ghi
thêm (usage theo ngày, purge) ở migration SQL riêng, giống bảng nền tảng của metap. Làm khi có sản phẩm
thứ hai cần.

### Lỗi thật tìm thấy khi làm: `DROP INDEX` không ghi schema

Test e2e cho đường nâng cấp (index unique kiểu cũ, toàn bảng → kiểu mới theo tenant, cùng tên) không hội
tụ: mỗi lần reconcile vẫn còn 2 op. Gốc: `executor.go` sinh
`DROP INDEX CONCURRENTLY IF EXISTS "<tên>"` **không có schema**. Bảng table-per-entity nằm trong schema
sản phẩm (`entities`, `waf`, `jira`…) không có trong `search_path`, nên `DROP ... IF EXISTS` không tìm thấy
index và **im lặng không làm gì**; `CREATE ... IF NOT EXISTS` ngay sau đó cũng im lặng không làm gì vì
tên đã tồn tại. Đây là nguyên nhân gốc của gap "blanket constraint → partial index không hội tụ, chưa
root-cause" ghi trong `metap-demo-waf/CLAUDE.md` (finding thứ 6) và lặp lại ở `jira.projects.key`
(Phase 80, 82): cách xử lý tạm lúc đó là `DROP INDEX` bằng tay. Sửa: tên index trong `DROP` luôn đi kèm
schema của bảng.

### Ảnh hưởng tới sản phẩm đang dùng

- Mặc định unique đổi từ toàn bảng sang theo tenant. Lần reconcile đầu sau khi nâng sẽ dựng lại các
  index unique (drop + create concurrently, cùng tên). `metap-demo-waf` đặt `uniqueScope: global` cho
  `waf.zones.hostname` và `waf.domains.apexDomain` để giữ hành vi cũ; các field unique khác (`zoneId` của
  `waf.ddos_policies`, `jira.projects.key`) đúng nghĩa hơn khi theo tenant.
- `metap-lowcode` build và test pass không cần sửa.
- "Global" chỉ có hiệu lực trong một bảng vật lý: tenant `DedicatedDb` có bản sao bảng riêng, nên duy
  nhất toàn nền tảng qua nhiều database phải dùng registry ngoài tầng entity (WAAP: `waap_global`).

### Verify

- Unit: `reconciler/compile_test.go` (index tenant, unique theo tenant/global, composite, FK giữ index một
  cột), `metadata` validate.
- E2E trên PostgreSQL 16 thật (database mới migrate từ đầu và database đã có dữ liệu):
  `TestUniqueScopeTenantAndGlobalOnASharedTable` (cùng slug ở 2 tenant được, 2 lần trong 1 tenant bị chặn,
  hostname global bị chặn chéo tenant; nâng từ index kiểu cũ hội tụ sau 1 lượt — **fail trên code cũ với
  2 op mỗi lượt, pass sau khi sửa `DROP INDEX`**), `TestSystemFieldsAreWrittenOnlyBySetSystemFieldsWithoutVersionBumpOrAudit`,
  `TestAuditRowsRouteToMonthPartitionsAndDefaultRowsMoveWhenThePartitionIsCreated`; toàn bộ `go test ./...`
  với `DATABASE_URL` pass.
- `metap-demo-waf/data-plane/go` và `metap-lowcode/go` build, vet, test pass với bản này.
