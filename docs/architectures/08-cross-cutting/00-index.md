# 8. Cross-cutting Concepts

Các pattern và nguyên tắc áp dụng xuyên suốt nhiều building block, không thuộc sở hữu riêng của bất kỳ khối nào.

## Metadata-Driven Development

Field, list view, validation schema, workflow, và index/search hint của mọi entity đều được khai báo một lần duy nhất (`EntityDefinition`) rồi được biên dịch/kiểm tra hợp lệ như một artifact runtime (`MetadataCompiler`), thay vì được xem như config thụ động. Xem [05. Building Block View](../05-building-blocks/00-index.md).

## Transactional Outbox

Một business write và (các) event nó sinh ra được commit trong cùng một transaction PostgreSQL; một process publisher riêng biệt (`outbox-publisher`) drain và gửi chúng tới RabbitMQ thông qua `EventBus` trait (`metap-infra`). Đây là cơ chế duy nhất để side effect chạm tới RabbitMQ — không có service nào publish trực tiếp. Xem [06. Runtime View](../06-runtime/00-index.md).

## Multi-Tenancy

Mọi bảng nghiệp vụ đều mang `tenant_id`; mọi lời gọi `QueryPlanner`/`CrudService` đều được scope theo nó (`PermissionService::scoped_tenant`). Không tồn tại đường query xuyên tenant nào trong toàn bộ codebase. `scoped_tenant` nhận vào một `RequestContext` đầy đủ và báo lỗi thay vì âm thầm fallback về một tenant mặc định nếu `tenant_id` từng rỗng — một tenant rỗng tại điểm này chỉ có thể là một bug thật sự ở phía trên (auth extractor luôn suy ra một `tenant_id` thật từ một JWT đã verify trước khi bất kỳ đoạn code query-planning nào chạy), và một giá trị mặc định âm thầm sẽ biến bug đó thành kết quả query sai-nhưng-im-lặng trông giống như xuyên tenant, thay vì một lỗi rõ ràng, ồn ào — xem [09. Architecture Decisions](../09-adr/00-index.md).

Từ Phase 16 (2026-08-16), `CrudService` không còn mở transaction trực tiếp trên một `PgPool`
dùng chung — mọi transaction tenant-scoped đi qua `metap-control::Router::begin(tenant_id)`
(`crates/metap-control`), thứ tự quyết định *bảng nào thực sự lưu row của tenant này*:
`Schema` (`SET LOCAL search_path`, hiện chỉ chạy `schema_name="public"` trong thực tế — isolation
thật cần data-plane table-per-entity, chưa xây) hay `DedicatedDb` (một `PgPool` riêng cho
tenant đó, cache theo `dsn_secret_ref`, DSN được resolve qua `SecretStore`/`VaultStore`). Một
tenant chưa có row `control.tenants` thì fallback về hành vi tương thích ngược (schema `public`,
status `Active`). Chi tiết building block ở
[05. Building Block View](../05-building-blocks/03-core-services.md#control-plane-router-multi-tenancy).

## Permission Enforcement

RBAC (danh sách role được phép) kết hợp với ABAC tùy chọn (điều kiện thuộc tính), được đánh giá phía server, ở ba mức: mức entity (role này có được đụng vào entity này không, action gồm `read`/`create`/`update`/`delete`/`transition` — sửa field và chuyển workflow state là hai action tách biệt), mức field (field nào được đọc/ghi), mức record (row cụ thể nào được đọc/ghi, được dịch thành mệnh đề SQL `WHERE`). **Deny-by-default cho non-admin, deny-overrides-allow** (đổi từ opt-in-restriction ngày 2026-08-21 — xem [09. Architecture Decisions](../09-adr/00-index.md)): chưa có policy nào cho một `(entity, action)` thì **không ai được phép**; mỗi policy còn mang một `effect` (`allow`/`deny`) — cần ít nhất một `allow` khớp mới được phép, nhưng chỉ cần một `deny` khớp là bị từ chối ngay bất kể có bao nhiêu `allow` cũng khớp (`metap_permission::evaluate_policies`, seam chung cho cả 4 entry point enforcement). `POST /admin/policies/seed-defaults` bulk-tạo policy allow cho một role mới, tránh việc onboard chậm vì phải tạo policy từng action một. Role `admin` luôn bypass toàn bộ bước này. Một điều kiện record-level có thể tham chiếu sang record khác qua dotted attribute path (1-hop, chỉ cho thao tác trên record đơn — không áp dụng `list()`). Role được tra mới từ `user_roles` cho mỗi request, không bao giờ cache trên JWT. Sequence diagram đầy đủ (tạo user → đăng nhập → kiểm tra quyền) ở [06. Runtime View](../06-runtime/00-index.md)'s mục "Tạo user, đăng nhập, và kiểm tra quyền"; building block ở [05. Building Block View](../05-building-blocks/03-core-services.md#permission-service).

## Cookie Session and CSRF

Trước 2026-09-03, JWT chỉ sống trong React state (`useAuth().token`) — mất khi F5, không ai đọc
được cookie/localStorage nên miễn nhiễm XSS-đánh-cắp-token, đổi lại UX kém (đăng nhập lại mỗi lần
reload). Phase 64 đảo ngược quyết định đó (chủ dự án chủ động đổi, không phải sửa bug — xem
[09. Architecture Decisions](../09-adr/00-index.md)), chọn `HttpOnly` cookie + double-submit CSRF thay vì
`sessionStorage`/`localStorage` (2 lựa chọn kia JS đọc được, một payload XSS lấy thẳng được token).

**2 cookie, set cùng lúc mỗi lần login/OIDC callback thành công** (`POST /auth/login`,
`GET /auth/oidc/{tenantId}/callback`) — `crates/metap-http/src/cookies.rs`:
- `metap_session` — chính JWT, `HttpOnly` (JS không đọc được, kể cả qua XSS). Đây là điểm chính
  của việc bỏ giữ token trong React state.
- `metap_csrf` — giá trị random (UUID v4), **không** `HttpOnly` — JS đọc được, cố ý.

**Double-submit cookie, không phải token đồng bộ hoá kiểu session server-side**: client tự đọc
`document.cookie`'s `metap_csrf` và gắn vào header `X-CSRF-Token` trên mọi request (không chỉ
mutating — mở rộng 2026-09-03 sau audit 04 A#4, vì `GET /auth/token` cũng phát credential dù là
GET); server so 2 giá trị (header vs cookie) khớp mới cho qua **các method mutating** (không
GET/HEAD/OPTIONS — `metap-http/src/auth.rs`'s `requires_csrf_check`). Lý do cơ chế này chặn được
CSRF: một trang cross-site có thể khiến browser tự đính kèm `metap_session` (cookie luôn tự gửi
theo origin), nhưng **không đọc được** `metap_csrf` (same-origin policy chặn JS cross-site đọc
cookie của origin khác) nên không tạo được header đúng.

**`Authorization` header luôn thắng khi có mặt, hoàn toàn không đổi gì** — nhánh Bearer cũ (CLI,
`dev-tools mint-token`, service-to-service như `cron-scheduler`/`metap-grpc::GrpcBackend`) không
chạy qua cookie/CSRF ở đâu cả; `AuthContext` chỉ rơi vào nhánh cookie khi request **không có**
header đó — dấu hiệu của một request browser dùng `fetch(..., {credentials: "include"})`. Sequence
đầy đủ (2 tuyến song song) ở [06. Runtime View](../06-runtime/00-index.md)'s mục "Tạo user, đăng nhập, và kiểm
tra quyền".

**`graphql-gateway` là ngoại lệ có chủ đích, không tham gia cơ chế này** — service riêng, tự
keypair, tự CORS, "decode-only" (chỉ verify JWT, không mint) — mở rộng nó sang cookie nghĩa là
thêm CORS/cookie cross-origin cho 1 service, phạm vi lớn hơn nhiều so với nhu cầu thật. Giải pháp:
`GET /auth/token` (route mới trên service chính, dùng `AuthContext` nên chạy được dù caller vào
bằng cookie hay Bearer) mint 1 JWT ngắn hạn theo đúng identity của caller; `platform-ui`'s
`useGraphQLQuery` gọi route này ngay trước mỗi lần gọi gateway, không giữ token lại — đúng tinh
thần "không giữ credential sống lâu phía client" của cả đợt đổi này.

`POST /auth/logout` (route mới) xoá cả 2 cookie (`Max-Age=0`) — client không tự xoá được cookie
`HttpOnly`, không yêu cầu `AuthContext` (logout một session đã hết hạn/đã logout rồi là no-op hợp
lý, không cần 401 trước). `AppState.cookie_secure` (mặc định `true`) điều khiển flag `Secure` của
2 cookie — **phải tự set `false` cho binary dev chạy `http://localhost`** (không HTTPS), thiếu
bước này browser âm thầm không lưu cookie, giống hệt triệu chứng "login xong không giữ được session"
dù mọi thứ khác đều đúng (lỗ hổng phát hiện live 2026-09-03 ở `metap-demo-crm`/`metap-demo-jira`/
`metap-demo-waf` — cả 3 quên set field này khi phase 64 ship, xem `docs/roadmap/69-*.md`).

## Security Principles

- Route nghiệp vụ mặc định yêu cầu auth.
- Tenant scope là bắt buộc.
- Permission được enforce ở phía server.
- CORS dùng allowlist.
- HTML rich text phải được sanitize trước khi render.
- Secret không bao giờ nằm trong repository.
- Container nên chạy non-root trong production.
- Audit log cho các hành động nhạy cảm phải append-only.

## Performance Principles

- Giới hạn cứng cho page size. (Đã xong.)
- Keyset pagination cho record khối lượng lớn. (Đã xong — xem [05. Building Block View](../05-building-blocks/03-core-services.md#query-planner).)
- Background job cho export/print/report. (Hoãn lại, dựa trên trigger — xem [11. Risks and Technical Debt](../11-risks/00-index.md).)
- Query contract cho từng list view. (Đã xong.)
- Cache snapshot metadata và permission. (Đã xong — `PermissionSnapshot`, theo từng lời gọi, chủ ý không cache theo TTL/cross-request.)
- Index được khai báo sát với metadata. (Đã xong — `EntityField.indexed`/`unique`/`searchMode`, được `IndexReconciler` reconcile.)
- Tách workload reporting khỏi workload OLTP khi cần thiết. (Hoãn lại, dựa trên trigger — cùng mục với trên.)
