## Phase 91: Apollo Federation v2 capability cho `metap-graphql` (opt-in, additive) (2026-09-22)

Trigger: chủ dự án muốn GraphQL là giao thức chuẩn cho toàn bộ ứng dụng (trừ vài endpoint bắt buộc
REST), đồng thời hỏi liệu hướng này có mở đường cho Apollo Federation khi `metap-lowcode` tách
thành microservices thật hay không.

### Phát hiện kỹ thuật (verify trực tiếp trong source đã vendor local)

`async-graphql` v7.2.1 (đúng version workspace đang pin) — cụ thể **module `dynamic`**, chính là
module `metap-graphql` đang dùng để build schema runtime từ `MetadataRegistry` — **đã hỗ trợ
Federation v2 sẵn**, không cần đổi cách build schema, không cần cargo feature riêng:
- `Object::key(fields)` — đánh dấu `@key`.
- `SchemaBuilder::entity_resolver(fn)` — resolver cho `_entities(representations: [_Any!]!)`.

### Thiết kế (additive, không đổi hành vi caller hiện có)

`crates/metap-graphql/src/schema.rs`: `build_schema_parts`/`build_schema` giữ nguyên 100% (mọi
caller hiện tại — `metap-graphql-http` mặc định, `metap-graphql-gateway`, test suite — SDL/hành vi
không đổi 1 bit, có test regression xác nhận). Thêm 2 hàm public mới,
`build_schema_parts_with_federation`/`build_schema_with_federation`, gọi chung 1 hàm private
`build_schema_parts_inner(..., federation: bool)` — cùng pattern additive đã dùng nhiều lần trong
repo này (`reconcile`/`reconcile_with_scope`, `find_oidc_user`/`find_external_user`).

Khi `federation == true`: mỗi entity `Object` được thêm `.key("id")` (mọi entity đều có `id` UUID
trong `RecordDto` envelope — khoá tự nhiên sẵn có, không cần cấu hình per-entity), và schema được
đăng ký `entity_resolver` — resolver này gọi **đúng** `RecordBackend::get(entity_name, id,
context)` y hệt resolver `Query.{camel}(id)` đã có, nên thừa hưởng nguyên vẹn permission/
tenant-scoping, không có code path riêng nào bypass được.

`crates/metap-graphql-http/src/lib.rs`: thêm `router_with_federation(state, limits, federation:
bool)`, `router(state, limits)` hiện tại trở thành wrapper gọi với `federation: false`. Crate này
không tự đọc env var — binary nào muốn bật tự đọc `metap_runtime::env::flag_enabled
("GRAPHQL_FEDERATION_ENABLED")` trong `main.rs` của chính nó rồi truyền vào (cùng shape
`GRPC_ENABLED`). **Không service nào trong tổ chức này gọi với `true` — capability đã có, chưa ai
bật.**

`crates/metap-graphql-gateway` (BFF hiện tại, stitch bằng cách gọi REST `/metadata/entities` mỗi
upstream) **hoàn toàn không đụng tới** — quyết định có chủ đích, không phải bỏ sót. Federation ở
cấp subgraph chỉ mở ra lựa chọn *sau này*: thay composite gateway tự viết bằng 1 router chuẩn
Federation (Apollo Router hoặc tương đương) khi `metap-lowcode` thật sự tách microservices — đó là
quyết định kiến trúc riêng, chưa làm ở đây.

### Phát hiện thêm lúc verify sống: giới hạn thật của `async-graphql`'s union resolution

Thiết kế ban đầu định cho 1 representation không resolve được (typename lạ, id sai, hoặc bị từ
chối quyền) trả về `null` đúng vị trí trong list `_entities` (đúng tinh thần spec Federation cho
phép partial result). **Verify sống lộ ra: không làm được theo cách đó** — union output type
(`_Entity`) trong `async-graphql`'s dynamic module bắt buộc mọi item phải là `FieldValue::WithType`;
`FieldValue::NULL` trần (không kèm type) hit thẳng `resolve_list`'s `try_join_all` fail-fast, làm
**lỗi cả câu `_entities`**, không chỉ null đúng 1 vị trí — không có per-item-null path cho union
trong version này. Đã sửa thiết kế: representation không resolve được trả **lỗi GraphQL thật**
(tái dùng `service_result_to_gql` đã có sẵn trong file, cùng shape `extensions.code`/`status`/
`fieldErrors` mọi resolver khác đã dùng) thay vì giả vờ null — dữ liệu không bao giờ rò rỉ, chỉ
khác ở chỗ lỗi thay vì null im lặng. Ghi lại trực tiếp trong code comment tại
`crates/metap-graphql/src/schema.rs`'s `entity_resolver` để không ai lặp lại giả định sai này.

### Verify

`cargo build/clippy(-D warnings)/fmt --check --workspace` sạch (`metap-graphql-gateway` build sạch
không cần sửa gì, xác nhận API cũ không đổi). `cargo test --workspace` sạch (0 fail). E2e mới
(`crates/metap-graphql/tests/graphql_schema_postgres.rs`, `--ignored`, Postgres thật):
- `non_federated_schema_has_no_federation_types` — `build_schema` (hàm cũ) không có `_entities`/
  `@key` trong SDL, chứng minh mặc định không đổi.
- `federated_schema_exposes_key_directive_and_resolves_entities_through_normal_permission_checks`
  — SDL phẳng (`schema.sdl()`, cái `GET /graphql/schema.graphql` trả) có field `_entities`/
  `_service` nhưng **không** có `@key` (chỉ field `_service.sdl` — SDL kiểu Federation thật, cái
  router Federation thật sự query — mới có `@key`, verify cả 2 dạng); admin query `_entities` trả
  đúng record thật; role không có quyền đọc bị từ chối bằng lỗi GraphQL, không lộ dữ liệu.

### Còn nợ / không làm trong phase này

- Chưa bật `GRAPHQL_FEDERATION_ENABLED` ở bất kỳ `main.rs` nào (`metap-demo-waf`,
  `metap-demo-jira`, `metap-lowcode`, `templates/metap-app`) — chờ trigger thật (thời điểm
  `metap-lowcode` thật sự tách microservices).
- Chưa giới thiệu Apollo Router hay bất kỳ Federation-router nào.
- Chuyển các nhóm route REST còn lại (`/admin/*`, `/cron/*`, `/dashboards/*`, `/preferences/*`,
  `/platform/config*`, `/admin/config*`, `/public/config`, `/users`, `/oauth/*`) sang GraphQL —
  phạm vi riêng, plan khác, phiên sau. `attachments` (upload/download nhị phân) giữ REST vĩnh viễn
  (quyết định đã chốt cùng phiên này) — thêm vào danh sách exception cố định cùng `/auth/*`/
  webhook-receiving endpoints.
