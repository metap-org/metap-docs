# Tiny deployment profile (single binary, không RabbitMQ)

- **Trạng thái:** proposed — chưa có trigger, cần quyết định sản phẩm trước
- **Người đề xuất:** ghi lại từ thảo luận kiến trúc, `docs/team-charter.md` ("Định hướng đang ghi
  nhận, chưa có trigger" #3), đã đặt tên trước ở `docs/modular-spi-architecture.md`'s Deployment
  Profiles
- **Track sở hữu:** Backend Ops-Infra
- **Phase roadmap liên quan:** không thuộc phase nào

## Vấn đề / động lực

Một profile triển khai nhẹ hơn: 1 binary duy nhất, SQLite thay Postgres, in-memory `EventBus` thay
RabbitMQ — nhắm use case self-host/single-tenant nhỏ không muốn vận hành cả cụm Postgres+RabbitMQ.

**Đây không phải gap kỹ thuật — là câu hỏi sản phẩm chưa trả lời: Metap có nhắm khách self-host
không?** Chính `docs/modular-spi-architecture.md`'s Deployment Profiles đã khuyến nghị "Option 1:
giữ một triết lý deployment duy nhất" cho giai đoạn hiện tại.

## Phạm vi

**Trong phạm vi (nếu được kích hoạt):**
- (chưa thiết kế — phụ thuộc câu trả lời "có nhắm self-host không" trước)

**Ngoài phạm vi:**
- Không đổi triết lý deployment mặc định (Postgres + RabbitMQ) cho SaaS multi-tenant — Tiny chỉ
  là 1 profile thêm, không thay thế.

## Tiêu chí chấp nhận

<Chưa xác định — phụ thuộc quyết định sản phẩm trước tiên.>

## Ranh giới kiến trúc bị đụng tới

- **`docs/architectures/02-constraints.md`'s ràng buộc "Postgres/RabbitMQ-duy-nhất"** — cần sửa
  trực tiếp constraint này, không chỉ thêm code.
- **`QueryPlanner` (`metap-query`)** — dialect hiện tại là Postgres-specific
  (`jsonb_extract_path_text`, keyset pagination cú pháp Postgres) — cần audit toàn bộ trước khi
  khẳng định SQLite chạy được, không giả định.
- **`EventBus` trait** (`metap-infra`) — đã là trait, 1 impl in-memory mới về mặt kỹ thuật không
  khó (interface sẵn sàng), nhưng ngữ nghĩa "outbox pattern" (đảm bảo publish tin cậy) cần xem lại
  khi không còn broker thật.
- Cần ADR riêng nếu được kích hoạt — đây là thay đổi giả định nền tảng (Postgres/RabbitMQ luôn có
  mặt), không phải một tính năng cộng thêm.

## Rủi ro / phụ thuộc

- **Chưa có trigger, và trigger ở đây là quyết định sản phẩm** (nhắm thị trường self-host), không
  phải nhu cầu kỹ thuật — không nên bắt đầu audit `QueryPlanner`/viết ADR cho tới khi câu hỏi đó
  được trả lời rõ ràng.
- Rủi ro chia đôi effort bảo trì (2 dialect DB, 2 EventBus impl) nếu làm mà không có nhu cầu thị
  trường thật đủ lớn để bù lại.
