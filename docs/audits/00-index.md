# Index — audit đã chạy cho `metap`

Theo dõi audit nào đã chạy, phần nào đã fix/verify, phần nào còn treo. Cập nhật file này mỗi khi
1 audit mới chạy hoặc trạng thái 1 finding thay đổi — không xoá audit cũ, chỉ cập nhật cột trạng
thái.

| # | File | Phạm vi | Trạng thái |
|---|---|---|---|
| 02 | [`02-full-codebase-audit.md`](02-full-codebase-audit.md) | Review từng dòng toàn bộ `crates/`+`packages/platform-react` (6 agent độc lập, 2026-08-26) | 12 finding nghiêm trọng nhất (10 "ưu tiên xử lý" + 2 HIGH workflow) đã verify độc lập + fix — chi tiết [`../roadmap/41-audit-2-fixes.md`](../roadmap/41-audit-2-fixes.md). Phần còn lại (mọi MEDIUM/LOW, phụ lục `apps/*`) **chưa verify/fix** |
| 03 | [`03-metap-core-architecture-audit.md`](03-metap-core-architecture-audit.md) | Kiến trúc: layering, ranh giới crate, doc-vs-reality drift, bề mặt bảo mật multi-tenant, ranh giới `metap`↔`metap-lowcode` (1 agent Opus, 2026-09-02) | **Toàn bộ 14/14 finding đã fix** 2026-09-02 — xem bảng bên dưới |

## Chi tiết audit 03 (đang xử lý)

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
