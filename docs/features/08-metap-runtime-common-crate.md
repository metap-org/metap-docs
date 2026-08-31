# Crate dùng chung `metap-runtime` — plumbing xuyên cắt, "cách repo này viết code"

- **Trạng thái:** done — crate đã tạo, 11 module có test, đã di trú toàn bộ call site cũ tìm được
  qua 5 vòng rà soát (4 vòng cùng ngày 2026-08-31, vòng 5 cùng ngày kế tiếp): vòng 1 sau khi chủ dự
  án xác nhận "tách vào contrib", vòng 2 sau câu hỏi "ngoài http_error thì còn gì common có thể
  vứt vào contrib k", vòng 3 chủ động sau phản hồi "rà tiếp đi... đừng bắt tao phải nhắc từng
  thứ" — tìm `openapi::insert`, vòng 4 sau khi chủ dự án nêu rõ mục tiêu "có runtime để viết
  backend tùy biến, core metap cũng phụ thuộc vào runtime đó" — thêm
  `rate_limit`/`trace`/`request_id`/`request_context`/`serve` + crate mới `metap-app`, vòng 5 sau
  câu hỏi "http, grpc hỗ trợ các header của istio/linkerd chưa" — thêm `trace_context` (W3C Trace
  Context, xem "Rà soát vòng 5" dưới, chi tiết đầy đủ ở
  `docs/roadmap/57-istio-linkerd-trace-context.md`).
  **Đổi tên `metap-contrib` → `metap-runtime` cùng ngày** (chủ dự án chọn, vai trò thật là SDK để
  viết backend/router tuỳ biến, không chỉ chỗ gom code trùng lặp) — đổi thư mục, package name,
  ~50 chỗ tham chiếu trong code/docs, alias facade `metap::contrib` → `metap::runtime`.
- **Người đề xuất:** chủ dự án, 2026-08-31 (cùng lúc yêu cầu tách `metap-lowcode` thành mono-repo
  microservices — xem ghi chú "Liên quan" cuối file)
- **Track sở hữu:** Backend Core
- **Phase roadmap liên quan:** Phase 53 (`docs/roadmap/53-metap-runtime-and-lowcode-microservices-plan.md`),
  Phase 56 (`docs/roadmap/56-metap-runtime-sdk-and-metap-app.md`), Phase 57
  (`docs/roadmap/57-istio-linkerd-trace-context.md`)

## Vấn đề / động lực

`metap` (repo core) và `../metap-lowcode` (repo SaaS low-code, path dependency vào `metap`) đều tự
viết tay cùng vài loại boilerplate xuyên cắt (HTTP client, bearer-token parsing, đọc env var) mỗi
nơi một kiểu, không có chỗ chung nào để tham chiếu "đây là cách repo này làm việc đó" — rủi ro thật
đã xảy ra: 2 chỗ dựng `reqwest::Client` không set timeout (xem "Rà soát" dưới), một upstream treo có
thể làm hang vô hạn.

## Rà soát (bằng chứng thật, đọc code trước khi viết crate — không đoán)

**4 ứng viên xác nhận có duplicate thật, đã đưa vào `metap-runtime` (`crates/metap-runtime/src/`)
và di trú toàn bộ call site tìm được:**

| Module | Vấn đề tìm thấy | Bằng chứng (file:line) | Call site đã di trú |
|---|---|---|---|
| `http_client` | `reqwest::Client::new()` không timeout — hang vô hạn nếu upstream treo | `graphql-gateway/src/schema_builder.rs:49`, `metap-jwks/src/client.rs:30` (không timeout) vs `cron-scheduler/src/main.rs:37` (tự set 30s, không chỗ nào tái dùng) | `graphql-gateway` (2 chỗ), `metap-jwks`, `cron-scheduler` |
| `bearer` | Cùng logic `strip_prefix("Bearer ")` viết tay 3 lần, 3 error type khác nhau | `metap-http/src/auth.rs:93-95` (axum), `graphql-gateway/src/server.rs:54-56` (axum thủ công), `metap-grpc/src/auth.rs:65-67` (tonic) | `metap-http`, `graphql-gateway`, `metap-grpc` |
| `env::env_or`/`require_env` | Idiom `env::var(X).ok().and_then(...).unwrap_or(default)` lặp | `metap-infra/src/config.rs` (~29 lần), `graphql-gateway/src/config.rs` (6 lần, độc lập, không tái dùng `AppConfig`) | `metap-infra` (host/ttl/port SMTP + `DATABASE_URL`/`RABBITMQ_URL` required), `graphql-gateway` (host/port), `cron-scheduler` (tick/batch) |
| `env::optional` (thêm lúc di trú — không có trong rà soát ban đầu) | Idiom `env::var(X).ok().filter(\|s\| !s.is_empty())` lặp — phát hiện khi thực sự đọc code để migrate `metap-infra/src/config.rs` mới thấy đây là idiom **riêng**, khác `env_or`, và lặp rộng hơn dự kiến | `metap-infra/src/config.rs` (17 lần), `dev-tools/src/main.rs` (1 lần), `cron-scheduler/src/main.rs` (1 lần), `graphql-gateway/src/config.rs` (1 lần) — 23 lần tổng, 4 file | `metap-infra` (17 field optional), `cron-scheduler`, `dev-tools` |

**Phát hiện thêm khi di trú `dev-tools/src/main.rs`** (không có trong rà soát ban đầu — file
chưa được quét kỹ lúc đó): 9 lần lặp y hệt `std::env::var(X).map_err(\|_\| anyhow::anyhow!("{X} is
required"))?` (`DATABASE_URL` × 5 subcommand, `VAULT_ADDR`/`VAULT_TOKEN`/`AWS_SECRETS_REGION`/
`AWS_SECRETS_ACCESS_KEY`/`AWS_SECRETS_SECRET_KEY`/`GCP_SECRETS_PROJECT_ID` × 1 mỗi cái) — di trú
hết sang `env::require_env`. Bài học: rà soát ban đầu chỉ quét vài crate trọng tâm (HTTP/gRPC
transport), bỏ sót `dev-tools` (CLI, không phải service) — lúc thực sự migrate, đọc lại toàn bộ
call site mới lộ ra.

**Rà soát vòng 2 (cùng ngày, sau câu hỏi "ngoài http_error thì còn gì common có thể vứt vào contrib
k") — 3 ứng viên thêm, cũng đã di trú xong:**

| Module | Vấn đề tìm thấy | Bằng chứng (file:line) | Call site đã di trú |
|---|---|---|---|
| `http_error` | `service_error_response`/`internal_error_response` (shape `{"error":{"code":...}}`) đã tập trung tốt ở `metap-http::error` — nhưng `graphql-gateway` (dù đã depend `metap-http`) tự viết `unauthorized()` riêng, trả body plain-text thay vì đúng shape | `graphql-gateway/src/server.rs:41-43` (bản cũ) vs `metap-http/src/error.rs` | Chuyển 2 hàm sang `metap-runtime::http_error` (nguồn thật), `metap-http::error` giờ chỉ re-export; `graphql-gateway` gọi thẳng `metap-runtime` |
| `shutdown::signal` | Y hệt 17 dòng (`ctrl_c` + SIGTERM) ở 3 binary; `graphql-gateway` tự viết bản thiếu SIGTERM — gap thật, không chỉ trùng lặp | `cron-scheduler/src/main.rs:79-99`, `notification-worker/src/main.rs:27-47`, `outbox-publisher/src/main.rs:36-56` (y hệt) vs `graphql-gateway/src/server.rs:138-140` (chỉ ctrl_c) | 4 binary: `cron-scheduler`, `notification-worker`, `outbox-publisher`, `graphql-gateway` |
| `cors::build` | Logic "origins rỗng→`CorsLayer::new()`, khác thì parse + `allow_credentials(true)`" gần như y hệt, chỉ methods/headers khác | `metap-http/src/lib.rs:57-72` vs `graphql-gateway/src/server.rs:100-109` | `metap-http`, `graphql-gateway` (methods/headers vẫn tham số riêng — 2 nơi cho phép khác nhau thật) |

Phát hiện thêm ngoài lề lúc migrate `outbox-publisher`: 2 chỗ `env::var(X).ok().and_then(...).unwrap_or(default)` (`OUTBOX_POLL_MS`/`OUTBOX_BATCH_SIZE`) cũng bị 2 vòng rà soát trước bỏ sót — di trú luôn sang `env::env_or`, thêm `outbox-publisher`/`notification-worker` vào danh sách crate phụ thuộc `metap-runtime`.

**Rà soát vòng 3 (chủ động, sau phản hồi "rà tiếp đi... đừng bắt tao phải nhắc từng thứ" — không
chờ user chỉ từng thứ nữa) — 1 ứng viên thêm, đã di trú:**

| Module | Vấn đề tìm thấy | Bằng chứng (file:line) | Call site đã di trú |
|---|---|---|---|
| `openapi::insert` | Hàm 3 dòng `paths.insert(path.to_string(), item)` y hệt — path fragment thật mỗi crate build khác nhau, chỉ helper insert giống | `metap-http/src/openapi_paths.rs`, `metap-lowcode-http/src/openapi_paths.rs`, `metap-control-http/src/openapi_paths.rs` | Cả 3 — `metap-lowcode-http`/`metap-control-http` (repo `metap-lowcode`) lần đầu thêm `metap-runtime` làm dependency |

**Rà soát vòng 4 (chủ dự án nêu rõ mục tiêu: "có runtime để viết backend tùy biến, core metap
cũng phụ thuộc vào runtime đó") — 2 loại thay đổi, cả 2 có bằng chứng thật:**

| Module | Vấn đề tìm thấy | Bằng chứng (file:line) | Call site đã di trú |
|---|---|---|---|
| `rate_limit::build`/`trace::build` | Y hệt inline trong `metap-http::build_router`; `graphql-gateway` không có rate-limit/tracing-span nào cả — gap tính năng thật, không chỉ trùng lặp | `metap-http/src/lib.rs` (bản gốc) | `metap-http` di trú, `graphql-gateway` thêm mới (đóng gap) |
| `request_id`/`request_context` | Move nguyên vẹn khỏi `metap-http` — không caller ngoài nào tham chiếu tên cũ (verify bằng grep) | `metap-http/src/request_id.rs`, `metap-http/src/request_context.rs` (đã xoá) | `metap-http`, `graphql-gateway` |
| `serve::run` | `TcpListener::bind` + `axum::serve(...).with_graceful_shutdown(...)` + log "listening" — lặp ở mọi `main.rs` serve HTTP | `metap-demo-crm`, `metap-demo-jira`, `templates/metap-app`, `graphql-gateway` (4 file, không phải 1) | Cả 4 |

**Crate mới `metap-app` (không phải module `metap-runtime`)** — `bootstrap_platform(&AppConfig) ->
PlatformParts` (Postgres pool, `Router`, `PermissionService`, JWT keypair), lặp thật ở 3
`main.rs` (`metap-demo-crm`, `metap-demo-jira`, `templates/metap-app` — verify bằng grep thật, không
đoán). **Không thể đưa vào `metap-runtime`**: cần `metap-control`/`metap-permission`/
`metap-cache`/`metap-infra`, mà 4 crate đó đã depend `metap-runtime` — nhét vào sẽ tạo vòng lặp
dependency thật. Đi qua `EnterPlanMode`/`ExitPlanMode` trước khi code (chủ dự án tự phát hiện rủi
ro vòng lặp trước khi tôi kịp báo) — chốt: `metap-app` là crate riêng, cùng tầng facade `metap`
(depend hết core crate, không ai depend ngược — verify `cargo tree -p metap-infra | grep
metap-app` rỗng). Crate-level doc comment của nó liệt kê 5 primitive có sẵn (router tuỳ biến qua
`extra_routes`, `Router::pool_for`, `AuthContext`/`AdminContext`, `outbox::enqueue`,
`EventBus::subscribe`) cho việc viết backend tuỳ biến — không viết code trừu tượng mới, chỉ chỉ
đường, trỏ `metap-lowcode-http` làm ví dụ thật.

**Xác nhận thêm phát hiện lúc verify `templates/metap-app`**: build trực tiếp trong thư mục đó
không chạy được (Cargo.toml dùng git-URL placeholder `{{metap_git}}` cho `cargo generate`, không
phải path dependency) — verify bằng cách copy sang scratchpad, thay placeholder bằng path
dependency cục bộ trỏ `../metap/crates/metap`, rồi build/clippy/test thật. Phát hiện thêm 1 bug
có sẵn không liên quan (không phải do lượt này gây ra): `tests/http_server.rs`'s `EntityField`
literal thiếu field `storage` (thêm khi `EntityField` có field đó, test không được cập nhật theo)
— fix luôn vì chặn verify.

**Rà soát vòng 5 (chủ dự án hỏi "http, grpc hỗ trợ các header của istio/linkerd chưa", rồi xác
nhận mục tiêu "muốn tích hợp được Istio trong tương lai, vì metap-lowcode sau này sẽ rất lớn") —
1 module mới, chi tiết đầy đủ ở `docs/roadmap/57-istio-linkerd-trace-context.md`:**

| Module | Vấn đề tìm thấy | Bằng chứng (file:line) | Call site đã di trú/thêm |
|---|---|---|---|
| `trace_context` | Không đọc/forward chuẩn tracing nào của service mesh (B3, W3C `traceparent`); `x-request-id` tự sinh mới thay vì pass-through — phá log correlation Envoy/Istio ingress cần | `request_id.rs` (bản cũ, không đọc `x-request-id` đến), `metap-grpc/src/client.rs`'s `signed_request` (chỉ gắn `authorization`, không gắn gì cho tracing) | `request_id::generate_request_ids` (parse `traceparent` + sửa `x-request-id` pass-through), `trace::build` (thêm field span `trace_id`/`span_id`), `http_client::attach_trace_context` (helper opt-in, chưa wire caller nào — cả 3 caller hiện tại chạy boot-time/background), `metap-grpc::client::attach_traceparent` (tự động, điểm nghẽn duy nhất `signed_request`) |

Chọn W3C Trace Context (`traceparent`), không phải B3 — chuẩn IETF, cả Istio và Linkerd đều hỗ
trợ, khớp tự nhiên với field `trace_id`/`span_id` `tracing::Span` đã có, không cần thêm SDK
OpenTelemetry (sidecar mesh tự tạo/export span khi header propagate đúng). Ambient qua
`tokio::task_local!`, cộng thêm chứ không thay `x-trace-id`/`RequestIds` cũ (khác biệt và lý do
giữ song song 2 cơ chế: xem `docs/roadmap/57-istio-linkerd-trace-context.md`).

**Đã kiểm tra, KHÔNG đưa vào — đã tập trung tốt sẵn hoặc không đủ giá trị để trừu tượng hoá:**
- gRPC client + service-JWT attach — chỉ 1 định nghĩa (`metap-grpc::client::GrpcBackend`),
  `graphql-gateway` tái dùng đúng.
- JWT decode — tập trung ở `metap_peripherals::decode_access_token`, dùng chung bởi
  `metap-http`/`metap-grpc`/`graphql-gateway`.
- `AuthContext`/`AdminContext` — định nghĩa 1 lần ở `metap-http/src/auth.rs`, mọi route ở
  `metap-lowcode-http`/`metap-control-http` import lại.
- List/pagination boilerplate ngoài `QueryPlanner` — không route nào tự parse query param tay.
- `scoped_tenant(&context)` gọi ~30 chỗ — không phải logic lặp, chỉ gọi 1 method chung
  (`PermissionService::scoped_tenant`) đã có sẵn 1 nơi; mỗi call site là business logic khác nhau.
- `resolve_pool` (scoped_tenant + pool_for + map lỗi, ở `metap-lowcode-http`) — chỉ 1 call site
  thật; `metap-control-http` đúng khi dùng `state.pool` thẳng (data của nó không tenant-scoped).
- `security_headers`/`TraceLayer`/rate-limit (Governor) — `security_headers` đã tập trung tốt
  (`metap-http`/`metap-graphql-http`/`graphql-gateway` cùng import 1 hàm); `TraceLayer`/Governor
  chỉ tồn tại ở `metap-http`, không có bản thứ 2 để tách — `graphql-gateway`/`metap-lowcode-http`
  thiếu rate-limit/tracing-span là **gap tính năng**, không phải trùng lặp, khác phạm vi.
- Response envelope thành công `Json(json!({"data": X})).into_response()` — lặp ~50 lần nhưng
  không có logic gì (khác `service_error_response` có default-message/field-errors/logging), và
  hình dạng không đồng nhất thật (`{"data": X}` trơn / `(CREATED, ...)` / `{"data", "page"}` phân
  trang) — 1 helper chung không phủ hết, lợi ích quá nhỏ (~10 ký tự/chỗ) so với việc phải làm 2-3
  biến thể. Không tách.
- Backoff schedule (`metap-infra::event_bus::resilient::backoff_delay` vs
  `metap-cron::store::run_lifecycle::backoff_delay`) — cùng công thức "base × 2^exponent, có cap"
  nhưng tham số hoá khác nhau thật (1 cái lịch cố định không cấu hình được cho reconnect
  EventBus, 1 cái nhận `retry_backoff_seconds` cấu hình được cho retry cron job) và kiểu trả về
  khác (`std::time::Duration` vs `chrono::Duration`) — mỗi hàm chỉ 3-4 dòng, gộp lại sẽ cần
  signature đủ tổng quát để làm code phức tạp hơn là gọn lại. Không tách.
- Request-id injection (`metap-http/src/request_id.rs`) — chỉ 1 crate dùng, không có bản copy nào
  ở nơi khác nên chưa đủ bằng chứng để tách.
- **`metap-runtime` không thể gộp vào chính facade `metap`** — facade đã depend
  `metap-infra`/`metap-http`/... (để re-export), mà các crate đó giờ depend `metap-runtime`; gộp
  vào facade sẽ tạo vòng lặp dependency thật, Cargo từ chối compile. Giữ nguyên là crate riêng,
  nằm dưới `metap-infra`/`metap-http`/... trong dependency graph.

## Phạm vi

**Trong phạm vi (đã làm, 4 vòng rà soát cùng ngày 2026-08-31 + 1 vòng cùng ngày kế tiếp):**
- `crates/metap-runtime` giờ 11 module: `http_client::{build, default_client, attach_trace_context}`,
  `bearer::parse_bearer`, `env::{env_or, require_env, optional}`, `http_error::{service_error_response,
  internal_error_response}`, `shutdown::signal`, `cors::build`, `openapi::insert`,
  `rate_limit::build`, `trace::build`, `request_id`/`request_context` (2 module đầy đủ, không
  phải hàm lẻ), `serve::run`, `trace_context` (W3C Trace Context, vòng 5) — 28 unit test,
  `cargo clippy -D warnings` sạch.
- Framework-agnostic có chủ đích (`bearer::parse_bearer` không biết axum/tonic là gì) để cả
  `metap-http` (axum) lẫn `metap-grpc` (tonic) đều gọi được, tự map `None` sang error type riêng.
- Di trú toàn bộ call site tìm được sang dùng chung — **12 crate/binary** giờ phụ thuộc
  `metap-runtime`: 10 ở repo `metap` (`metap-infra`, `metap-http`, `metap-grpc`, `metap-jwks`,
  `graphql-gateway`, `cron-scheduler`, `dev-tools`, `outbox-publisher`, `notification-worker`,
  `metap-app` — crate mới, xem dưới) + 2 ở repo `../metap-lowcode` (`metap-lowcode-http`,
  `metap-control-http`). Facade `metap` re-export `metap_runtime as runtime` (cùng mẫu
  `metap::infra`).
- **Crate mới `crates/metap-app`** — `bootstrap_platform(&AppConfig) -> PlatformParts`, cùng tầng
  facade `metap` (không phải module của `metap-runtime` — xem lý do vòng lặp dependency ở "Rà
  soát vòng 4" dưới). Facade re-export `metap_app as app`; `bootstrap_platform`/`PlatformParts`
  cũng có trong `metap::prelude`. Migrate 3 call site thật: `metap-demo-crm`, `metap-demo-jira`,
  `templates/metap-app`'s `main.rs`.
- **1 thay đổi hành vi runtime thật, có chủ đích**: `metap-jwks::JwksClient`/
  `graphql-gateway`'s schema-fetch client giờ có timeout 30s (trước đó không có, đúng bug survey
  tìm ra) — request tới 1 upstream/JWKS server treo giờ fail sau 30s thay vì treo vô hạn.
- **1 thay đổi hành vi nhỏ, chấp nhận được**: `bearer::parse_bearer` trim khoảng trắng thừa quanh
  token trước khi decode (`metap-http`/`metap-grpc`/`graphql-gateway`'s `authenticate`) — bản cũ
  không trim, 1 token có khoảng trắng thừa trước đây bị từ chối decode, giờ decode được nếu nội
  dung token hợp lệ. Không đổi độ an toàn (cùng token, chỉ khoan dung whitespace hơn).

**Ngoài phạm vi (chưa làm, follow-up riêng — cần chốt trước khi làm):**
- `GrpcBackend`'s channel+interceptor construction thành helper dùng chung — chưa tách vì hiện chỉ
  1 caller thật (`graphql-gateway`); tách khi có caller thứ 2.
- 1 cơ chế truyền `tenant_id`/service-auth qua network boundary cho `services/lowcode-admin-api`/
  `services/control-api` (Phase 53's T3/T4, `metap-lowcode`) — chưa cần vì T3/T4 chưa code.
- `port`'s filter đặc biệt (`p > 0`, `metap-infra/src/config.rs`), `auth_jwt_public_key_path`/
  `auth_jwt_private_key_path`'s "set nhưng rỗng vẫn là lỗi" — **cố ý không di trú**, vì `env_or`/
  `require_env` không tái tạo đúng 2 hành vi khác biệt này; di trú sẽ đổi hành vi validate.

## Tiêu chí chấp nhận

- `cargo build -p metap-runtime` và `cargo test -p metap-runtime` sạch — đạt (28/28 test pass).
- `cargo clippy -p metap-runtime -- -D warnings` sạch — đạt.
- `cargo build --workspace`/`cargo test --workspace` (89 `test result: ok`, 0 fail, 224 test pass)
  /`cargo clippy --workspace --all-targets -- -D warnings`/`cargo fmt --all --check` ở cả `metap`
  và `metap-lowcode` sau khi di trú 12 crate/binary qua 4 vòng + thêm `trace_context` ở vòng 5 —
  đạt, sạch toàn bộ.
- `cargo test -p metap-grpc` (5 test, gồm 2 test mới cho `attach_traceparent`) — đạt.
- `cargo tree -p metap-infra | grep metap-app` rỗng — xác nhận `metap-app` không tạo vòng lặp
  dependency — đạt.
- `../metap-demo-crm`, `../metap-demo-jira` build lại sạch sau migrate sang `bootstrap_platform` —
  đạt. `templates/metap-app` verify qua bản copy scratchpad (Cargo.toml gốc dùng git-URL
  placeholder `{{metap_git}}` cho `cargo generate`, không build trực tiếp được) — build/clippy/test
  sạch, phát hiện + fix 1 bug có sẵn không liên quan (`EntityField` thiếu field `storage` trong
  `tests/http_server.rs`).

## Ranh giới kiến trúc bị đụng tới

Không đụng layering hiện có (`docs/architectures/05-building-blocks.md`) — đây là một crate
plumbing ngang hàng `metap-infra`, không phải một layer mới. Không cần ADR riêng cho việc thêm
crate này (không phải quyết định kiến trúc gây tranh cãi) — nhưng phần "microservices cho
`metap-lowcode`" đi kèm yêu cầu ban đầu **có** đụng một ADR đã ghi (`09-adr.md` dòng 55, "Không
tách microservice cho hướng SaaS multi-tenant") — xem `docs/roadmap/53-metap-runtime-and-lowcode-microservices-plan.md` để biết phần đó đang ở trạng thái nào (chờ chủ dự án quyết định hướng, chưa
làm).

## Rủi ro / phụ thuộc

- Đã migrate (xem "Tiêu chí chấp nhận") — 2 thay đổi hành vi runtime thật đã liệt kê rõ ở "Phạm
  vi", cả 2 đều là cải thiện (timeout mặc định, khoan dung whitespace), không phải regression.
- Hướng "mono-repo microservices" cho `metap-lowcode` đã được chọn (xem Phase 53,
  `../metap-lowcode/docs/architecture.md`) — khi T3/T4 (2 binary service mới) thật sự viết
  `main.rs`, đó sẽ là caller thứ 2/3 cho một helper "dựng `AppState` tối thiểu" — lúc đó tách vào
  `metap-runtime` là đúng lúc (chưa làm bây giờ, chỉ 1 caller thật hiện có).
