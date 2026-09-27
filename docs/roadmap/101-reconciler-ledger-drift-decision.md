## Phase 101: chốt hướng "ledger drift" ở `metap-reconciler` — quyết định không xây thêm code (2026-09-27)

Trigger: chủ dự án yêu cầu "chốt hướng 2" — câu hỏi kiến trúc "item 2" bỏ ngỏ từ
`../metap-demo-waf/CLAUDE.md`'s "Root-cause fix, 3 parts" (viết lúc điều tra 8th/9th finding,
2026-09-17): có nên bắt `metap-reconciler::introspect()` re-derive toàn bộ tín hiệu hội tụ trực
tiếp từ `pg_catalog` (bỏ hẳn niềm tin vào ledger), hay giữ ledger nhưng thêm 1 cross-check định kỳ
rẻ tiền (gợi ý ban đầu: vào `reconciler-orchestrator`'s poll loop có sẵn).

### Rà lại trước khi quyết — không chọn nguyên xi 1 trong 2 gợi ý cũ

**Re-derive toàn bộ**: bị loại — audit rộng đã làm cùng ngày với 8th/9th finding (2026-09-17)
xác nhận mọi tín hiệu hội tụ khác `introspect.rs`/`diff.rs` đọc (index/FK/unique constraint) **đã
luôn** tự lấy tươi từ `pg_catalog` mỗi lần gọi, không tin ledger ở đâu cả — chỉ riêng trường hợp
sync trigger là có tin ledger, và đã được sửa tận gốc (item 1, Phase 84): `introspect()` giờ luôn
gọi `sync_trigger_exists()` kiểm `pg_trigger`/`pg_proc` thật mỗi lần reconcile, không điều kiện gì —
tự hồi phục ngay ở lần `reconcile()` kế tiếp, không cần ledger đúng hay sai. Không còn lý do bắt
toàn bộ crate trả giá hiệu năng cho 1 trường hợp đã đóng.

**Cross-check định kỳ trong `reconciler-orchestrator`**: rà lại kỹ hơn thấy gợi ý này **sai chỗ** —
`reconciler-orchestrator` chỉ tick qua entity **low-code đã publish** (`metap_lowcode::
get_published`, lấy từ `reconciler_entity_deployments`). Cả 3 sự cố thật (6th/7th/8th finding) đều
xảy ra trên entity **code-authored** (`waf.zones`, `waf.ddos_policies`, ...) — loại entity này
**không bao giờ** nằm trong hàng đợi của orchestrator trừ khi ai đó tự tay chạy `dev-tools
enqueue-reconcile`. Nghĩa là nếu làm đúng theo gợi ý cũ, cross-check sẽ kiểm tra đúng những entity
chưa từng gãy, và bỏ qua hoàn toàn loại entity đã gãy cả 3 lần. Muốn cross-check thật sự hữu ích
phải xây 1 watchdog định kỳ hoàn toàn mới, quét toàn bộ entity code-authored xuyên mọi service —
lớn hơn nhiều so với "thêm 1 check vào vòng lặp có sẵn" như bullet cũ ngụ ý.

### Quyết định

**Không xây thêm code/hạ tầng polling mới lúc này.** Lý do:
- Cơ chế tự-verify-qua-`pg_catalog` mỗi lần `reconcile()` (item 1, đã xong) đã đóng đúng lỗ hổng đã
  xảy ra — chưa có sự cố thứ 4 nào kể từ Phase 84 để biện minh cho việc xây thêm.
- Entity code-authored vẫn được `reconcile()` lại mỗi lần service restart (deploy thật) — 1 điểm
  tự-verify thật, dù không liên tục.
- `dev-tools enqueue-reconcile` là lối thoát thủ công sẵn có nếu nghi ngờ có drift giữa 2 lần
  restart.
- Rủi ro còn lại (ai đó/cái gì xoá trigger/extension ngoài vòng đời reconcile, giữa 2 lần restart)
  là **rủi ro vận hành** (cần giám sát hạ tầng — cảnh báo khi có `DROP TRIGGER`/`DROP EXTENSION`
  ngoài kế hoạch trên DB này), không phải lỗ hổng thiết kế của `metap-reconciler` — cờ rõ để không
  ai ngộ nhận đã có người lo, không phải im lặng bỏ qua.

Đã cập nhật `../metap-demo-waf/CLAUDE.md`'s "Root-cause fix, 3 parts" — item 2 giờ ghi "decided
2026-09-27, no new code" thay vì "flagging, not deciding", kèm đầy đủ lý do rà lại ở trên.

### Việc không làm trong phase này

- Không đổi gì trong `metap-reconciler` — quyết định là giữ nguyên code, chỉ chốt hướng bằng docs.
- Không xây watchdog/giám sát hạ tầng — cờ là việc của đội vận hành/hạ tầng, ngoài phạm vi
  `metap-reconciler` (và ngoài phạm vi phase này).
- `migrate.rs::copy_generic_records`'s latent gap (đã ghi trong `../CLAUDE.md`) — rà lại xác nhận
  mô tả cũ vẫn đúng nguyên: "latent defense-in-depth gap, not a live bug today", không có gì để sửa
  thêm (ranh giới tin cậy hợp lệ cho 1 CLI tool gọi thủ công, không phải bug) — không đổi.
