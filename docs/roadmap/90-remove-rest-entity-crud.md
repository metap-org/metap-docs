## Phase 90: bỏ REST entity CRUD ở `metap` core, GraphQL-only (2026-09-21)

Trigger: chủ dự án — "Hiện tại t muốn bỏ hết giao thức http cho router, chỉ dùng graphql thôi,
http dùng cho webhook, để đỡ phức tạp hoá application", "Phần auth có thể giữ rest nhé, còn lại
luôn luôn graphql, sao này có thể add federation gateway cho micoservice sẽ hợp lý hơn là viết
federation tay như này".

### Khảo sát trước khi làm: nhóm nào là "structural", không đụng được ngay

Trước khi sửa, rà toàn bộ 16 route group của `metap-http::build_router` để tách "entity CRUD generic
(GraphQL đã cover đủ)" khỏi "structural — REST vì lý do giao thức, không phải tiện lợi":

- **OAuth2 Authorization Server** (`/oauth/authorize`/`/token`/`/revoke`,
  `/.well-known/oauth-authorization-server`) — RFC 6749/7009 tự nó là giao thức HTTP, không có khái
  niệm "GraphQL OAuth2 server".
- **JWKS/OAuth discovery** (`/.well-known/*`) — cùng lý do, well-known URI là giao thức HTTP chuẩn.
- **`/health`/`/metrics`** — health-check/scrape convention của hạ tầng (load balancer, Prometheus),
  không phải application API.
- **Attachment upload/download** — binary file, GraphQL không có primitive truyền file thô hợp lý
  (multipart hoặc pre-signed URL vẫn là REST/HTTP thuần).
- **`/admin/*`, `/cron/*`, `/dashboards/*`, `/preferences/*`, `/platform/config*`,
  `/admin/config*`, `/public/config`, `/users`, workflow-events** — chưa có tương đương GraphQL, cần
  xây riêng, ngoài phạm vi phiên này.

Chốt qua trao đổi với chủ dự án: **giữ nguyên nhóm structural ở trên, làm trước phần (a)** — bỏ
REST entity CRUD generic (`routes::records`, `/api/:entity*`) trong `metap` core, vì GraphQL
(`metap-graphql`/`metap-graphql-http`) đã có parity đầy đủ từ Phase 49/50, không cần xây gì mới
trước. Phần xây GraphQL parity cho nhóm admin/cron/dashboards/... rồi bỏ REST của chúng là phase
sau, chưa bắt đầu.

### Việc đã làm (`metap` core)

- Xoá thẳng `crates/metap-http/src/routes/records.rs` (`/api/:entity*` —
  list/get/create/update/delete/transition/aggregate) — không deprecate dần, xoá file luôn vì
  GraphQL đã cover đủ 7 thao tác này từ trước.
- `metap_metadata::generate_openapi_document` bỏ tham số `entities: &[EntitySummary]`, không còn
  sinh path `/api/{entity}*` per-entity nữa — `GET /metadata/openapi.json` giờ chỉ còn 3 path tĩnh
  (`/metadata/entities`, `/metadata/entities/{entity}`, `/metadata/actions`). `GET
  /graphql/schema.graphql` (đã có từ trước, xem "Metadata types stay generated" trong
  `metap/CLAUDE.md`) là điểm discovery schema tương đương cho GraphQL.
- **Phát hiện 1 dependent REST nội bộ thật trong chính repo `metap`**: `crates/metap-cron-scheduler`'s
  `workflow_transition`/`bulk_query_action` job executor gọi thẳng `/api/:entity/...` của một binary
  downstream qua HTTP — nếu không sửa, xoá `routes::records` sẽ làm cron-scheduler gãy ngay khi có
  job kiểu này chạy (không phải lỗi biên dịch, lỗi runtime lúc gọi 404). Migrate sang
  `metap-grpc::client::GrpcBackend` (transport nội bộ service-to-service `metap` đã có sẵn từ Phase
  49/50, không xây gì mới) — `ExecutorConfig.target_grpc_backend: Option<Arc<dyn RecordBackend>>`,
  cấu hình qua `CRON_TARGET_GRPC_ADDR`; thiếu hoặc không kết nối được thì degrade per-job (job đó
  fail, log rõ), không refuse boot cả process — cùng nguyên tắc executor này đã áp dụng cho các
  credential optional khác (`CRON_LOGIN_URL`/...).
- Viết lại 3 file e2e test của `metap-http` (`http_server.rs`, `jwt_security_postgres.rs`,
  `cookie_session_postgres.rs`) để đi qua `/graphql` (mount kiểu downstream binary thật sẽ mount —
  qua `build_router`'s `extra_routes`, `metap-graphql-http` giờ là dev-dependency của `metap-http`,
  an toàn vì dev-dependency không tạo cycle build thật) thay cho route REST đã xoá. 2 route dùng để
  thay thế test cookie/CSRF không liên quan entity CRUD (`GET /auth/me`) vì `requires_csrf_check` xét
  theo HTTP method (GET/HEAD/OPTIONS an toàn), mà GraphQL luôn đi qua POST — không thể dùng `/graphql`
  để test nhánh "GET không cần CSRF" nữa, phải chọn 1 route REST còn tồn tại thật sự là GET.
- 1 test bị xoá hẳn thay vì port: `updating_the_auth_context_entity_record_via_patch_invalidates_the_cache_automatically`
  — chủ đề của nó (tự động invalidate `ContextAttributesCache` khi PATCH qua REST) không còn tồn tại
  ở đâu trong codebase sau khi xoá `routes::records`, xem mục "khoảng hở phát hiện" dưới.

### Khoảng hở phát hiện, không phải do phiên này tạo ra — chỉ không còn cách né

`invalidate_context_cache_if_auth_context_entity` (tự động invalidate `ContextAttributesCache` khi
một bản ghi entity được cấu hình làm `AUTH_CONTEXT_ENTITY` bị update) **chưa bao giờ tồn tại ở đâu
ngoài `routes::records::update_record`** — `metap-graphql`'s mutation `update{Type}` và
`metap-grpc`'s RPC `update` chưa từng gọi hàm này. Trước phiên này, REST vẫn còn tồn tại song song
nên khoảng hở chỉ lộ ra khi ai đó update qua GraphQL/gRPC; giờ REST bị xoá, đường update duy nhất
còn lại (GraphQL) **luôn** đi qua khoảng hở này — cache có thể stale tới khi hết TTL hoặc gọi tay
`POST /admin/users/{userId}/context/invalidate`. Không sửa trong phiên này — fix đúng nghĩa cần thêm
side effect này vào `CrudService`/`RecordBackend::update` (mọi call site ở mọi repo downstream), quy
mô lớn hơn phạm vi "chỉ xoá REST" của phase này. Ghi lại ở `metap/CLAUDE.md`'s `metap-http` bullet,
chưa mở issue riêng — chủ dự án tự quyết định độ ưu tiên.

### Còn nợ (phase sau, chưa bắt đầu)

Xây GraphQL parity rồi bỏ REST cho: `/admin/*` (users/policies), `/cron/*`, `/dashboards/*`,
`/preferences/*`, `/platform/config*`/`/admin/config*`/`/public/config`, `/users`, `/oauth/*`
(giữ REST vĩnh viễn — giao thức, xem trên), attachments (giữ REST vĩnh viễn — binary), workflow-events.
Khoảng hở `invalidate_context_cache_if_auth_context_entity` ở trên cũng chưa sửa.

### Verify

`cargo build --workspace`, `cargo clippy --workspace --all-targets -- -D warnings`, `cargo test
--workspace` (unit, không cần DB) đều sạch trên `metap`. Chưa chạy `-- --ignored` (e2e thật, cần
Postgres) trong phiên này — môi trường phiên không có Postgres sẵn.
