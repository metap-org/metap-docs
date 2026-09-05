# Phiên đăng nhập hết hạn — chưa tự đăng xuất ra `/login`, chưa có cơ chế tự refresh

- **Trạng thái:** done (phần A, 2026-09-05) / proposed (phần B, chờ chọn hướng)
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

`platform-ui`: `api/sessionEvents.ts` mới (event bus thuần, không phụ thuộc React) — `onSessionExpired`/`notifySessionExpired`. `api/client.ts`'s `apiFetch` và `api/graphqlClient.ts`'s `flush` gọi `notifySessionExpired()` khi nhận 401 (trừ `/auth/login`/`/auth/me`/`/auth/logout` ở REST, và trừ mọi call có `token` riêng ở GraphQL — xem giải thích trong code). `auth/AuthContext.tsx` subscribe sự kiện này, ép `refetchQueries(["currentUser"])` ngay khi nhận — `status` chuyển "anonymous" ngay lập tức thay vì chờ window refocus, `RequireAuth` (đã có sẵn ở `metap-demo-waf`) tự nhảy về `/login`. Không cần sửa gì ở `metap-demo-waf` hay bất kỳ app nào khác dùng `platform-ui` — fix nằm hoàn toàn ở tầng chia sẻ. Verify: `pnpm typecheck`/`pnpm lint` sạch (0 lỗi, 6 warning đều pre-existing không liên quan), dev server (`metap-demo-waf-data-plane-dev-web-1`) hot-reload các file đổi không lỗi, `curl` root trả 200. Chưa commit/push — chờ xác nhận. Phần B (tự refresh) vẫn `proposed`, chờ chọn hướng trước khi code (đụng ngữ nghĩa bảo mật ở `metap` core).
