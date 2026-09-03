# Index — audit đã chạy cho `metap`

Theo dõi audit nào đã chạy, phần nào đã fix/verify, phần nào còn treo. Cập nhật file này mỗi khi
1 audit mới chạy hoặc trạng thái 1 finding thay đổi — không xoá audit cũ, chỉ cập nhật cột trạng
thái.

| # | File | Phạm vi | Trạng thái |
|---|---|---|---|
| 02 | [`02-full-codebase-audit.md`](02-full-codebase-audit.md) | Review từng dòng toàn bộ `crates/`+`packages/platform-react` (6 agent độc lập, 2026-08-26) | 12 finding nghiêm trọng nhất (10 "ưu tiên xử lý" + 2 HIGH workflow) đã verify độc lập + fix — chi tiết [`../roadmap/41-audit-2-fixes.md`](../roadmap/41-audit-2-fixes.md). Phần còn lại (mọi MEDIUM/LOW, phụ lục `apps/*`) **chưa verify/fix** |
| 03 | [`03-metap-core-architecture-audit.md`](03-metap-core-architecture-audit.md) | Kiến trúc: layering, ranh giới crate, doc-vs-reality drift, bề mặt bảo mật multi-tenant, ranh giới `metap`↔`metap-lowcode` (1 agent Opus, 2026-09-02) | **Toàn bộ 14/14 finding đã fix** 2026-09-02 — xem bảng bên dưới |
| 04 | [`04-auth-protocols-gateway-audit.md`](04-auth-protocols-gateway-audit.md) | **Bảo mật + kiến trúc** của auth (cookie session/CSRF/Bearer/Basic/OIDC/JWT), 4 giao thức giao tiếp (REST/gRPC/GraphQL/RabbitMQ), `graphql-gateway` — chỉ core `metap`, `metap-lowcode` để lại lần sau (2026-09-03) | **17 finding — 3 đã fix** (A#1 HIGH, B#4, B#5, cùng ngày). Phần A bảo mật (1 HIGH, 4 MEDIUM, 5 LOW), Phần B kiến trúc (1 HIGH, 3 MEDIUM, 3 LOW). Xem 2 bảng bên dưới |

## Chi tiết audit 04 — Phần A, bảo mật

| # | Mức độ | Vấn đề | Trạng thái |
|---|---|---|---|
| 1 | **HIGH** | SSRF có phản hồi qua `webhook` target của cron — tenant admin đọc được nội bộ mạng + cloud metadata | **Đã fix** 2026-09-03 — `cron-scheduler`'s `executor/ssrf_guard.rs` mới (chặn private/loopback/link-local/CGNAT/ULA + IPv4-mapped IPv6, allowlist host tuỳ chọn, cấm header `Authorization`/`Cookie`), client webhook riêng với `redirect::Policy::none()`. 11 unit test |
| 2 | MEDIUM | `users_email_unique` unique toàn cục trên `email`, không phải `(tenant_id, email)` | Chưa fix — cần ADR, ràng buộc này đang chịu lực cho thiết kế login |
| 3 | MEDIUM | Cổng gRPC không rate limit, `optional_serve` luôn plaintext (`tls_config: None`) | Chưa fix |
| 4 | MEDIUM | `GET /auth/token` phát credential nhưng miễn CSRF (vì là GET), chỉ còn CORS đỡ | Chưa fix — sửa kèm 1 chỗ ở `@metap/platform-ui` |
| 5 | MEDIUM | `forwarded_bearer_token` fallback im lặng sang service account | Chưa fix — chưa có call site dính, nhưng thất bại tương lai sẽ âm thầm |
| 6 | LOW | `POST /auth/logout` không kiểm CSRF → logout-CSRF | Chưa fix — tradeoff đã cân nhắc, ghi lại cho đủ |
| 7 | LOW | Gateway hardcode `SchemaLimits::default()`, không chỉnh qua env | Chưa fix |
| 8 | LOW | JWT không có `jti`/revocation; `aud`/`iss` là hằng số chung toàn mesh | Chưa fix — nặng hơn từ khi token nằm trong cookie 1h |
| 9 | LOW | Tên test `empty_origins_uses_permissive_default` nói ngược hành vi thật (restrictive) | Chưa fix |
| 10 | LOW | `metap-jwks` vẫn là code chết — `optional_serve` không có đường dùng `TokenVerifier::Jwks` | Chưa fix |

## Chi tiết audit 04 — Phần B, kiến trúc

| # | Mức độ | Vấn đề | Trạng thái |
|---|---|---|---|
| B1 | **HIGH** | Gateway là aggregator tĩnh + fail-closed toàn phần: 1 upstream chết → không boot; publish low-code → gateway vẫn phục vụ schema cũ tới khi restart tay | Chưa fix — mâu thuẫn trực tiếp với lời hứa "hot-swap không restart" của nền tảng |
| B2 | MEDIUM | Error mất thông tin dần qua từng hop — `field_errors` bị nén thành một con số đếm trong chuỗi text | Chưa fix — 2 chỗ sửa độc lập (gRPC `Status::with_details`, GraphQL `extensions`) |
| B3 | MEDIUM | Bề mặt năng lực lệch: REST ~13 nhóm route, gRPC/GraphQL chỉ records → gateway thực chất là records-only BFF | Chưa fix — tối thiểu phải ghi rõ ranh giới vào doc |
| B4 | MEDIUM | Cookie/CSRF session (ship 2026-09-03) có **0 test** — `metap_session`/`x-csrf-token` không xuất hiện trong `crates/*/tests/` | **Đã fix** 2026-09-03 — tách `requires_csrf_check`/`csrf_matches` thành hàm thuần (6 unit test, không cần DB) + `tests/cookie_session_postgres.rs` (7 e2e) |
| B5 | LOW | `attach_trace_context` không có caller nào → trace liền qua gRPC nhưng đứt qua mọi hop REST | **Đã fix** 2026-09-03 — chẩn đoán ban đầu sai (xem đính chính trong audit): fix thật là `dispatch::execute` mở root trace mỗi job run, rồi mới gắn vào 3 callback REST |
| B6 | LOW | Cùng tên field khác kiểu giữa REST (`assigneeId` = uuid string) và GraphQL (`assigneeId` = object lồng) | Chưa fix — chỉ cần ghi doc |
| B7 | LOW | Gateway giữ email+password thật của N upstream trong env, không xoay vòng được | Chưa fix — đọc cùng A#5 và A#10 |

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
