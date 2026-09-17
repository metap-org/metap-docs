# Index — audit đã chạy cho `metap`

Theo dõi audit nào đã chạy, phần nào đã fix/verify, phần nào còn treo. Cập nhật file này mỗi khi
1 audit mới chạy hoặc trạng thái 1 finding thay đổi — không xoá audit cũ, chỉ cập nhật cột trạng
thái.

| # | File | Phạm vi | Trạng thái |
|---|---|---|---|
| 02 | [`02-full-codebase-audit.md`](02-full-codebase-audit.md) | Review từng dòng toàn bộ `crates/`+`packages/platform-react` (6 agent độc lập, 2026-08-26) | 12 finding nghiêm trọng nhất (10 "ưu tiên xử lý" + 2 HIGH workflow) đã verify độc lập + fix — chi tiết [`../roadmap/41-audit-2-fixes.md`](../roadmap/41-audit-2-fixes.md). Phần còn lại (mọi MEDIUM/LOW, phụ lục `apps/*`) **chưa verify/fix** |
| 03 | [`03-metap-core-architecture-audit.md`](03-metap-core-architecture-audit.md) | Kiến trúc: layering, ranh giới crate, doc-vs-reality drift, bề mặt bảo mật multi-tenant, ranh giới `metap`↔`metap-lowcode` (1 agent Opus, 2026-09-02) | **Toàn bộ 14/14 finding đã fix** 2026-09-02 — xem bảng bên dưới |
| 04 | [`04-auth-protocols-gateway-audit.md`](04-auth-protocols-gateway-audit.md) | **Bảo mật + kiến trúc** của auth (cookie session/CSRF/Bearer/Basic/OIDC/JWT), 4 giao thức giao tiếp (REST/gRPC/GraphQL/RabbitMQ), `graphql-gateway` — chỉ core `metap`, `metap-lowcode` để lại lần sau (2026-09-03) | **16/17 finding đã đóng** 2026-09-13 (A#1/A#4/A#7/A#9/A#10/B1(HIGH)/B7 fix, A#2/A#3-TLS/B3/B6 xác nhận cố ý qua ADR, A#3-rate-limit/A#5/A#8/B2/B4/B5 fix). Còn mở: A#6 (tradeoff đã cân nhắc, không cần fix thêm). Xem 2 bảng bên dưới |
| 06 | [`06-tenant-isolation-and-audit-trail-audit.md`](06-tenant-isolation-and-audit-trail-audit.md) | Sweep có hệ thống (1 agent Opus, 2026-09-17) vào 2 vùng chưa từng audit và đều vừa ship: per-tenant schema isolation (feature 35) + crate `metap-audit` (audit 05); có rà thêm bề mặt sinh SQL và tàn dư Phase 86 | **Chưa fix gì — 3 HIGH mới**. #1 `Router::pool_for` bỏ qua schema tenant (5 call site `metap-lowcode` chưa từng được tính, 1 trên mọi HTTP handler); #2 tenant provision sau feature 35 không login được + `users_email_unique` mất tính toàn cục mà audit 04 A#2 dựa vào; #3 `/audit-events` bypass field-level permission. #1/#2 cùng gốc: feature 35 phá bất biến `schema_name == "public"` mà 3 chỗ khác vẫn giả định — **chưa nổ, sẽ nổ ở lần `provision_schema_tenant` tiếp theo** |
| 05 | [`05-crud-audit-trail-gap.md`](05-crud-audit-trail-gap.md) | `CrudService`'s audit trail cho nghiệp vụ enterprise — tìm qua trao đổi thường (không phải audit sweep có hệ thống), 2026-09-13 | **Đã code và verify xong (2026-09-13)** — crate `metap-audit` mới, trait `AuditTrailStore` pluggable (Postgres mặc định, cho phép DB/storage khác), `EntityDefinition.audit` per-entity opt-in, `CrudService::with_audit(...)` ghi audit sau khi cả 4 write (`create`/`update`/`delete`/`transition`) commit; `reason: Option<&str>` thêm tường minh, rippled qua REST/gRPC/GraphQL + downstream (`metap-demo-waf` 7 call site). `workflow_events` giữ nguyên, không đổi. Smoke-test thật qua REST trên `metap-demo-jira` (`jira.projects`) xác nhận đúng. Còn treo: mask field nhạy cảm trước khi ghi audit — chưa làm |

## Chi tiết audit 04 — Phần A, bảo mật

| # | Mức độ | Vấn đề | Trạng thái |
|---|---|---|---|
| 1 | **HIGH** | SSRF có phản hồi qua `webhook` target của cron — tenant admin đọc được nội bộ mạng + cloud metadata | **Đã fix** 2026-09-03 — `cron-scheduler`'s `executor/ssrf_guard.rs` mới (chặn private/loopback/link-local/CGNAT/ULA + IPv4-mapped IPv6, allowlist host tuỳ chọn, cấm header `Authorization`/`Cookie`), client webhook riêng với `redirect::Policy::none()`. 11 unit test |
| 2 | MEDIUM | `users_email_unique` unique toàn cục trên `email`, không phải `(tenant_id, email)` | **Xác nhận cố ý** 2026-09-13 (ADR trực tiếp, chủ dự án uỷ quyền quyết định) — ghi doc tại `metap_peripherals::auth::verify_credentials`: ràng buộc này đang chịu lực cho `POST /auth/login` (không có tenant picker), tách theo tenant cần redesign UI, không phải chỉ đổi query |
| 3 | MEDIUM | Cổng gRPC không rate limit, `optional_serve` luôn plaintext (`tls_config: None`) | **Rate limit đã fix** 2026-09-13 — `metap-grpc::rate_limit::RateLimitLayer` mới (token-bucket riêng cho tonic, cùng tham số mặc định gateway HTTP đang dùng 200ms/burst 300), 3 unit test. **Plaintext mặc định xác nhận cố ý** (ADR) — ghi doc trên `serve()`: mesh-internal port, mTLS là việc của sidecar, không phải thiếu sót |
| 4 | MEDIUM | `GET /auth/token` phát credential nhưng miễn CSRF (vì là GET), chỉ còn CORS đỡ | **Đã fix** 2026-09-03 — `cookies::credential_issuing_request_allowed` (gate riêng cho endpoint phát credential, Bearer không bị ảnh hưởng) + `apiFetch` gắn CSRF header cho mọi request thay vì chỉ non-GET |
| 5 | MEDIUM | `forwarded_bearer_token` fallback im lặng sang service account | **Đã fix** 2026-09-13 — `metap-grpc::client::pick_token` log `tracing::debug!` khi rơi vào fallback, không đổi hành vi |
| 6 | LOW | `POST /auth/logout` không kiểm CSRF → logout-CSRF | Chưa fix — tradeoff đã cân nhắc, ghi lại cho đủ |
| 7 | LOW | Gateway hardcode `SchemaLimits::default()`, không chỉnh qua env | **Đã fix** 2026-09-03 (Phase 66) — gateway đọc `GRAPHQL_MAX_DEPTH`/`GRAPHQL_MAX_COMPLEXITY` từ env (nó không có Postgres pool); service có pool đọc cùng 2 khoá đó từ `platform_configs` qua crate mới `metap-config` |
| 8 | LOW | JWT không có `jti`/revocation; `aud`/`iss` là hằng số chung toàn mesh | **`jti` đã thêm** 2026-09-13 — `mint_jwt` sinh `Uuid::new_v4()` mỗi token, `AccessClaims.jti: Option<String>` phía decode (Option để token cũ trước bản vá vẫn decode được). Hạ tầng revocation vẫn **chưa xây** — cố ý, chưa có tính năng thật cần đến (xem doc comment `mint_jwt`) |
| 9 | LOW | Tên test `empty_origins_uses_permissive_default` nói ngược hành vi thật (restrictive) | **Đã fix** 2026-09-13 — đổi tên thành `empty_origins_uses_restrictive_default`, ghi rõ lý do trong test |
| 10 | LOW | `metap-jwks` vẫn là code chết — `optional_serve` không có đường dùng `TokenVerifier::Jwks` | **Đã fix** 2026-09-13 — `OptionalServeConfig.token_verifier_override: Option<Arc<TokenVerifier>>` mới; `../metap-demo-waf`'s 3 service (`zones`/`scanning`/`alerting-service`) xoá được đoạn tự chế đã có sẵn (bypass `optional_serve` để tự build `TokenVerifier::Jwks`), gọi thẳng `optional_serve` với `token_verifier_override: state.token_verifier.clone()` |

## Chi tiết audit 04 — Phần B, kiến trúc

| # | Mức độ | Vấn đề | Trạng thái |
|---|---|---|---|
| B1 | **HIGH** | Gateway là aggregator tĩnh + fail-closed toàn phần: 1 upstream chết → không boot; publish low-code → gateway vẫn phục vụ schema cũ tới khi restart tay | **Đã fix** 2026-09-13 — `UpstreamCache`/`GatewaySchemaCache` mới (TTL 30s, `moka`, cùng pattern `RegistryCache`): mỗi upstream refresh độc lập, không bao giờ quên schema thành công gần nhất; 1 upstream never-reachable chỉ rớt field `Reference` trỏ tới nó, không chặn upstream khác; `GET /health` + GraphQL `_gatewayHealth` báo trạng thái. Verify: 2 e2e test mới (`gateway_e2e_postgres.rs`) + live trên `../../metap-demo-waf` (stop/start `scanning-service` thật, gateway tự phục hồi trong 1 TTL, không restart). Chi tiết `../roadmap/85-*.md` |
| B2 | MEDIUM | Error mất thông tin dần qua từng hop — `field_errors` bị nén thành một con số đếm trong chuỗi text | **Đã fix** 2026-09-03 — `ErrorDetails` JSON trong `Status::details` (lossless, có fallback cho peer không nói envelope này) + GraphQL `extensions` (`code`/`status`/`fieldErrors`) |
| B3 | MEDIUM | Bề mặt năng lực lệch: REST ~13 nhóm route, gRPC/GraphQL chỉ records → gateway thực chất là records-only BFF | **Xác nhận cố ý** 2026-09-13 (ADR) — ghi doc ở đầu `metap-grpc`/`metap-graphql`: phạm vi BFF/service-to-service, không phải thiếu sót |
| B4 | MEDIUM | Cookie/CSRF session (ship 2026-09-03) có **0 test** — `metap_session`/`x-csrf-token` không xuất hiện trong `crates/*/tests/` | **Đã fix** 2026-09-03 — tách `requires_csrf_check`/`csrf_matches` thành hàm thuần (6 unit test, không cần DB) + `tests/cookie_session_postgres.rs` (7 e2e) |
| B5 | LOW | `attach_trace_context` không có caller nào → trace liền qua gRPC nhưng đứt qua mọi hop REST | **Đã fix** 2026-09-03 — chẩn đoán ban đầu sai (xem đính chính trong audit): fix thật là `dispatch::execute` mở root trace mỗi job run, rồi mới gắn vào 3 callback REST |
| B6 | LOW | Cùng tên field khác kiểu giữa REST (`assigneeId` = uuid string) và GraphQL (`assigneeId` = object lồng) | **Xác nhận cố ý** 2026-09-13 (ADR) — ghi doc trên `metap_metadata::FieldKind::Reference`: mỗi giao thức đúng idiom riêng, đổi REST theo GraphQL sẽ breaking wire-format |
| B7 | LOW | Gateway giữ email+password thật của N upstream trong env, không xoay vòng được | **Đã fix** 2026-09-13 — `UpstreamConfig.service_password_secret_ref` mới (opt-in, cùng lúc với B1), resolve qua `metap_control::SecretStore` (Vault/AWS/GCP/env) mỗi refresh cycle thay vì đọc literal env var — xoay credential ở backend có hiệu lực trong vòng 1 TTL, không redeploy. `service_password` literal vẫn là default, không migrate bắt buộc |

## Chi tiết audit 03 (đã xử lý xong)

| # | Mức độ | Vấn đề | Trạng thái |
|---|---|---|---|
| 1 | HIGH | ABAC bị bỏ qua ở `routes/attachments.rs`/`workflow_events.rs` | **Đã fix** 2026-09-02 — `CrudService::check_record_permission` mới |
| 2 | HIGH | `ServiceTokenSource` retry loop sleep 2430s thay vì 30s | **Đã fix** 2026-09-02 |
| 3 | MEDIUM | `cron-scheduler`'s `CRON_SERVICE_JWT` static-JWT pattern | **Đã fix** 2026-09-02 — dùng `ServiceTokenSource` (chuyển sang `metap-runtime`) |
| 4 | MEDIUM | Facade `metap` thiếu re-export 5 crate | **Đã fix** 2026-09-02 |
| 5 | MEDIUM | `templates/metap-app` hướng dẫn không compile được | **Đã fix** 2026-09-02 |
| 6 | MEDIUM | `graphql-gateway` kéo cả `metap-http` chỉ để dùng `security_headers` | **Đã fix** 2026-09-02 — chuyển sang `metap-runtime` |
| 7 | MEDIUM | `metap-permission`→`metap-cache` kéo `redis` hard-dep toàn workspace | **Đã fix** 2026-09-02 — feature-gate `redis-backend` |
| 8 | MEDIUM | CLAUDE.md thiếu 3 crate (`metap-attachments`/`metap-auth`/`metap-dashboards`) | **Đã fix** 2026-09-02 |
| 9 | LOW | 5 claim sai cụ thể trong CLAUDE.md | **Đã fix** 2026-09-02 |
| 10 | LOW | `PostgresPolicyStore` sai chỗ theo doc; `metap-metadata→metap-permission` inversion | **Đã fix** 2026-09-02 (doc, gộp chung sửa #9) |
| 11 | LOW | Field/record permission allow-by-default chưa ghi tài liệu | **Đã fix** 2026-09-02 — doc comment ở `permission_snapshot.rs` + CLAUDE.md |
| 12 | LOW | `entity.name` chưa validate charset | **Đã fix** 2026-09-02 — `compiler.rs::validate()` |
| 13 | LOW | ~40 path `apps/crm-server`/`apps/jira-server` lỗi thời + 1 cross-ref hỏng | **Đã fix** 2026-09-02 — 25 file `.rs` sửa qua bulk replace, `error.rs`'s cross-ref sửa riêng (2 dòng tường thuật lịch sử ở CLAUDE.md cố tình giữ nguyên) |
| 14 | LOW | Rate-limit/`/metrics`/`/metadata/entities` — tradeoff chưa ghi tài liệu | **Đã fix** 2026-09-02 — doc comment ở `rate_limit.rs`/`metrics.rs`/`metadata.rs` |

**Ghi chú xác minh**: `cargo build --workspace` + `cargo test --workspace` sạch (89/89 test suite `ok`, 0 fail) sau khi fix cả 14 mục. Riêng finding #1 (ABAC bypass) chưa có e2e test riêng cho `attachments`/`workflow-events` từ trước (gap coverage có sẵn, không phải do lần sửa này) — fix dựa trên tái dùng đúng pattern `CrudService::update`/`delete` đã verify sống nhiều lần, không phải suy đoán.
