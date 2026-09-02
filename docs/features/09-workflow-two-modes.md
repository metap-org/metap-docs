# Workflow hai chế độ (in-process + cross-module)

- **Trạng thái:** proposed — chưa có trigger
- **Người đề xuất:** ghi lại từ thảo luận kiến trúc, `docs/team-charter.md` ("Định hướng đang ghi
  nhận, chưa có trigger" #1)
- **Track sở hữu:** Backend Core
- **Phase roadmap liên quan:** không thuộc phase nào

## Vấn đề / động lực

Một logical workflow model (trigger/condition/activity) chạy được ở cả hai deployment shape mà
không phải viết lại: **in-process** (một module tự xử lý workflow của chính nó, gọi hàm trực
tiếp) và **cross-module** (một module publish event, module khác subscribe và chạy activity của
riêng nó). Hôm nay `metap-cron`'s workflow-automation layer (Phase 17) chỉ chạy in-process — một
`cron_jobs` row dispatch tới đúng 1 target trong cùng deployment.

## Ghi chú thảo luận (2026-09-03, chưa phải thiết kế)

Chủ dự án vẫn chưa có use case thật ("chưa nghĩ ra usecase... đang suy nghĩ thêm") — mục dưới đây
chỉ ghi lại mạch suy nghĩ minh hoạ để không quên, **không phải trigger cụ thể**, không kích hoạt
việc bắt đầu thiết kế (xem điều kiện trigger ở mục "Rủi ro / phụ thuộc" bên dưới, chưa đổi).

- **Ví dụ minh hoạ (giả định, không phải entity thật trong repo)**: thương mại điện tử —
  `order → checkout → fulfillment → delivery` — mỗi bước có thể là 1 hệ thống/module khác nhau,
  nhưng nghiệp vụ là 1 flow xuyên suốt. Nếu có use case thật dạng này trong repo (vd giữa
  `metap-demo-crm`/`metap-demo-jira`/`metap-demo-waf`), đó mới là trigger đủ điều kiện.
- **Yêu cầu visualize đi kèm**: nếu #1 thành hình, canvas ở
  [`10-workflow-visualize.md`](10-workflow-visualize.md) (đã `done`) sẽ cần phân biệt được — nét
  đứt/nét thường hoặc màu khác — giữa transition thuần nội bộ 1 module và transition
  trigger/được-trigger-bởi module khác. Cần 1 field mới trên `WorkflowTransition`
  (`crates/metap-metadata`) đánh dấu tính chất này trước khi `WorkflowDiagram` vẽ được — chưa có
  hôm nay.
- **Câu hỏi mở về outbox**: hiện `metap-workflow` bắn outbox event vô điều kiện trên **mọi**
  transition (không tốn kém do outbox-pattern ghi cùng transaction, publish async — không phải vấn
  đề hiệu năng), phục vụ `notification-worker`/`metap-cron`'s `WaitEvent` chain subscribe. Câu hỏi
  chủ dự án nêu: có cần phân biệt transition nào "đáng publish có mục đích" (cross-module) với
  transition thuần nội bộ không? Field đánh dấu ở ý trên (nếu làm) có thể dùng chung cho cả 2 mục
  đích — vừa vẽ khác màu trên canvas, vừa quyết định event nào thật sự cần publish rõ ràng cho
  module khác, thay vì bắn hết như hiện tại.

## Phạm vi

**Trong phạm vi:**
- (chưa thiết kế — brief này chỉ ghi nhận vấn đề, chưa có đề xuất giải pháp cụ thể)

**Ngoài phạm vi:**
- Không thay thế `WorkflowTransition.guard`/state machine hiện tại (`crates/metap-metadata`) —
  đó là state-machine-guard, không phải workflow engine.
- Không thay thế `metap-cron`'s `TargetType::Steps`/`WaitEvent` (Phase 17, đã xong) — ý này là mở
  rộng cách một chuỗi activity *chạy xuyên module*, không phải viết lại cách nó chạy trong 1 module.

## Tiêu chí chấp nhận

<Chưa xác định — cần trigger cụ thể trước khi viết tiêu chí kiểm chứng được.>

## Ranh giới kiến trúc bị đụng tới

**Đối lập trực tiếp với `docs/architectures/09-adr.md`'s kết luận hiện tại**: `WorkflowRuntime`
được liệt là một trong các Capability SPI (Service Provider Interface) **chưa có trigger, chưa nên
xây** — modular monolith vẫn là lựa chọn đúng cho `metap`-core (xem ADR, "Không tách microservice
cho hướng SaaS multi-tenant"). Làm ý này nghĩa là đảo ngược một phần kết luận đó — **cần ADR mới**
giải thích trigger cụ thể trước khi bắt đầu thiết kế, không chỉ feature brief.

## Rủi ro / phụ thuộc

- **Trigger cần thiết**: một module thứ hai thật sự cần chạy cross-module workflow (không phải
  giả định trước) — chưa xảy ra trong repo hôm nay.
- Rủi ro thiết kế sai nếu làm trước khi có use case thật: dễ over-engineer 1 abstraction chưa
  biết hình dạng thật của bài toán thứ 2 cần nó.
- Phụ thuộc gián tiếp `docs/features/02-workflow-engine.md` (engine in-process hiện tại) — bất kỳ
  thiết kế cross-module nào cũng nên compose lên trên nó, không thay thế.
