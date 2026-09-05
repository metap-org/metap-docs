# GeneratedList — Export (CSV/JSON) cho dữ liệu đã tải

- **Trạng thái:** done (2026-09-05) — `platform-ui` commit `078e967`, CSV/JSON export qua
  `DropdownMenu` cạnh Refresh, có hint rõ "chỉ dòng đã tải" trong UI.
- **Người đề xuất:** rà soát `docs/frontend-checklist.md` (2026-09-04), mục 1 "Dynamic Table" >
  Export.
- **Track sở hữu:** Frontend Platform
- **Phase roadmap liên quan:** chưa gắn phase

## Vấn đề / động lực

`GeneratedList` không có nút export/CSV/JSON download nào trong `platform-ui/src`. Vì
`GeneratedList` đã virtualize + giữ toàn bộ dòng đã tải trong bộ nhớ (React Query cache,
`useApiInfiniteQuery`), export dữ liệu **đã tải** không cần đổi backend hay dựng atom mới ở
`design-system` — đúng nhóm "làm được ngay" mà checklist gợi ý.

## Phạm vi

**Trong phạm vi:**
- Nút "Export" cạnh nút Refresh/New hiện có (`GeneratedList.tsx:230-251`).
- Xuất **đúng các dòng đã tải** (loaded rows) — cố tình **không** hứa hẹn export toàn bộ N bản ghi
  khớp filter trên server, cùng quyết định có chủ đích đã áp dụng cho row-selection/bulk-delete
  ("select all = chỉ các dòng đã tải", 2026-09-04) để nhất quán hành vi trong cùng 1 component.
- 2 format: CSV và JSON, tên cột/khoá theo `listView.fields`.
- UI phải nói rõ (label/tooltip) rằng export chỉ lấy dữ liệu đang hiển thị/đã tải, tránh hiểu nhầm
  giống rủi ro đã ghi nhận ở tính năng row-selection.

**Ngoài phạm vi:**
- Export toàn bộ result set qua server (cần endpoint streaming riêng ở backend — không phải "làm
  ngay", brief riêng nếu có trigger).
- Export theo format khác (Excel/PDF).

## Tiêu chí chấp nhận

- Bấm "Export CSV" → tải về 1 file `.csv` chứa đúng các dòng đang có trong danh sách (đã loaded),
  đúng thứ tự cột theo `listView.fields`, escape đúng dấu phẩy/xuống dòng trong giá trị field.
- Bấm "Export JSON" → tải về 1 file `.json` là mảng object tương ứng, field name khớp
  `listView.fields`.
- Danh sách rỗng (0 dòng đã tải) → nút Export ở trạng thái disabled, giống cách các nút khác của
  `GeneratedList` xử lý trạng thái rỗng.
- Không có request mạng nào phát sinh khi bấm Export (dữ liệu lấy thẳng từ cache đã có).
- Hover/focus vào nút Export thấy tooltip/label nói rõ phạm vi "chỉ các dòng đã tải".

## Ranh giới kiến trúc bị đụng tới

`platform-ui/src/list/GeneratedList.tsx` — thuần UI+logic, đúng chỗ `platform-ui`. Không cần atom
mới ở `design-system` nếu `Button`/`IconButton` hiện có đủ dùng; nếu cần icon export riêng, kiểm
tra icon set `@metap/ui` đang dùng trước khi tự vẽ SVG trong `platform-ui` (vi phạm rule
UI → design-system). Không đụng backend, không cần ADR.

## Rủi ro / phụ thuộc

Rủi ro chính là kỳ vọng người dùng ("export" thường ngầm hiểu là toàn bộ dữ liệu khớp filter, không
phải chỉ trang đã cuộn tới) — cần chủ dự án xác nhận phạm vi này trước khi approve, tương tự cách
row-selection/bulk-delete đã phải làm rõ quyết định này công khai trong UI, không chỉ trong docs.
Không phụ thuộc phase/feature khác.
