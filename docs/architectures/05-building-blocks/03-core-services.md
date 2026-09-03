# 5.3 Whitebox: Core Services

[← 5. Building Block View](00-index.md)

## Metadata Registry

Sở hữu các entity definition:

- fields
- list views
- workflow
- index/search/sort hints

Metap validate và compile metadata như một runtime artifact hạng nhất, thay vì coi nó là một mô tả schema thụ động. `MetadataCompiler` thực thi điều này tại thời điểm `MetadataRegistry::register()` — field trùng lặp, tham chiếu field/filter/sort của listView bị treo (dangling), giá trị enum thiếu, và workflow shape sai định dạng đều khiến quá trình khởi động thất bại, chứ không phải đợi đến request đầu tiên. Mỗi entity có một hash xác định (deterministic) cho hình dạng của nó (`MetadataCompiler::hash`, gồm cả guard condition của từng workflow transition kể từ 2026-08-17), được expose dưới dạng `version` tại `GET /metadata/entities`; `MetadataDriftService` so sánh hash đó với hash được ghi nhận lần gần nhất mỗi khi boot và chỉ cảnh báo — không bao giờ crash — khi có drift, phản ánh đúng tinh thần graceful-degradation của health check. Cùng bản chiếu metadata an toàn đó cũng là nguồn cho tài liệu OpenAPI được sinh ra tại `GET /metadata/openapi.json` (viết tay trong `metap-metadata/src/openapi.rs`, được đồng bộ thủ công với các struct trong `entity.rs` — Rust không có bước runtime-reflection tương đương Zod).

## Control Plane (Router, Multi-Tenancy)

`metap-control` (`docs/multi-tenant-platform-design.md` §2.2, `docs/roadmap.md` Phase 16) — control plane cho SaaS multi-tenancy, sở hữu:

- **`control.tenants`** — registry một dòng mỗi tenant (`id`, `tier`, `strategy`, `schema_name`/`dsn_secret_ref`, `status`, `trial_expires_at`), qua `TenantRegistry` trait (`PostgresTenantRegistry` là impl duy nhất) + `RegistryCache` (moka, TTL 30s) phía trước để tránh query lại mỗi request.
- **`TenantStatus`** — `Provisioning`/`Active`/`Migrating`/`Suspended`/`Expired`/`Deleted` (terminal, chỉ set qua `DELETE /platform/tenants/{id}`, không bao giờ set qua `PATCH .../status`). Mỗi status không phải `Active` map sang một mã lỗi HTTP cụ thể ở `CrudService` (`router_unavailable`) — `Suspended`/`Expired` → 403, `Migrating`/`Provisioning` → 503, `Deleted` → 404 — thay vì rơi vào nhánh lỗi 500 chung.
- **`TenantStrategy`** — `Schema { schema_name }` (trial, thực tế luôn ghim `"public"` — isolation thật cần data-plane table-per-entity, chưa xây) hoặc `DedicatedDb { dsn_secret_ref }` (paid, đã có isolation vật lý thật).
- **`Router::begin(tenant_id)`** — điểm duy nhất mở một transaction tenant-scoped, thay cho `CrudService` nhận thẳng một `PgPool`. Với `Schema`: `SET LOCAL search_path` trên connection mượn từ pool chung, scoped theo transaction (không thể rò sang request tiếp theo dùng lại cùng connection — bẫy nghiêm trọng nhất của thiết kế này, đã fix explicit). Với `DedicatedDb`: mở/tái dùng một `PgPool` riêng cho tenant đó, cache theo `dsn_secret_ref` (moka, idle TTL 10 phút). Một tenant chưa có row `control.tenants` fallback về hành vi tương thích ngược: `{status: Active, strategy: Schema("public")}`.
- **`SecretStore`** trait (`EnvStore`/`VaultStore`) — resolve DSN cho `DedicatedDb`. `EnvStore` đọc thẳng biến môi trường tên đúng bằng `dsn_secret_ref`. `VaultStore` (`crates/metap-control/src/vault_store.rs`) gọi Vault KV v2 qua HTTP, hỗ trợ token tĩnh hoặc AppRole (ưu tiên AppRole nếu có cả hai) kèm auto-renewal (`renew_self` trước, fallback login lại bằng AppRole chỉ khi renew thất bại — tránh lỗi với role có `secret_id_num_uses=1`). Chi tiết vận hành + các câu hỏi production còn bỏ ngỏ ở [07. Deployment View](../07-deployment/00-index.md#secret-manager--secretstore--vaultstore-2026-08-17--2026-08-21).
- **`PostgresPolicyStore`** — sống ở `metap-control` chứ không phải `metap-permission` (thuần lý do dependency-cycle: `metap-permission -> metap-control` sẽ khép vòng lặp `metap-control -> metap-peripherals -> metap-metadata -> metap-permission`; trait `PolicyStore` vẫn ở `metap-permission`), mỗi lời gọi route qua `Router::begin`.
- **Provisioning** — `provision_schema_tenant`/`provision_dedicated_db_tenant` (ghi row `control.tenants`, chạy migration lên DB riêng khi `dedicated_db`, tạo admin user đầu tiên) dùng chung giữa `dev-tools provision-tenant` (CLI) và `POST /platform/tenants` (`metap-control-http`, gate bởi `PlatformAdminContext` — tenant sentinel `PLATFORM_TENANT_ID` + role `"platform_admin"`, khác `AdminContext` chỉ ủy quyền trong tenant của chính người gọi).
- **Delete/deprovision** (`DELETE /platform/tenants/{id}`) — chỉ detach routing (set `status: Deleted`, đóng dedicated pool nếu có), **không** tự động `DROP DATABASE` cho tenant `dedicated_db`, **không** tự động xóa data cho tenant `schema` — quyết định có chủ ý, tránh mất dữ liệu không thể hoàn tác qua một lời gọi API.

## CRUD Service

CRUD tổng quát cho các metadata entity (`metap-crud::CrudService`), là thứ duy nhất mà routes gọi để thao tác trên record.

Trách nhiệm:

- validate dữ liệu bằng validator dẫn xuất từ field metadata (`metap-crud/src/validation.rs`, thay thế cho các Zod schema riêng theo từng entity — không có một object validation-schema viết tay riêng biệt)
- thực thi permission thông qua `PermissionService`
- gọi query planner (`metap-query::plan_list`) cho list/search
- mở mọi transaction qua `Router::begin(tenant_id)` (xem "Control Plane" ở trên), không bao giờ nhận thẳng một `PgPool`
- lưu trữ record
- enqueue outbox event
- gọi các workflow function khi cần

## Permission Service

Lớp permission (`metap-permission::PermissionService`) sở hữu:

- tenant scope
- role assignment — động, lưu trong DB theo từng `(tenant_id, user_id)`, được grant/revoke ngay tại runtime qua HTTP API có bảo vệ admin (`crates/metap-http/src/routes/admin.rs`, bọc `metap-peripherals::assign_role`/`revoke_role`/`list_users`); bản thân JWT chỉ là một khẳng định danh tính trần trụi (bare identity assertion), không mang theo role
- policy storage — một allow-list theo role kết hợp với một attribute condition tùy chọn (`PolicyCondition`), đứng sau trait `PolicyStore` (`PostgresPolicyStore`, sống ở `metap-control` — xem "Control Plane" ở trên — là implementation duy nhất hiện nay). Mỗi policy còn mang một `effect` (`allow`/`deny`, cột `policies.effect`, mặc định `"allow"`) — `metap_permission::evaluate_policies` fold mọi policy khớp (role gate + condition) thành một `PolicyVerdict`: **`Deny` nếu có ít nhất một policy `deny` khớp** (thắng tuyệt đối, bất kể có bao nhiêu `allow` cũng khớp), ngược lại `Allow` nếu có ít nhất một `allow` khớp, ngược lại `NoMatch`. **Deny-by-default cho non-admin** (đổi từ opt-in-restriction ngày 2026-08-21): một `(entity, action)` chưa có policy nào thì `NoMatch` → bị từ chối, không phải được phép mặc định như trước — `admin` luôn bypass toàn bộ; `POST /admin/policies/seed-defaults` là công cụ bulk-tạo policy allow cho một role/entity mới, tránh việc onboard chậm.
- action ở entity-level có 5 giá trị: `read`/`create`/`update`/`delete`/`transition` — sửa field (`update`) và chuyển workflow state (`transition`) là hai action tách biệt (trước 2026-08-21 dùng chung `update`, gộp hai quyền lại làm một).
- field-level permission — che (mask) khi đọc và chặn khi ghi, được gắn vào mọi call site của `CrudService` (`list`/`create`/`update`/`transition`)
- record-level permission — attribute condition được dịch thành mệnh đề `WHERE` (`metap-query::condition_to_sql::record_policy_where_clause`, `(allow1 OR allow2...) AND NOT (deny1 OR deny2...)` khi có cả hai effect) và AND vào `plan_list` khi đọc, cộng thêm một kiểm tra cùng hình dạng (`PermissionSnapshot::can_perform_record_condition`) trước khi ghi.
- **cross-record condition** (2026-08-21) — một điều kiện record-level có thể tham chiếu sang record khác qua dotted attribute path (vd `"referredBy.status"`). `metap-permission` vẫn thuần túy/đồng bộ: `evaluate_condition` traverse path lồng nhau trên subject đã có sẵn; `PermissionSnapshot::required_relation_fields(action)` chỉ đọc segment đầu path để báo cần fetch quan hệ nào, trả rỗng khi không cần (không tốn gì). `CrudService::enrich_record_for_actions` là nơi duy nhất có I/O — fetch đúng 1-hop qua field `FieldKind::Reference` rồi merge vào một **bản sao** subject, chỉ chạy ở 4 method single-record (`get`/`update`/`transition`/`delete`); `list()` không hỗ trợ (cần `QueryPlanner` JOIN, chưa xây) — `metap-query` reject rõ ràng nếu policy dùng cho `list()` chứa dotted attribute, thay vì âm thầm không bao giờ khớp (nguy hiểm với policy `deny`).
- giải thích/debug policy — `PolicyExplainer` tạo ra một trace chỉ-đọc của mọi policy đã được xét và lý do, được expose qua endpoint mô phỏng `POST /admin/policies/explain` có bảo vệ admin
- một `PermissionSnapshot` theo từng call gom các policy của một tenant/entity vào một lần fetch DB duy nhất, dùng lại xuyên suốt một lần gọi `CrudService` — cố ý không phải là cache theo kiểu cross-request/TTL

Ban đầu chỉ là một scaffold cho phép mọi thứ để kiến trúc có thể chạy được (trong codebase TS gốc); ranh giới service đã được cố định ngay từ đầu và logic thật sự ở trên giờ đã lấp đầy nó, được port lại 1:1 sang Rust, rồi được siết lại đáng kể ngày 2026-08-21 (deny-by-default, effect, cross-record — ba gap được tìm ra qua một lần review permission engine, xem [09. Architecture Decisions](../09-adr/00-index.md)).

## Query Planner

`metap-query::plan_list` biến các view/query contract an toàn thành SQL — đây là nơi *duy nhất* các query list/filter/sort được chuyển thành SQL.

Quy tắc:

- mọi list đều có giới hạn tối đa
- mọi business query đều bao gồm tenant scope
- frontend không thể gửi các toán tử truy vấn database tùy ý
- các field filter/sort phải được khai báo trong metadata
- các báo cáo tốn kém dùng report service riêng hoặc background job (hoãn lại, kích hoạt theo trigger — xem [11. Risks and Technical Debt](../11-risks/00-index.md))

Xây dựng trên nền đó:

- **Hot field indexes.** `EntityField.indexed`/`unique` điều khiển `IndexReconciler` (`metap-peripherals`), tự động đồng bộ các partial expression index theo từng entity trên `records` lúc boot (`CREATE INDEX CONCURRENTLY IF NOT EXISTS`, best-effort) và qua một lệnh gọi thủ công tương đương `pnpm index:reconcile`. Biểu thức được index phải khớp byte-for-byte với biểu thức filter/sort của chính query đó (`jsonb_extract_path_text`, không phải toán tử `->>` tương đương về mặt ngữ nghĩa) nếu không Postgres sẽ không bao giờ chọn nó.
- **Full-text search.** `EntityField.searchMode: "fts"` (opt-in; mặc định vẫn là substring/ILIKE) khớp qua `to_tsvector('simple', ...) @@ plainto_tsquery('simple', ...)`, được hậu thuẫn bởi một GIN index — cùng cơ chế `IndexReconciler` như trên.
- **Keyset pagination.** Một cursor mờ (opaque), mã hóa base64 (`metap-query/src/cursor.rs`, client không bao giờ diễn giải nó) được validate theo sort *đã được resolve* (sau fallback) và chuyển thành điều kiện `WHERE` dạng keyset; một cursor dành cho sai sort, hoặc bị hỏng định dạng, sẽ trả về `400`, không bao giờ được chấp nhận âm thầm hay gây ra `500`.

## Workflow Functions

Workflow là metadata-driven (`metap-workflow`, các free function thay vì struct — không có state cần giữ qua từng call):

- state field
- initial state
- terminal states
- transitions
- actions

Transitions là các thao tác atomic có optimistic locking (một write bị lệch version sẽ làm request thất bại, chứ không phải làm sai state), được bảo vệ bởi một `PolicyCondition` — cùng hình dạng khai báo mà policy đã dùng (`metap-permission::PolicyCondition`), không phải một function, vì Rust không có khái niệm tương đương server-side-predicate-function để port từ thiết kế TS gốc (xem doc comment của `metap-metadata::entity::WorkflowTransition` để biết lý do). Mọi transition đều được ghi vào bảng audit append-only `workflow_events` và phát ra một outbox event `<entity>.workflow.transitioned` sau khi commit — side effect chỉ luôn đi qua outbox, không bao giờ publish trực tiếp.

## Outbox and EventBus

Các transaction của API ghi outbox row vào PostgreSQL (`metap-infra::outbox::enqueue`, cùng transaction với business write). Một publisher (`outbox-publisher`, một binary riêng) drain các row này và publish sang RabbitMQ thông qua trait `EventBus` (`metap-infra::EventBus`; `RabbitEventBus` là implementation duy nhất hiện nay) — việc publish nằm sau một interface (xem [09. Architecture Decisions](../09-adr/00-index.md)).

Điều này bảo vệ hệ thống khỏi mất business event khi RabbitMQ tạm thời không khả dụng.

