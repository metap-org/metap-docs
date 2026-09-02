# Schema versioning cho entity

- **Trạng thái:** proposed — chưa có trigger
- **Người đề xuất:** ghi lại từ thảo luận kiến trúc, `docs/team-charter.md` ("Định hướng đang ghi
  nhận, chưa có trigger" #6)
- **Track sở hữu:** Backend Core
- **Phase roadmap liên quan:** không thuộc phase nào

## Vấn đề / động lực

Metadata và record cùng mang `schema_version`, physical mapping field → JSONB path đổi theo
version (vd `v2: data->>'total'` vs. `v3: data->'pricing'->>'total'`) — cho phép đổi field shape
mà không phải migrate toàn bộ record cũ ngay lập tức.

## Phạm vi

**Trong phạm vi (nếu được kích hoạt):**
- (chưa thiết kế — 2 câu hỏi ở mục Rủi ro cần trả lời trước)

**Ngoài phạm vi:**
- Không phải cột `version` đã có trong `records` (optimistic locking) — cần tên khác hoàn toàn để
  tránh đụng độ, chưa chọn tên.

## Tiêu chí chấp nhận

<Chưa xác định — chưa có trigger.>

## Ranh giới kiến trúc bị đụng tới

**Đối lập trực tiếp cách `QueryPlanner` hoạt động hôm nay**: `field_expression()`/
`sort_field_expression()` (`crates/metap-query/src/condition_to_sql.rs`,
`crates/metap-query/src/query_planner.rs`) map thẳng 1 `field_name` sang 1 path JSONB **cố định
duy nhất**, không biết đến version. Làm ý này nghĩa là viết lại giả định nền tảng đó.

## Rủi ro / phụ thuộc

- **2 câu hỏi chưa trả lời, phải chốt trước khi viết spec:**
  1. Filter/sort xử lý sao khi nhiều record cùng entity nhưng khác `schema_version` nằm trong
     cùng 1 query? (vd `sort theo total` khi 1 nửa record là `v2` 1 nửa là `v3`, path JSONB khác
     nhau).
  2. Tên cột version mới, tránh đụng `version` đã dùng cho optimistic locking trong `records`
     (`crates/migrations/0000_*.sql`).
- **Trigger**: 1 entity thật cần đổi field shape mà không muốn migrate toàn bộ record cũ ngay lập
  tức — chưa entity nào trong repo cần điều này.
- Đây là 1 trong các ý rủi ro kỹ thuật cao nhất trong danh sách "chưa có trigger" — nên cân nhắc kỹ
  trước khi bắt đầu dù có trigger, vì đụng module `metap-query` mà ADR (`09-adr.md`) đã gọi là
  "Postgres-dialect seam rủi ro cao nhất".
