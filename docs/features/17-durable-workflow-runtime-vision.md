# Tầm nhìn dài hạn: State Machine + Workflow tách biệt, Durable Workflow Runtime

- **Trạng thái:** vision — không phải feature sẵn sàng lên spec, ghi lại để không quên hướng
- **Người đề xuất:** ghi lại 2026-08-20 từ brainstorm, `docs/team-charter.md` ("Định hướng đang
  ghi nhận, chưa có trigger", mục cuối)
- **Track sở hữu:** Backend Core
- **Phase roadmap liên quan:** không thuộc phase nào — đào sâu #09 (workflow hai chế độ) và #10
  (workflow visualize), không phải ý độc lập mới

## Vấn đề / động lực

Roadmap 5 level tham khảo cho hướng platform dài hạn:

1. **Metadata-driven CRUD** — hiện tại, xong.
2. **Metadata-driven application** (Jira/Confluence/CRM/ERP dựng chủ yếu bằng entity/field/view
   definition, không viết CRUD riêng cho từng loại) — cần metadata mô tả thêm
   Action/Command/Event/Condition/State/Transition/Policy/Workflow/Trigger/Job/Schedule, không
   chỉ Entity/Field/Relation/View như hôm nay.
3. **Metadata-driven workflow** — 1 engine Workflow đúng nghĩa.
4. **Durable workflow runtime** (retry/timeout/timer/signal/replay/idempotency, kiểu Temporal) —
   khác hẳn về độ khó so với `WorkflowEngine` hiện tại (chỉ atomic transition + guard + audit,
   không có gì persist execution state giữa các bước).
5. **Distributed workflow platform**.

**Metap hôm nay đứng ở level 1, mới bắt đầu chạm level 2 qua Phase 11 (low-code metadata).**

## Trigger đã xảy ra cho phần nhỏ hơn — Phase 17 đã xây, không phải toàn bộ tầm nhìn này

Quan trọng: **State Machine và Workflow là hai primitive tách biệt, compose với nhau** — State
Machine (state/transition/guard/action) mô tả 1 entity đang ở đâu, được chuyển đi đâu; Workflow
(trigger/condition/activity/timer/event) mô tả 1 chuỗi việc có thể kéo dài, vừa lắng nghe
state-transition làm trigger, vừa gọi ngược 1 transition như action.

Rà code thật (không suy đoán) đã tìm ra: `crates/metap-workflow/src/lib.rs` (272 dòng) chỉ là 3
hàm thuần (`get_initial_status`/`find_transition`/`run_guard`) — không giữ execution state, không
cần refactor gì để "tách" nó, nó vốn không biết gì về trigger/schedule/activity. Bất ngờ hơn:
`metap-cron` đã là ~70% hạ tầng Workflow cần (trigger schedule, activity gọi transition/webhook,
reliable dispatch qua outbox, audit). **Trigger đã xảy ra 2026-08-21** cho đúng phần này — đã xây
xong cả 3 increment (Phase 17, `docs/features/02-workflow-engine.md`, done).

**Phần còn lại của tầm nhìn 5-level (level 3 trọn vẹn, level 4/5) vẫn chưa có trigger** — đây là
brief giữ lại hướng dài hạn, không phải backlog sẵn sàng code.

## Phạm vi

Không áp dụng — đây là ghi nhận tầm nhìn, không phải spec sẵn sàng implement.

## Tiêu chí chấp nhận

Không áp dụng.

## Ranh giới kiến trúc bị đụng tới

Level 4 (durable workflow runtime) đối lập gần như hoàn toàn với thiết kế hiện tại: cần persist
execution state giữa các bước (điều `metap-workflow` cố tình không có), retry/timeout/signal/
replay/idempotency ở mức primitive — gần với việc viết 1 engine mới hoàn toàn (kiểu Temporal) hơn
là mở rộng những gì đang có. Sẽ cần ADR riêng, nhiều tuần thiết kế trước khi code, khi (và nếu) có
trigger thật.

## Rủi ro / phụ thuộc

- **Chưa có trigger cho level 3 trọn vẹn/level 4/5** — chỉ nên tham khảo hướng, không bắt đầu.
- Phụ thuộc #09 (workflow hai chế độ) và #10 (workflow visualize) — 2 ý đó là bước nhỏ hơn, gần
  hơn trên cùng con đường tới tầm nhìn này.
