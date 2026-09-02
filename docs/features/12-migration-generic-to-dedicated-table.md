# Migration path: generic `records` table → bảng riêng cho 1 entity

- **Trạng thái:** proposed — chưa có trigger
- **Người đề xuất:** ghi lại từ thảo luận kiến trúc, `docs/team-charter.md` ("Định hướng đang ghi
  nhận, chưa có trigger" #4)
- **Track sở hữu:** Backend Core
- **Phase roadmap liên quan:** liên quan `docs/features/04-table-per-entity.md` (đã `done`, wire
  vào Phase 21) nhưng đây là ý khác — di chuyển 1 entity ĐANG SỐNG trên `records` sang bảng riêng
  mà không mất dữ liệu, không phải tạo entity mới trực tiếp trên bảng riêng.

## Vấn đề / động lực

`docs/features/04-table-per-entity.md` đã chứng minh table-per-entity chạy được thật
(`metap-demo-jira`) cho entity **mới tạo trực tiếp** trên bảng riêng. Câu hỏi khác, chưa trả lời:
1 entity đã có dữ liệu thật trên bảng `records` chung (JSONB), muốn chuyển sang bảng riêng vì lý do
hiệu năng (đạt ngưỡng ~10M row, `multi-tenant-platform-design.md` §3.1) — quy trình migrate dữ
liệu đang sống, không downtime hoặc downtime tối thiểu, là gì?

## Phạm vi

**Trong phạm vi (nếu được kích hoạt):**
- (chưa thiết kế cụ thể — cần 1 entity thật đo được nghẽn hiệu năng trước khi viết spec, không
  chuẩn bị sẵn trước theo đúng nguyên tắc `docs/team-charter.md` đã nêu)

**Ngoài phạm vi:**
- Không phải bản thân table-per-entity mechanism (`metap-reconciler`, đã xong) — đây là *migration
  path* cho dữ liệu đã tồn tại, dùng lại mechanism đó làm đích đến.

## Tiêu chí chấp nhận

<Chưa xác định — theo đúng `docs/architectures/05-building-blocks.md`'s Data Model Strategy Step
3, chỉ viết spec khi có tín hiệu hiệu năng đo được thật của 1 entity cụ thể, không phải chuẩn bị
sẵn trước khi cần.>

## Ranh giới kiến trúc bị đụng tới

- `metap-reconciler` (đích đến, đã có sẵn — `compile`/`introspect`/`diff`/`executor`/`backfill`).
- Cần thiết kế: đọc toàn bộ record JSONB hiện có trên `records` cho entity đó, ghi sang bảng
  riêng mới (dùng `backfill::run_batched_update`-style checkpointed loop đã có cho việc khác, hay
  cần 1 loop riêng), giữ đúng optimistic locking (`version`) và outbox event không bị gián đoạn
  giữa chừng.
- Cutover: đổi `EntityDefinition.table_name` từ `"records"` sang bảng riêng — cần đồng bộ đúng
  thời điểm (feature flag? maintenance window ngắn?) để không có request nào đọc/ghi nhầm bảng
  giữa lúc migrate.

## Rủi ro / phụ thuộc

- Rủi ro mất dữ liệu/downtime nếu thiết kế sai — đây là migration dữ liệu sản xuất thật, không
  phải thêm tính năng mới, cần review kỹ hơn mức bình thường khi có trigger.
- Phụ thuộc `docs/features/04-table-per-entity.md` (đích đến migration) — đã sẵn sàng.
