## Phase 60: `GeneratedList`/`RecordDetail` — sửa layout, thêm back-link, tận dụng diện tích màn hình (2026-09-01)

Trigger: chủ dự án phản hồi trực tiếp sau khi dùng qua UI thật — "màn list xấu thế, màn detail
thì không có phần back lại màn list, diện tích đang không được tối ưu, bé quá". 3 việc cụ thể,
đều ở `platform-ui` (dùng chung cho mọi entity, mọi app downstream).

## 1. Bug alignment cột thật ở `GeneratedList` — nguyên nhân chính của "xấu"

Đọc code trước khi sửa: mỗi data row được `@tanstack/react-virtual` render với
`position: absolute` (kỹ thuật virtualize chuẩn) bên trong 1 `<table>` dùng
`table-layout: auto` (mặc định của trình duyệt, `Table` component chưa từng set gì khác).
`table-layout: auto` tính độ rộng cột chỉ dựa trên các row **còn nằm trong normal flow** — mọi
data row đã bị lấy ra khỏi flow bởi `position: absolute`, nên trình duyệt chỉ còn đúng row header
(`<thead>`) để tính độ rộng cột, data cell thực tế không hề khớp cột với header. Đây gần như chắc
chắn là nguyên nhân chính khiến list "xấu"/lệch, không phải chỉ là thiếu style.

Sửa bằng `table-fixed` (`Table className="table-fixed"`) — với layout này, trình duyệt chỉ đọc độ
rộng ở đúng 1 row đầu tiên của bảng (row label, không phải row filter) để quyết định độ rộng mọi
cột, mọi row khác (kể cả row absolute) tự động ăn theo, không phụ thuộc việc row đó có nằm trong
flow hay không. Cột "actions" được gán độ rộng cố định (`ACTIONS_COLUMN_WIDTH = 140`px) trên đúng
row label; các cột dữ liệu còn lại chia đều phần còn lại theo hành vi mặc định của
`table-layout: fixed` khi không set độ rộng riêng. Thêm `truncate` cho data cell để nội dung dài
không phá layout dưới độ rộng cột cố định.

Polish thêm (cùng lúc, cùng nguyên nhân "xấu"): mũi tên sort đổi từ ký tự `▼`/`▲` thô sang icon SVG
nhỏ nhất quán style với các icon khác trong `@metap/ui` (`Chip`'s nút xoá cũng dùng pattern SVG
inline tương tự); `Input`/`Select` ở row filter thu nhỏ (`h-8` thay `h-10` mặc định) — trước đó 2
row header (label + filter) đều dùng control cỡ mặc định làm phần header nặng/dày bất thường so
với data row.

## 2. `RecordDetail` — thêm back-link

Không có bất kỳ cách nào quay lại list từ trang detail ngoài nút Back của trình duyệt — xác nhận
đúng như phản hồi. Thêm link "← Back to {label}" (`navAdapter.toRecordList(entityName)`, hàm đã
có sẵn — `handleDelete` vốn đã dùng để điều hướng sau khi xoá, chỉ chưa ai lộ ra làm link UI) ngay
trên tiêu đề. Key i18n mới `common.backToList` (2 locale `en`/`vi`, theo đúng convention
`{{label}}` interpolation `noListView` đã dùng).

## 3. Tận dụng diện tích màn hình

- `GeneratedList`: thử bỏ hẳn `max-w` (full-bleed `w-full`) trước — chủ dự án gửi ảnh chụp thật
  sau khi build/chạy: `table-fixed` (mục 1) chia đều cột đúng như thiết kế, nhưng trên màn hình
  rộng thật, cột dữ liệu ngắn (`1`, `12`, `waf`, `allow`) bị kéo giãn cách nhau rất xa — nhìn trống
  trải, tệ hơn cả trước khi sửa. **Chốt lại**: `max-w-6xl` cố định (~1152px) → `max-w-7xl`
  (~1280px, vẫn `mx-auto` để giữ canh giữa) — tăng vừa phải, không full-bleed. Bài học: với
  `table-fixed` chia đều cột, độ rộng container càng lớn thì cột nội dung ngắn càng bị kéo giãn vô
  lý — bảng admin nhiều cột ngắn (số, badge, tên ngắn) cần 1 trần chiều rộng hợp lý, không phải
  "càng rộng càng tốt". Chiều cao vùng scroll đổi từ `h-[600px]` cố định (không đổi theo viewport)
  sang `h-[calc(100vh-280px)]` + `min-h-[420px]` làm sàn cho màn nhỏ. Con số 280px là ước lượng
  phần chrome cố định phía trên (header shell 60px + padding + tiêu đề) — ghi rõ trong code là
  heuristic, chưa verify hết mọi kích thước màn hình.
- `RecordDetail`: `max-w-3xl` (768px, khá chật cho 1 grid 2 cột) → `max-w-5xl` (1152px); giữ
  nguyên grid 2 cột (`sm:grid-cols-2`) — không thêm cột thứ 3 vì `getFieldLayoutHint`'s `span`
  hiện chỉ tính cho lưới 2 cột, đổi số cột sẽ làm field `span: 2` không còn nghĩa là "hết chiều
  rộng" nữa, rủi ro layout sai không đáng đánh đổi trong phase này.

**Cố ý không đụng `AppShellLayout`** (shell dùng chung cho mọi trang, không chỉ list/detail) — sửa
`main`/header thành full-height flex để bảng tự co giãn "đẹp" hơn theo viewport là khả thi nhưng
rủi ro cao hơn nhiều (ảnh hưởng mọi trang khác chưa audit hết ở cả `apps/crm-fe`/`apps/jira-fe`),
nên chọn cách ít rủi ro hơn (`calc(100vh-...)` cục bộ trong `GeneratedList`) cho phase này.

## Xác minh

`platform-ui`: `typecheck`/`lint`/`format:check` sạch. `apps/crm-fe`/`apps/jira-fe`'s `web/`:
`tsc -b` sạch (2 consumer thật của cả `GeneratedList`/`RecordDetail`). **Không tự verify bằng
browser** — đúng chính sách "Frontend verification policy" (viết code, chạy typecheck/lint, giao
lại cho chủ dự án tự kiểm trên browser thật), đặc biệt quan trọng ở phase này vì thay đổi chủ yếu
là visual/layout — con số `calc(100vh-280px)` là ước lượng, có thể cần chỉnh lại sau khi nhìn thấy
thật trên màn hình.
