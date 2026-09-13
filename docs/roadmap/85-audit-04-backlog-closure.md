## Phase 85: Đóng backlog audit 04 — gateway hot-swap thật (B1), credential rotation (B7), cookie cross-origin (`metap-lowcode`), và 8 finding còn lại (2026-09-13)

Trigger: sau khi fix 3 bug `metap-reconciler` sống trên `metap-demo-waf` (Phase 84), chủ dự án hỏi
tiếp "kiến trúc `metap` còn lỗ hổng gì không, audit tiếp tuần này", và chỉ ra `metap-docs` chưa
được cập nhật đúng convention. Rà lại thấy audit
[`04-auth-protocols-gateway-audit.md`](../audits/04-auth-protocols-gateway-audit.md) (2026-09-03)
còn 11/17 finding chưa đóng, đứng đầu là B1 (HIGH) — gateway boot-time-only, fail-closed toàn
phần, mâu thuẫn trực tiếp với lời hứa "hot-swap metadata, không restart" của nền tảng. Chủ dự án
ủy quyền quyết định thẳng cả 4 item cần ADR thay vì hoãn tiếp ("Làm hết, kể cả 4 item cần ADR").

### B1 — gateway TTL-cached, fault-tolerant, không còn fail-closed

**Trước đây**: `crates/metap-graphql-gateway`'s `connect_upstreams` khám phá schema mọi upstream
đúng 1 lần lúc boot, `?`-chain xuyên suốt — 1 upstream chậm/hỏng làm hỏng boot của TOÀN BỘ gateway,
và một entity low-code mới publish không bao giờ xuất hiện ở gateway tới khi có người restart tay.

**Bây giờ** (`crates/metap-graphql-gateway/src/schema_builder.rs`, viết lại gần như toàn bộ):
- `UpstreamCache` — một cái mỗi upstream, refresh TTL-gated (`moka::future::Cache`, cùng pattern
  `metap_control::RegistryCache`), giữ `last_good: ArcSwapOption<UpstreamSchema>` không bao giờ bị
  xoá bởi một lần refresh thất bại, cộng `status: ArcSwap<UpstreamStatus>` ghi lại reachability
  mới nhất.
- `build_composite` — dựng lại registry hợp nhất từ bất cứ thứ gì mỗi `UpstreamCache` đang giữ.
  **Quy tắc rớt field**: một field `Reference` chỉ bị rớt khi `ref_entity` thuộc về một upstream
  CHƯA TỪNG thành công lần nào — một upstream đã từng thấy rồi mới rớt xuống vẫn giữ nguyên hình
  dạng schema cũ (một query live vào nó chỉ nhận lỗi backend bình thường, như trước fix này).
  2 upstream tranh cùng tên entity trong 1 chu kỳ: cái thứ 2 bị `MetadataRegistry::register` từ
  chối, log lại và bỏ qua cho chu kỳ này, không làm hỏng cả lần rebuild.
- `GatewaySchemaCache` — bọc `build_composite` trong TTL cache riêng (30s), infallible — kể cả khi
  `.finish()` tự nó lỗi (bug thật trong cách map type của chính crate này) vẫn trả về 1 schema rỗng
  hợp lệ thay vì crash.
- Độ suy giảm hiện ra 2 chỗ: `GET /health` (`{"status":"ok"|"degraded","checks":{"upstreams":[...]}}`,
  không auth, không lộ error text) và GraphQL `Query._gatewayHealth` (đằng sau auth Bearer của
  chính gateway, full chi tiết — `{upstreams, droppedFields}`, trả qua `Json` scalar, tái dùng
  đúng cơ chế `aggregate` field đã có sẵn, không cần tự dựng type GraphQL lồng nhau).
- Refresh theo kiểu lazy per-request qua `moka`, không có background task — đúng tiền lệ
  `RegistryCache`.

**Breaking change đi kèm, đã fix luôn**: `build_with_extensions`'s `extend` closure đổi từ
`FnOnce` sang `Fn` (giờ gọi lại mỗi chu kỳ refresh, không chỉ 1 lần lúc boot) — chạm sang
`../metap-demo-waf`'s `data-plane/graphql-gateway/src/main.rs`, sửa tại chỗ gọi (`zones`/
`scanning`/`alerting` giờ `.clone()` mỗi lần thay vì move 1 lần).

**Verify**: 2 e2e test mới/sửa trong `tests/gateway_e2e_postgres.rs`
(`one_dead_upstream_does_not_block_the_others_entities` — 1 upstream chưa từng sống, upstream còn
lại vẫn phục vụ đủ, `_gatewayHealth`/health struct đúng; test cũ cập nhật theo API mới). **Verify
sống trên `../../metap-demo-waf`'s docker stack thật đang chạy** (không phải mock): `docker stop
scanning-service` → `GET /health` trả `"status":"degraded"`, `scanning.reachable:false`, gateway
KHÔNG restart (0 container restart) → `docker start scanning-service` → trong vòng 1 TTL (30s),
`GET /health` tự quay lại `"status":"ok"` không cần restart gateway.

### B7 — credential rotation cho gateway qua `SecretStore`

Đi kèm B1 vì cùng sửa `connect_one_upstream`. `UpstreamConfig` thêm
`service_password_secret_ref: Option<String>` (opt-in, env `UPSTREAM_<N>_SERVICE_PASSWORD_SECRET_REF`
hoặc key YAML `servicePasswordSecretRef`) — khi có, mật khẩu thật resolve tươi từ
`metap_control::SecretStore` (Vault/AWS/GCP/env, cùng `build_secret_store` mọi binary khác đã
dùng) mỗi chu kỳ refresh, thay vì đọc literal `service_password` 1 lần lúc boot. Xoay credential ở
backend có hiệu lực trong vòng 1 TTL, không redeploy. `service_password` literal vẫn là default,
không migrate bắt buộc — mỗi upstream cần ít nhất 1 trong 2.

**Refactor phụ**: `metap_control::build_secret_store` trước đây nhận `&metap_infra::AppConfig`
nguyên khối (đòi `DATABASE_URL`/`RABBITMQ_URL` — 2 thứ gateway này không có). Tách ra
`SecretStoreConfig` (10 field liên quan) + `impl From<&AppConfig>` cho 3 call site cũ
(`metap-app`, `metap-cron-scheduler`, `metap-dev-tools`) + `SecretStoreConfig::from_env()` mới cho
gateway. `metap-control` gained a `metap-runtime` dependency cho `env::optional`.

**Verify**: 3 unit test mới (`schema_builder::tests`) qua `resolve_service_password` (đã tách
thành hàm thuần, test không cần login/HTTP thật) — literal path, secret-ref path (qua
`EnvStore`, chứng minh secret-ref thắng khi cả 2 đều set), và lỗi rõ ràng khi thiếu cả 2. 2 unit
test mới ở `config.rs` cho validation. Verify sống: `../../metap-demo-waf`'s stack (dùng literal
password) rebuild + boot lại bình thường sau khi refactor, không regression.

### Cookie cross-origin fix (`../../metap-lowcode/services/control-plane-graphql`)

Không phải finding của audit 04 (audit đó chỉ soi core `metap`) nhưng cùng lớp bug và cùng đợt
làm việc này — `impersonateTenant`/`exitImpersonation` forward `Set-Cookie` của `control-api` qua
gateway verbatim; `metap_http::cookies::session_cookies` cố ý build cookie host-only (không có
`Domain`), nên forward verbatim làm cookie bind vào origin của GATEWAY, không phải `control-api`.
Local dev không sao (RFC 6265 không scope theo port, `localhost` 2 port khác nhau đã share cookie
sẵn) nhưng deployment thật với 2 hostname khác nhau sẽ hỏng.

**Fix**: `Config.cookie_domain: Option<String>` mới (env `COOKIE_DOMAIN`, mặc định unset = giữ
nguyên hành vi cũ 100%). `rest::rewrite_cookie_domain` (dùng `axum_extra::extract::cookie::Cookie`)
áp dụng trong `post_forwarding_set_cookie` — điểm duy nhất cả 2 resolver đều đi qua, nên
`schema.rs` không cần đổi gì. Parse lỗi → forward nguyên văn (log warn), không bao giờ âm thầm mất
cookie.

**Verify**: 3 unit test thật (`rest::tests`) — thêm `Domain` khi chưa có, thay `Domain` cũ, forward
nguyên văn khi input không parse được (`cookie::ParseError::MissingPair`). `cargo check`/`clippy
--workspace -- -D warnings` sạch trên toàn bộ `metap-lowcode` (phát hiện thêm 1 call site
`build_secret_store` chưa cập nhật ở `services/reconciler-orchestrator` khi refactor `SecretStoreConfig`
cho B7 — sửa cùng lúc). **Chưa verify sống bằng `impersonateTenant` thật qua HTTP** — không có
stack `metap-lowcode` nào đang chạy lúc này để dựng lại toàn bộ (control-api + control-plane-graphql
+ Postgres + tenant thật); đường code khi `COOKIE_DOMAIN` unset (mặc định) là no-op xác nhận được
qua đọc code (`match` rẽ nhánh `None` trả `raw.to_string()` y hệt hành vi trước fix) — verify sống
với `COOKIE_DOMAIN` set thật nên làm lần tới khi có stack chạy.

### 8 finding còn lại đóng cùng đợt (chi tiết trong
[`../audits/00-index.md`](../audits/00-index.md)'s 2 bảng)

- **Fix thật**: A#9 (tên test CORS sai ngược), A#5 (log khi `pick_token` fallback im lặng), A#10
  (`optional_serve` giờ nhận `TokenVerifier::Jwks` qua `token_verifier_override` — xoá luôn đoạn
  code 3 service WAF tự chế để bypass hạn chế cũ), A#3-rate-limit (`metap-grpc::rate_limit`, layer
  riêng cho tonic — `tower_governor` không dùng được thẳng vì hardcode `axum::body::Body`), A#8
  (`jti: Uuid` thêm vào JWT claims, decode side `Option` để token cũ vẫn chạy).
- **Xác nhận cố ý qua ADR** (ghi doc, không đổi code): A#2 (`users_email_unique` toàn cục — login
  không có tenant picker), A#3-TLS (`tls_config: None` mặc định — mesh lo mTLS, không phải thiếu
  sót), B3 (gRPC/GraphQL chỉ records, không phải full REST surface — phạm vi BFF cố ý), B6
  (`assigneeId` khác kiểu REST vs GraphQL — mỗi giao thức đúng idiom riêng).
- A#6: tradeoff đã cân nhắc từ audit gốc, xác nhận lại vẫn đúng, không cần đổi gì.

### Verify tổng thể

`cargo check`/`cargo clippy --workspace --all-targets -- -D warnings` sạch trên cả `metap` lẫn
`../../metap-demo-waf/data-plane` sau mỗi bước. `cargo test --workspace` (unit) sạch.
`cargo test -p metap-graphql-gateway -- --ignored` (e2e, cần `DATABASE_URL`) sạch — 2/2 pass.
Verify sống nhiều lần trên docker stack thật của `metap-demo-waf` đang chạy song song với phiên
làm việc này (không phải dựng riêng cho audit) — `cargo watch` tự rebuild theo từng lần sửa, xác
nhận không có commit nào làm gateway thật ngừng boot.
