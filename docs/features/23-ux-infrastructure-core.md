# UX Infrastructure nền tảng — URL state, global loading/error, local storage, responsive

- **Trạng thái:** done (2026-09-05) — `platform-ui` commit `078e967`. Cả 5 mục trong phạm vi đã
  code: URL state qua `useSearchParams`, global loading indicator, `ErrorBoundary` (build từ
  `Alert`/`Button` có sẵn, không cần atom mới), `useLocalStorage`, nav mobile thu gọn hamburger.
- **Người đề xuất:** rà soát `docs/frontend-checklist.md` (2026-09-04), mục 6 "UX Infrastructure"
  — 10/11 mục đang `[ ]`. Tách 4 mục "hạ tầng, rủi ro thấp, ảnh hưởng rộng" ra khỏi 3 mục còn lại
  cần quyết định phạm vi trước (xem
  [24. Command palette & state persistence](24-command-palette-and-state-persistence.md)) —
  giống cách Phase 01 từng tách "có trigger nhưng chưa scope" khỏi phần đã rõ việc.
- **Track sở hữu:** Frontend Platform
- **Phase roadmap liên quan:** chưa gắn phase

## Vấn đề / động lực

Nhóm UX Infrastructure của checklist gần như trắng — chỉ "Browser back/forward" (mặc định của
`react-router`) là `[x]`. 4 mục dưới đây đều là hạ tầng thuần FE (không đụng backend/metadata,
không cần quyết định sản phẩm gây tranh cãi), ảnh hưởng tới **mọi** app downstream cùng lúc vì
sống ở `platform-ui`, nên xếp vào nhóm "làm được ngay":

- **URL state / deep linking**: filter/sort của `GeneratedList` sống trong `useState`
  (`GeneratedList.tsx:78-81`), mất khi F5 hoặc share link.
- **Global loading state**: loading xử lý per-screen (`Spinner` riêng từng component), không có
  tín hiệu cấp app khi có request đang chạy ở nền.
- **Global error handling**: `AppShellLayout.tsx` chưa có `ErrorBoundary` — 1 lỗi runtime ở
  component con làm trắng cả trang thay vì chỉ hỏng đúng vùng đó.
- **Responsive/mobile**: chỉ 3 chỗ dùng breakpoint Tailwind trong toàn `platform-ui/src`
  (`RecordDetail.tsx`/`RelatedRecordsPanel.tsx`); `GeneratedList`'s table cố định width và
  `AppShellLayout`'s header nav chưa xử lý màn hẹp (không hamburger/collapse).
- **Local storage abstraction**: 0 tham chiếu `localStorage`/`sessionStorage` trong toàn bộ
  `platform-ui/src` — thêm 1 tiện ích nhỏ, generic, dùng lại được cho các nhu cầu persist-phía-
  client sau này (kể cả feature 24).

## Phạm vi

**Trong phạm vi:**
- `GeneratedList` sync filter/sort qua `useSearchParams` (react-router) thay vì `useState` nội bộ
  — đổi URL trực tiếp hoặc F5 phải restore đúng trạng thái.
- 1 provider/indicator loading cấp app (kiểu progress-bar mỏng ở top, dựa trên React Query's
  `useIsFetching()` toàn cục) — **bổ sung**, không thay thế `Spinner` per-screen hiện có.
- `AppShellLayout` bọc 1 `ErrorBoundary` — lỗi runtime ở nội dung trang hiện fallback UI + nút
  "Thử lại" (reset boundary) thay vì trắng trang toàn bộ shell.
- 1 utility `storage.get`/`storage.set` (try/catch quanh mọi lần đọc/ghi — private mode/storage bị
  chặn không được làm crash app) trong `platform-ui/src/lib/` hoặc tương đương.
- `AppShellLayout`'s header nav collapse thành hamburger dưới 1 breakpoint (`md` Tailwind);
  `GeneratedList`'s table cuộn ngang trong khung riêng (`overflow-x: auto`) thay vì vỡ layout
  trên màn hẹp.

**Ngoài phạm vi:**
- Local storage abstraction dùng để persist **cái gì** (filter preset, tab đang mở, ...) — đó là
  quyết định sản phẩm, để ở [feature 24](24-command-palette-and-state-persistence.md); feature
  này chỉ dựng cái utility, chưa gắn business logic persist nào lên nó.
- Keyboard shortcuts / command palette — feature 24.
- Audit accessibility toàn diện (skip-link, `aria-live` cho async state) — task lớn hơn, cần 1
  audit riêng nếu muốn làm nghiêm túc, không lẫn vào đây.

## Tiêu chí chấp nhận

- Đổi filter/sort trên `GeneratedList` → URL query string đổi theo; copy URL đó mở tab mới → thấy
  đúng filter/sort đã set. F5 trang hiện tại → filter/sort không mất.
- Nút Back của trình duyệt sau khi đổi filter → quay lại đúng trạng thái filter trước đó (không
  nhảy hẳn ra khỏi trang, không lẫn với record-ID route param hiện có).
- Bật Network throttling, thao tác gọi bất kỳ query nào trong app → thấy global loading indicator
  hiện; tắt throttling → indicator biến mất khi hết fetch.
- Ép 1 component con throw lỗi (test tay) → `ErrorBoundary` hiện fallback UI + nút "Thử lại"; bấm
  nút đó → boundary reset, không cần reload cả trang.
- `storage.get`/`storage.set` không throw khi `localStorage` bị chặn (test bằng chế độ ẩn danh có
  chặn site data, hoặc mock `localStorage` throw).
- Thu nhỏ viewport xuống dưới breakpoint mobile → `AppShellLayout`'s nav thu về hamburger, mở ra
  đóng vào được; `GeneratedList` không vỡ layout ngang (cuộn được, không tràn trang).

## Ranh giới kiến trúc bị đụng tới

`platform-ui/src/shell/AppShellLayout.tsx`, `platform-ui/src/list/GeneratedList.tsx`, 1 utility
mới (`platform-ui/src/lib/storage.ts` hoặc tương đương), 1 `ErrorBoundary` component mới. **Cần
soi đúng ranh giới UI → design-system / UI+logic → platform-ui khi dựng `ErrorBoundary`**: phần
*hiển thị* fallback (card/message/nút "Thử lại") nên là 1 atom mới ở `@metap/ui` (thuần trình bày,
không biết gì về React error-boundary lifecycle), còn class component bắt lỗi (`componentDidCatch`,
logic reset) thuộc `platform-ui`, import atom kia để render — không viết cả 2 phần gộp chung trong
`platform-ui` như 1 khối, đúng rule đã nêu trong yêu cầu tính năng này. Không cần ADR (không đổi
API/contract nào ra ngoài `platform-ui`).

## Rủi ro / phụ thuộc

Global loading indicator cần đặt `useIsFetching()` ở đâu trong tree (trong `AppShellLayout` hay 1
provider riêng mount ở `main.tsx` của từng app) — quyết định nhỏ lúc code, không phải rủi ro kiến
trúc. Không phụ thuộc phase/feature khác; feature 24 (command palette) có thể tái dùng
`storage.ts` từ đây nên nên làm feature này trước nếu cả 2 được approve.
