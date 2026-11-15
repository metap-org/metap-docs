# 111. List: phân trang số, search, quyền theo entity, widget nhập liệu

**Trạng thái:** làm xong 2026-11-15 trên nhánh `claude/metap-docs-features-g1zjqn` của metap, design-system, platform-ui, waap (PR chưa merge). Chưa kiểm tra trên trình duyệt (theo quy ước FE: người dùng tự xem).

## Vấn đề (chủ dự án nêu)

- Màn list không có ô search, không có tổng số/trang kiểu `1 2 3 … last` (chỉ cuộn vô hạn theo cursor, API không trả tổng).
- Nút (New, Xóa) và menu không theo quyền ở màn list; chỉ trang chi tiết dùng `capabilities`.
- Thiếu widget: switch, multi checkbox, combobox, radio, tags.
- waap tự dựng từng trang (list, search, filter, phân trang) thay vì dùng `GeneratedList`.

## Backend (metap)

- `ListInput` thêm `Offset`, `Search`, `WithTotal`. REST: `?offset=&search=&withTotal=true`; gRPC: cùng khoá trong `query`, `ListResponse` thêm `total`/`has_total`/`can_*`; GraphQL: đối số `offset`, `search`, `withTotal`, field `total`, `capabilities` trên `…Connection`.
- `offset` và `cursor` loại trừ nhau; `offset` tối đa `query.MaxListOffset` (100000), vượt thì dùng cursor (OFFSET sâu phải quét và bỏ mọi dòng bị bỏ qua). Trang theo cursor không trả `total` (đếm "sau cursor" không phải thứ người gọi muốn).
- `search` OR qua mọi field `searchable` kiểu string/enum (ILIKE hoặc tsvector nếu `searchMode: fts`); entity không có field nào thì `400 search_not_supported`.
- `ListResult.Capabilities` (`canCreate/canUpdate/canDelete`, quyền cấp entity) và `GET /metadata/entity-access` (read/create/update/delete cho mọi entity của tenant) để FE ẩn nút/menu.
- `EntityField.Widget`: `switch|checkbox|select|radio|combobox|textarea|multiselect|checkboxGroup|tags`, `Validate` kiểm tra hợp kiểu (switch chỉ cho boolean, ...). Multi-value lưu `json` là mảng chuỗi; `multiselect`/`checkboxGroup` kiểm từng phần tử thuộc `enumValues`, `tags` tự do.
- Test: `list_paging_postgres_test.go`, `validation_widget_test.go`, `compiler_widget_test.go`, `TestEntityAccess…`.

## Frontend

- `GeneratedList`: phân trang số (design-system `Pagination`, nhãn truyền được), tổng số, chọn số dòng/trang, ô search (chỉ hiện khi entity có field searchable), URL đồng bộ `q/page/size`; New/xóa từng dòng/xóa hàng loạt theo capability.
- `useEntityAccess` + `ShellNavItem.entity`: ẩn mục menu khi không có quyền đọc.
- `FieldInput` theo `field.widget`; ô multi-value hiển thị danh sách thay vì JSON.
- Điểm mở rộng để app viết ít code: `title`, `actions`, `rowHref`, `cellRenderers`, `baseFilters`, `newHref`, `readOnly`. Màu badge theo giá trị enum đã có sẵn trong metadata (`FieldDisplayHint.enumTones`).
- waap: `ZonesPage` còn ~25 dòng (trước ~150), cột/tone lấy từ metadata của `waf.zones`; `enabled` là switch, `protectionMode`/`sensitivity` là radio; menu ẩn theo quyền đọc entity.

## Còn lại

- Các trang waap khác (Incidents, Findings, Alerting, panel rule/access list) vẫn hand-built; chuyển dần sang `GeneratedList` + `cellRenderers`.
- Phân trang số vẫn kèm đếm `COUNT(*)` mỗi lần đổi trang: bảng rất lớn (`security_events`, audit) nên giữ cursor.
- Menu của trang admin/cron vẫn gate theo role, chưa theo entity.
- `generated-types.ts` (platform-ui) được vá tay thêm `widget` vì sinh từ OpenAPI cần chạy server; sinh lại khi tiện.
