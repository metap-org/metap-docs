# Audit 05 — Audit trail của `CrudService` chưa đủ cho nghiệp vụ enterprise

> **Phạm vi (2026-09-13):** không phải 1 audit sweep có hệ thống như 02/03/04 — phát hiện qua trao
> đổi thường (đang bàn tradeoff JSONB/storage của `metadata.records`, chủ dự án hỏi ngược lại yêu
> cầu enterprise thật: "mỗi thay đổi phải track được ai đổi, đổi gì trên nghiệp vụ nào, thời gian,
> lý do sở cứ"). Ghi lại làm 1 audit riêng vì đúng shape "gap tìm thấy, có bằng chứng code cụ thể,
> chưa quyết hướng fix, chưa code" — theo đúng convention của `04-*.md`. **Chưa code gì cả.**

## Tóm tắt

| # | Mức độ | Vấn đề |
|---|---|---|
| 1 | **HIGH** (nghiệp vụ enterprise/compliance) | Không có audit trail tổng quát cho bất kỳ write nào — `create`/`update`/`delete` không ghi gì cả; `transition()` chỉ ghi `workflow_events`, mà bảng đó **không phải audit trail**, chỉ là ledger transition của workflow engine (xem mục 2) |
| 2 | Làm rõ, không phải bug | `workflow_events` phục vụ đúng mục đích hẹp của nó (state machine transition, cho chính workflow engine dùng lại) — **không nên vá thêm cột (`reason`,...) để biến nó thành audit trail**; `reason`/before-after nếu cần phải nằm trong `AuditTrailStore` mới (mục 1), `transition()` gọi cả 2 — `record_event` (giữ nguyên, ledger transition) và `AuditTrailStore` (mới, audit trail thật) — không gộp chung |

---

## 1. HIGH — Không có audit trail tổng quát cho bất kỳ write nào

**Bằng chứng**: `crates/metap-crud/src/crud_service/update.rs` — không có bất kỳ lời gọi nào tới
`record_event`/ghi bảng audit nào trong toàn bộ hàm `update()` (`create()`/`delete()` cũng vậy).
`transition()` (`transition.rs:207-208`) là chỗ DUY NHẤT trong cả codebase ghi lại gì đó khi có
thay đổi — nhưng ghi vào `workflow_events`, mà bảng đó chỉ là ledger transition của workflow
engine (state cũ → state mới, để chính engine/observability của state machine dùng), **không phải
audit trail nghiệp vụ** — không có field-level before/after, không có reason, và quan trọng nhất:
chỉ tồn tại cho entity nào có `workflow`, hoàn toàn không phủ `create`/`update`/`delete` thường.

**Hệ quả thật**: sửa tên khách hàng, đổi giá, đổi bất kỳ field nào — dù có workflow hay không —
không để lại dấu vết nào tra cứu lại được ai đổi/đổi gì/giá trị cũ-mới. Với nghiệp vụ enterprise
cần audit trail đầy đủ cho MỌI thay đổi, đây là gap chặn thẳng yêu cầu đó.

**Hướng đã chốt với chủ dự án (2026-09-13)**: 1 audit/change-log tổng quát áp dụng cho **cả 4**
write (`create`/`update`/`delete`/`transition` — `transition()` gọi audit trail mới này SONG SONG
với `record_event` hiện có, không thay thế nó, vì 2 bảng phục vụ 2 mục đích khác nhau — xem mục
2), thiết kế theo đúng "swappable backend qua trait" đã có tiền lệ khắp codebase này (`EventBus`,
`SecretStore`, `ObjectStore`, `Cache` — mỗi cái đều là 1 trait + nhiều impl chọn qua config, không
hardcode 1 backend cụ thể):

- Crate mới `metap-audit` (cùng tier với `metap-cache`/`metap-storage` — không business-entity
  knowledge, plain library): trait `AuditTrailStore` (`record(entry: AuditEntry) -> Result<()>`,
  cộng vài query method đọc lại theo `entity`/`record_id`/khoảng thời gian) + struct `AuditEntry`
  (`tenant_id, entity, record_id, actor, action, occurred_at, reason: Option<String>, before:
  Option<JsonObject> (diff field-level, không snapshot toàn bộ record — tránh nhân đôi storage,
  cùng tinh thần tradeoff JSONB T2/T3 đã bàn), after: Option<JsonObject>`).
- **"Per-entity"**: mỗi `EntityDefinition` tự khai báo có audit hay không (field mới, ví dụ
  `audit: Option<EntityAuditConfig>`, cùng dạng khai báo per-entity như `workflow`/
  `unique_constraints` đã có) — không phải mọi entity đều bắt buộc audit.
- **"Postgres hoặc storage khác, add thêm DB khác cũng được"**: `PostgresAuditTrailStore` là impl
  mặc định (bảng chung `audit_trail_events`, hoặc per-entity dedicated table kiểu
  `qualified_table_name_for` reconciler đã dùng — chọn khi thiết kế chi tiết), nhận `PgPool` riêng
  của nó thay vì bắt buộc dùng chung `Router`/pool của tenant — cho phép trỏ audit trail sang 1
  Postgres instance khác hẳn (cách ly compliance) mà không cần code khác. Backend khác (ghi ra
  object storage qua `metap-storage::ObjectStore` có sẵn dạng JSONL/Parquet theo ngày, hay
  Elasticsearch cho log tìm kiếm được) chỉ cần thêm 1 impl mới của cùng trait, không đổi
  `CrudService`.
- **Hook vào `CrudService`**: `Option<Arc<dyn AuditTrailStore>>` field mới, opt-in — cùng khuôn
  `PermissionService::with_cache` (constructor riêng, không đổi `CrudService::new`'s chữ ký hiện
  có) — không cấu hình thì hành vi y hệt hôm nay, không audit gì (giữ nguyên mọi downstream chưa
  cần tính năng này).

## 2. Làm rõ — `workflow_events` là ledger transition của workflow engine, không phải audit trail

**Bằng chứng**: `crates/migrations/0001_panoramic_firestar.sql:1-11` (`CREATE TABLE
"workflow_events"`) — cột: `id, tenant_id, entity, record_id, action, from_state, to_state, actor,
created_at`. So với `outbox_events` (`0000_green_jean_grey.sql`, có `payload jsonb, published_at,
attempts, last_error`) thì `workflow_events` không có shape outbox — nhưng cũng không có
`reason`/before-after value, tức không đủ shape để làm audit trail nghiệp vụ. Đây là 1 bảng có
mục đích rất cụ thể: cho workflow engine tự tra lại lịch sử transition của chính state machine nó
quản lý (debug guard/validator, hiển thị "workflow diagram" lịch sử trên UI,...).

**Kết luận, không phải hướng fix**: **không vá thêm cột vào `workflow_events`**. Nó cứ giữ nguyên
vai trò hẹp của nó. `reason`/before-after cho hành động `transition()` (nếu cần) đi qua
`AuditTrailStore` mới ở mục 1 — `transition()` sẽ gọi cả `record_event` (giữ nguyên, không đổi) và
audit trail mới (thêm), 2 lời gọi độc lập, ghi vào 2 bảng phục vụ 2 mục đích khác nhau.

## Ảnh hưởng đi kèm nếu làm

- **Breaking API**: `create()`/`update()`/`delete()`/`transition()` đều cần thêm tham số mới (ít
  nhất `reason: Option<String>`) — chạm REST/gRPC/GraphQL (`metap-http`, `metap-grpc`,
  `metap-graphql` đều gọi `CrudService` trực tiếp) và mọi downstream binary (`metap-demo-jira`,
  `metap-demo-waf`, `metap-lowcode`'s services).
- **Bảng audit mới sẽ tăng vô hạn theo chủ đích** (không phải bug) — đây chính là loại "history
  table" đã bàn trong phiên trước (xem `metadata.records` bloat investigation cùng ngày): **không**
  nên áp retention/prune cho loại bảng này, đúng như chủ dự án đã chỉnh — enterprise compliance cần
  giữ vĩnh viễn (hoặc theo thời hạn pháp lý dài), không phải tối ưu ổ cứng bằng cách xoá.

## Trạng thái

**Đã code và verify xong (2026-09-13)** — crate `metap-audit` mới (`AuditTrailStore` trait,
`PostgresAuditTrailStore`, `diff_json_objects`), `EntityDefinition.audit: Option<EntityAuditConfig>`
(chỉ 1 flag `enabled: bool`, opt-in per-entity — không có chọn backend riêng theo entity, xem lý do
bên dưới), `CrudService::with_audit(...)` (constructor thứ 2, additive, `CrudService::new` cũ không
đổi) ghi 1 dòng vào `metadata.audit_trail_entries` sau khi transaction nghiệp vụ đã commit, cho cả 4
write (`create`/`update`/`delete`/`transition`). Tham số `reason: Option<&str>` được thêm tường
minh (không nhét vào `data`/`payload`) trên cả 4 method, rippled qua REST/gRPC/GraphQL và toàn bộ
downstream repo có gọi trực tiếp (`metap-demo-waf`'s 7 call site).

Trả lời 3 câu hỏi còn treo ở trên khi code thật:
- **Shape `EntityAuditConfig`**: chỉ `{ enabled: bool }` — không cho chọn backend/store riêng theo
  entity. Backend nào (Postgres/DB khác) là quyết định ở tầng deployment (1
  `Arc<dyn AuditTrailStore>` truyền vào `CrudService::with_audit` lúc boot), không phải per-entity —
  tránh over-engineering cho nhu cầu chưa ai yêu cầu.
- **Diff field-level**: `diff_json_objects(before, after)` — so 2 `JsonObject` phẳng (shallow,
  top-level key), record `before` lấy từ record đã đọc sẵn trong `update()` (cho check
  `expected_version`); `create()` diff với object rỗng; `delete()` không diff (record đã xoá không
  còn "sau" để so) — dùng `action: Delete` để thể hiện, `diff: {}`.
  `AuditTrailStore` impl không tự tính diff, `CrudService` tính rồi truyền `AuditEntry` đã có diff.
- **Mask field nhạy cảm trước khi ghi**: **chưa làm, vẫn còn treo** — audit trail hiện ghi diff thô,
  chưa áp field-level permission mask. Ai đọc được audit log (admin nào, tenant nào) là câu hỏi bảo
  mật riêng cho 1 finding sau, chưa trong scope này.

Đánh đổi transactional-consistency (audit ghi *sau* khi transaction nghiệp vụ đã commit, không cùng
transaction — vì `Arc<dyn AuditTrailStore>` object-safe trỏ DB khác không thể share transaction của
caller) — **chấp nhận best-effort cho v1** theo quyết định của chủ dự án. Nếu sau này cần
at-least-once thật, hướng nâng cấp đã ghi sẵn: route qua `outbox_events` (outbox pattern đã có sẵn
trong platform) thay vì gọi `AuditTrailStore` trực tiếp.

Đã verify: `cargo test --workspace` + `clippy --workspace --all-targets -- -D warnings` sạch trên
`metap`; e2e thật (Postgres) cho `metap-crud`/`metap-graphql`/`metap-graphql-gateway`/`metap-grpc`
đều pass; `metap-demo-waf/data-plane` (3 service) build/clippy/e2e sạch sau khi thêm `reason=None`
vào 7 call site thật; `metap-demo-jira` build/clippy sạch, và **đã smoke-test thật qua REST** (chạy
`jira-server` thật, mint token thật, `POST`/`PATCH`/`DELETE /api/jira.projects` thật với `reason`,
xác nhận đúng 3 dòng audit — create/update/delete — xuất hiện trong `metadata.audit_trail_entries`
với diff/reason/action đúng). `jira.projects` là entity ví dụ thật đã bật `audit.enabled = true`
trong `metap-demo-jira/src/entities/project_entity.rs`.
