## Phase 64: Session sống qua reload — cookie `HttpOnly` thay JWT-trong-React-state (2026-09-03)

Trigger: chủ dự án hỏi thẳng "jwt/auth session đang k lưu ở đâu trên browser reload page là mất,
có cách nào xử k". Đây là đảo ngược một quyết định đã ghi rõ trong `CLAUDE.md`
("The token lives only in memory (React state) and is lost on refresh — that's deliberate, not a
bug") — không phải sửa bug, mà chủ dự án chủ động đổi quyết định đó. Trước khi code, hỏi lại chọn
giữa `sessionStorage`/`localStorage`/`HttpOnly` cookie + CSRF; chốt cookie + CSRF (bảo mật tốt nhất
với XSS, đổi lại việc lớn nhất cả 2 phía).

Đụng **2 repo**: `metap` (backend) và `platform-ui` (frontend). Không đụng `design-system`.

## Backend (`metap`)

Thêm `axum-extra` (feature `cookie`) vào `metap-http` — không tự viết parser cookie tay, đây là
code liên quan bảo mật.

**`crates/metap-http/src/cookies.rs`** (mới) — 2 cookie set cùng lúc mỗi lần login:
- `metap_session` — chính JWT, `HttpOnly` (JS không đọc được — đây là điểm chính của việc bỏ giữ
  token trong React state: một XSS payload không còn lấy được token nữa).
- `metap_csrf` — giá trị random (UUID v4), **không** `HttpOnly`. Cùng với việc `AuthContext` bắt
  buộc header `X-CSRF-Token` khớp giá trị cookie này trên mọi request cookie-authenticated có
  mutate (không phải GET/HEAD/OPTIONS) — kiểu double-submit cookie chuẩn: một request giả mạo
  cross-site có thể khiến browser tự đính `metap_session`, nhưng không đọc được `metap_csrf` để
  đặt vào header.

**`crates/metap-http/src/auth.rs`**'s `AuthContext` — thêm nhánh fallback: nếu request **không có**
header `Authorization` (dấu hiệu của request từ browser dùng `fetch(..., {credentials: "include"})`
thay vì tự set header), đọc cookie session, chạy check CSRF nếu mutating, decode JWT y hệt luồng
Bearer. Path Bearer/Basic cũ **hoàn toàn không đổi** — vẫn ưu tiên nếu có header, dùng cho CLI/
service-to-service/`dev-tools mint-token` output.

**`routes/auth.rs`**:
- `login`/`oidc_callback` giờ set 2 cookie trên response (kèm `Set-Cookie`), JSON body vẫn giữ
  nguyên `{data: {token}}` cho caller không phải browser.
- `oidc_callback`: **bỏ hẳn `#token=...` URL fragment**, thay bằng set cookie trực tiếp trên chính
  response redirect rồi điều hướng thẳng. Đây là cải tiến so với thiết kế fragment cũ (audit 02's
  B3 fix, mới ngày hôm trước) — cookie qua `Set-Cookie` header không bao giờ chạm vào URL/history
  của browser, kể cả tạm thời, trong khi fragment vẫn phải "hiện ra rồi mới xoá". `OidcCallbackPage`
  viết lại lần 2 trong 2 ngày liên tiếp — lần này không đọc gì từ URL nữa, chỉ phản ứng theo
  `useAuth().status`.
- `POST /auth/logout` (route mới) — xoá cả 2 cookie (`Max-Age=0`), không yêu cầu `AuthContext`
  (logout một session đã hết hạn/đã logout rồi là no-op hợp lý, không cần 401 trước).
- `GET /auth/token` (route mới) — xem mục "graphql-gateway" bên dưới.

**`AppState`** thêm field `pub cookie_secure: bool` (default `true` trong `AppState::new`, không
đổi signature — mọi call site hiện có, kể cả ở `metap-demo-crm`/`metap-demo-jira` ngoài phạm vi
session này, không bị breaking change). Dev binary chạy `http://localhost` cần `Secure=false` thì
tự set `state.cookie_secure = false;` (mọi field đều `pub`, cùng convention `object_store`).

**Phát hiện giữa chừng — `graphql-gateway` không đi qua cookie được**: `crates/graphql-gateway` là
service riêng, tự keypair, tự router, "decode-only" theo đúng mô tả trong `CLAUDE.md` — chỉ nhận
`Authorization: Bearer`, không có khái niệm cookie nào cả. Không mở rộng gateway sang cookie (thêm
CORS/cookie cross-origin cho 1 service riêng là phạm vi lớn hơn nhiều so với yêu cầu). Giải pháp:
thêm `GET /auth/token` — route mới, dùng `AuthContext` (chạy được dù caller vào bằng cookie hay
Bearer), mint 1 JWT mới ngắn hạn theo đúng identity của caller. `platform-ui`'s `useGraphQLQuery`
gọi route này **ngay trước mỗi lần** gọi gateway, không giữ token lại — đúng tinh thần của cả đợt
đổi này (không giữ credential sống lâu phía client).

## Frontend (`platform-ui`)

**`AuthContext.tsx`** viết lại hoàn toàn: bỏ `{token, setToken}`, thay bằng
`{status, markAuthenticated, logout}`. `status` (`"unknown"|"authenticated"|"anonymous"`) suy ra từ
1 `useQuery` gọi `GET /auth/me` — **chạy không điều kiện**, không gate theo chính `status` nó tạo ra
(nếu không sẽ deadlock: query xác định status lại chờ status). `useCurrentUser()` (đã có sẵn, dùng ở
`AppShellLayout` lấy role/email) giờ đọc **cùng query key** `["currentUser"]` — react-query dedupe
tự động nên không tốn thêm request nào dù nhiều nơi cùng gọi.

**`api/client.ts`**'s `apiFetch` bỏ tham số `token`, tự thêm `credentials: "include"` (để browser
gửi/lưu cookie kể cả khi FE và API khác origin thật, không chỉ dev-proxy) và tự đính header
`X-CSRF-Token` cho mọi request mutating (đọc cookie `metap_csrf` qua `document.cookie`) — không
call site nào phải tự nhớ làm việc này nữa. `useApiQuery`/`useApiMutation`/`useApiInfiniteQuery`
đổi gate `enabled` từ `token !== null` sang `status === "authenticated"`.

**12 call site** trực tiếp dùng `useAuth().token` được dọn: `RecordDetail`, `GeneratedList`,
`WorkflowActionBar` (bỏ tham số `token` khỏi `apiFetch` tay), `adminApi.ts` (6 hàm), `LocaleProvider`
(đổi gate `if (!token)` → `if (status !== "authenticated")`), `LoginForm` (`setToken` →
`markAuthenticated`, gọi sau khi backend đã set cookie xong), `AppShellLayout` (`setToken(null)` →
`await logout()`, giờ là async nên gọi qua `void handleLogout()`), `OidcCallbackPage` (viết lại,
xem trên).

**`graphqlClient.ts`/`useGraphQLQuery.ts`** — khác biệt so với phần còn lại: **giữ nguyên** tham số
`token` (không bỏ), vì đây là client gọi `graphql-gateway`, không phải API chính. `useGraphQLQuery`
tự gọi `GET /auth/token` lấy token tươi ngay trước mỗi query — component gọi nó
(`RelatedRecordsPanel`) không đổi gì ở call site.

## Phát hiện ngoài lề, đã xử lý

- **Disk quota hết giữa chừng** — `cargo test --workspace` lỗi linker (`Bus error`) do writable
  disk allowance của session đã hết (không phải máy hỏng, `df` báo sai theo đúng cảnh báo môi
  trường). `cargo clean` giải phóng 32.5GB (`.shared-target`, dùng chung với 4 repo khác theo đúng
  quy ước `CLAUDE.md`) rồi build/test lại sạch.

## Xác minh

`metap`: `cargo build/clippy --all-targets/test --workspace` sạch cả 3 (e2e `-- --ignored` chưa
chạy, cần Postgres/RabbitMQ). `platform-ui`: `typecheck`/`lint`/`format:check` sạch, lint về đúng
baseline 6 warning có sẵn từ trước, 0 warning ở file đụng tới.

**Chưa test bằng browser thật** (đúng frontend verification policy) — đặc biệt luồng OIDC (không
dựng được IdP thật trong session), và hành vi CSRF/cookie thật qua trình duyệt (cross-tab logout,
cookie `Secure` trên `http://localhost`, v.v.) nên chủ dự án tự kiểm trước khi merge.

## Còn lại

- Regenerate `platform-ui/src/metadata/generated-types.ts` không liên quan phase này (đó là nợ từ
  audit 02's C1, vẫn đang chờ 1 demo app chạy được).
- Chưa thêm cơ chế thu hồi session sớm (revocation list) — JWT hết hạn tự nhiên sau
  `TOKEN_TTL_SECONDS` (1h) như cũ, `POST /auth/logout` chỉ xoá cookie ở phía trình duyệt gọi nó,
  không vô hiệu hoá token đã phát hành nếu nó đã bị sao chép ra nơi khác (giới hạn vốn có của JWT
  stateless, không phải thứ đợt này gây ra hay có ý định giải quyết).
