## Phase 69: Dev tooling (docker-compose/mold) + fix phase 64's downstream fallout (2026-09-03)

Trigger: chủ dự án hỏi "chưa có docker compose dev... rust build đỡ nặng mỗi khi change k" — bắt
đầu là việc thuần dev-tooling, nhưng lúc dựng dev-compose cho `metap-demo-waf` và thử đăng nhập
thật thì lộ ra: [[64-cookie-session-persistence]] (đổi `useAuth().token` → `useAuth().status`,
cookie `HttpOnly` thay JWT-trong-React-state) **chưa được lan xuống 3 app tiêu dùng**
(`metap-demo-crm`, `metap-demo-jira`, `metap-demo-waf`) — cả 3 vẫn giữ code cũ, gãy hoàn toàn luồng
login. Phase 64's "Chưa test bằng browser thật" note chính là lý do lỗi này không bị bắt lúc đó.

Đụng **7 chỗ**: `metap`, `metap-lowcode`, `metap-demo-crm`, `metap-demo-jira`, `metap-demo-waf`,
`platform-ui`, và root `metap-org` (không phải git repo, xem cuối bài).

## 1. Dev tooling — build nhẹ hơn, dev loop nhanh hơn

**Build speed** (`.cargo/config.toml` + `Cargo.toml`'s `[profile.dev]`, **giống hệt nhau** ở cả 5
repo Rust — `metap`, `metap-lowcode`, `metap-demo-crm`, `metap-demo-jira`,
`metap-demo-waf/data-plane`, bắt buộc giống nhau để không phá cache dùng chung
`.shared-target`):
- Linker `mold` (`-C link-arg=-fuse-ld=mold`) — nhanh hơn hẳn `ld` mặc định, ít RAM hơn, quan
  trọng vì máy dev chỉ ~9.7GB RAM.
- `debug = "line-tables-only"` + `split-debuginfo = "unpacked"` — giảm lượng debug info phải
  emit/copy mỗi lần link, vẫn giữ line number cho backtrace.

**Hot-reload dev compose** (mới, root `metap-org`): `Dockerfile.dev`/`Dockerfile.dev-web` +
`docker-compose.dev.base.yml` (template `rust-dev`/`web-dev` dùng chung qua `extends:`, vì YAML
anchor không cross-file được) — mỗi repo có `docker-compose.dev.yml` riêng
(`metap-demo-crm/`, `metap-demo-jira/`, `metap-demo-waf/data-plane/`) bind-mount source +
`.shared-target` vào container, chạy `cargo watch`/`vite` thay vì build lại image production mỗi
lần đổi code. `metap-demo-waf/data-plane/docker-compose.dev.yml` phủ đủ cả 5 mảnh (3 service
pillar + `graphql-gateway` + `web`), có `healthcheck` (`curl /health`, interval 30s/10 retries) để
`graphql-gateway` chờ đúng lúc 3 service kia sẵn sàng thay vì chỉ chờ container start.

Sự cố gặp khi dựng — đều đã fix, để lại làm bài học:
- `docker-compose.dev.yml` với `docker-compose.yml` (production) **cùng thư mục** →
  Compose tự suy project name theo tên thư mục → trùng nhau → tái dùng nhầm image production (sai
  ENTRYPOINT). Fix: đặt `name:` tường minh cho từng file dev.
- Đường dẫn `--watch` của `cargo watch` tính sai số cấp `../` (containers nested sâu hơn host).
- `pnpm dev -- --host` bị pnpm chèn thêm `--` làm Vite không nhận `--host` thật — đổi sang gọi
  thẳng `pnpm exec vite --host 0.0.0.0`.
- `pnpm install` trong container không có TTY, gặp `node_modules` khác host → abort thay vì hỏi
  — thêm `CI=true`.
- `web`'s dev-server proxy (`vite.config.ts`) hardcode `http://localhost:3000/3010/3020/4000` —
  không dùng `network_mode: host` (mất cách ly network) mà sửa `vite.config.ts` đọc
  `ZONES_URL`/`SCANNING_URL`/`ALERTING_URL`/`GATEWAY_URL` từ env, container dùng bridge network +
  port-map bình thường, trỏ qua service DNS name nội bộ compose.
- Container `web` chạy root ghi `node_modules` (bind-mount) → host user hết quyền sửa, phải
  `sudo chown` lại — biết trước, ghi rõ trong comment file.

## 2. Fix downstream fallout của phase 64 (cookie-session)

Không phải thay đổi thiết kế mới — chỉ là lan nốt phase 64 xuống 3 app đã sót:

- **`RequireAuth`** (local trong từng app's `App.tsx`, không phải component chung của
  `platform-ui`) vẫn `const { token } = useAuth()` — API mới không còn `token`, luôn `undefined`
  → login xong vẫn bị đá về `/login`. Sửa cả 3 app dùng `status` (`"unknown"` → chờ, không redirect
  vội; `"anonymous"` → mới redirect — tránh race bounce ngay sau khi vừa login xong).
- **`AppState.cookie_secure`** mặc định `true` (`AppState::new`) — cookie set flag `Secure`, 5
  binary dev đều chạy `http://localhost` (không HTTPS) nên browser âm thầm không lưu cookie. Thêm
  `state.cookie_secure = false;` vào cả 5 `main.rs` (crm-server, jira-server, zones/scanning/
  alerting-service) — đúng như phase 64 tự ghi chú "Dev binary cần tự set field này".
- **`apiFetch(url, token, init)`** — API mới chỉ còn `(path, init)`. Riêng `metap-demo-jira`'s 6
  trang custom (comment, worklog, watcher, attachment, transition, dashboard) vẫn gọi kiểu cũ 3
  tham số — `token` (string) bị nhét nhầm vào chỗ `init` (object), mọi action ghi dữ liệu ở các
  trang đó gửi sai request (mất `method`/`body`). Sửa cả 9 call site; `AttachmentsPanel` (multipart
  upload/download, không qua được `apiFetch`) tự thêm `credentials: "include"` + tự đọc cookie
  `metap_csrf` gắn `X-CSRF-Token` cho request ghi (upload).

`metap-demo-crm` không có breakage kiểu (3) — không có trang custom nào tự gọi `apiFetch` với
`token`.

## 3. `platform-ui`: workflow diagram + vị trí action bar (yêu cầu UX riêng)

- **`WorkflowDiagram`**: đổi route cạnh từ cubic-Bézier cong sang orthogonal (chỉ `L`, không `C`)
  — cùng waypoint cũ (vốn đã nằm đúng trục ngang/dọc, trừ nhánh "2 cột liền kề" phải đổi cách tách
  các cạnh song song từ lệch dọc sang lệch ngang trong gutter để không làm đoạn thẳng bị chéo).
  Không đổi sang Canvas — SVG vẫn đúng lựa chọn ở quy mô vài node/cạnh này (theme CSS free, tooltip
  `<title>` free, khớp tự nhiên với JSX/React reconciler, nét luôn sắc dù zoom).
- **`RecordDetail`**: `WorkflowActionBar` ("Visualize workflow" + nút đổi state) dời từ dưới cùng
  (sau bảng field + related records) lên ngay dưới header/`WorkflowStepper` — theo yêu cầu không
  muốn action bị chôn dưới 1 khúc cuộn.

## 4. Thử theme (`metap-themes`, repo mới chưa có trong `../CLAUDE.md`)

`metap-themes` — 5 package CSS-variable override (`@metap/theme-{enterprise,storefront,saas,
fintech,creative}`) trên nền `@metap/ui`, không rebuild `@metap/ui`, chỉ redefine lại đúng tên
biến (`--primary`, `--radius`, `--font-sans`...). Wire thử `@metap/theme-enterprise` vào
`metap-demo-waf/data-plane/web` (`link:../../../metap-themes/packages/enterprise`, import
`theme.css` sau `@metap/ui/style.css`, set `data-theme="enterprise"` cố định — chưa phải UI chọn
theme, chỉ là bản thử 1 theme). Đây là bản thử/demo, không phải quyết định chốt theme cho sản
phẩm.

## Xác minh

- `metap`/`metap-demo-crm`/`metap-demo-jira`: `cargo check` sạch.
- `metap-demo-crm`/`metap-demo-jira`/`metap-demo-waf/data-plane/web`: `tsc -b`/`tsc --noEmit` sạch.
- `platform-ui`: `tsc --noEmit` sạch.
- `metap-demo-waf/data-plane` dev stack: chạy thật qua `docker compose ... up --build` — cả 5
  container lên, healthcheck pass, `graphql-gateway` login + fetch schema qua service DNS name
  thành công (`entities=9`), `web` Vite dev server trả 200, module `theme.css` resolve OK.
- **Chưa test bằng browser thật** (đúng frontend verification policy) cho `metap-demo-crm`/
  `metap-demo-jira` (chỉ verify bằng `cargo check`/`tsc`, không chạy container thật) và cho luồng
  login/theme của `metap-demo-waf` (đã chạy container + curl xác nhận HTTP 200/health, nhưng chưa
  xác nhận bằng mắt qua browser) — chủ dự án tự kiểm.

## Còn lại

- `graphql-gateway` chưa có trong `docker-compose.dev.yml` của `metap-demo-crm`/`metap-demo-jira`
  (2 app đó không có gateway riêng, không áp dụng).
- 4 theme còn lại (`storefront`/`saas`/`fintech`/`creative`) chưa thử ở đâu — chỉ mới xem qua app
  demo độc lập trong `metap-themes/apps/demo`.
- Root `metap-org` **không phải git repo** — `Dockerfile.dev`, `Dockerfile.dev-web`,
  `docker-compose.dev.base.yml` (3 file dùng chung cho cả dev-compose setup) nằm ở đó, **không
  thuộc repo nào cả, không commit/push được**. Nếu muốn track, cần quyết định thuộc repo nào (có
  thể tạo git repo riêng cho `metap-org` root, hoặc copy vào 1 trong các repo hiện có) — chưa làm,
  để chủ dự án quyết.
- `metap-demo-crm`/`metap-demo-jira` **không có git remote** — mọi commit ở 2 repo này chỉ nằm
  local, không đẩy lên đâu được (khác `metap`/`metap-lowcode`/`metap-demo-waf`/`platform-ui`, đã
  push thẳng `master`/`main` theo yêu cầu phiên này).
