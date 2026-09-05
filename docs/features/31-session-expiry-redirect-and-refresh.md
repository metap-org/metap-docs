# Phiên đăng nhập hết hạn — chưa tự đăng xuất ra `/login`, chưa có cơ chế tự refresh

- **Trạng thái:** done (2026-09-05, cả 2 phần)
- **Người đề xuất:** phát hiện từ báo cáo người dùng 2026-09-05 (`metap-demo-waf` — nhưng gốc rễ nằm ở `platform-ui`/`metap` core, không riêng WAF)
- **Track sở hữu:** Frontend Platform (phần A) + Backend Core (phần B, đụng `metap-http`/`metap-peripherals`)
- **Phase roadmap liên quan:** không thuộc phase nào

## Vấn đề / động lực

Báo cáo: "login hết sessions nhưng k bay ra ngoài, và chưa có cơ chế tự refresh". Root-cause hai vấn đề riêng biệt:

**A. Không tự điều hướng về `/login` khi session hết hạn.** `RequireAuth` (`metap-demo-waf/.../App.tsx`) là một layout route mount **một lần duy nhất** (từ refactor 2026-09-04, để tránh remount `AppShellLayout` mỗi lần điều hướng) và chỉ đọc `AuthContext`'s `status` để quyết định redirect. `status` được suy ra từ query `["currentUser"]` (`GET /auth/me`), và theo default của react-query, query này chỉ tự refetch khi mount / window refocus / reconnect — **không refetch khi một API call khác giữa chừng nhận về 401**. `apiFetch` (`platform-ui/src/api/client.ts`) và `graphqlFetch` (`platform-ui/src/api/graphqlClient.ts`) khi nhận 401 chỉ `throw` (`ApiError`/`Error`), không có đường nào báo ngược cho `AuthContext`. Kết quả: cookie đã hết hạn ở backend, nhưng `status` trong React vẫn kẹt ở `"authenticated"` cho tới khi người dùng tự refresh trang hoặc đổi tab (trigger window focus) — đúng như báo cáo "không bay ra ngoài".

**B. Chưa có cơ chế tự refresh.** Xác nhận qua grep toàn bộ `metap-http/src/routes/`, `metap-peripherals/src/`, `metap-auth/src/`: không có route/hàm nào tên `refresh`/`sliding`. `auth.sessionTtlSeconds` (`metap-config/src/keys.rs:188`, mặc định 3600s) là TTL **cố định** cho cả JWT lẫn `Max-Age` cookie (`crates/metap-http/src/routes/auth.rs`'s `session_ttl_seconds`, dùng khi `mint_token` ở `login`/`oidc_callback`/`issue_token`). Không có endpoint nào gia hạn phiên — `GET /auth/token` (đã có) là thứ khác: mint một Bearer token ngắn hạn **cho riêng `graphql-gateway`**, không đụng tới cookie session, không kéo dài phiên chính. Nghĩa là một người dùng đang thao tác liên tục vẫn bị văng ra sau đúng 1 giờ (mặc định), không có cảnh báo trước, không có cách nào gia hạn.

## Phạm vi

**Trong phạm vi:**
- **Phần A** (`platform-ui`, không đụng backend): `apiFetch`/`graphqlFetch` phát một sự kiện "session invalidated" khi nhận 401 (trừ chính `/auth/login`/`/auth/me`/`/auth/logout` — 401 ở 3 route này là kỳ vọng/đã tự xử lý). `AuthContext` lắng nghe sự kiện này và ép `["currentUser"]` refetch ngay lập tức, khiến `status` chuyển "anonymous" ngay khi có 401 đầu tiên thay vì chờ window refocus — `RequireAuth` (đã có logic redirect sẵn) sẽ tự nhảy về `/login` mà không cần sửa `metap-demo-waf`.
- **Phần B** (đề xuất, chưa code): sliding session — gia hạn cookie/JWT khi người dùng còn hoạt động, để một phiên đang dùng liên tục không tự hết hạn giữa chừng. Hướng đề xuất: gọi lại `GET /auth/me` định kỳ (ví dụ mỗi 5 phút, chỉ khi tab đang active) và cho route này **re-mint + re-set cookie** với TTL mới mỗi lần gọi thành công — không cần endpoint mới, chỉ đổi hành vi `me()` ở `auth.rs`. Vẫn giữ một TTL tuyệt đối tối đa (ví dụ 24h kể từ lúc login) để một session không sống vô hạn chỉ vì tab luôn mở.

**Ngoài phạm vi:**
- Mô hình refresh-token riêng (access token ngắn hạn + refresh token dài hạn) — đổi kiến trúc lớn hơn mức cần thiết cho model cookie-JWT hiện tại.
- Đồng bộ trạng thái đăng xuất giữa nhiều tab (BroadcastChannel/storage event) — vấn đề khác, không phải cái đang báo cáo.
- Refresh cho OIDC provider riêng — provider-specific, không đụng trong brief này.

## Tiêu chí chấp nhận

- (Phần A) Xoá cookie session thủ công (giả lập hết hạn) rồi bấm bất kỳ hành động nào gọi API (REST hoặc GraphQL) trong app đang mở sẵn → app tự điều hướng về `/login` ngay, không cần refresh trang hay đổi tab.
- (Phần A) 401 từ chính `/auth/login` (sai mật khẩu) và từ `/auth/me` lúc chưa đăng nhập không gây vòng lặp gọi lại vô ích.
- (Phần B, sau khi chọn hướng) Một phiên gọi API đều đặn (khoảng cách giữa 2 lần gọi < TTL) không tự hết hạn giữa chừng; một phiên bị bỏ không quá TTL vẫn hết hạn như cũ.

## Ranh giới kiến trúc bị đụng tới

- `platform-ui`: `api/client.ts`, `api/graphqlClient.ts`, `auth/AuthContext.tsx` — thêm một module sự kiện nhỏ thuần logic (`api/sessionEvents.ts`), không thêm state UI framework-specific nào ra ngoài React, không đụng entity-specific logic.
- `metap` core (`crates/metap-http/src/routes/auth.rs`, có thể `crates/metap-http/src/auth.rs`) nếu làm phần B — đổi ngữ nghĩa "phiên": từ TTL cố định sang sliding có trần tuyệt đối. Cân nhắc chi phí: mint JWT mỗi lần gia hạn không rẻ bằng đọc cookie thuần, nhưng tần suất thấp (mỗi 5 phút/tab active) nên chấp nhận được. Có thể cần note trong ADR nếu team muốn ghi nhận đổi hướng bảo mật này chính thức.

## Rủi ro / phụ thuộc

- Phần B là thay đổi ngữ nghĩa bảo mật (session lifecycle) — cần chốt hướng (sliding qua `/auth/me` vs. endpoint `/auth/refresh` riêng) trước khi code, không tự quyết.
- Phần A không phụ thuộc backend, rủi ro thấp, làm ngay được.

## Ghi chú (phần A, done 2026-09-05)

`platform-ui`: `api/sessionEvents.ts` mới (event bus thuần, không phụ thuộc React) — `onSessionExpired`/`notifySessionExpired`. `api/client.ts`'s `apiFetch` và `api/graphqlClient.ts`'s `flush` gọi `notifySessionExpired()` khi nhận 401 (trừ `/auth/login`/`/auth/me`/`/auth/logout` ở REST, và trừ mọi call có `token` riêng ở GraphQL — xem giải thích trong code). `auth/AuthContext.tsx` subscribe sự kiện này, ép `refetchQueries(["currentUser"])` ngay khi nhận — `status` chuyển "anonymous" ngay lập tức thay vì chờ window refocus, `RequireAuth` (đã có sẵn ở `metap-demo-waf`) tự nhảy về `/login`. Không cần sửa gì ở `metap-demo-waf` hay bất kỳ app nào khác dùng `platform-ui` — fix nằm hoàn toàn ở tầng chia sẻ. Verify: `pnpm typecheck`/`pnpm lint` sạch (0 lỗi, 6 warning đều pre-existing không liên quan), dev server (`metap-demo-waf-data-plane-dev-web-1`) hot-reload các file đổi không lỗi, `curl` root trả 200. Đã commit + push, PR `platform-ui#5` + `metap-docs#5`.

## Ghi chú (phần B, done 2026-09-05)

Chốt số theo đề xuất đã thống nhất: TTL mỗi lần gia hạn = `auth.sessionTtlSeconds` hiện có của tenant (không đổi, mặc định 1h), FE poll `GET /auth/me` mỗi 20 phút khi tab active, trần tuyệt đối 24h qua key mới `auth.sessionAbsoluteMaxSeconds` (`PlatformGlobal`-tier, không cho tenant tự nới — đây là trần an toàn của platform).

`metap` core:
- `metap-config/src/keys.rs` — key mới `AUTH_SESSION_ABSOLUTE_MAX_SECONDS`, `PlatformGlobal`, default 86400, bound [3600, 2_592_000] (trùng ceiling với `AUTH_SESSION_TTL_SECONDS`, lý do giống nhau: chưa có token revocation).
- `crates/metap-http/src/cookies.rs` — cookie thứ 3, `metap_session_started_at` (`HttpOnly`, giá trị epoch giây), set 1 lần lúc login/OIDC callback, **không bao giờ ghi đè lại** — `Max-Age` của chính nó bằng đúng trần tuyệt đối tại thời điểm set, nên browser tự rụng cookie này khi hết trần, không cần server tự tính toán "đã bắt đầu từ bao lâu" ở đâu khác. `clear_session_cookies` giờ trả về 3 cookie (thêm cookie này) để logout xoá sạch.
- `crates/metap-http/src/routes/auth.rs` — `login`/`oidc_callback` set thêm cookie này. `me()` giờ có thêm bước `try_refresh_session`: nếu request có đủ 3 cookie (`session`/`csrf`/`started_at`) và chưa vượt trần tuyệt đối, mint token mới với TTL đầy đủ, set lại session+csrf cookie (tái dùng đúng giá trị CSRF cũ, không rotate). Không có 3 cookie này (Bearer/Basic/CLI caller) hoặc đã vượt trần → không làm gì, `me()` vẫn trả identity như cũ.
- `attach_cookies` đổi từ nhận đúng 1 cặp cookie sang nhận `impl IntoIterator` để dùng chung cho cả trường hợp 2 và 3 cookie.

`platform-ui`:
- `auth/AuthContext.tsx` — `useQuery` của `["currentUser"]` thêm `refetchInterval: 20 * 60 * 1000` (`refetchIntervalInBackground` mặc định `false`, tự dừng khi tab ẩn).

Verify: `cargo build`/`cargo clippy --all-targets -- -D warnings`/`cargo test` cho `metap-http`+`metap-config` sạch (18+12 unit test pass, e2e cần Postgres/RabbitMQ nên không chạy trong lượt này — chưa verify sống qua e2e/browser thật). `platform-ui`'s `tsc`/`oxlint` sạch. Container `zones-service` (đang chạy live qua `cargo watch`, path-depend vào `../metap`) tự rebuild lại theo đúng code mới, không panic, tiếp tục phục vụ `/auth/login`/`/auth/me` bình thường (log thật quan sát được, không phải giả lập). Chưa verify sống bằng cách thật sự để 1 session hết hạn rồi chờ tự gia hạn/tự văng ra — để người dùng tự kiểm tra qua trình duyệt.
