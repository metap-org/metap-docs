## Phase 74: JWKS (Ed25519) — trust root nhiều service, thay 1 file RSA keypair chia sẻ (2026-09-04)

Trigger: câu hỏi thẳng "sửa jwk có giúp bỏ được cái auth/token kia không / có cách nào tốt hơn
hiện tại" khi đang review `metap-demo-waf`. Rà lại phát hiện `metap-demo-waf`'s 3 service
(`zones-service`/`scanning-service`/`alerting-service`) + `waf-graphql-gateway` đang mint/verify
session bằng cách **copy 1 file private key RSA** (`dev-jwt-private.pem`) vào cả 4 container
(`docker-compose.dev.yml`'s `- ./keys:/app/keys:ro`) — không có đường rotate an toàn (đổi key
phải cutover cứng đồng thời cả 4 nơi), và container nào rò rỉ volume đó cũng lộ signing key cho
toàn sản phẩm. `metap-jwks`/`metap-jwks-http` (crate có sẵn từ Phase 49, JWKS trust root Ed25519,
chưa binary nào trong `metap-org` dùng thật) đúng là câu trả lời — doc comment gốc của crate còn
nêu ví dụ minh hoạ chính là "a WAAP deployment's many sub-modules".

## Đã làm

- **`TokenVerifier`/`TokenSigner`** (dispatch Static-RS256 vs JWKS-EdDSA, cùng `AccessClaims`) di
  dời từ `metap-grpc::auth` sang `metap-jwks` (crate mới `metap_jwks::{verifier, signer}`) — không
  đặt ở `metap-runtime` vì `metap-jwks` đã phụ thuộc `metap-runtime`, đặt ngược lại sẽ tạo cycle.
  `metap-grpc::auth::authenticate` giờ gọi lại `decode_with_verifier` thay vì tự match — hành vi
  không đổi, test cũ vẫn xanh.
- **`metap-http::AppState`** thêm 2 field mới `token_verifier`/`token_signer`
  (`Option<Arc<metap_jwks::{TokenVerifier,TokenSigner}>>`, mặc định `None`) + method
  `verify_token`/`mint_token` — dispatch JWKS khi có, fallback RSA tĩnh khi không, y hệt hành vi
  cũ. **Additive thuần túy**: không đổi `AppState::new`'s signature, `metap-demo-crm`/
  `metap-demo-jira`/`templates/metap-app` build sạch không cần sửa gì (verify trực tiếp).
- **`crates/graphql-gateway`** (`metap` core): `GatewayConfig` thêm `jwks_url`/`cookie_auth_enabled`
  (env `JWKS_URL`/`COOKIE_AUTH_ENABLED`, cả 2 mặc định tắt) — `JWKS_URL` và
  `AUTH_JWT_PUBLIC_KEY_PATH` loại trừ lẫn nhau.
- **Cookie-auth opt-in cho `graphql-gateway`** (cùng phiên, cùng động lực giảm round-trip
  `/auth/token` trước mỗi lần gọi GraphQL): `/graphql` chấp nhận session cookie trực tiếp (double-
  submit CSRF y hệt REST, luôn bắt buộc vì `POST` không phân biệt được query/mutation) khi
  `COOKIE_AUTH_ENABLED=true`, Bearer vẫn ưu tiên trước khi có. Tách hằng số
  `SESSION_COOKIE_NAME`/`CSRF_COOKIE_NAME`/`CSRF_HEADER_NAME` + `requires_csrf_check`/
  `csrf_matches` từ `metap-http::cookies` sang `metap_runtime::cookie_auth` (2 nơi dùng chung —
  đúng ngưỡng gom vào `metap-runtime` đã áp dụng cho mọi thứ khác trong crate đó). `@metap/
  platform-ui`'s `graphqlFetch`/`useGraphQLQuery` bỏ hẳn bước `GET /auth/token` khi `token` không
  truyền vào — verify sống: query qua cookie không kèm Bearer thành công, thiếu CSRF header bị
  reject đúng ("missing or invalid csrf token").
- **`dev-tools gen-jwks-key [dir] [kid]`** — sinh Ed25519 keypair (`JwksKeyPair::generate`), ghi
  `dev-jwks-private.der` (raw PKCS8, không PEM — theo đúng cách `metap-jwks` làm việc) +
  `dev-jwks-kid.txt`. `gen-keys` (RSA) vẫn còn nguyên, chạy song song làm fallback.
- **`metap-demo-waf`**: cả 3 service cùng giữ 1 EdDSA key (khớp mô hình share-1-key hiện tại — **cố
  ý chưa chuyển sang 1-issuer-duy-nhất**, xem "Chưa làm" dưới), `zones-service` publish
  `/.well-known/jwks.json` (mount qua `fallback_service`, không phải `route_service`/
  `nest_service("/")` — axum 0.8 refuse cả 2 cho 1 `Router` nguyên khối, tìm ra lúc chạy live).
  `waf-graphql-gateway` verify qua `JwksClient` trỏ về endpoint đó — kế thừa `JWKS_URL`/
  `COOKIE_AUTH_ENABLED` từ `metap`'s `graphql-gateway` core vì nó bọc đúng module `config`/`server`
  đó (`schema_builder::build_with_extensions`, xem Phase 73), không cần sửa gì thêm ở binary WAF
  riêng.

## Xác minh

Build/test toàn bộ crate liên quan (`metap-runtime`/`metap-jwks`/`metap-jwks-http`/`metap-http`/
`metap-grpc`/`graphql-gateway`) sạch, kể cả e2e với Postgres/RabbitMQ thật — chỉ 3 fail ở
`platform_config_postgres.rs` xác nhận **có sẵn trước phiên này** (test lại y hệt khi `git stash`
code mới, do DB dev dùng chung bị nhiễm state qua nhiều lần chạy trong ngày, không liên quan JWKS).
Verify sống qua container thật: login → JWT header đổi `alg: RS256` → `alg: EdDSA, kid: <uuid>` →
`/auth/me`/REST cả 3 service/`graphql-gateway` (forward + gRPC) đều accept token mới.

## Chưa làm / phạm vi cố ý bỏ qua

**Chưa đi hết đường "giảm blast radius" JWKS hứa hẹn** — quyết định phạm vi chốt trước khi code
(hỏi lại người dùng, chọn phương án ít rủi ro hơn): cả 3 service WAF vẫn cùng giữ 1 private key,
không chuyển sang mô hình 1-issuer-duy-nhất (chỉ 1 service giữ key, còn lại chỉ verify qua HTTP).
Được lợi: rotation an toàn 3 bước (`JwksKeyStore::add_key`/`promote`/`remove_key`, chưa test rotation
thật qua container sống, chỉ có unit test của `metap-jwks` tự nó). Không được lợi: số tiến trình
giữ private key vẫn là 3, y hệt mô hình RSA cũ — nếu cần siết blast radius thật sự (đúng tinh thần
gốc JWKS được thiết kế cho), bước tiếp theo là đổi topology mint, kèm sửa
`graphql-gateway::schema_builder::connect_upstreams` (hiện mỗi upstream tự login riêng, cần gộp về
1 issuer) — chưa có trigger.
