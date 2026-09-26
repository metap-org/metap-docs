## Phase 95: Migrate nốt backlog REST còn lại (`admin`/`cron`/`dashboards`/`preferences`/`config`/`users`/oauth-clients) sang GraphQL, xoá REST (2026-09-26)

Trigger: sau Phase 91-94 (Federation capability + `platform-ui`/`metap-demo-jira` GraphQL migration),
chủ dự án yêu cầu migrate nốt backlog REST Phase 90 để lại ("tiếp theo migrate nốt") — `metap/
CLAUDE.md`'s danh sách "deliberately not removed in this phase": `/admin/*`, `/cron/*`,
`/dashboards/*`, `/preferences/*`, `/platform/config*`, `/admin/config*`, `/public/config`,
`/users`, `/oauth/*`.

### Phát hiện quan trọng nhất — khác hẳn Phase 90

Research trước khi code phát hiện: **không route nào trong backlog này đi qua
`CrudService`/`MetadataRegistry`** — mỗi nhóm dùng 1 service bespoke riêng (`metap_peripherals`,
`PermissionService`, `metap_cron`, `metap_oauth_server`, `metap_dashboards`, `metap_config`).
Nghĩa là không có GraphQL nào "tự sinh miễn phí" từ schema builder hiện có (khác hẳn entity CRUD
Phase 90) — mọi field phải viết tay. Cũng phát hiện `metap/CLAUDE.md`'s cách liệt kê `/admin/*` và
`/cron/*` là 2 mục riêng là mô tả nhầm — `routes/cron.rs`'s handlers đều mount dưới
`/admin/cron-jobs*`, chỉ tách file cho gọn code, không phải 1 URL prefix riêng.

**`/oauth/*` tách làm 2 loại thật sự khác nhau**: 5/8 route trong `oauth2.rs`
(`/oauth/authorize`, `/oauth/authorize/decision`, `/oauth/token`, `/oauth/revoke`,
`/.well-known/oauth-authorization-server`) là giao thức OAuth2/OIDC thật (RFC 6749/7009/8414),
được gọi trực tiếp bởi OAuth2 client library/browser bên thứ 3 — **không bao giờ có thể thay bằng
GraphQL**, thêm vào danh sách exception REST vĩnh viễn cạnh `/auth/*`/attachments/workflow-events.
Chỉ 3 route admin-CRUD (`/admin/oauth/clients*`) migrate sang GraphQL.

### Dò frontend consumer trước khi xoá REST — khác Phase 90 ở điểm này

Phase 90 xoá REST TRƯỚC khi frontend kịp migrate, gãy 2 app (`platform-ui`, `metap-demo-jira`),
phải dò tìm sửa lại ở Phase 93/94. Lần này dò trước (Explore agent quét toàn bộ `platform-ui`/
`metap-demo-jira/web`/`metap-demo-waf/data-plane/web`/`metap-lowcode`/`metap-control-plane-web`):

| Nhóm | Có consumer? | File |
|---|---|---|
| `/admin/users`, `/admin/users/{id}/roles` | Có | `platform-ui/src/admin/adminApi.ts` + `UsersAdminPage.tsx` |
| `/admin/users/{id}/context/invalidate` | Không | — |
| `/admin/policies`, `/admin/policies/matrix` | Có | `adminApi.ts` + `PoliciesAdminPage.tsx`/`PermissionMatrix.tsx`/`AdvancedPoliciesPanel.tsx`/`PermissionSearch.tsx` |
| `/admin/policies/seed-defaults`, `/admin/policies/explain` | Không | — |
| `/admin/cron-jobs*` | Có | `adminApi.ts` + `CronJobsAdminPage.tsx` |
| `.../runs/{runId}/workflow-run` | Không | — |
| `/admin/oauth/clients*` | Không | — |
| `/dashboards/me`, `/dashboards/tenant-default` | Có | `metap-demo-jira/web`'s `CustomizableDashboardPage.tsx` (app-local) |
| `/preferences` | Có | `platform-ui/src/i18n/LocaleProvider.tsx` (shared) |
| `/platform/config*` | Không | — |
| `/admin/config*` | Có | `metap-demo-waf/data-plane/web`'s `SettingsPage.tsx` (app-local) |
| `/public/config` | Không (kể cả login page) | — |
| `/users` (picker) | Có | `platform-ui/src/auth/useTenantUsers.ts` (shared) |

Mọi call site FE đều qua `apiFetch`/`useApiQuery`/`useApiMutation` — không raw `fetch`.

### Thiết kế backend

**Resolver viết tay đặt ở `metap-graphql-http`, không phải `metap-graphql`** — `metap-graphql`
phải giữ backend-agnostic. `metap-graphql/src/schema.rs`'s `build_schema_parts`/
`build_schema_parts_with_federation` đã có sẵn đúng seam (doc comment tự mô tả: trả về
`(SchemaBuilder, Object query, Object mutation)` trước `.finish()` cho caller có resolver riêng
add thêm field). Module mới `metap-graphql-http/src/platform_fields.rs`:
`add_platform_fields(query, mutation) -> (query, mutation)` — mỗi field `Field::new(...)` dùng lại
đúng service call REST đã dùng (`state.pool`/`state.router`/`state.permissions`/`state.config`/
`state.context_attributes_cache`, tất cả có sẵn qua `AppState` attach làm schema-wide `.data()`
cạnh `backend`, an toàn vì các field này không hot-swap như `crud`/`metadata`).
`SchemaHolder::build` (`lib.rs`) đổi từ gọi `build_schema`/`build_schema_with_federation` sang gọi
`build_schema_parts(_with_federation)` + `add_platform_fields` + tự `.register().finish()`.

**Output là JSON scalar cho mọi field** (không model đầy đủ GraphQL object type cho
`Policy`/`CronJob`/`OAuthClient`/...) — nhất quán với `capabilities`/`filter`/`aggregate` đã dùng
kiểu này trong entity schema. Model đầy đủ là việc làm thêm sau, không bắt buộc cho parity chức
năng — mọi field/mutation gọi đúng service function REST đã gọi và giữ nguyên field
name/casing JSON, nên response cũ và field GraphQL mới parse giống hệt nhau.

**Auth per-field, không phải per-route** — REST mỗi route có 1 extractor riêng
(`AuthContext`/`AdminContext`/`PlatformAdminContext`) chặn trước khi handler chạy; GraphQL chỉ có
1 cổng `AuthContext` chung ở `/graphql`. `require_admin`/`require_platform_admin`
(`platform_fields.rs`) tái tạo đúng check 2 extractor đó (`context.is_admin()`, check
`PLATFORM_TENANT_ID` + role `platform_admin`) ngay trong resolver.

**`publicConfig`** (`/public/config`'s thay thế) sống trên **1 schema riêng, không xác thực**
(`metap-graphql-http::public_schema`, mount `POST /graphql/public`, không dùng `AuthContext`
extractor) — schema này chỉ có đúng 1 field, không có gì khác reachable từ đó cả. Đây là thuộc
tính an toàn quan trọng nhất của thiết kế — không phải guard runtime trên từng field khác.

### Bug thật tìm ra khi verify sống, không phải chỉ vấn đề migration

1. **`ConfigStore::set_tenant` từ chối mọi key `secret`-tier vô điều kiện** (`NotWritable`, "holds
   a credential; write it through the secret path, not as a config value") — orchestration đúng
   (`validate_tenant_secret` → `SecretStore::put_secret` → `set_tenant_secret_marker`, theo đúng
   doc comment của chính `metap-config`) **chưa từng được wire vào
   `routes::tenant_config::set_tenant_config`** — xác nhận qua `git show` trên REST handler đã xoá
   trước khi kết luận đây là bug do migration gây ra. Nghĩa là credential-tier config key
   (`docs/features/18-config-tiers-db-backed.md` slice 3) **chưa bao giờ set được qua HTTP nào cả**
   từ trước tới giờ. Sửa đúng lần đầu trong `setTenantConfig`/`resetTenantConfig`
   (`resetTenantConfig` cũng gọi `SecretStore::delete_secret` khi clear — REST's doc comment hứa
   "revoke, không chỉ unlink" nhưng code cũ chưa từng làm việc này).
2. **2 lỗi `TypeRef` tự viết**: `TypeRef::named_list_nn(T)` = `[T]!` (list bắt buộc, item optional)
   — không phải `[T!]` như tưởng — dùng nhầm cho 4 argument optional-list-of-string (`roles`/
   `actions`/`allowedScopes`), khiến GraphQL validation từ chối request không truyền argument đó.
   Đúng phải là `TypeRef::named_nn_list(T)` = `[T!]` (list optional, item bắt buộc). Phát hiện qua
   e2e thật, không phải code review.

### Verify

`cargo build/clippy(-D warnings)/fmt --check/test --workspace` sạch. E2e thật qua Postgres thật
(`--ignored`): port lại toàn bộ 4 file test REST cũ sang GraphQL thay vì xoá
(`crates/metap-http/tests/{platform_config_postgres,tenant_config_postgres,tenant_secret_postgres,
oauth2_authorization_server_postgres}.rs`, cộng `http_server.rs`'s 2 call site dùng
`/admin/policies`/context-invalidate) — giữ nguyên ý định test cũ, chỉ đổi transport; phát hiện và
sửa luôn 1 assertion đã stale từ trước migration này (shape `secret`/`set`/`secretRef` cũ không
khớp `TenantConfigItemDto` thật từ sau utoipa migration `2dfa9a7`). Thêm file test mới
`crates/metap-graphql-http/tests/platform_fields_postgres.rs` cho 4 nhóm 2 file kia không cover
(users/roles, cron jobs, dashboards, preferences) — 4 test, bao gồm cả path admin-gate 403 và
validation 400. Tổng cộng toàn bộ test liên quan phase này pass lặp lại nhiều lần (idempotent).

**Phát hiện thêm, không liên quan phase này**: `crates/metap-http/tests/http_server.rs`'s
`full_http_lifecycle_over_a_real_server_and_a_real_jwt` fail (delete-rồi-get vẫn trả record thay
vì 404) và `auth_context_entity_enriches_org_scoped_policies...` fail sớm hơn vì `get`/`list`
resolver trong `metap-graphql/src/schema.rs` dùng shape lỗi bare (`"{status}: {message}"`, không
`extensions`) khác với `service_result_to_gql`'s shape đầy đủ mutation dùng — cả 2 file này hoàn
toàn không đụng bởi phase này (`git diff` xác nhận), pre-existing, không sửa ở đây.

### Docs

`metap/CLAUDE.md`: xoá backlog cũ, ghi rõ `/admin/*`+`/cron/*` là 1 group, thêm 5 route OAuth2
protocol vào exception REST vĩnh viễn, mở rộng bullet `metap-graphql`/`metap-graphql-http` mô tả
`platform_fields`/`public_schema`. `metap-oauth-server`'s bullet cập nhật HTTP surface (3 route
admin-CRUD chuyển GraphQL, 5 route protocol ở lại).

### Còn nợ / không làm trong phase này

- Chưa migrate frontend (`platform-ui`/`metap-demo-jira`/`metap-demo-waf`) — làm ngay sau phase
  này trong cùng phiên, xem cập nhật README từng repo.
- Chưa model GraphQL object type đầy đủ cho `Policy`/`CronJob`/`OAuthClient`/... (JSON scalar là
  đủ cho parity chức năng).
- Chưa test qua browser thật (đúng chính sách repo).
- `metap-graphql/src/schema.rs`'s `get`/`list` error-shape thiếu `extensions` — pre-existing, phát
  hiện qua `http_server.rs`, không sửa (ngoài phạm vi phase này).
