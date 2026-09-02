## Phase 61: `metap-demo-waf/data-plane` — tách 3 microservice theo pillar + GraphQL gateway (2026-09-01)

Trigger: chủ dự án chốt hướng "metap-waf sẽ thiết kế microservice, GraphQL để call trên nhiều
microservices đó" — quyết định kiến trúc lớn, đi ngược ADR gốc của `metap` ("không tách
microservice cho CRUD hot path"). Đi qua `EnterPlanMode`/`ExitPlanMode` trước khi code, cùng 2
vòng làm rõ scope với chủ dự án:

1. **Xác nhận phạm vi thật**: "microservice" nghĩa là tách nhỏ chính `data-plane` (1 binary duy
   nhất, 9 entity 4 pillar) theo pillar (DDoS+Firewall/Scanning/Alerting), không chỉ gắn GraphQL
   gateway lên cấu trúc 3-plane (data-plane/control-plane/edge-plane) đã có sẵn.
2. **Chủ dự án tự chỉ ra 1 trục tách bị bỏ sót**: đọc `data-plane/docs/09-access-control.md` xác
   nhận sản phẩm vốn đã định hình "2 portal riêng — Admin Portal (platform-level, xuyên tenant) và
   Customer Portal (per-tenant)". Plan chốt lại **chỉ thiết kế phần Customer Portal backend** (3
   pillar service); Admin Portal backend (`Plan`/`Subscription` billing, chưa xây) là service
   khác, phạm vi khác, ghi lại nhưng không thiết kế ở phase này.

## Rào cản kỹ thuật thật, phát hiện lúc code (không phải suy đoán trước)

**T1 (spike trước khi tách code)** đọc `MetadataRegistry::validate_references()`
(`metap/crates/metap-metadata/src/registry.rs:130-166`) xác nhận: entity đích của 1 field
`Reference` phải có `EntityDefinition` đầy đủ trong CÙNG registry — không chỉ tên. Nhưng route
`/api/:entity*` (`metap-http/src/routes/records.rs`) là route generic tra `Path(entity)` thẳng
vào registry lúc request, không có khái niệm "đăng ký chỉ để validate, không serve CRUD". Vậy
đăng ký `zone_entity()` ở `scanning-service`/`alerting-service` chỉ để pass validate sẽ **đồng
thời** lộ CRUD `waf.zones` đầy đủ qua route generic của 2 service đó — phá đúng nguyên tắc "1
service sở hữu Zone" mà cả việc tách hướng tới.

**Quyết định** (chủ dự án chọn sau khi nghe trade-off): `zoneId` ở `scanning-service`/
`alerting-service` là **`String` thuần, không phải `Reference`** — đúng workaround
`waf.security_events.triggeredById`/`waf.incidents.assignedTo` đã dùng cho trường hợp tương tự
(target không registered cùng registry). Mất FK/cascade-delete-guard/`relatedDisplay` tự động
cho 2 field này; `zones-service`'s nội bộ (`DdosPolicy.zoneId`/`FirewallRule.zoneId`) vẫn giữ
`Reference`/FK thật vì cùng service.

Do không còn ràng buộc FK cross-service, việc 3 service dùng chung 1 Postgres DB/tenant
(`Router::pool_for`) giờ chỉ còn là lựa chọn vận hành đơn giản hơn (khớp `control.tenants` hiện
giả định 1 tenant = 1 DB), không phải yêu cầu kỹ thuật bắt buộc — vẫn giữ vì lý do đó.

## Kiến trúc cuối

`data-plane/` đổi từ 1 Cargo package thành **workspace 3 member**:

| Service | Sở hữu | Port (HTTP / gRPC) |
|---|---|---|
| `services/zones-service` | `waf.zones`, `waf.ddos_policies`, `waf.firewall_rules` | 3000 / 3001 |
| `services/scanning-service` | `waf.scan_jobs`, `waf.scan_findings` | 3010 / 3011 |
| `services/alerting-service` | `waf.security_events`, `waf.incidents`, `waf.alert_policies`, `waf.alert_notifications` | 3020 / 3021 |

Mỗi service tự `main.rs` (path-dep 4 cấp `../../../../metap/crates/metap`, sâu hơn
`metap-lowcode`'s 3 cấp vì thêm 1 tầng `services/`), modernize sang `bootstrap_platform`/
`metap::grpc::optional_serve` (Phase 56/58) thay boot sequence tay cũ — rút ngắn đáng kể so với
`data-plane`'s `main.rs` gốc. Mỗi entity module compile vào đúng 1 binary sở hữu nó, không có
crate dùng chung (khác `metap-lowcode` — ở đây không cần chia sẻ code logic, chỉ chia sẻ
database). Test mới mỗi service: `owns_exactly_its_own_N_entities` (thay
`all_nine_entities_are_auto_registered` cũ) — regression guard cho đúng bug T1 tìm được, không
chỉ đếm entity mà còn implicit xác nhận không entity nào của service khác lọt vào registry.

**GraphQL gateway**: tái dùng nguyên trạng `metap/crates/graphql-gateway` — đọc code xác nhận đã
entity-agnostic hoàn toàn (cấu hình qua `UPSTREAM_<N>_*` env, tự khám phá schema qua
`GET /metadata/entities`), không cần sửa gì. Kế hoạch: 1 instance riêng cho WAF (không gộp
upstream với `metap-demo-crm`/`metap-demo-jira`), 3 upstream trỏ vào 3 service trên — **chỉ dùng
cho query cross-service, read-only** (giới hạn kế thừa nguyên trạng: gateway chỉ decode-only
auth, mọi call xuống upstream dùng 1 service JWT tĩnh, không propagate identity người gọi thật —
mutation qua gateway sẽ chạy permission theo identity service, không phải role người dùng thật,
nên mutation vẫn đi thẳng REST vào đúng service).

**Admin Portal backend** (`Plan`/`Subscription` billing + bảng gán quyền cross-tenant cho Admin
Portal) — ghi nhận là service khác, phạm vi khác, chưa thiết kế/code, khi build đi theo pattern
platform-level đã có sẵn (`metap-control`'s `PLATFORM_TENANT_ID`/`PlatformAdminContext`,
`metap-lowcode`'s `services/control-api`).

## ADR

Thêm bullet exception riêng cho WAF vào `docs/architectures/09-adr.md` (theo đúng khuôn Phase 53
đã làm cho `metap-lowcode`) — khác biệt quan trọng ghi rõ: đây **vẫn là CRUD hot path thật** (ADR
gốc vẫn áp dụng về nguyên tắc), lý do tách là chu kỳ scale/traffic khác nhau giữa các pillar chứ
không phải tránh ACID phân tán (ACID vẫn giữ được nhờ chung DB) — khác hẳn lý do `metap-lowcode`.

## Xác minh

`cargo build/test/clippy -D warnings --workspace` sạch cho cả 3 service (từ `data-plane/`, sau
khi turn `Cargo.toml` thành workspace root). 3 unit test mới `owns_exactly_its_own_N_entities`
pass — xác nhận registry mỗi service đúng, không entity nào lọt sai chỗ.

**T3 verify sống qua HTTP thật (2026-09-01, cùng phiên)** — cả 3 service chạy thật (Postgres dev
sẵn có, tenant/user có sẵn từ đợt test 2026-08-30 trước khi tách), token JWT thật qua
`dev-tools mint-token`:
- `GET /api/waf.zones/:id` ở `zones-service` → 200, đọc đúng Zone có sẵn.
- `POST /api/waf.scan_jobs` ở `scanning-service` với `zoneId` (String) trỏ id Zone đó → 201,
  tạo thành công — xác nhận field `String` không FK hoạt động đúng, không lỗi ràng buộc nào.
- `GET`/`POST /api/waf.zones` ở cả `scanning-service` **và** `alerting-service` → cả 2 đều
  `404 entity_not_found` — bằng chứng thật (không chỉ suy luận từ đọc code) rằng route CRUD
  `waf.zones` không hề tồn tại ở 2 service không sở hữu nó, kể cả thử ghi (`POST`).
- `GET /api/waf.ddos_policies` ở `zones-service` → `relatedDisplay.zoneId: "shop.example.com"`
  vẫn tự động resolve đúng — xác nhận `Reference`/FK nội bộ (`DdosPolicy.zoneId` → `Zone`, cùng
  service) hoàn toàn không bị ảnh hưởng bởi việc tách.

**T4 GraphQL gateway instance thật (2026-09-01, cùng phiên)** — `data-plane/graphql-gateway/`
(config-only, tái dùng nguyên trạng binary `metap/crates/graphql-gateway`, không code mới): keypair
riêng cho instance này (decode-only), 3 `UPSTREAM_<N>_*` trỏ vào 3 service, `SERVICE_JWT` dùng
chung 1 token (3 service share keypair). Chạy thật (`cargo run --manifest-path
../../metap/crates/graphql-gateway/Cargo.toml`) → boot log xác nhận `schema built, entities=9`
(gộp đúng 9 entity từ 3 upstream). **Bằng chứng BFF thật** (đúng kiểu test e2e Phase 50 đã làm
cho crm+jira): 1 GraphQL query duy nhất gộp `wafZones` (từ `zones-service`) +
`wafScanJobsList` (từ `scanning-service`) + `wafIncidentsList` (từ `alerting-service`) → 1
response JSON có đủ cả 3 phần dữ liệu thật.

**T5 routing FE theo tiền tố entity (2026-09-01, cùng phiên)** — `web/vite.config.ts` viết lại:
1 middleware tự forward request theo tiền tố tên entity (`waf.scan_*` → `scanning-service`,
`waf.security_events`/`waf.incidents`/`waf.alert_*` → `alerting-service`, còn lại →
`zones-service`), cộng 1 middleware riêng gộp `GET /metadata/entities` (không có entity trong
path — nguồn cho nav/danh sách entity của FE) từ cả 3 service thành 1 response, vì đây là endpoint
duy nhất không thuộc về đúng 1 service. **Bug thật tìm được khi test** (không phải đọc code suy
luận): thử dùng Vite's built-in `server.proxy`'s option `router` (hàm chọn target động) trước —
`curl` thật lộ ra Vite 8 luôn dùng `target` tĩnh của object, bỏ qua `router` hoàn toàn (khác tài
liệu Vite/`http-proxy` cũ mô tả) — mọi entity ngoài `zones-service` bị route sai, trả về
`404 entity_not_found` giống hệt lỗi ở service không sở hữu entity đó. Sửa bằng middleware tự viết
(forward qua `fetch`, không dùng `server.proxy` cho phần theo entity nữa). Verify lại bằng `curl`
thật qua dev server: `/api/waf.scan_jobs`/`/api/waf.incidents`/`/api/waf.zones` đều route đúng
service, `/metadata/entities/waf.scan_jobs` trả đúng metadata (field `zoneId` đúng `kind: "string"`
— xác nhận thay đổi T1 lan tới tận FE-facing response), `/metadata/entities` gộp đủ 9 entity.
`tsc -b`/`oxlint`/`prettier --check` sạch.

**T6 màn Zone overview thật, gọi GraphQL gateway từ browser (2026-09-01, cùng phiên)** — trước
khi build, làm rõ với chủ dự án: `platform-ui` **chưa có lớp fetch GraphQL nào**, kể cả tổng quát
(chỉ có REST qua `apiFetch`/`useApiQuery`). Chủ dự án chọn: thêm 1 lớp dùng chung
(`useGraphQLQuery`/`graphqlFetch`, cùng vai trò `useApiQuery`/`apiFetch` bên REST) vào
`platform-ui` trước, thay vì viết thẳng trong 1 trang. Thêm `platform-ui/src/api/
{graphqlClient,useGraphQLQuery}.ts` (fetch thuần, không thêm dependency `graphql-request`/
Apollo/urql — khớp cách `apiFetch` cũng không dùng thư viện HTTP nào ngoài `fetch`).

`ZoneOverviewPage` (`data-plane/web/src/demo/ZoneOverviewPage.tsx`, route `/zones/:zoneId/
overview`, link từ trang chủ) — 1 query GraphQL gộp Zone + DdosPolicy + FirewallRule (từ
`zones-service`) + ScanJob gần nhất (`scanning-service`) + Incident gần nhất (`alerting-service`).
Thêm `/graphql` vào `web/vite.config.ts`'s proxy (1 target cố định, không cần route theo entity
vì gateway tự resolve).

**2 bug thật tìm được lúc verify (không phải đọc code)**:
1. Vite 8's `server.proxy`'s `router` option (dùng cho `/api`/`/metadata/entities` ở T5) không
   tôn trọng giá trị trả về — luôn dùng `target` tĩnh của object, mọi entity ngoài
   `zones-service` route sai (`404`). Sửa bằng middleware tự viết (`configureServer`, forward
   qua `fetch` tay), verify lại bằng `curl` thật.
2. Gateway ban đầu tự sinh 1 keypair riêng (decode-only) — token đăng nhập thật (`/auth/login`,
   ký bởi key CHUNG của 3 service WAF) **không decode được ở gateway** (`401 invalid or expired
   token`), vì gateway kiểm theo key khác. Phát hiện khi verify bằng đúng token đăng nhập thật
   (không phải token test riêng đã dùng cho T4). Sửa bằng trỏ `AUTH_JWT_PUBLIC_KEY_PATH` của
   gateway về đúng key chung 3 service — giờ 1 token đăng nhập bình thường dùng được cho cả REST
   lẫn `/graphql`, không cần "token gateway" riêng.

Xác minh: `platform-ui` `typecheck`/`lint`/`format:check` sạch. `data-plane/web`'s `tsc -b`/
`oxlint`/`prettier --check` sạch. Query GraphQL đầy đủ của `ZoneOverviewPage` chạy qua `curl` tới
`web/`'s dev server `/graphql` **bằng đúng token đăng nhập thật** (không phải token mint riêng
cho test) — trả về dữ liệu thật từ cả 3 service trong 1 response.

**Còn lại chưa làm**: Traefik/reverse-proxy thật cho production (dev-server routing của T5 chỉ
dùng được lúc `pnpm dev`), Admin Portal backend riêng (`Plan`/`Subscription`).

## Đính chính sau khi xong T1-T6 (2026-09-01, cùng phiên) — "mất FK" ghi ban đầu quá lời

Kiểm tra lại sau khi cả 6 task xong: cả 9 entity WAF đều `table_name: "records"` (bảng chung,
JSONB), **chưa từng dùng table-per-entity** — kể cả trước khi tách. `metap-reconciler::compile()`
(nơi thật sự build FK Postgres) **chỉ chạy cho entity table-per-entity** (`data-plane`'s
`main.rs` — cả bản gốc lẫn 3 service mới — chỉ gọi `check_metadata_drift`/`reconcile_indexes`
từ `metap-peripherals`, không bao giờ gọi `metap_reconciler::reconcile()`). Vậy **chưa từng có
FK Postgres thật nào** cho `zoneId` ở sản phẩm này, kể cả lúc còn là 1 binary duy nhất — phần
"giữ chung DB để giữ FK/cascade-delete-guard" ghi ở mục "Rào cản kỹ thuật cốt lõi"/ADR là diễn
giải sai, cần đính chính.

**Đúng là gì mất khi hạ `zoneId` xuống `String`**:
- `Reference`'s validate (`metap-crud/src/validation.rs`) chỉ check định dạng UUID hợp lệ, không
  check bản ghi có tồn tại thật — không phải nguồn integrity đáng kể.
- **Mất `relatedDisplay` tự động** — tiện hiển thị (vd hostname thay vì UUID trần), người dùng
  FE tự gọi thêm hoặc dùng GraphQL gateway để "join".
- **Mất delete-guard `find_referencing_record`** cho riêng field này — nhưng guard này
  (`referencing_fields(metadata: &MetadataRegistry, ...)`, `metap-crud/src/crud_service/
  helpers.rs`) chỉ quét registry của **tiến trình đang chạy lệnh xoá**, không bao giờ biết tới
  entity registered ở service khác — nghĩa là **guard này đã không bao giờ bảo vệ được tham
  chiếu cross-service**, dù giữ `zoneId` là `Reference` hay không. Tách service tự nó đã xoá khả
  năng bảo vệ này (xoá 1 Zone ở `zones-service` không bị chặn dù `scanning-service`/
  `alerting-service` còn ScanJob/SecurityEvent/Incident trỏ tới) — đây là **gap thật, độc lập
  với lựa chọn String/Reference**, chưa có giải pháp (cần 1 cơ chế cross-service kiểm tra trước
  khi xoá nếu muốn, chưa làm).

**Kết luận không đổi**: quyết định `zoneId: String` ở `scanning-service`/`alerting-service` vẫn
đúng — lý do quyết định (đăng ký `zone_entity()` ở service khác sẽ lộ CRUD `waf.zones`) vẫn đứng
vững, độc lập với chuyện FK. Chỉ phần lý giải phụ ("giữ DB chung để không mất FK") bị sai, đã sửa
lại ở ADR (`09-adr.md`) và ghi lại đầy đủ ở đây.
