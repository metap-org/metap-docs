## Phase 88: OAuth2 cho core — login provider + Authorization Server thật (2026-09-17)

Trigger: chủ dự án yêu cầu trực tiếp "bổ sung giúp t oauth2 cho core" ngay sau khi merge audit 06.
Câu hỏi làm rõ trước khi code (`metap-auth` đã có `AuthProviderKind::Oidc` — OAuth2 login qua OIDC
rồi): chủ dự án chọn **cả hai chiều còn thiếu**, không phải một — (1) `AuthProviderKind::OAuth2`,
đăng nhập tenant qua provider chỉ có access_token, không có id_token/discovery; (2) metap tự làm
**OAuth2 Authorization Server**, cấp token cho ứng dụng bên thứ ba thay mặt user. Cả hai đều nằm
trong `metap` core (không phải app cụ thể) — đúng nghĩa "cho core" chủ dự án nói.

### Chiều 1 — `AuthProviderKind::OAuth2` (login provider)

Thêm biến thể thứ 4 song song `Local`/`Basic`/`Oidc` trong `metap-auth`, module riêng
(`oauth2_login.rs`) — khác OIDC đúng một điểm: không có discovery document, không có `id_token`
ký sẵn để verify, danh tính lấy qua việc gọi **userinfo endpoint** đã cấu hình bằng access_token
vừa đổi được (GitHub's OAuth App là ví dụ kinh điển của kiểu provider này). `OAuth2LoginConfig`
thêm `userinfo_url`, `subject_field`/`email_field` (default `"id"`/`"email"`, cấu hình được vì mỗi
provider đặt tên field khác nhau) so với `OidcConfig`.

Tổng quát hoá `find_oidc_user`/`jit_provision_oidc_user` thành `find_external_user`/
`jit_provision_external_user` nhận thêm `provider: &str` — hai hàm cũ giờ là wrapper mỏng gọi hàm
mới với `"oidc"`, không caller nào đổi. Không cần đổi schema: `users.auth_provider` vốn đã là
cột free-text (`0020_users_oidc_columns.sql`), `"oauth2"` chỉ là một giá trị mới hợp lệ; unique
index `users_tenant_external_subject_idx` đã khoá theo `(tenant_id, auth_provider,
external_subject)` nên cùng một `external_subject` dưới 2 provider khác nhau không bao giờ đụng
nhau — có test riêng xác nhận việc này.

`crates/metap-http/src/routes/auth.rs` thêm `GET /auth/oauth2/{tenant_id}/login` +
`GET /auth/oauth2/{tenant_id}/callback`, cấu trúc bám sát `oidc_login`/`oidc_callback` gần như
nguyên văn (cùng cách JIT-provision, cùng cách mint session/cookie). Tái dùng thẳng
`state.oidc_flow_cache` (không dựng cache thứ hai) — key CSRF token luôn ngẫu nhiên mật mã học nên
2 flow chia sẻ 1 cache an toàn; field `nonce` của `OidcFlowEntry` chỉ để rỗng cho flow này (không
có id_token để chống replay bằng nonce).

### Chiều 2 — metap là Authorization Server thật (`metap-oauth-server`)

Crate mới, thuần thư viện (không HTTP, giống `metap-cron`/`metap-dashboards`) — `client.rs`
(đăng ký/CRUD client, hash secret), `code.rs` (authorization code, TTL 60s, single-use nguyên tử),
`refresh.rs` (refresh token, xoay vòng + phát hiện reuse), `pkce.rs` (S256), `token.rs` (sinh/hash
token đối xứng dùng chung cho cả 3 loại credential trên). `crates/metap-http/src/routes/oauth2.rs`
là mặt HTTP: `GET /oauth/authorize`, `POST /oauth/token`, `POST /oauth/revoke`,
`GET /.well-known/oauth-authorization-server`, cộng `POST|GET /admin/oauth/clients` +
`DELETE /admin/oauth/clients/{id}` (đăng ký/thu hồi client, `AdminContext`).

**Phạm vi cố ý cắt gọn cho v1, ghi rõ chứ không lặng lẽ bỏ**:
- Chỉ 2 grant type: `authorization_code` (+ PKCE bắt buộc cho public client) và `refresh_token`.
  `client_credentials` client gọi tới bị từ chối đúng mã lỗi RFC 6749 (`unsupported_grant_type`)
  thay vì giả vờ hỗ trợ — grant này cần một danh tính "service user" để mint token thay mặt, chưa
  provision trong phase này.
- Không có màn hình consent riêng: một session đã đăng nhập gọi `/oauth/authorize` coi như đồng ý
  luôn — đúng mức tin cậy của một registry chỉ admin của chính tenant đó tạo được client, nhưng là
  gap thật với OAuth2 "chuẩn". Việc frontend chưa được yêu cầu trong phiên này.
- Mọi lỗi ở `/oauth/authorize`, kể cả lỗi phát sinh **sau khi** `client_id`/`redirect_uri` đã xác
  minh hợp lệ, trả thẳng response lỗi thay vì redirect kèm `?error=...` như RFC 6749 khuyến nghị
  cho nhóm lỗi đó — đơn giản hoá có chủ đích, ghi lại trong doc comment của route.
- Không có job dọn dẹp code/refresh-token hết hạn — khác `metadata.audit_trail_entries` (không
  prune là yêu cầu compliance), đây thuần là gap chưa làm, để lại cho phase sau.

**Access token là JWT nền tảng bình thường**, mint qua đúng trust root `mint_jwt`/`metap-jwks` mọi
session khác dùng — để `AuthContext` verify được ngay, không cần đường decode mới. Thêm 2 claim
tuỳ chọn `scope`/`clientId` (additive, `Option<String>`, token cũ không có field này vẫn decode
bình thường — đã verify thực nghiệm hành vi `#[derive(Deserialize)]` của serde với `Option<T>`
vắng mặt trước khi tin vào nó). Không đổi chữ ký `mint_jwt`/`mint_with_signer` sẵn có (nhiều caller
downstream) — thêm hàm chị em `mint_oauth_access_token`/`mint_scoped_service_or_user_jwt` mint
claim mở rộng, `mint_jwt` gọi lại hàm chung với `None, None`. `AuthContext` (`metap-http`) gộp
`scope`/`clientId` từ claim vào `RequestContext.context_attributes` đã có sẵn (field vốn dựng cho
đúng việc này — `AUTH_CONTEXT_ENTITY`) thay vì thêm field mới vào `RequestContext` — struct đó có
48 điểm khởi tạo literal xuyên workspace + repo downstream, đúng loại cascade audit 06's
`EntityAuditConfig` từng nhắc tới.

**Cố ý không tự động enforce `scope` vào RBAC/ABAC** — phần đó vẫn quyết định hoàn toàn bởi role
của user thật đứng sau token (giống hệt một session login thường), `scope` chỉ *hiện diện* để một
policy ABAC có thể đọc qua `fromContext.oauthScope` nếu tenant muốn siết thêm. Đây là quyết định
sản phẩm cần chủ dự án chốt (route nào cần enforce cứng theo scope, ánh xạ scope→action ra sao) —
ghi lại như câu hỏi mở, không tự quyết.

**Revoke chỉ thật cho refresh token**, không cho access token đã phát — đúng giới hạn `mint_jwt`'s
doc comment đã nêu từ trước ("không có hạ tầng kiểm tra thu hồi"). `POST /oauth/revoke` (RFC 7009)
luôn trả 200 kể cả khi token không khớp gì (đúng spec — không được dùng làm oracle dò token hợp
lệ). Refresh token xoay vòng mỗi lần dùng (`replaced_by`, chuỗi liên kết xuôi thời gian); trình lại
một token **đã bị thay** bị coi là dấu hiệu bị đánh cắp — thu hồi toàn bộ chuỗi sống còn lại của
cặp (client, user) đó ngay lập tức.

**`oauth_clients`/`oauth_authorization_codes`/`oauth_refresh_tokens` cố ý không đưa vào
`metap_control::tenant_schema::TENANT_SCOPED_TABLES`** dù cả 3 đều có cột `tenant_id` — cùng lý do
audit 06 finding #2 loại `users`/`user_roles`: `POST /oauth/token` xác định tenant **từ chính dòng
client_id/code/refresh-token tìm được**, không có tenant picker nào ở endpoint này để tách theo
schema — y hệt lý do khiến clone `users` phá `POST /auth/login`.

### Sự cố thật gặp lúc verify sống, không phải lý thuyết

1. **`CREATE TABLE` không schema-qualify landing sai schema.** Giả định ban đầu (viết trong migration)
   là default `search_path` của database (`public, metadata, control`, từ `0028`) sẽ đưa bảng mới
   vào `metadata` — sai: `search_path` liệt kê `public` **trước**, nên `CREATE TABLE oauth_clients`
   không qualify tạo thẳng vào `public`, ngược hẳn với mọi bảng platform khác từ Phase 82 tới giờ.
   Phát hiện bằng cách chạy migration thật rồi `\dt` trực tiếp, không suy luận từ đọc code — đúng
   như migration `0032_audit_trail_entries.sql` đã làm đúng ngay từ đầu (`CREATE TABLE
   metadata.audit_trail_entries`, schema-qualify tường minh). Sửa: qualify cả 3 `CREATE
   TABLE`/`REFERENCES`/`CREATE INDEX` thẳng vào `metadata.*`.
2. **`axum::response::Redirect::to` trả 303, không phải 302** — assumption ban đầu trong test/doc
   comment (copy theo cảm giác chung "redirect = 302"). Grep thẳng vendored source
   (`axum-0.8.x/src/response/redirect.rs`) xác nhận `Redirect::to` dùng `StatusCode::SEE_OTHER`
   (303); `Redirect::temporary`/`permanent` mới là 307/308. `oidc_login`/`oidc_callback` sẵn có
   trong repo mang cùng annotation `status = 302` sai y hệt — **không sửa** 2 chỗ đó (ngoài phạm vi
   phiên này, hành vi thật của chúng không đổi, chỉ mô tả utoipa sai) nhưng route mới
   (`oauth2_login`/`oauth2_callback`) sửa đúng thành 303 ngay từ đầu.

### Verify

`cargo build --workspace` + `clippy --workspace --all-targets -- -D warnings` + `fmt --all --check`
sạch. `cargo test --workspace` (unit): không fail nào phát sinh. E2e sống trên Postgres 16 native
(không Docker trong môi trường), DB reset sạch trước mỗi lượt chạy:
- `metap-oauth-server` (7 test, thuần thư viện): code single-use, `redirect_uri` sai bị từ chối mà
  không đốt code hợp lệ, refresh token xoay vòng + reuse-detection thu hồi cả chuỗi, revoke qua
  client sai bị chặn.
- `metap-http` (3 test, qua HTTP thật — `Form` extractor, HTTP Basic client auth, `AdminContext`):
  toàn bộ chu trình authorize → token → refresh → replay-bị-chặn qua server thật; public client
  thiếu PKCE bị từ chối; client secret sai bị từ chối.
- `metap-auth` (6 test, wiremock cho IdP giả + 2 test chạm Postgres thật): round-trip đăng nhập
  qua userinfo (cả `id` dạng số lẫn dạng chuỗi), field cấu hình sai bị báo lỗi thay vì âm thầm
  dùng giá trị rác, JIT-provisioning không tạo trùng, cùng `external_subject` dưới 2 provider khác
  nhau không đụng nhau.
- 2 fail môi trường có sẵn từ trước (`vault_store.rs`, `tenant_secret_postgres.rs`, cần Vault qua
  Docker — bị chặn ở môi trường này) xác nhận không liên quan bằng `git stash` chạy lại trên
  `master`, kết quả giống hệt.

### Còn nợ, ghi rõ để không quên

- `client_credentials` grant — cần thiết kế danh tính "service user" gắn với client trước khi làm.
- Màn hình consent thật (frontend) — hiện tại mọi caller đã đăng nhập coi như đồng ý.
- Enforce `scope` vào RBAC/ABAC một cách hệ thống — hiện chỉ *hiện diện* qua `context_attributes`,
  chưa có route nào tự động gate theo nó; cần chủ dự án chốt hướng.
- Job dọn code/refresh-token hết hạn (2 bảng này không phải loại "never prune" như audit trail).
- `GET /.well-known/oauth-authorization-server` trả path tương đối, không phải absolute URI như
  RFC 8414 yêu cầu nghiêm ngặt — nền tảng chưa có config base-URL công khai nào để dùng.

`metap-demo-waf`/`metap-demo-jira`/`metap-demo-crm` chưa app nào bật 2 tính năng này — cả hai đều
opt-in (route group `oauth2` mới trong `RouteGroups`, cần migration `0034` áp trước; login provider
cần 1 dòng `tenant_auth_configs` mới), không app nào tự động đổi hành vi.
