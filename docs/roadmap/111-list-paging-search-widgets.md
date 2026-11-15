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
- Test: `list_paging_postgres_test.go`, `validation_widget_test.go`, `compiler_widget_test.go`, `TestFullHTTPLifecycleOverARealServerAndARealJWT` (phần `entity-access`).

## Frontend

- `GeneratedList`: phân trang số (design-system `Pagination`, nhãn truyền được), tổng số, chọn số dòng/trang, ô search (chỉ hiện khi entity có field searchable), URL đồng bộ `q/page/size`; New/xóa từng dòng/xóa hàng loạt theo capability.
- `useEntityAccess` + `ShellNavItem.entity`: ẩn mục menu khi server trả lời rõ là không có quyền đọc (`useEntityDenied`). Chưa biết (đang tải, request lỗi, entity không có trong map) thì vẫn hiện; nút bấm thì ngược lại, chưa biết là chưa cho (`useCanAccessEntity`).
- `FieldInput` theo `field.widget`; ô multi-value hiển thị danh sách thay vì JSON.
- Điểm mở rộng để app viết ít code: `title`, `actions`, `rowHref`, `cellRenderers`, `baseFilters`, `newHref`, `readOnly`. Màu badge theo giá trị enum đã có sẵn trong metadata (`FieldDisplayHint.enumTones`).
- waap: `ZonesPage` còn ~25 dòng (trước ~150), cột/tone lấy từ metadata của `waf.zones`; `enabled` là switch, `protectionMode`/`sensitivity` là radio; menu ẩn theo quyền đọc entity.

## Sửa sau review (2026-11-15, trước khi merge)

- **`GET /metadata/entity-access` chưa hề được viết** dù mô tả PR metap có nêu; `useCanAccessEntity` trả `false` khi chưa có dữ liệu nên mọi mục menu gắn entity của waap (Zones, Rules, Access lists, Incidents, Findings, Alerting) sẽ biến mất với mọi người, kể cả admin. Đã thêm endpoint (`routes_metadata.go`, cùng kiểm tra cấp entity như đường đọc/ghi, test trong `http_server_postgres_test.go`). Mỗi service chỉ trả entity của mình, nên waap gộp 3 service trong Vite dev (`mergeEntityAccess`, giống `mergeEntityList`); nginx prod chỉ tới zones-service (cùng khoảng trống đã ghi cho `/metadata/entities`). Vì vậy menu chuyển sang "chỉ ẩn khi bị từ chối rõ ràng", để map thiếu hoặc lỗi không làm rỗng menu.
- **`?page=` trên URL bị mất khi tải lại**: effect `setPage(1)` theo sort/filter/search chạy ngay sau bước đọc URL và lại chạy khi search debounce xong. Bỏ effect đó; mỗi control (sort, filter, search, số dòng/trang) tự về trang 1 trong handler, còn đổi entity/`baseFilters` reset trước bước đọc URL.
- **Chọn hàng loạt giữ qua các trang**: selection chỉ xoá khi đổi entity/sort/filter, nên sang trang khác rồi "Xóa đã chọn" sẽ xoá cả dòng không còn trên màn hình. Giờ xoá cả khi đổi trang, search, số dòng/trang.
- `RecordWorkspace`: đổi tab chỉ giữ `tab`, bỏ `q/page/sort/filter` của list ở tab cũ (tab mới đọc nhầm thành của mình).
- waap `ZonesPage`: trả lại địa chỉ origin dưới hostname và cảnh báo vàng khi zone chưa có cấu hình bảo vệ (qua `cellRenderers`).
- platform-ui: bỏ `tsconfig.tsbuildinfo` bị commit nhầm, thêm `*.tsbuildinfo` vào `.gitignore`.

## Còn lại

- Các trang waap khác (Incidents, Findings, Alerting, panel rule/access list) vẫn hand-built; chuyển dần sang `GeneratedList` + `cellRenderers`.
- Phân trang số vẫn kèm đếm `COUNT(*)` mỗi lần đổi trang: bảng rất lớn (`security_events`, audit) nên giữ cursor.
- Menu của trang admin/cron vẫn gate theo role, chưa theo entity.
- Nút New trong một tab liên quan của `RecordWorkspace` mở form tạo trống, chưa điền sẵn khoá liên kết (vd. `zoneId`).
- Prod (nginx) chưa gộp `/metadata/entity-access` từ 3 service: menu của entity scanning/alerting không ẩn theo quyền ở prod (vẫn hiện, server vẫn chặn).
- `generated-types.ts` (platform-ui) được vá tay thêm `widget` vì sinh từ OpenAPI cần chạy server; sinh lại khi tiện.
