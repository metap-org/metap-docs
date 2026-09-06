# Migration path: generic `records` table → bảng riêng cho 1 entity

- **Trạng thái:** approved — trigger + hướng thiết kế đã chốt 2026-09-06, chưa code
- **Người đề xuất:** ghi lại từ thảo luận kiến trúc, `docs/team-charter.md` ("Định hướng đang ghi
  nhận, chưa có trigger" #4); **kích hoạt 2026-09-06** — chủ dự án chốt chủ động đóng khoảng hở
  hiệu năng này ngay, không chờ 1 entity thật đo được nghẽn ("không được để lỗ hổng hiệu năng ở
  đây") — xem ADR cập nhật, `docs/architectures/09-adr/00-index.md`.
- **Track sở hữu:** Backend Core
- **Phase roadmap liên quan:** liên quan `docs/features/04-table-per-entity.md` (đã `done`, wire
  vào Phase 21) nhưng đây là ý khác — di chuyển 1 entity ĐANG SỐNG trên `records` sang bảng riêng
  mà không mất dữ liệu, không phải tạo entity mới trực tiếp trên bảng riêng.

## Vấn đề / động lực

`docs/features/04-table-per-entity.md` đã chứng minh table-per-entity chạy được thật
(`metap-demo-jira`) cho entity **mới tạo trực tiếp** trên bảng riêng. Câu hỏi khác: 1 entity đã có
dữ liệu thật trên bảng `records` chung (JSONB), muốn chuyển sang bảng riêng — quy trình migrate dữ
liệu đang sống là gì?

**Quyết định cơ chế (2026-09-06, chủ dự án chốt trực tiếp):** **downtime chấp nhận được** — tắt
hẳn service trong lúc migrate, không cần zero-downtime dual-write/shadow-read. Lý do: metap còn ở
giai đoạn dev/thiết kế, chưa có dữ liệu sản xuất thật ("metap mới phase dev") — độ phức tạp thiết
kế/rủi ro bug của dual-write (2 nguồn ghi phải luôn nhất quán, rollback giữa chừng khó) không đáng
so với cái giá phải trả chỉ để né vài giây/vài phút downtime không ai đang phụ thuộc vào lúc này.
**Quyết định này đảo lại khi metap có khách hàng/dữ liệu sản xuất thật** — lúc đó phải quay lại
thiết kế zero-downtime.

## Phạm vi

**Trong phạm vi:**
1. Tool/lệnh migrate 1 entity (khả năng sẽ là `dev-tools migrate-to-dedicated-table <entity>` hay
   tương tự, chưa chốt tên) — cho entity bất kỳ đang sống trên `records`, không giới hạn
   accounting/inventory như định hướng cũ trong `docs/architectures/05-building-blocks/04-data-model.md`.
2. Quy trình 1 lần (không phải reconciler chạy liên tục):
   a. `metap-reconciler::reconcile()` tạo bảng đích (mechanism đã có, không đổi — `compile()` +
      `introspect()` + `diff()` + `executor::execute`).
   b. Batch-copy: đọc toàn bộ record JSONB hiện có của entity đó trên `records`, ghi sang bảng
      mới — cần viết mới (không tái dùng thẳng `backfill::run_batched_update`, hàm đó phục vụ
      "sửa cột trong cùng 1 bảng", không phải "copy bảng nguồn khác sang bảng đích khác" — nhưng
      giữ cùng nguyên tắc checkpointed/tự lành sau crash).
   c. Đổi `EntityDefinition.table_name` từ `"records"` sang bảng đích (`qualified_table_name_for`).
   d. Khởi động lại service.
3. Không cần feature flag/dual-write — service tắt trong bước b/c nên không có request nào đọc/ghi
   nhầm bảng giữa lúc migrate.

**Ngoài phạm vi:**
- Không phải bản thân table-per-entity mechanism (`metap-reconciler`, đã xong) — đây là *migration
  path* cho dữ liệu đã tồn tại, dùng lại mechanism đó làm đích đến.
- Zero-downtime migration — ngoài phạm vi cho tới khi có dữ liệu sản xuất thật (xem "Quyết định cơ
  chế" trên).

## Tiêu chí chấp nhận

- Migrate 1 entity thật (test trên `metap-demo-crm`, ví dụ `crm.customers`) từ `records` sang bảng
  riêng, không mất record nào, `version` (optimistic locking) giữ nguyên giá trị sau migrate.
- Service dừng trong lúc migrate, khởi động lại phục vụ đúng từ bảng mới — verify bằng request
  thật trước/sau (list/get/create/update/delete/transition đều hoạt động đúng).
- `workflow_events`/outbox không bị gián đoạn theo entity đó — record tạo/sửa *trước* lúc dừng
  service verify vẫn còn nguyên vẹn *sau* migrate.

## Ranh giới kiến trúc bị đụng tới

- `metap-reconciler` (đích đến, đã có sẵn — `compile`/`introspect`/`diff`/`executor`).
- Batch-copy loop mới (chưa có sẵn — checkpointed, tự lành nếu crash giữa chừng, không cần
  rollback vì chạy lúc service đã tắt, không có ghi đồng thời).
- `EntityDefinition.table_name`: đổi trong code (recompile+redeploy), không phải data-driven —
  giống cách `metap-demo-jira`'s entity code-authored set `table_name` hôm nay.
- Có thể cần thêm 1 binary/subcommand mới trong `dev-tools` hoặc 1 crate ops riêng (giống
  `db-migrate`) — chưa chốt.

## Rủi ro / phụ thuộc

- Rủi ro mất dữ liệu nếu batch-copy sai (bỏ sót record, sai kiểu dữ liệu lúc convert JSONB path
  sang cột typed) — cần test kỹ trên dữ liệu thật trước khi chạy trên môi trường có ý nghĩa.
- Phụ thuộc `docs/features/04-table-per-entity.md` (đích đến migration) — đã sẵn sàng.
- Quyết định downtime-acceptable là tạm thời theo giai đoạn dự án — phải revisit khi metap tiến
  gần production thật (xem ADR).
