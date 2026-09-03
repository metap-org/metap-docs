# Audit 04 — Auth, giao thức giao tiếp, và gateway (core `metap`)

> **Phạm vi (2026-09-03):** logic auth (cookie session/CSRF/Bearer/Basic/OIDC/JWT), 4 giao thức
> giao tiếp (REST, gRPC, GraphQL, RabbitMQ/outbox), và `crates/graphql-gateway`. Chỉ core
> `metap` — `metap-lowcode` **cố tình để lại** cho lần review sau (yêu cầu của chủ dự án).
>
> **Khác 2 audit trước**: [`02`](02-full-codebase-audit.md) đọc từng dòng toàn bộ `crates/`
> (2026-08-26), [`03`](03-metap-core-architecture-audit.md) soi kiến trúc/layering/doc-drift
> (2026-09-02, 14/14 finding đã fix). Audit này đi theo **đường đi của một request** qua từng
> giao thức, và đặc biệt soi phần **cookie session + CSRF mới ship 2026-09-03** (Phase 64) —
> ship sau audit 03 đúng 1 ngày nên chưa từng được review.
>
> Phương pháp: đọc code thật (không suy từ doc), không build (tránh phình `../.shared-target`
> theo quy tắc 40GB của CLAUDE.md), không sửa code — chỉ report. Mọi finding đều có file:line
> cụ thể để đối chiếu.

> **Cấu trúc:** [Phần A](#phần-a--bảo-mật) là bảo mật (10 finding), [Phần B](#phần-b--kiến-trúc)
> là kiến trúc (7 finding) — cùng phạm vi, khác góc nhìn. Phần B không lặp lại Phần A: nó soi
> tính nhất quán giữa 4 giao thức, độ hoàn chỉnh của gateway, và những cơ chế "có code nhưng chưa
> nối vào đâu".

# Phần A — Bảo mật

## Tóm tắt

| # | Mức độ | Vùng | Vấn đề |
|---|---|---|---|
| 1 | **HIGH** | Giao thức / cron | SSRF **có phản hồi** qua `webhook` target: tenant admin đọc được nội bộ mạng + cloud metadata |
| 2 | MEDIUM | Auth / multi-tenant | `users_email_unique` unique **toàn cục trên `email`**, không phải `(tenant_id, email)` |
| 3 | MEDIUM | Giao thức / gRPC | Cổng gRPC không có rate limit, mặc định plaintext — bất đối xứng với REST |
| 4 | MEDIUM | Auth / CSRF | `GET /auth/token` phát credential nhưng miễn CSRF (vì là GET), chỉ còn CORS đỡ |
| 5 | MEDIUM | Gateway / identity | `forwarded_bearer_token` fallback **im lặng** sang service account |
| 6 | LOW | Auth / CSRF | `POST /auth/logout` không kiểm CSRF → logout-CSRF |
| 7 | LOW | Gateway | `SchemaLimits::default()` hardcode ở gateway, không chỉnh được qua env |
| 8 | LOW | Auth / JWT | Không có `jti`/revocation; `aud`/`iss` là hằng số chung toàn mesh |
| 9 | LOW | Giao thức / CORS | Tên test `empty_origins_uses_permissive_default` sai với hành vi thật (restrictive) |
| 10 | LOW | Auth / JWKS | `metap-jwks` vẫn là code chết — `optional_serve` không có đường dùng `TokenVerifier::Jwks` |

---

## 1. HIGH — SSRF có phản hồi qua cron `webhook` target

**Vị trí:** `crates/cron-scheduler/src/executor/webhook.rs:25-53`, tạo job qua
`crates/metap-http/src/routes/cron.rs` (`POST /admin/cron-jobs`, gate bằng `AdminContext`).

`run_webhook` nhận `url`/`method`/`headers`/`body` thẳng từ `cron_jobs.target_config` — một giá
trị **tenant admin tự ghi** qua API — rồi gọi đi mà không kiểm gì cả:

```rust
let mut req = http.request(method, &cfg.url).json(&body);
for (key, value) in &cfg.headers {
    req = req.header(key, value);
}
```

Không có allowlist scheme/host, không chặn dải private/link-local, không chặn redirect
(`metap_runtime::http_client::build` chỉ set `timeout`, giữ nguyên default của `reqwest` là
follow tối đa 10 redirect — nên có allowlist ở URL đầu vào cũng vẫn bypass được).

Điều làm mức độ nặng lên: **nội dung phản hồi được trả về cho người tạo job**:

```rust
Ok(json!({ "status": status.as_u16(), "body": truncate(&text, 2000) }))
```

Giá trị này lưu vào `cron_job_runs` và đọc lại được qua `GET /admin/cron-jobs/{id}/runs`. Đây là
**read SSRF**, không phải blind SSRF — khác hẳn về mức độ khai thác.

**Tác động thực tế:** trong mô hình SaaS multi-tenant, tenant admin là *khách hàng*, không phải
operator. Với finding này, một khách hàng có thể:
- đọc cloud metadata (`http://169.254.169.254/...` → IAM credential trên AWS/GCP),
- quét và đọc service nội bộ trong VPC (kể cả admin API của service khác, RabbitMQ management,
  Prometheus, `metap-control-http`),
- tự đặt header `Authorization` tuỳ ý khi gọi các đích nội bộ đó.

Tức là vượt ranh giới tenant — đúng thứ mà toàn bộ `Router`/`PermissionService` được xây để chặn.

**Điểm sáng đã đúng sẵn:** `workflow_transition`/`bulk_query_action` **không** dính lỗi này —
chúng dùng `config.target_base_url` (env, operator kiểm soát,
`crates/cron-scheduler/src/executor/workflow_transition.rs:45,88,105`), nên service token của
`ServiceTokenSource` không rò rỉ qua đường tenant-controlled. Chỉ `webhook` là hở.

**Đề xuất** (không tự làm, cần chốt trước): allowlist theo host/domain cấu hình ở cấp operator
(env), resolve DNS rồi chặn IP private/loopback/link-local **sau khi resolve** (chống DNS
rebinding), `redirect::Policy::none()` cho riêng client webhook, chặn header `Authorization`/
`Cookie` do tenant đặt, và cân nhắc không trả `body` về `cron_job_runs` (chỉ giữ status code).

---

## 2. MEDIUM — `users_email_unique` unique toàn cục, không theo tenant

**Vị trí:** `crates/migrations/0009_users.sql:10`

```sql
CREATE UNIQUE INDEX "users_email_unique" ON "users" USING btree ("email");
```

Unique trên **một mình `email`**, không phải `(tenant_id, email)` — dù bảng có `tenant_id` và có
index riêng cho nó ngay dòng dưới. Hệ quả:

- **Một email không thể là user của 2 tenant.** Với một platform mà multi-tenancy là lõi (có hẳn
  `control.tenants`, `Router`, `TenantStrategy`), đây là giới hạn sản phẩm thật, không phải chi
  tiết kỹ thuật. Va ngay cả với chính thiết kế của repo: platform admin là user trong
  `PLATFORM_TENANT_ID`; cùng một người muốn vừa là platform admin vừa là user của một tenant thật
  thì phải dùng 2 email khác nhau.
- **OIDC JIT provisioning fail 500** khi email đã tồn tại: `jit_provision_oidc_user`
  (`crates/metap-auth/src/lib.rs:124-144`) INSERT thẳng, không bắt lỗi unique → trả
  `internal_error_response` với thông báo mờ mịt, thay vì một lỗi rõ ràng.
- **Không nhất quán giữa các tenant strategy**: `DedicatedDb` có DB riêng nên index này là
  per-database — cùng một email có thể tồn tại vừa ở pool chung vừa ở DB riêng. Và
  `POST /auth/login` khi không truyền `tenantId` chỉ tra pool chung
  (`crates/metap-http/src/routes/auth.rs:79`), nên user của tenant dedicated **bắt buộc** phải
  truyền `tenantId` — hành vi này đúng nhưng đang phụ thuộc vào một ràng buộc DB không ai nêu ra.

Ràng buộc này đang **chịu lực** cho thiết kế login hiện tại (tra theo email không cần tenant thì
mới không nhập nhằng), nên đổi nó không phải sửa 1 dòng SQL — phải quyết luôn: login bằng
`(email, tenantId)` bắt buộc, hay thêm bước chọn tenant. Cần ADR, không nên tự đổi.

---

## 3. MEDIUM — gRPC thiếu rate limit và mặc định plaintext

**Vị trí:** `crates/metap-grpc/src/serve.rs:50-62` (`serve`), `82-108` (`optional_serve`).

REST (`metap_http::build_router`) và gateway đều có `rate_limit` (200ms/300 burst),
`security_headers`, `request_id`, `trace`. Cổng gRPC **không có lớp nào** — chỉ
`Server::builder().add_service(...).serve(addr)`. Cùng một `CrudService`, cùng 6 thao tác
list/get/create/update/transition/delete, nhưng một đường có throttle còn đường kia không.

Thêm nữa, `optional_serve` — đường mà **mọi binary thật đang dùng** (`../metap-demo-crm`,
`../metap-demo-jira`) — luôn truyền `tls_config: None` và luôn `TokenVerifier::Static`. Khả năng
mTLS mà `serve()` có sẵn không có đường nào chạm tới từ helper này, nên trên thực tế gRPC nội bộ
đang chạy **plaintext**, dựa hoàn toàn vào tin cậy mức mạng.

Không phải lỗ hổng nếu cổng gRPC thực sự chỉ ở trong VPC kín — nhưng đó là giả định về deployment
chưa được ghi ở đâu cả, trong khi REST thì không cần giả định đó. Đề xuất tối thiểu: ghi rõ giả
định này vào doc comment của `serve.rs` (giống cách `metrics.rs` đã ghi cho `/metrics` ở audit 03
finding #14), và cho `optional_serve` nhận `Option<ServerTlsConfig>` để deployment nào cần thì bật.

---

## 4. MEDIUM — `GET /auth/token` phát credential nhưng miễn CSRF

**Vị trí:** `crates/metap-http/src/routes/auth.rs:154-165` (`issue_token`),
`crates/metap-http/src/auth.rs:116-123` (miễn CSRF cho GET/HEAD/OPTIONS).

`GET /auth/token` mint một JWT Bearer **sống 1 giờ, mang đúng identity của caller**, và xác thực
bằng cookie. Vì là `GET`, nó rơi vào nhánh miễn CSRF:

```rust
if !matches!(parts.method, Method::GET | Method::HEAD | Method::OPTIONS) { /* CSRF check */ }
```

Miễn CSRF cho GET là đúng nguyên tắc chung (GET không được đổi state) — nhưng route này không đổi
state, nó **phát credential**, một loại rủi ro khác hẳn. Thứ duy nhất chặn một trang lạ đọc được
token là CORS: `metap_runtime::cors::build` chỉ `allow_credentials(true)` kèm danh sách origin
tường minh, và origin rỗng thì fail-closed (xem finding #9) — nên **hiện tại không khai thác
được**. Nhưng nó biến toàn bộ an toàn của endpoint nhạy cảm nhất thành phụ thuộc vào một biến môi
trường cấu hình đúng, cộng thêm việc mọi origin trong allowlist đều phải sạch XSS/không bị chiếm
subdomain.

**Đề xuất:** yêu cầu header `X-CSRF-Token` cho riêng route này dù nó là GET (hoặc đổi thành POST).
Chi phí: `apiFetch` trong `@metap/platform-ui` hiện chỉ gắn CSRF header cho non-GET, nên cần sửa
kèm 1 chỗ ở FE — nhỏ, và làm cho endpoint tự đứng vững thay vì dựa vào CORS.

---

## 5. MEDIUM — `forwarded_bearer_token` fallback im lặng sang service account

**Vị trí:** `crates/metap-grpc/src/client.rs:86-91` (`pick_token`),
`crates/metap-control/src/auth_context.rs:130-134`.

```rust
match ctx.forwarded_bearer_token.as_deref() {
    Some(forwarded) => Cow::Borrowed(forwarded),
    None => Cow::Owned((*service_token.current()).clone()),
}
```

Chỉ **duy nhất** `graphql-gateway/src/server.rs::authenticate` set `forwarded_bearer_token`. Mọi
đường auth phía server (`resolve_request_context` — dùng chung cho REST, gRPC, GraphQL in-process,
cả Bearer lẫn cookie) đều để `None`, với comment thừa nhận thẳng: *"A future 2nd-hop forward from
one of these servers could thread the raw token in here too, but nothing needs that today."*

Hôm nay đúng là chưa có call site nào dính — `metap-graphql-http` luôn dùng `state.crud`
in-process. Vấn đề là **hình dạng của thất bại khi điều đó thay đổi**: ngày nào một service mount
backend remote, mọi request đi qua nó sẽ *âm thầm* chạy dưới quyền service account thay vì quyền
người dùng thật. Không lỗi compile, không log, không 403 — chỉ là permission rộng hơn dự kiến.
Đây là loại lỗi đắt nhất để phát hiện sau.

**Đề xuất:** làm cho fallback phải khai báo tường minh — ví dụ `ServiceTokenSource` thành
`Option` trên `GrpcBackend`, hoặc một cờ `allow_service_account_fallback` mà call site phải bật.
Boot-time schema discovery của gateway (call site hợp lệ duy nhất hiện nay) bật cờ đó; mọi call
site chạy trong ngữ cảnh request thì không, và thiếu forwarded token là lỗi rõ ràng.

---

## 6. LOW — `POST /auth/logout` không kiểm CSRF

**Vị trí:** `crates/metap-http/src/routes/auth.rs:112-115`

Route cố tình không nhận `AuthContext` (lý do đã ghi rõ trong doc comment và hợp lý: bắt auth ở
route mà việc duy nhất của nó là "xoá session" sẽ 401 đúng lúc cookie vừa hết hạn). Hệ quả kèm
theo: không có cửa nào chạy CSRF check, nên một trang lạ có thể ép browser nạn nhân logout. Tác
hại chỉ là phiền (bị đăng xuất), không mất dữ liệu, không chiếm tài khoản — nên LOW. Ghi lại để
biết đây là tradeoff đã cân nhắc chứ không phải sót.

Cùng nhóm: `POST /auth/login` cũng không thể có CSRF (chưa có session) → login-CSRF kinh điển
(ép nạn nhân đăng nhập vào tài khoản của kẻ tấn công). Chuẩn ngành cũng chấp nhận; nêu cho đủ.

---

## 7. LOW — Gateway hardcode `SchemaLimits::default()`

**Vị trí:** `crates/graphql-gateway/src/schema_builder.rs:143`

```rust
let schema = build_schema(&registry, backend.clone(), SchemaLimits::default())?;
```

`depth: 10, complexity: 1000` (`crates/metap-graphql/src/schema.rs:44-53`), và chính doc comment
của `Default` nói đây là *"starting guardrails, not a permanent tuning"*. Gateway lại là bề mặt
public nhất (một schema gộp nhiều service, mỗi entity thêm resolver), nên nó là chỗ **cần chỉnh
được nhất** mà lại là chỗ duy nhất không chỉnh được — mọi thứ khác trong gateway đều đọc từ env.
Đề xuất: thêm `GRAPHQL_MAX_DEPTH`/`GRAPHQL_MAX_COMPLEXITY` vào `GatewayConfig`.

---

## 8. LOW — JWT không có `jti`/revocation; `aud`/`iss` chung toàn mesh

**Vị trí:** `crates/metap-peripherals/src/auth.rs:179-190` (`mint_jwt`), `214-224`
(`decode_access_token`).

Phần verify **làm đúng**: RS256 ghim cứng (không dính `alg: none`/HS-confusion), `aud`/`iss` đều
được validate, có leeway. Hai điểm còn lại đáng ghi:

- **Không có `jti`, không có revocation list.** Token rò rỉ thì dùng được đến hết `exp` (1h) +
  leeway (20s), không có cách thu hồi. Đã được biết (comment ở `auth.rs:130-133` nhắc Phase 20),
  nhưng nay **nặng hơn trước**: từ 2026-09-03 token nằm trong cookie sống 1h thay vì chỉ trong
  React state mất khi reload, nên cửa sổ dùng lại của một token bị rò đã rộng ra thật sự.
- **`aud`/`iss` là hằng số dùng chung** (`JWT_AUDIENCE = "metap-api"`, `JWT_ISSUER = "metap"`) cho
  *mọi* service. Cộng với mô hình chia chung keypair mà gateway đang dựa vào để forward token,
  điều này nghĩa là: token mint cho service A dùng được ở service B và ở gateway, không phân biệt.
  Ranh giới tin cậy thật sự là "ai giữ private key", còn `aud` không thu hẹp thêm gì. Đúng như
  thiết kế đã chọn, nhưng nên ghi rõ — vì `aud` tồn tại thường khiến người đọc tưởng nó đang phân
  tách service.

---

## 9. LOW — Tên test CORS nói ngược với hành vi thật

**Vị trí:** `crates/metap-runtime/src/cors.rs` (test `empty_origins_uses_permissive_default`).

Khi `cors_origins` rỗng, `build` trả `CorsLayer::new()`. Đã đối chiếu source tower-http 0.6.11:
`CorsLayer::new()` là cấu hình **restrictive** (mọi trường `Default::default()`, không phát header
CORS nào); `permissive()` là constructor **khác**. Tức hành vi thật là fail-closed — đúng và an
toàn — nhưng tên test nói ngược lại.

Đáng sửa vì đây là hàm mà cả `metap-http` lẫn `graphql-gateway` dùng chung, và finding #4 ở trên
đang **dựa vào chính hành vi fail-closed này** làm lớp phòng thủ. Một người đọc tin theo tên test
có thể kết luận sai rằng để trống `CORS_ORIGINS` nghĩa là mở toang.

---

## 10. LOW — `metap-jwks` vẫn là code chết

`optional_serve` (`crates/metap-grpc/src/serve.rs:82-108`) luôn dựng `TokenVerifier::Static`, và
doc comment nói rõ binary nào cần `Jwks` thì phải tự gọi `serve()`. Không binary nào làm điều đó.
`metap-http::AuthContext` cũng không có nhánh JWKS. Nên toàn bộ cơ chế 3 bước rotate key của
`metap-jwks`/`metap-jwks-http` chưa từng chạy ở đâu.

Không phải bug — đã được ghi là opt-in trong CLAUDE.md. Nêu ở đây vì nó liên quan trực tiếp
finding #5 và #8: JWKS chính là câu trả lời cho "gateway và upstream không chung keypair", tức là
tiền đề để bỏ được mô hình forward-token-chung-key hiện tại. Nó đang tồn tại nhưng chưa có đường
để dùng.

---

## Những chỗ đã kiểm và **đúng** (không phải finding)

Ghi lại để lần audit sau không phải kiểm lại từ đầu:

- **Mọi RPC gRPC đều authenticate** — cả 6 (`crates/metap-grpc/src/service.rs:41,76,89,108,128,152`),
  không sót cái nào.
- **Mọi route module đều có auth extractor** trừ `health`/`metrics` (cố ý, đã ghi doc ở audit 03
  #14). `cron` dùng `AdminContext` cho toàn bộ handler.
- **Chỉ 2 chỗ nhận `tenantId` từ client** (`/auth/login`, `/auth/providers`), đều có lý do rõ.
  Basic auth **từ chối** thay vì thay thế khi `user.tenant_id` lệch `X-Tenant-Id`
  (`crates/metap-http/src/auth.rs:219-221`) — đúng cách fix của audit 02 finding #2.
- **SQL identifier injection đã bịt**: `field.name`/`entity.name` bị ép charset
  `^[A-Za-z][A-Za-z0-9_]*$` ở `crates/metap-metadata/src/compiler.rs:81-96` trước khi tới chỗ nội
  suy `format!("\"{}\"", field.name)` trong `metap-query`. Có regression test.
- **`EntityField.computed` không đụng SQL** — `render_expression`
  (`crates/metap-metadata/src/computed.rs`) là template string thuần Rust, render trong
  `metap-crud`, không bao giờ vào câu SQL.
- **OIDC an toàn ở các điểm kinh điển**: PKCE + nonce + `state` (state bind với tenant, check ở
  `routes/auth.rs:286-293`), JIT provisioning khớp theo `sub` của IdP **không phải email**
  (`crates/metap-auth/src/lib.rs:130-137`) — nên không có đường chiếm tài khoản qua email trùng.
- **Login chống enumeration bằng timing**: `dummy_hash()`
  (`crates/metap-peripherals/src/auth.rs:52-57`) bắt nhánh "email không tồn tại" trả cùng chi phí
  argon2 với nhánh sai mật khẩu.
- **Cookie session dựng đúng chuẩn**: `HttpOnly` cho session, `SameSite=Lax`, host-only (không set
  `Domain`), double-submit CSRF với giá trị random **độc lập với JWT** (nên rò JWT qua log không
  đủ để forge CSRF header), `Max-Age(0)` khi clear giữ nguyên attribute để browser thật sự xoá.
- **Thứ tự ưu tiên Authorization header > cookie** (`crates/metap-http/src/auth.rs:92-127`) là thứ
  giữ cho CSRF check không bị vòng qua: header không thể bị browser tự gắn vào request cross-site.
- **`metap-graphql-http` dùng lại chính `AuthContext` của REST** nên GraphQL kế thừa nguyên cookie
  + CSRF + JWT, không có đường auth thứ hai để lệch.

## Ghi chú về mức độ

Xếp #1 là HIGH vì nó vượt ranh giới tenant trong mô hình SaaS (khách hàng đọc được hạ tầng của
operator), dù cần role admin — đúng ngưỡng mà `PermissionService`/`Router` được xây để giữ.
Không có finding nào ở mức cho phép **người dùng chưa đăng nhập** làm gì vượt quyền: các bề mặt
public (`/health`, `/metrics`, `/metadata/openapi.json`) đều đã được rà và ghi tradeoff từ audit
03.

---

# Phần B — Kiến trúc

Không lặp lại audit 03 (layering, ranh giới crate, doc-drift — đã fix 14/14). Góc nhìn ở đây:
**đi theo một capability qua cả 4 giao thức và xem nó có nguyên vẹn không**, cộng với những cơ
chế đã viết xong nhưng chưa có đường chạy.

| # | Mức độ | Vấn đề |
|---|---|---|
| B1 | **HIGH** | Gateway là aggregator **tĩnh + fail-closed toàn phần**, mâu thuẫn với chính lời hứa "metadata động, hot-swap không restart" |
| B2 | MEDIUM | Độ trung thực của error **giảm dần qua từng hop** — `field_errors` bị nén thành một con số |
| B3 | MEDIUM | Bề mặt năng lực lệch hẳn giữa các giao thức: REST ~13 nhóm route, gRPC/GraphQL chỉ có records |
| B4 | MEDIUM | Cookie/CSRF session (ship 2026-09-03) có **0 test** |
| B5 | LOW | `attach_trace_context` không có caller nào — tracing thủng đúng ở các hop REST |
| B6 | LOW | Cùng tên field, khác kiểu giữa REST và GraphQL (`assigneeId`) |
| B7 | LOW | Gateway giữ email+password thật của N upstream trong env, không xoay được |

---

## B1. HIGH — Gateway tĩnh và fail-closed toàn phần

**Vị trí:** `crates/graphql-gateway/src/schema_builder.rs:48-149`, `src/main.rs:34-42`.

`build()` chạy **đúng một lần lúc boot**, và mọi bước đều `?`:

- login vào từng upstream (dòng 59-72),
- `GET` metadata của từng upstream (75-97),
- `GrpcBackend::connect` từng upstream (101-111),
- `registry.validate_references()` **toàn cục** (140),
- `build_schema` (143).

Ba hệ quả kiến trúc, không phải bug riêng lẻ:

1. **Boot all-or-nothing.** Một upstream chậm/đang restart → gateway *không khởi động được*, dù
   các upstream khác hoàn toàn khoẻ. Trong k8s đây là CrashLoopBackOff: gateway sập theo upstream
   yếu nhất. Một BFF thường phải làm ngược lại — phục vụ phần schema nào lấy được, đánh dấu phần
   còn lại là degraded.
2. **Không re-discovery.** Publish một low-code entity ở upstream → chính `/graphql` của upstream
   đó thấy ngay (`metap-graphql-http`'s `SchemaHolder` so `Arc::ptr_eq` rồi rebuild lazy,
   `crates/metap-graphql-http/src/lib.rs:33-70`), nhưng **gateway vẫn phục vụ schema cũ cho tới
   khi có người restart tay**. Cùng một repo đã giải đúng bài toán này ở chỗ khác rồi mà gateway
   không được hưởng.
3. **Coupling qua `validate_references()` toàn cục.** Một entity có `Reference` trỏ sang entity
   nằm ở upstream chưa được khai báo trong `UPSTREAM_<N>_*` → gateway không boot. Nghĩa là thêm
   một quan hệ cross-service ở tầng metadata sẽ *âm thầm biến một upstream thành dependency bắt
   buộc lúc boot của gateway*, không ai khai báo điều đó ở đâu cả.

Đây là mâu thuẫn kiến trúc lớn nhất tìm được: toàn bộ giá trị cốt lõi của nền tảng là *metadata
thay đổi lúc runtime, không cần restart* (`ArcSwap` ở `AppState.metadata`, hot-reload sau
publish/rollback, low-code) — nhưng thành phần đứng trước mặt client lại đóng băng metadata tại
thời điểm boot.

**Đề xuất** (cần chốt hướng, không tự làm): cho gateway một `SchemaHolder` tương đương — poll
`GET /metadata/entities` của từng upstream theo TTL (hoặc nghe outbox event), rebuild schema khi
đổi, và cho phép boot ở trạng thái một phần (upstream lỗi → log + loại khỏi schema + retry nền,
thay vì `?`). `validate_references()` khi đó nên chạy trên tập entity thực sự resolve được, không
phải điều kiện boot.

---

## B2. MEDIUM — Error mất thông tin dần qua từng hop

Đi theo một lỗi validation (tạo record thiếu field bắt buộc) từ `CrudService` ra tới client
GraphQL qua gateway:

| Hop | Dạng | Còn giữ được gì |
|---|---|---|
| `CrudService` | `ServiceResult::Err { status, error, message, field_errors }` | đủ: mã lỗi, thông điệp, **lỗi theo từng field** |
| → gRPC (`status.rs:14-31`) | `tonic::Status` | `Code` + **một chuỗi text**; `field_errors` bị nén thành `" (3 field error(s))"` — chỉ còn **số lượng** |
| → `GrpcBackend` (`client.rs:115-125`) | `ServiceResult` dựng lại | status số suy ngược từ `Code`; mọi thứ không map được → `502` |
| → GraphQL (`schema.rs:56-62`) | `GqlError::new(format!("{status}: {}", ...))` | **một chuỗi duy nhất**, không `extensions`, không mã lỗi máy đọc được |

Client GraphQL qua gateway nhận về `"422: Validation failed (3 field error(s))"` — không biết
field nào sai, không có mã để phân nhánh xử lý. Trong khi đúng thứ nền tảng này quảng cáo là
"validation sinh tự động từ metadata" — nó chỉ được giao **đủ trên REST**.

Hai chỗ sửa độc lập nhau:
- gRPC: đưa `field_errors` vào `Status::with_details` (hoặc `google.rpc.BadRequest`) thay vì đếm.
- GraphQL: dùng `GqlError::new(...).extend_with(|_, e| e.set("code", ...))` để có `extensions`
  đúng chuẩn GraphQL, thay vì nhét status vào đầu chuỗi message.

---

## B3. MEDIUM — Bề mặt năng lực lệch hẳn giữa các giao thức

Đếm thật:

- **REST** (`metap_http::build_router`): ~13 nhóm route — records, attachments, workflow-events,
  metadata, auth, admin/users, admin/policies, admin/cron-jobs, dashboards, preferences, users,
  health, metrics.
- **gRPC** (`crates/metap-grpc/proto/metap_crud.proto:18-24`): **1 service, 6 rpc** —
  List/Get/Create/Update/Transition/Delete. Chỉ records.
- **GraphQL**: cũng chỉ records (Query/Mutation sinh từ entity).

Nghĩa là `graphql-gateway` — thứ được gọi là "the real BFF" — về bản chất là **records-only
BFF**. Một màn hình thật cần đính kèm file, xem lịch sử workflow, hay đọc dashboard config thì
vẫn phải gọi thẳng REST của từng service, tức là client vẫn phải biết topology mà gateway sinh ra
để che đi.

Không nhất thiết sai (records là phần khó nhất và là phần đáng gộp nhất), nhưng hiện **không chỗ
nào nói ra ranh giới này** — `CLAUDE.md` và doc gateway đều mô tả nó như một BFF gộp toàn phần.
Đề xuất rẻ nhất: ghi rõ "records-only" vào doc comment của `graphql-gateway` và vào
`docs/vision.md`, rồi quyết có mở rộng proto hay không như một quyết định riêng.

---

## B4. MEDIUM — Cookie/CSRF session không có test nào

`metap_session`/`x-csrf-token` **không xuất hiện trong bất kỳ file nào dưới `crates/*/tests/`**
(grep = 0 kết quả). Cơ chế ship 2026-09-03 gồm: xác thực qua cookie, double-submit CSRF, thứ tự
ưu tiên header > cookie, clear cookie khi logout — không có test nào phủ.

Đáng chú ý vì `metap-http` **đã có sẵn** `tests/jwt_security_postgres.rs` cho đúng loại việc này,
nên đây là lệch so với chuẩn của chính repo, không phải "repo này vốn không test auth". Bốn ca
tối thiểu nên có, mỗi ca là một dòng phòng thủ thật đang không được canh:

1. request có cookie hợp lệ + không có `Authorization` → 200 (đường cookie hoạt động),
2. request POST có cookie nhưng **thiếu** `X-CSRF-Token` → 401 (chốt chặn CSRF thật sự chạy),
3. request POST có cookie + CSRF header **sai giá trị** → 401,
4. request có `Authorization` header hợp lệ + cookie rác → 200 (header thắng, và không bị CSRF
   check chặn nhầm).

Ca 2 quan trọng nhất: nếu điều kiện `if !matches!(parts.method, GET|HEAD|OPTIONS)` bị ai đó sửa
nhầm, không có gì báo — toàn bộ đường cookie mất phòng thủ CSRF mà test suite vẫn xanh.

---

## B5. LOW — `attach_trace_context` chưa có caller nào

**Vị trí:** `crates/metap-runtime/src/http_client.rs:22-35`.

Trace context **có** được thiết lập cho mọi request đến: `request_id.rs:55` gọi
`trace_context::scope(trace_ctx, next.run(request))`. Và hop gRPC **có** propagate:
`metap-grpc/src/client.rs:98-107` đọc `trace_context::current()` rồi gắn `traceparent`.

Nhưng `attach_trace_context` — bản tương đương cho hop REST — **không được gọi ở đâu cả** (grep
toàn `crates/`, chỉ có định nghĩa và test của chính nó). Nên các hop REST xuyên process không
mang trace đi:

- `cron-scheduler` → `/api/:entity/...` của app (hop vận hành đáng quan tâm nhất, vì đây là chỗ
  job chạy nền tác động lên dữ liệu thật),
- `graphql-gateway` → `GET /metadata/entities` của upstream,
- `metap-jwks` → refresh JWKS.

Kết quả: trace liền mạch qua gRPC, đứt qua REST. Đây là loại lỗ hổng quan sát chỉ lộ ra đúng lúc
đang debug sự cố production.

> **Đính chính khi fix (2026-09-03):** bản audit đầu viết "sửa rẻ: gọi `attach_trace_context` ở 3
> call site trên" — **sai**. Gọi thêm ở 3 chỗ đó thôi thì vẫn là no-op, vì cả 3 đều chạy **ngoài**
> mọi request scope: `cron-scheduler` tiêu thụ từ RabbitMQ, gateway fetch metadata lúc boot,
> `metap-jwks` refresh nền — nên `trace_context::current()` trả `None` ở cả ba. Lỗ hổng thật là
> **không có gì tạo trace context ngoài request HTTP**, chứ không phải "quên gọi hàm".
> Cách fix đã áp dụng: `dispatch::execute` mở một **root trace context mới cho mỗi job run**
> (`from_headers` với header rỗng sinh trace id mới), rồi mới gắn `attach_trace_context` vào 3
> callback REST của `workflow_transition`/`bulk_query_action` + webhook — giờ một record bị cron
> ghi truy ngược được về đúng job run gây ra nó. Gateway metadata-fetch và JWKS refresh **cố ý để
> nguyên**: chúng là thao tác boot/nền một lần, một root trace cho mỗi lần đó gần như không có giá
> trị vận hành nào.

---

## B6. LOW — Cùng tên field, khác kiểu giữa REST và GraphQL

Một field `Reference` tên `assigneeId`:

- **REST**: trả về string uuid, cộng `relatedDisplay.assigneeId` = tên hiển thị (Mode 2,
  `CrudService::hydrate_related_display`).
- **GraphQL**: `assigneeId` là **object lồng** kiểu `User`, resolve qua DataLoader
  (`metap-graphql/src/schema.rs:141-175`); muốn lấy id thô phải viết `assigneeId { id }`.

Cách của GraphQL **tốt hơn** (batch đúng, chọn field tuỳ ý, không cần `relatedDisplay`). Vấn đề
thuần nhất quán: cùng một tên field mang hai kiểu khác nhau tuỳ giao thức, và tên `...Id` giờ trỏ
tới một object chứ không phải id. Một client dùng cả hai (đúng kịch bản `@metap/platform-ui` gọi
REST và gọi gateway) phải giữ hai mô hình trong đầu. Chỉ cần ghi rõ trong doc là đủ — đổi tên
field ở GraphQL (`assignee` cho object, `assigneeId` cho id thô) là phương án sạch hơn nhưng phá
vỡ tương thích.

---

## B7. LOW — Gateway giữ credential người dùng thật cho N upstream

**Vị trí:** `crates/graphql-gateway/src/schema_builder.rs:59-64`, `config.rs`.

Mỗi upstream cần `UPSTREAM_<N>_SERVICE_EMAIL` + `_SERVICE_PASSWORD` — gateway đăng nhập như một
*người dùng thật* rồi giữ token. Đây là bản sửa đúng cho sự cố JWT tĩnh hết hạn (2026-09-02), nên
không phải bước lùi. Nhưng ở góc kiến trúc: gateway trở thành nơi tập trung N cặp credential dài
hạn, không có cơ chế xoay vòng, và mọi upstream buộc phải chấp nhận đăng nhập bằng password.

`metap-jwks` chính là câu trả lời cho lớp bài toán này (mỗi service ký bằng key riêng, tin nhau
qua JWKS, không chia sẻ bí mật) nhưng chưa nối được vào đâu (xem A#10). Nêu ở đây để hai finding
được đọc cùng nhau: A#10 nói "JWKS là code chết", B7 nói **vì sao đáng nối** — nó gỡ luôn cả ràng
buộc chung-keypair mà cơ chế forward token của gateway đang phải dựa vào (A#5).

---

## Những chỗ kiến trúc đã kiểm và **đúng**

- **`RecordBackend` là seam đúng chỗ.** Cùng một resolver GraphQL chạy được với `CrudService`
  in-process lẫn `GrpcBackend` remote mà không biết mình đang chạy với cái nào — và
  `metap-graphql-http` với gateway thật sự dùng chung code resolver, không phải hai bản song song.
  Đây là phần kiến trúc tốt nhất của cụm này.
- **Permission không thể lệch giữa các giao thức**, vì `GrpcRecordService` gọi thẳng vào
  `CrudService` chứ không tự kiểm tra lại — REST/gRPC/GraphQL dùng chung đúng một đường enforcement.
- **`ListResponse` của gRPC và `{Type}Connection` của GraphQL cố ý cùng hình dạng**
  (`records`/`nextCursor`/`hasMore`), có ghi lý do ở doc comment — hai transport phi-REST không
  trôi khỏi nhau.
- **Masking field-level là miễn phí ở mọi giao thức** vì mọi resolver đọc `RecordDto` đã
  `filter_readable_fields` từ trước, không có đường vòng nào đọc dữ liệu chưa mask.
- **Coverage test ở các seam giao thức là thật**, không phải chỉ unit test: có e2e Postgres riêng
  cho gateway, gRPC (cả server lẫn client backend), GraphQL (schema và HTTP mount). Lỗ hổng duy
  nhất tìm được là cookie/CSRF (B4).
- **Fail-fast khi hai upstream trùng tên entity** (`registry.register` từ chối, `schema_builder.rs:130`)
  — đúng cách, phát hiện lúc boot thay vì trả nhầm dữ liệu lúc chạy.
