# `GeneratedList` — chọn cột hiển thị, cột bắt buộc khai ở entity

- **Trạng thái:** done, 2026-09-06
- **Người đề xuất:** chủ dự án, 2026-09-06 — trực tiếp yêu cầu trong phiên làm việc
- **Track sở hữu:** Frontend Platform + Backend Core (`EntityListView` là contract `metap-metadata`)
- **Phase roadmap liên quan:** không thuộc phase nào

## Vấn đề / động lực

`GeneratedList` (`@metap/platform-ui`) trước đây luôn render **toàn bộ** `listView.fields` làm
cột — không có cách nào để người dùng ẩn bớt cột không cần xem trên màn hình cụ thể của họ (list
nhiều field dễ tràn ngang, đặc biệt bảng >6-7 cột trên viewport hẹp). Yêu cầu cụ thể: thêm option
chọn cột nào hiển thị/ẩn, phân biệt cột nào **bắt buộc** (không cho ẩn, do entity khai) vs cột nào
người dùng tự do bật/tắt, và trạng thái đã chọn lưu ở FE (`localStorage`), không phải trên server.

## Phạm vi

**Trong phạm vi:**
- `EntityListView` (`crates/metap-metadata/src/entity.rs`) thêm `required_fields: Vec<String>`
  (wire: `requiredFields`) — subset của `fields`, cùng hợp đồng "không validate, cứ tin metadata
  khai đúng" mà `filters` đã có. Rỗng (mặc định, `#[serde(default)]`) = mọi cột đều ẩn được, đúng
  hành vi cũ.
- `GeneratedList.tsx`: nút "Cột hiển thị" (icon 2 cột) trong toolbar, mở `DropdownMenu` liệt kê
  toàn bộ `listView.fields` dạng checkbox (`DropdownMenuCheckboxItem`) — cột nằm trong
  `requiredFields` bị khoá `checked` + `disabled`, có `title` giải thích vì sao không ẩn được; cột
  còn lại tự do toggle, menu không tự đóng khi tick (để bật/tắt nhiều cột 1 lượt).
- Cột ẩn/hiện áp dụng đồng bộ cho cả 3 chỗ render dùng `listView.fields` để giữ `gridTemplateColumns`
  đúng số cột: header, hàng filter (khi mở), và cell dữ liệu từng row.
- Trạng thái lưu `localStorage`, key theo `entityName` + `listView.name` (`platform-ui.list.hiddenColumns.<entity>.<view>`)
  — lưu **tập cột ẩn**, không phải tập cột hiện, để field mới entity thêm sau này mặc định hiện ra
  thay vì bị ẩn oan vì chưa từng được lưu.

**Ngoài phạm vi (chắc chắn):**
- Export CSV/JSON (`exportLoadedRecords`/`handleExportAll`) vẫn xuất **toàn bộ** `listView.fields`,
  không theo cột đang ẩn/hiện trên màn hình — export là xuất dữ liệu, ẩn cột chỉ là gọn màn hình,
  không nên vô tình làm mất cột trong file xuất ra vì người dùng đang ẩn nó để xem cho dễ.
- Sync trạng thái cột giữa các thiết bị/trình duyệt (vd qua `/preferences`, cơ chế đã có cho locale)
  — yêu cầu rõ ràng là lưu ở FE (`localStorage`), không phải trên server.
- Cho phép người dùng đổi **thứ tự** cột (kéo-thả) — chỉ ẩn/hiện, không đổi thứ tự
  `listView.fields` khai trong metadata.

## Tiêu chí chấp nhận

- `cargo test -p metap-metadata` xanh (không có test riêng cho `required_fields` — hành vi thuần
  serde giống hệt `filters`, không thêm rủi ro logic mới ở tầng Rust).
- `cargo build --workspace --tests --benches` xanh ở cả 3 workspace path-depend vào `metap`:
  `metap`, `metap-demo-waf/data-plane`, `metap-lowcode` — 40 chỗ dựng `EntityListView` struct
  literal (8 entity thật của WAF + template scaffold + phần còn lại là test/bench fixture trong
  `metap`) đều cập nhật thêm `required_fields: vec![]` để compile lại, không đổi hành vi.
- `GET /metadata/openapi.json` từ `crm-server` đang chạy phản ánh đúng `requiredFields` trong
  schema `EntityListView` (verify sống qua `curl`, không chỉ đọc code).
- `platform-ui`'s `generated-types.ts` regenerate qua `pnpm generate:types` (chạy thật, không tay
  sửa).
- `tsc --noEmit`/`oxlint`/`prettier --check` sạch trên `GeneratedList.tsx` + `resources.ts` (i18n
  key `common.columns`/`common.columnsRequiredHint`, cả `en`/`vi`).
- Chưa entity thật nào khai `requiredFields` — mặc định rỗng nghĩa là mọi cột hiện tại vẫn ẩn được
  tự do, hành vi cũ giữ nguyên cho tới khi một entity cụ thể khai cột nào bắt buộc.
- Chưa browser-test (đúng frontend verification policy) — để user tự kiểm thao tác ẩn/hiện +
  reload trang xem có nhớ đúng lựa chọn qua `localStorage` không.

## Ranh giới kiến trúc bị đụng tới

`crates/metap-metadata/src/entity.rs` + `openapi.rs` (Backend Core, thêm field mới, không đổi field
cũ nào — không cần ADR, cùng tiền lệ `filters`). `platform-ui/src/list/GeneratedList.tsx` +
`src/i18n/resources.ts` (Frontend Platform). 2 entity thật của `metap-demo-waf` không đổi hành vi,
chỉ cập nhật literal cho compile — chưa entity nào chủ động khai `requiredFields` khác rỗng.

## Rủi ro / phụ thuộc

Không phụ thuộc feature khác. Rủi ro chính đã lường trước: `localStorage` có thể bị chặn (Safari
private mode) — `loadHiddenColumns`/`saveHiddenColumns` nuốt lỗi, coi như "chưa ẩn cột nào" thay vì
crash. Một entry `hiddenFields` cũ trỏ vào field đã bị xoá khỏi `listView.fields` tự động biến mất
vì `visibleFields` chỉ lọc trên `listView.fields` hiện tại — không cần dọn rác `localStorage` thủ
công.
