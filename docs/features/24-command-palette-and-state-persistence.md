# Command palette, keyboard shortcuts, state persistence — cần 1 vòng scoping trước khi code

- **Trạng thái:** done (2026-09-05), phạm vi v1 rút gọn — tự chốt luôn phương án tối thiểu đã đề
  xuất trong brief thay vì chờ vòng scoping riêng (`Cmd/Ctrl+K` mở palette, chỉ điều hướng nav
  item theo role, không entity/record search, không state persistence). `design-system` commit
  `2c0c139` (atom `CommandPalette` mới — xác nhận lúc code là chưa có primitive
  Command/Combobox-palette nào), `platform-ui` commit `078e967` (`AppCommandPalette`, mount trong
  `AppShellLayout`). State persistence (mục còn lại của tên brief) **chưa làm** — vẫn cần quyết
  định "persist cái gì" như brief đã nêu, để ngỏ cho lần sau nếu có nhu cầu thật.
- **Người đề xuất:** rà soát `docs/frontend-checklist.md` (2026-09-04), mục 6 "UX Infrastructure"
  — 3 mục còn lại sau khi tách phần hạ tầng rõ việc sang
  [23. UX Infrastructure nền tảng](23-ux-infrastructure-core.md).
- **Track sở hữu:** Frontend Platform
- **Phase roadmap liên quan:** chưa gắn phase

## Vấn đề / động lực

Khác 4 mục ở feature 23 (có thể code thẳng, không cần quyết định gì gây tranh cãi), 3 mục dưới đây
đều mở ra 1 UX pattern **hoàn toàn mới** cho `platform-ui` (chưa có tiền lệ nào), nên cần chốt
phạm vi trước khi code — đúng tinh thần feature 01 đã áp dụng khi gặp việc "có trigger nhưng chưa
scope":

- **Keyboard shortcuts** — 0 hit cho hotkey/cmdk/keydown-meta pattern trong toàn bộ
  `platform-ui/src`.
- **Command palette** — chưa có; cần quyết định danh sách action nào đăng ký (điều hướng? tìm
  entity? action admin?) trước khi biết UI cần những gì.
- **State persistence** — hiện tại locale là preference **duy nhất** được persist, và persist
  **server-side** qua `GET/PUT /preferences` (`i18n/LocaleProvider.tsx:33-58`), không phải
  `localStorage`. Mở rộng ra client-side persist cần quyết định persist **cái gì** trước (đụng
  chung câu hỏi với "Saved views" ở nhóm Dynamic Table — không tự mở rộng thành 1 feature Saved
  views đầy đủ ở đây nếu chưa có trigger riêng cho nó).

## Phạm vi

**Chưa chốt — cần 1 vòng scoping riêng trước khi implement.** Ghi lại câu hỏi cần trả lời, không
phải đáp án:

- Command palette v1 nên có action nào? Đề xuất tối thiểu để bàn: điều hướng tới mọi trang trong
  `navItems` của `AppShellLayout` (Dashboard/Zones/.../Settings...) + tìm nhanh theo tên entity
  (không phải tìm theo record — fuzzy search nội dung record cần index riêng, khác phạm vi).
- Phím tắt mở palette: `Cmd/Ctrl+K` là convention phổ biến, ít tranh cãi — có thể chốt luôn không
  cần họp bàn riêng.
- State persistence: phạm vi tối thiểu nào đáng làm trước? Ví dụ "nhớ tab đang active gần nhất
  của 1 record có nhiều tab" (phụ thuộc feature 20 nếu được làm trước) hẹp hơn nhiều so với
  "Saved views" đầy đủ (lưu cả filter/sort/column state) — 2 mức độ này khác hẳn effort, cần chọn
  rõ trước khi code, không mặc định làm hết.

**Ngoài phạm vi (chắc chắn, không cần bàn thêm):**
- Fuzzy search theo nội dung record cụ thể (cần index tìm kiếm riêng ở backend).
- Tuỳ biến phím tắt theo từng user.
- "Saved views" đầy đủ (filter/sort/column state per-view, đặt tên, chia sẻ) — nếu có nhu cầu
  thật, nên có brief riêng, không gộp vào đây.

## Tiêu chí chấp nhận

Chưa có — sẽ điền khi feature này chuyển `proposed` → `approved`, sau vòng scoping ở trên. Đề xuất
tối thiểu để tham khảo lúc scoping (không phải cam kết):

- `Cmd/Ctrl+K` mở palette từ bất kỳ trang nào trong shell.
- Gõ tên 1 trang trong nav rồi Enter → điều hướng đúng trang đó.
- `Esc` đóng palette, không đổi route.

## Ranh giới kiến trúc bị đụng tới

Palette cần 1 primitive overlay + input + list lọc được (kiểu "cmdk") — **kiểm tra xong `design-
system/src/components/*` (2026-09-05): chưa có primitive Command/Combobox-kiểu-palette nào**, chỉ
có `autocomplete`/`suggest-input`/`dropdown-menu`/`dialog` gần nhất. Theo đúng rule "UI →
design-system, UI+logic → platform-ui" của dự án: **phần input+overlay+list lọc (thuần trình bày,
không biết action nào tồn tại) phải build ở `@metap/ui` trước**, rồi `platform-ui` mới compose
logic đăng ký action/điều hướng lên trên. Đây là lý do chính khiến feature này nặng hơn 4 mục ở
feature 23 — không chỉ wiring, mà cần thêm atom mới ở tầng dưới.

## Rủi ro / phụ thuộc

Rủi ro cao hơn các feature 19-23 vì mở 1 UX pattern mới hoàn toàn — nên approve xong 1 vòng scoping
ngắn trước khi code, không gộp chung PR với feature 23 dù cùng nhóm 6 trong checklist. Nếu feature
23 được làm trước, tái dùng `storage.ts` từ đó cho phần "nhớ trạng thái" thay vì tự viết lại.
