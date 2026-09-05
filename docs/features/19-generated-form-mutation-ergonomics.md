# GeneratedForm — dirty state, unsaved-changes guard, partial update thật, optimistic update

- **Trạng thái:** done (2026-09-05) — dirty-state/partial-update/optimistic-update/beforeunload
  guard đã code, `platform-ui` commit `078e967`. In-app navigation blocker (`useBlocker`) xác
  nhận không dùng được (mọi app dùng `<BrowserRouter>` thường, không phải data router) — bỏ khỏi
  scope, chỉ còn `beforeunload`, đã ghi rõ trong code.
- **Người đề xuất:** rà soát `docs/frontend-checklist.md` (2026-09-04), mục 2 "Generic CRUD" >
  Update — 4 gap generic cho mọi entity qua `GeneratedForm`, không phải case-by-case, nên gộp
  thành 1 feature thay vì 4 việc rời rạc.
- **Track sở hữu:** Frontend Platform
- **Phase roadmap liên quan:** chưa gắn phase

## Vấn đề / động lực

4 gap dưới đây đều nằm ở tầng mutation/form-state chung (`platform-ui/src/form/GeneratedForm.tsx`,
`platform-ui/src/api/useApiMutation.ts`), không đụng entity cụ thể nào — sửa 1 lần áp dụng cho mọi
app downstream:

- **Partial update chưa thật partial**: `updateMutation` gọi đúng `PATCH /api/{entity}/{id}`
  nhưng payload gửi lại cả object `formData` dựng lại từ mọi key (`GeneratedForm.tsx:86-99,
  102-104`), không phải diff chỉ field đã đổi.
- **Không có dirty state**: đã có tiền lệ cô lập ở `admin/policies/PermissionMatrix.tsx:264,
  342-349` (track `dirty` boolean + label "You have unsaved changes" + disable Save) nhưng chưa
  nhân rộng cho `GeneratedForm`.
- **Không có unsaved-changes guard**: không `beforeunload`/router-blocker nào — label ở
  `PermissionMatrix.tsx` chỉ thông báo thụ động, không chặn điều hướng.
- **Không có optimistic update**: cả `create`/`update` đều `await` mutation rồi mới
  refetch/navigate; `useApiMutation.ts` chưa có `onMutate`/rollback.

## Phạm vi

**Trong phạm vi:**
- `GeneratedForm` tính diff giữa `formData` hiện tại và `existing.data` (khi edit) trước khi gọi
  `PATCH` — chỉ gửi field đã đổi thật.
- Dirty-state tracking tổng quát hoá từ pattern `PermissionMatrix.tsx`, dùng chung cho mọi
  `GeneratedForm` instance (create lẫn edit).
- Unsaved-changes guard: chặn điều hướng trong-app khi form dirty (xác nhận trước khi rời), cộng
  `beforeunload` cho refresh/đóng tab.
- Optimistic update **chỉ cho update mutation** — cập nhật UI ngay khi submit, rollback về giá trị
  cũ nếu request fail.

**Ngoài phạm vi:**
- Optimistic update cho `create`/`delete` — `create` cần khái niệm tempId phức tạp hơn, `delete`
  rủi ro UX (mất dữ liệu nhìn thấy) cao hơn lợi ích cho 1 vòng đầu; để riêng nếu có nhu cầu sau.
- Undo/redo stack sau khi rollback — rollback về giá trị cũ trước khi submit là đủ.
- Saved views / lưu lại filter — khác gap, thuộc nhóm Dynamic Table.

## Tiêu chí chấp nhận

- Sửa 1 field trên `GeneratedForm` (edit mode) rồi bấm link điều hướng sang route khác → có hộp
  thoại xác nhận rời trang; huỷ thì ở lại đúng trang cũ với dữ liệu chưa mất.
- Sửa 1 field rồi refresh/đóng tab → trình duyệt hiện cảnh báo `beforeunload` gốc.
- Mở Network tab, sửa đúng 1 field rồi submit ở edit mode → request `PATCH` body chỉ chứa field
  đó (và id/version nếu cần cho optimistic-lock), không chứa các field không đổi.
- Update mutation: UI (ví dụ `RecordDetail` đang mở) phản ánh giá trị mới **ngay khi bấm Save**,
  trước khi response về; tắt mạng giả lập lỗi → UI rollback đúng về giá trị cũ, có toast báo lỗi.
- Không regression: flow `create`/`delete` hiện tại (không đổi optimistic) vẫn hoạt động y hệt
  trước, kể cả toast thành công/lỗi đã có.

## Ranh giới kiến trúc bị đụng tới

Chỉ trong `platform-ui/src/form/GeneratedForm.tsx` + `platform-ui/src/api/useApiMutation.ts` —
thuần UI+logic tái dùng, đúng chỗ `platform-ui`, không đụng `design-system` (không cần atom mới),
không đụng backend. Không cần ADR.

## Rủi ro / phụ thuộc

`react-router` cung cấp `useBlocker` cho unsaved-changes guard trong-app, nhưng hook này **yêu
cầu data router** (`createBrowserRouter`), không hoạt động với `<BrowserRouter>` truyền thống.
`metap-demo-waf/data-plane/web/src/main.tsx` (và nhiều khả năng cả `metap-demo-crm`/
`metap-demo-jira`) đang dùng `<BrowserRouter>` cổ điển — cần xác nhận lại thật ở từng app trước
khi code, vì nếu đúng vậy, guard trong-app phải làm bằng cách khác (chặn `onClick` của
`NavigationAdapter.Link`/`navigate`, không phải `useBlocker`) — phạm vi/khó độ có thể tăng nếu
phải sửa cách `Link`/`navigate` hoạt động trong `NavigationContext`. Không phụ thuộc phase/feature
khác.
