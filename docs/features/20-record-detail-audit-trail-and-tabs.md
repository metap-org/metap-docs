# RecordDetail — audit trail (workflow_events) + Tabs

- **Trạng thái:** proposed
- **Người đề xuất:** rà soát `docs/frontend-checklist.md` (2026-09-04), mục 2 "Generic CRUD" >
  Detail — "Audit information" được chính checklist gợi ý là gap thấp công sức nhất trong nhóm
  ưu tiên (dữ liệu backend đã có sẵn từ Phase 5, chỉ thiếu FE consume).
- **Track sở hữu:** Frontend Platform
- **Phase roadmap liên quan:** chưa gắn phase

## Vấn đề / động lực

`workflow_events` (audit log append-only, ghi mọi transition — Phase 5) tồn tại ở backend từ lâu,
và theo `metap/CLAUDE.md`'s route-group list, `metap-http` đã có sẵn route
`/api/:entity*/workflow-events`. Nhưng `platform-ui/src` có **0 tham chiếu** tới nó — audit log đã
tồn tại, chỉ chưa lộ ra UI.

Cùng lúc, `RecordDetail.tsx` chưa dùng `Tabs` (atom Radix-based đã có ở `@metap/ui`, mới chỉ dùng ở
`admin/PoliciesAdminPage.tsx`) — field hiện render flat trong 1 grid 2 cột dài. Gộp 2 gap vào 1
feature vì cùng 1 chỗ sửa: thêm tab "Lịch sử" là lý do tự nhiên để đưa `Tabs` vào `RecordDetail`
lần đầu, tránh phải sửa `RecordDetail.tsx` hai lần cho hai gap liên quan.

## Phạm vi

**Trong phạm vi:**
- `RecordDetail` chuyển sang bố cục `Tabs`: tab "Chi tiết" (nội dung hiện tại, không đổi) + tab
  "Lịch sử" (audit).
- Tab "Lịch sử" gọi `GET /api/:entity/:id/workflow-events` (cần đọc lại
  `crates/metap-http/src/routes/*` tương ứng để xác nhận đúng path/response shape thật trước khi
  code — chưa từng có FE nào gọi route này, không giả định shape).
- Hiển thị mỗi event: thời gian, người thực hiện, `from → to` state, action — sắp giảm dần theo
  thời gian.
- Entity không có `stateField` (không phải workflow entity) → không hiện tab "Lịch sử", không gọi
  API thừa.
- Loading/empty state cho tab "Lịch sử" (record mới tạo, chưa từng transition).

**Ngoài phạm vi:**
- Field groups có cấu trúc (gap khác của Detail — cần khái niệm "group" mới trong
  `entityLayout.ts`, là quyết định thiết kế riêng, không lẫn vào feature này).
- Activity feed tổng quát hơn audit log thuần workflow (ví dụ comment/mention) — chưa có backend
  nào hỗ trợ, khác phạm vi.
- Sửa/mở rộng backend nếu response thiếu field cần thiết — xem "Rủi ro" bên dưới.

## Tiêu chí chấp nhận

- Mở 1 record của entity có workflow → thấy 2 tab "Chi tiết"/"Lịch sử", tab "Chi tiết" mặc định
  active, không đổi hành vi hiện có.
- Chuyển sang tab "Lịch sử" → thấy danh sách transition thật của record đó, đúng thứ tự thời gian
  giảm dần, không trộn record khác.
- Record vừa tạo (chưa transition lần nào) → tab "Lịch sử" hiện empty state, không lỗi.
- Entity không có workflow (không `stateField`) → `RecordDetail` không hiện tab "Lịch sử" và
  không gọi API `/workflow-events` (kiểm bằng Network tab).

## Ranh giới kiến trúc bị đụng tới

`platform-ui/src/detail/RecordDetail.tsx` + 1 hook mới (kiểu `useWorkflowEvents`, cạnh
`platform-ui/src/api/useApiQuery.ts` hiện có). Route backend `/api/:entity*/workflow-events` đã
tồn tại theo tài liệu, nhưng **cần đọc code thật ở `crates/metap-http` trước khi code FE** để xác
nhận response shape — không phải ranh giới mới nếu route đã đúng như tài liệu mô tả, nhưng nếu
thiếu field thì lan sang HTTP-API Surface (xem Rủi ro).

## Rủi ro / phụ thuộc

Nếu response của `/workflow-events` không có tên hiển thị người thực hiện (chỉ `user_id`, không
có email/tên), tab "Lịch sử" sẽ phải tự resolve tên qua `GET /users` riêng (thêm 1 round-trip),
hoặc backend cần bổ sung field — trường hợp sau lan sang track HTTP-API Surface, cần sign-off
chéo track theo `docs/team-charter.md` và đánh giá lại phạm vi trước khi tiếp tục. Không phụ thuộc
phase/feature nào khác.
