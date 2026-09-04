## Phase 76: `metap-demo-waf` — loạt bug thật lộ ra khi chạy portal sống lần đầu sau Phase 71-73 (2026-09-04)

Trigger: chạy thử live (Docker) toàn bộ portal ngay sau khi pull xong Phase 70-73 — lần đầu có
người thật click qua UI mới thay vì chỉ build/test tĩnh. Mỗi bug dưới đây là 1 phát hiện độc lập
lúc thao tác thật trên browser, không phải review code.

## Đã sửa

- **`graphql-gateway` container chạy nhầm binary — bug nghiêm trọng nhất, gây ra hầu hết lỗi
  "Unknown field" người dùng thấy.** `docker-compose.dev.yml`/`docker-compose.yml` build/chạy
  binary generic `metap/crates/graphql-gateway` (`working_dir: /workspace/metap`) thay vì
  `waf-graphql-gateway` (binary riêng của WAF, Phase 73, đủ 7 mutation custom + giờ thêm
  `aggregate` generic từ core — xem Phase 75). Sửa `working_dir`/`command` (dev) và
  `build.context`/`dockerfile` (production — tạo mới `graphql-gateway/Dockerfile` cho binary này,
  trước đó không có cách build production). Kèm sửa 1 đoạn comment trong `metap` core's
  `crates/graphql-gateway/Dockerfile` từng gợi ý đúng kiểu nhầm lẫn này (`docker build ... -t
  waf-graphql-gateway ../../metap` — build binary generic rồi gắn nhãn như thể là bản riêng).
- **Race điều hướng sau login/logout** (`@metap/platform-ui`) — `LoginForm`/`AppShellLayout`'s
  `handleLogout` gọi `navigate()` ngay sau `markAuthenticated()`/`logout()` mà không chờ
  `["currentUser"]` refetch xong; `queryClient.clear()` không cập nhật `status` đồng bộ, nên
  `RequireAuth` ở route đích có thể đọc trúng `status` cũ ("anonymous" ngay sau login, "authenticated"
  ngay sau logout) và tự đá ngược lại. Sửa: `markAuthenticated`/`logout` giờ `await
  queryClient.refetchQueries(["currentUser"])` trước khi resolve — caller `await` rồi mới
  `navigate()`, không còn race.
- **Vòng lặp render-storm ở Dashboard/Analytics/Zone overview** — `daysAgo()` tính timestamp mới
  mỗi lần render (không `useMemo`), giá trị đó nằm trong `queryKey` của `useAggregate` → mỗi render
  = query key mới → cache miss → refetch → re-render → lặp lại, tới khi bị `graphql-gateway`'s rate
  limit (429). Sửa bằng `useMemo` ở 3 file (`DashboardPage`/`AnalyticsPage`/`ZoneOverviewTab`).
- **`/metadata/entities` trả rỗng cho mọi caller thật** — `vite.config.ts`'s `mergeEntityList`
  (middleware gộp danh sách entity từ 3 service) chỉ forward header `authorization`, không forward
  `cookie` — từ khi frontend chuyển hẳn sang cookie-session (không còn giữ Bearer trong JS), mọi
  request `/metadata/entities` tới middleware này mất auth khi relay sang 3 service, cả 3 trả 401,
  middleware âm thầm coi `data: []`. Sửa: forward thêm `cookie`.
- **Xoá zone luôn bị từ chối** — `zone_delete_guard` (kiểm tra cross-service reference trước khi
  xoá) gọi `SCANNING_URL`/`ALERTING_URL`, nhưng 2 biến này chỉ được set cho service `web` (dùng cho
  mục đích khác — Vite proxy) trong cả 2 file docker-compose, không set cho `zones-service` (dùng
  gọi server-to-server thật) → fallback về `http://localhost:3010` vô nghĩa trong container riêng
  → lỗi network, guard từ chối xoá vì "không xác minh được reference". Sửa: thêm 2 biến này vào
  `zones-service`'s `environment` ở cả 2 file compose.
- **`AppShellLayout` mount lại mỗi lần chuyển trang** — `App.tsx` bọc `<AppShellLayout>` bên trong
  từng `<Route>` riêng lẻ (`page()`/`adminPage()` helper) thay vì dùng layout-route của React
  Router — mỗi navigation unmount/remount cả shell, khiến `useCurrentUser()`
  (`GET /auth/me`)/`useCurrentUserEmail()` (`GET /users` fallback) refetch lại dù đã cache
  (`staleTime: Infinity` không ngăn được remount-triggered refetch). Sửa: chuyển sang
  `<Route element={<RequireAuth/>}>` + `<Outlet/>`, shell chỉ mount 1 lần cho cả phiên.

## Thêm mới

- **"Visualize workflow" (SVG diagram)** ở `ZoneDetailPage`/`IncidentDetailPage` — 2 trang chi
  tiết tự viết riêng (thay cho `RecordDetail` generic cũ) chỉ có nút transition phẳng, thiếu hẳn
  diagram trực quan mà `RecordDetail`/`WorkflowActionBar` (đường "Raw records") đã có sẵn. Thêm
  bằng cách tái dùng thẳng `WorkflowDiagram` (`@metap/platform-ui`, không code mới ở tầng vẽ) +
  `useEntity`/`useEntityLabels` để lấy `EntityWorkflow` metadata. Do đường GraphQL app này dùng
  không trả `RecordCapabilities` (chỉ REST/`RecordDetail` có), mọi transition hiển thị đều đánh dấu
  `available: true` — backend vẫn là nơi enforce thật (guard `activate` vẫn chặn đúng khi
  `hasConfig`/`verificationStatus` chưa đủ điều kiện — xem ví dụ thật gặp lúc test: zone không có
  DDoS policy/firewall rule nào bị chặn activate với lỗi `guard_failed`, đúng thiết kế).

## Xác minh

Toàn bộ sửa trên verify sống qua container Docker thật (không chỉ build tĩnh) — login/logout,
Dashboard không còn spam GraphQL, `/metadata/entities` trả đủ 9 entity, xoá zone thành công (tạo
zone test rồi xoá, HTTP 200), `aggregate`/`verifyZoneDns` cả 2 hoạt động đúng qua đúng binary
`waf-graphql-gateway`. `tsc -b`/`oxlint` sạch cho mọi file frontend đổi. **Chưa browser-test** theo
đúng frontend verification policy (để người dùng tự kiểm bằng browser thật) — verify ở đây là qua
`curl` mô phỏng đúng path browser (cookie/CSRF/proxy) cộng log container, không phải Playwright.
