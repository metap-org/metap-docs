# Roadmap — Index nhanh (pending / done / cần quyết định)

[← Roadmap đầy đủ](../roadmap.md) (narrative + verify sống từng phase) · [← Architecture](../architectures/index.md)

Trang này là bản **quét nhanh theo trạng thái** — bổ sung cho `../roadmap.md` (đã có bảng đầy đủ
theo *thứ tự phase*, không phân nhóm theo trạng thái). Cột "Ghi chú" ở bảng dưới là trích nguyên
văn (rút gọn) từ chính status text của `../roadmap.md` — không viết lại nội dung lịch sử, chỉ sắp
xếp lại theo nhóm để quét nhanh hơn. Muốn bối cảnh/quyết định/verify đầy đủ của 1 phase, luôn vào
file phase đó (hoặc `../roadmap.md`), đừng dừng lại ở ghi chú rút gọn này.

Phân loại **Done-partial** dựa trên tín hiệu trực tiếp trong status text ("Đang làm"/"In-progress"/
"Làm một phần"/"phần lớn"/"chưa cut over"/"chưa có trigger" đứng cạnh phần đã xong) — với các phase
mà status text dài, có thể có nhiều chi tiết hơn không thể hiện hết trong 140 ký tự rút gọn; nếu
nghi ngờ 1 phase nào đó bị xếp sai nhóm, coi file phase đó là nguồn thật, không phải bảng này.

## 🔴 Cần quyết định kiến trúc quan trọng (chưa chốt, có thật — không phải backlog thường)

Khác với 3 bảng bên dưới (theo dõi *phase đã hoặc đang code*), mục này liệt kê các điểm cần
**quyết định sản phẩm/kiến trúc trước khi code** — chưa có phase nào vì chưa có trigger:

| Việc | Vì sao cần quyết định trước, không code thẳng | Nguồn |
|---|---|---|
| **Phase 9 — Multi-Service Evolution** (khi nào tách 1 module thành service/DB riêng) | Đã rà soát lại 2026-08-17, xác nhận chưa trigger nào xảy ra — nhưng đây là quyết định lớn (tách DB, chọn protocol service-to-service) cần chốt hướng trước khi bất kỳ module nào thực sự tách | [09-multi-service-evolution.md](09-multi-service-evolution.md), [04. Solution Strategy](../architectures/04-strategy/00-index.md) |
| Workflow 2 chế độ (in-process + cross-module) | Ảnh hưởng trực tiếp tới `metap-workflow`'s public API — đổi sau khi đã có consumer sẽ là breaking change | `docs/team-charter.md`'s "Định hướng đang ghi nhận, chưa có trigger" |
| Tiny deployment profile (single binary, không RabbitMQ) | Quyết định sản phẩm (có bán/hỗ trợ deployment mode này không) trước khi đụng `metap-infra`/`EventBus` | như trên |
| Migration path generic-table → bảng riêng (Data Model Strategy Step 3) | Ảnh hưởng schema + `QueryPlanner`/`CrudService` của mọi entity đã có | như trên, [05. Building Block View — Data Model](../architectures/05-building-blocks/04-data-model.md#data-model) |
| Schema versioning cho entity | Ảnh hưởng `MetadataCompiler`/hash-drift-check hiện có | như trên |
| Entity variant kiểu polymorphic/discriminated-union | Tự đánh giá là **rủi ro cao nhất trong backlog** (`docs/features/16-entity-variant-polymorphic.md`) | như trên |
| Metadata low-code theo từng Tenant | Hướng dài hạn cho "quy tắc cô lập schema cấp Tenant" (Phase 11C, 2026-08-22) — chưa chốt mô hình isolation | như trên |

**Lưu ý stale đã phát hiện khi soát lại (2026-09-04)**: `../roadmap.md`'s đoạn "Định hướng chưa
lên phase" liệt kê **7** ý ở trên bao gồm cả "computed/derived field" — nhưng
`docs/features/13-computed-derived-field.md` đã **done (2026-09-02)** theo chính
`docs/features/00-index.md`. Mục đó đã bỏ khỏi bảng trên (còn 6, không phải 7) — `../roadmap.md`
chưa được cập nhật lại đoạn đó, để nguyên (không sửa nội dung lịch sử của nó ở đây), chỉ không lặp
lại thông tin sai ở trang index này.

---

### ✅ Done (59)

| Phase | Ghi chú (trích nguyên văn từ `../roadmap.md`) |
|---|---|
| [0. Skeleton](00-skeleton.md) | Đã xong |
| [1. Production-shaped Platform Kernel](01-production-kernel.md) | Đã xong |
| [2. Metadata Compiler](02-metadata-compiler.md) | Đã xong |
| [3. Permission Engine](03-permission-engine.md) | Đã xong |
| [4. Query Planner V1](04-query-planner-v1.md) | Đã xong |
| [5. Workflow Engine V1](05-workflow-engine-v1.md) | Đã xong |
| [6. Frontend Core](06-frontend-core.md) | Đã xong (chưa verify trên browser) |
| [7. Module Migration Strategy](07-module-migration-strategy.md) | 4/4 module (crm.customers, sales.orders, inventory.movements, accounting.journal) |
| [11. Low-code Platform Backbone Architecture](11-lowcode-platform-backbone.md) | metadata audit log, migration-impact check, import/export, operational visibility (cross-entity audit feed) xong (2026-08-22); 2026-08-27:… |
| [13. Dynamic Cron Jobs](13-dynamic-cron-jobs.md) | Backend đã xong; admin UI đã xong (Phase 15) |
| [15. Shared App Shell (UI kit, real login, permission-aware components)](15-shared-app-shell.md) | Đã xong |
| [17. Metadata-driven Workflow Engine](17-metadata-workflow-engine.md) | chuỗi activity tuần tự, bảng `workflow_runs`) xong 2026-08-28; Increment 3 (`wait_event` — durable pause chờ 1 domain event khớp pattern,… |
| [18. Organization & Identity — P0](18-organization-identity-p0.md) | `RequestContext.context_attributes` (opt-in `AUTH_CONTEXT_ENTITY`, cache + invalidate endpoint), entity mẫu `hr.departments`/`hr.employees`… |
| [19. Table-per-entity](19-table-per-entity.md) | cả 4 entity code-authored của `crm-server` giờ đều dùng bảng riêng. `apps/jira-server` (Phase 21) vẫn là app demo khác dùng bảng riêng từ… |
| [22. `metap-storage` (object storage)](22-metap-storage.md) | `ObjectStore`/`S3ObjectStore` (SeaweedFS backend), wire thật ở Phase 27 |
| [23. `metap-cache` (caching layer)](23-metap-cache.md) | `Cache`/`MokaCache`/`RedisCache` (DragonflyDB), consumer đầu tiên: `PermissionService::with_cache` (policy-row cache) |
| [25. Tenant auth pluggable (Bearer + Basic + OIDC)](25-tenant-auth-pluggable.md) | cả 3/3 bước (crate `metap-auth`, bảng `tenant_auth_configs`, refactor local login; HTTP Basic per-request; OIDC redirect/callback + JIT… |
| [27. Generalize attachment thành năng lực platform (`metap-http`)](27-attachments-platform.md) | crate `metap-attachments` (2 cơ chế: bảng chung + bảng riêng theo entity), route generic `/api/{entity}/{id}/attachments*`, xoá bespoke cũ… |
| [28. `dev-tools` tenant-aware](28-dev-tools-tenant-aware.md) | `seed-admin`/`create-user` giờ đi qua `Router` (không còn ghi nhầm DB cho tenant `dedicated_db`), `mint-token` cảnh báo rõ khi default… |
| [33. Customizable dashboard — layout, widget catalog, org default](33-customizable-dashboard.md) | crate `metap-dashboards` (bảng `dashboard_configs`, 2 cấp: per-user override + per-tenant default admin-only), route generic `GET/PUT… |
| [34. Backend test kit — đóng nốt Semgrep CI + criterion nightly](34-backend-test-kit-completion.md) | Semgrep wired vào CI (blocking, `# nosemgrep` inline suppress finding false-positive duy nhất), criterion micro-benchmark cho… |
| [35. `metap-lowcode-http` Router-aware](35-lowcode-http-router-aware.md) | mọi handler resolve `Router::pool_for(tenant)` thay vì đọc thẳng `state.pool` (trước đây `DedicatedDb` tenant sẽ ghi nhầm entity low-code… |
| [37. EventBus — reconnect-with-backoff cho subscribe](37-eventbus-resilient-subscribe.md) | `metap_infra::run_resilient_consumer` (hàm mới) thay hết shape "bail khi mất kết nối, chờ process manager restart" ở cả 3 consumer thật… |
| [38. `metap-cron` — generic record-event triggers](38-generic-record-event-triggers.md) | trả lời câu hỏi "handler RabbitMQ dynamic/customize kiểu gì": cơ chế đã có sẵn (`on_transition` trigger, Phase 17), chỉ mở rộng thêm… |
| [39. Retry-with-backoff per-message + `TargetType::Email`](39-retry-with-backoff-and-email-target.md) | `metap_infra::RetryPolicy`/`ConsumedEvent::retry_or_give_up` (TTL+DLX delay-queue pattern, đọc `x-death` cho retry count) dùng đầu tiên ở… |
| [40. `HandlerRegistry` + fix bug mất `routing_key` khi retry](40-handler-registry-and-retry-routing-key-fix.md) | đóng nốt việc cuối trong list Phase 37 ("handler registry chung"): 1 process tự đăng ký nhiều handler tùy biến trên 1 subscription… |
| [41. Fix 12 bug từ AUDIT_2.md](41-audit-2-fixes.md) | audit toàn bộ `crates/` do 6 agent viết, 1 fork verify độc lập 12 finding nghiêm trọng nhất trước khi sửa (12/12 CONFIRMED). Bảo mật: SQL… |
| [42. Workflow engine — transition payload, `validator`, `set_fields`](42-workflow-engine-payload-validator-set-fields.md) | mô hình Jira-style condition/validator/post-function cho `WorkflowTransition`: transition giờ nhận `data` payload thật (trước chỉ có… |
| [43. Entity field validation — `min`/`max`, `minLength`/`maxLength`](43-field-min-max-length-validation.md) | `EntityField` thêm 4 field optional (`min`/`max` cho `Number`/`Money`, `minLength`/`maxLength` cho `String`), gap đã ghi nhận từ lâu trong… |
| [44. `reconciler-orchestrator` — chạy orchestrator thật lần đầu](44-reconciler-orchestrator-service.md) | crate mới `metap-reconciler-orchestrator` (binary `reconciler-orchestrator`), ticker thật đầu tiên chạy `metap-reconciler::orchestrator`'s… |
| [46. `rust-e2e` ra khỏi CI tự động + sửa gốc 2 test flaky](46-e2e-manual-and-flaky-test-fixes.md) | trigger: PR #6 (Workflow Engine Increment 2) hit CI đỏ 3 lần liên tiếp ở `rust-e2e`, mỗi lần một lỗi flaky khác nhau không liên quan diff,… |
| [47. Tách frontend library ra repo riêng — `@metap/platform-ui` + `@metap/ui`, gỡ Mantine khỏi `apps/crm-fe`/`apps/jira-fe`](47-platform-ui-design-system-split.md) | `packages/platform-react` (Mantine) move 100% sang repo riêng `../platform-ui` (`@metap/platform-ui`), build lại toàn bộ UI trên… |
| [48. Redesign UI phân quyền — ma trận RBAC (cơ bản) + condition builder ABAC (nâng cao)](48-permission-admin-ui-rbac-matrix-abac-builder.md) | plan trước khi code (`EnterPlanMode`). FE: type `PolicyCondition` thật (`admin/policyCondition.ts`, trước là `unknown`), ma trận… |
| [49. Nền tảng GraphQL + gRPC cho metap — chuẩn bị cho các microservice WAAP](49-graphql-grpc-jwks-foundation.md) | trigger thật: chủ dự án sắp xây "WAAP" (nhiều microservice thật, không phải demo) — đúng 2 trigger GraphQL BFF/gRPC đã ghi ở Phase 9 nhưng… |
| [50. BFF gateway thật xuyên microservice (`graphql-gateway`) — wire GraphQL/gRPC thật vào jira-server/crm-server](50-graphql-gateway-real-bff.md) | trigger: sau khi wire GraphQL (Phase 49's thư viện) vào `jira-server` lần đầu, chủ dự án chỉ ra đúng đây chưa phải BFF thật (1 service tự… |
| [51. Tách `apps/crm-server`/`crm-fe`/`jira-server`/`jira-fe` ra 2 repo riêng](51-example-apps-repo-split.md) | đúng việc Phase 47 để lại "cố ý chưa làm". 2 repo mới `../metap-demo-crm`/`../metap-demo-jira` (cấu trúc phẳng, không có tầng `data-plane/`… |
| [52. Tách tầng SaaS low-code control-plane ra repo riêng `metap-lowcode`](52-split-lowcode-saas-crates.md) | phân tích trước (`docs/features/07-split-lowcode-saas-crates.md`), lên plan qua `EnterPlanMode` rồi mới code. 4 crate di chuyển nguyên… |
| [54. Tách `docs/` ra repo riêng `metap-docs`](54-docs-repo-split.md) | copy nguyên trạng 76 file tracked (`git ls-files docs`) từ `metap` sang `../metap-docs`, giữ `docs/` làm subfolder (không flatten root) nên… |
| [55. `[workspace.dependencies]` cho dependency dùng chung](55-workspace-dependencies-centralization.md) | grep thật toàn bộ `crates/*/Cargo.toml` (không đoán) tìm 21 dependency được >= 2 crate dùng… |
| [56. `metap-runtime` trở thành SDK viết backend tuỳ biến + crate mới `metap-app`](56-metap-runtime-sdk-and-metap-app.md) | vòng rà soát thứ 4 cho `metap-runtime` (sau 3 vòng Phase 53): thêm `rate_limit`/`trace` (kéo khỏi `metap-http::build_router`, đóng gap… |
| [57. Chuẩn bị `metap-runtime`/`metap-grpc` cho tích hợp Istio/Linkerd — W3C Trace Context](57-istio-linkerd-trace-context.md) | trigger: chủ dự án hỏi "http, grpc hỗ trợ các header của istio/linkerd chưa", xác nhận mục tiêu "muốn tích hợp được Istio trong tương lai,… |
| [58. `submit_entity!`/`register_all_submitted()` — entity tự đăng ký qua `inventory`](58-entity-auto-registration.md) | entry ghi lại lịch sử, viết sau khi rà soát phát hiện gap tài liệu (code đã commit ở cả 4 repo trước khi entry này tồn tại).… |
| [59. Rà soát atom lạc chỗ ở `platform-ui` + thêm `FileUpload`](59-design-system-atom-audit.md) | chưa có người review nào ở `design-system` tính tới thời điểm này, quyết định cứ tiếp tục build thay vì chờ. Rà soát chủ động tìm tiếp cùng… |
| [60. `GeneratedList`/`RecordDetail` — sửa layout, thêm back-link, tận dụng diện tích màn hình](60-generated-list-detail-layout-fixes.md) | trigger: phản hồi trực tiếp sau khi dùng UI thật ("list xấu, detail không có back, diện tích không tối ưu"). Bug alignment cột thật tìm… |
| [62. `RelatedView` metadata + `RelatedRecordsPanel` — thay `ZoneOverviewPage` viết tay bằng khai báo](62-related-view-metadata.md) | trigger: chủ dự án hỏi thẳng "query không tự parse từ entity à" sau khi thấy `ZoneOverviewPage` viết tay ở Phase 61. Research trước khi lên… |
| [63. Audit `platform-ui` (login/phân quyền + `WorkflowDiagram`) và sửa 20 finding](63-platform-ui-audit-02-fixes.md) | trigger: chủ dự án dùng thử `WorkflowDiagram` trên browser thật, báo "mũi tên bị ghi đè rất xấu", kèm yêu cầu audit rộng hơn phần… |
| [64. Session sống qua reload — cookie `HttpOnly` thay JWT-trong-React-state](64-cookie-session-persistence.md) | trigger: chủ dự án hỏi thẳng "jwt/auth session đang k lưu ở đâu, reload mất, có cách nào xử k", đảo ngược một quyết định đã ghi rõ trong… |
| [65. Sửa 5 finding audit 04 (SSRF, CSRF token endpoint, error fidelity, coverage, trace)](65-audit-04-fixes.md) | trigger: audit 04 ra 17 finding, chủ dự án chọn xử lý theo lô (A#1+B#4+B#5, rồi B#2+A#4). A#1 (HIGH): `target_config` của cron job… |
| [66. `metap-config` — tunable nền tảng phân tầng, lưu trong Postgres](66-platform-config-tiers.md) | trigger: chủ dự án hỏi "config như này có thể set trong database, API admin/config, per tenant" khi đang vá audit 04 A#1. Crate mới… |
| [67. Tầng config per-tenant + bề mặt branding trước khi login](67-tenant-config-tier-public-theme.md) | bảng `tenant_configs`, `GET/PUT/DELETE /admin/config` (tenant lấy từ token, không bao giờ từ request), và `GET /public/config` không auth… |
| [68. Secret cho webhook — `SecretStore` tổng quát, config write-only, nới `FORBIDDEN_HEADERS`](68-webhook-secret-tier.md) | lát brief tự cảnh báo là dễ sai nhất vì nó sửa đúng guard audit 04 A#1 vừa đặt cùng ngày. `SecretStore` thêm… |
| [69. Dev tooling (docker-compose/mold) + fix phase 64's downstream fallout](69-dev-tooling-and-cookie-session-downstream-fixes.md) | trigger: câu hỏi thuần dev-tooling ("chưa có docker compose dev, rust build đỡ nặng k") lộ ra Phase 64 (cookie session) chưa lan xuống… |
| [70. Aggregation API (`plan_aggregate`)](70-aggregate-api.md) | `GROUP BY`/`COUNT`/`SUM` cho mọi entity — core trước đó không có khả năng aggregate nào; build/clippy/test verify xong 2026-09-04 |
| [71. WAF Customer Portal thật + endpoint "phải tự code"](71-waf-admin-portal.md) | portal 10 module theo IA zone-centric + verify-dns/test-origin/sync-config-state/delete-guard/correlate/alert/scan; build/test verify xong 2026-09-04 (48 lỗi tsc thật đã sửa) |
| [72. `control-plane` + `edge-plane`](72-control-edge-planes.md) | phần chặn request thật lần đầu có code: config distributor (subscribe + resync + ingest) và mitigation engine (hyper trần, không `metap`); build/clippy/test verify xong 2026-09-04, chưa chạy qua Postgres/Redis/RabbitMQ thật |
| [73. `metap-demo-waf`'s FE→BE giao thức chuẩn hoá về GraphQL](73-waf-graphql-protocol.md) | `data-plane/web` từ 100% REST sang gọi `waf-graphql-gateway`'s `/graphql` — CRUD generic + 8 field custom (verify DNS/scan/alert/`aggregate`), trừ `/auth/*`/`/preferences/*`/`/metadata/*`; extension point mới ở `metap` core (`build_schema_parts`) cho phép downstream thêm field ngoài CRUD; `tsc/oxlint/prettier/vite build` sạch, chưa browser-test |
| [74. JWKS (Ed25519) — trust root nhiều service](74-jwks-multi-service-trust-root.md) | `metap-demo-waf` chuyển từ share 1 file RSA keypair sang share 1 EdDSA key qua `metap-jwks`, rotation 3 bước; `metap-http`/`graphql-gateway` opt-in additive (RSA tĩnh vẫn fallback mặc định); cookie-auth opt-in cho `graphql-gateway` (bỏ round-trip `/auth/token`) |
| [75. `aggregate` lên thành capability generic của `RecordBackend`](75-aggregate-generic-record-backend.md) | Phase 70 chỉ REST — thêm `RecordBackend::aggregate` cho gRPC + GraphQL đơn-service, RPC `Aggregate` mới trong proto, `metap_query::AggregateSpec` gom parse logic dùng chung 3 transport |
| [76. `metap-demo-waf` — loạt bug thật lộ ra khi chạy portal sống lần đầu](76-waf-portal-live-bugfixes.md) | Bug nặng nhất: docker-compose chạy nhầm binary `graphql-gateway` generic thay vì `waf-graphql-gateway`; cộng race login/logout, render-storm Dashboard, `/metadata/entities` rỗng, zone-delete-guard thiếu env var, `AppShellLayout` mount lại mỗi navigation; thêm "Visualize workflow" SVG vào Zone/Incident detail |
| [77. `metap-demo-waf` — gộp GraphQL request, workflow diagram tương tác, bản dịch tiếng Việt](77-graphql-batching-workflow-diagram-i18n.md) | `graphql-gateway` chuyển sang `GraphQLBatchRequest` (backward-compatible), client gộp query cùng tick thành 1 request; `WorkflowDiagram` thêm zoom/pan/kéo-node/highlight hover; WAF thêm i18n en/vi đầy đủ + `LocaleSwitcher` mount vào `AppShellLayout` cho mọi app |

### 🟡 Done-partial / in-progress (8)

| Phase | Ghi chú (trích nguyên văn từ `../roadmap.md`) |
|---|---|
| [8. Hardening](08-hardening.md) | 2026-08-28: thêm `AwsSecretsManagerStore`/`GcpSecretManagerStore` (2 impl `SecretStore` mới cạnh `VaultStore`/`EnvStore`, cùng shape… |
| [10. Monorepo, npm publish](10-monorepo-npm-publish.md) | Làm một phần |
| [12. Rust Core Migration](12-rust-core-migration.md) | Đã quyết định; Migration Order (bước 1-9) đã xong trong `crates/`; chưa cut over sang production |
| [14. Multi-language (i18n)](14-i18n.md) | chưa có trigger |
| [16. Multi-tenant SaaS Control Plane & Data Plane](16-multi-tenant-saas.md) | 2026-08-16 → 2026-08-17); Giai đoạn 4: `VaultStore` (token) xong 2026-08-17, AppRole auth + auto-renewal + role lookup/RBAC qua Router… |
| [20. Backend test kit (regression/performance/security)](20-backend-test-kit.md) | security (`cargo audit`+CI, tenant-isolation/JWT/RBAC-ABAC test, CodeQL+Semgrep) và performance (k6 qua Docker + Grafana) xong; 2026-08-24… |
| [53. Đổi tên `metap-lowcode-platform`→`metap-lowcode`, thêm crate `metap-runtime`, mono-repo microservices cho `metap-lowcode`](53-metap-contrib-and-lowcode-microservices-plan.md) | đổi tên repo xong (đúng tên chủ dự án chọn), verify build sạch cả `metap-lowcode`+`metap-demo-crm`. `metap-runtime`… |
| [61. `metap-demo-waf/data-plane` — tách 3 microservice theo pillar + GraphQL gateway](61-waf-microservices-split.md) | trigger: chủ dự án chốt hướng "metap-waf thiết kế microservice, GraphQL gọi xuyên nhiều service". `EnterPlanMode` trước khi code, 2 vòng… |

### ⚪ Pending (trigger-based, chưa có trigger) (1)

| Phase | Ghi chú (trích nguyên văn từ `../roadmap.md`) |
|---|---|
| [9. Multi-Service Evolution](09-multi-service-evolution.md) | vẫn chưa trigger nào xảy ra, không có việc để làm |

