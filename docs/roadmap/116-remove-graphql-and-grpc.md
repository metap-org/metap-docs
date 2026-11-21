# 116. Bỏ GraphQL và gRPC: metap chỉ nói HTTP/JSON

**Trạng thái:** đã chốt hướng 2026-11-21 (chủ dự án). P0 (spike) xong 2026-11-21, xem mục "P0: đã làm". P1–P6 làm **một lượt** cùng ngày (chủ dự án chọn "chuyển 1 lượt": không chạy song song GraphQL/gRPC), xem mục "P1–P6: đã làm"; còn lại vài việc nhỏ ghi ở cuối mục đó. Tài liệu này là quyết định và kế hoạch. Thay quyết định "Entity CRUD ứng dụng: GraphQL-only" (2026-09-21, [roadmap 90](90-remove-rest-entity-crud.md)) trong [09. ADR](../architectures/09-adr/00-index.md).

## Vấn đề

metap là nền tảng **management**: CRUD có metadata, workflow, phân quyền, audit, quản trị user/tenant. Tải của nó là tải của một ứng dụng quản trị, không phải đường dữ liệu tốc độ cao. Phần nào của sản phẩm cần thông lượng cao thì đã tự đứng riêng, không đi qua metap: ở WAAP, edge plane (Go `net/http`, chỉ dùng `pkg/kit`) và analytics plane (service + ClickHouse riêng, ADR-0009 của waap) hoàn toàn không dùng luồng hay DB của metap.

Trong khi đó metap đang chạy **ba** giao thức cho cùng một việc: REST, gRPC và GraphQL. Hệ quả đo được (2026-11-21):

- **Mỗi thao tác custom viết ba lần:** message + RPC trong proto, handler gRPC (`pkg/grpcapi`), field GraphQL trong gateway (`pkg/graphqlgateway`: kiểu, tham số, `identityCall`, ánh xạ map). Lớp thứ ba chỉ dịch qua lại. Roadmap 115 thêm hơn 60 thao tác theo đúng mẫu này.
- **Khối lượng:**
  - gRPC: 10 service với 71 RPC (8 CRUD, 63 identity/admin), không có RPC streaming nào. `pkg/grpcapi` có khoảng 5.300 dòng viết tay và khoảng 10.500 dòng sinh từ proto, cộng bước `protoc` chạy trong docker (`gen.sh`).
  - GraphQL: `pkg/graphqlgateway` dài 4.404 dòng, gồm khoảng 92 field viết tay (63 identity/admin, 29 platform) ngoài phần CRUD sinh từ metadata.
- **Một lỗi đi qua ba lần dịch:** `ServiceError` → status gRPC kèm JSON `ErrorDetails` nhét trong trailer → `extensions` GraphQL. Mỗi trường mới của lỗi phải thêm ở cả ba chỗ (ví dụ `Details` ở metap#57).
- **Khó vận hành:** GraphQL dồn mọi thứ vào một `POST /graphql`, nên không rate-limit, log hay phân quyền theo đường dẫn được. gRPC cần HTTP/2 và client sinh từ proto, không gọi thử bằng curl được.
- **Lợi ích của GraphQL phần lớn không còn:**
  - Type của FE đã sinh từ OpenAPI (`/metadata/openapi.json` → `openapi-typescript`), không từ schema GraphQL.
  - Relation chéo service được resolve bằng `Get` từng id, tức vẫn là N lời gọi (chỉ có cache trong một request), không có batch.
  - REST `/api/{entity}` vốn đã có và đang được mount trong Go (`pkg/httpserver/routes_records.go`).

## Quyết định

**metap chỉ nói HTTP/JSON (REST).** Bỏ hẳn GraphQL (`pkg/graphqlgateway`, `/graphql`) và gRPC (`pkg/grpcapi`, proto, code sinh, `gen.sh`). Cả giao tiếp giữa các service cũng là HTTP/JSON.

Nguyên tắc đi kèm:

1. **Một thao tác mới tốn đúng một lớp:** một handler Go có kiểu input/output rõ ràng. Hợp đồng (OpenAPI) sinh từ chính các kiểu đó, không có IDL riêng và không có bước sinh code phía server.
2. **Một dạng lỗi:** `{"error":{"code","message","fieldErrors","details"}}` của `pkg/kit/httpx`, ở mọi route, mọi service.
3. **Một đường xác thực:** bearer token hoặc cookie phiên kèm CSRF double-submit (đã có), chuyển tiếp token người gọi khi service gọi service (`ForwardedBearerToken`, đã có).
4. **URL là route thật:** rate-limit, log, cache và phân quyền theo đường dẫn được.

## Thiết kế

### Entity CRUD

Giữ và hoàn thiện REST sẵn có: `GET/POST /api/{entity}`, `GET/PATCH/DELETE /api/{entity}/{id}`, `POST /api/{entity}/aggregate`, `POST /api/{entity}/{id}/transitions/{action}`, `GET /api/{entity}/{id}/audit-events`. Bổ sung những gì GraphQL đang cho FE:

| GraphQL đang cho | REST thay bằng |
|---|---|
| Chọn field | `?fields=a,b,c` (mặc định: mọi field đọc được) |
| Relation (`zone { hostname }`) | `?expand=zoneId` — cùng service thì service tự resolve; chéo service thì gateway resolve (xem dưới) |
| Lấy nhiều bản ghi theo id | `GET /api/{entity}?ids=a,b,c` (batch, dùng cho `expand`) |
| List: filter, sort, JQL, cursor, list view | Query string tương ứng; cú pháp chốt khi làm P1 |
| Capabilities | Giữ trong response như hiện nay |
| Action nghiệp vụ (`invokeAction`) | `POST /api/{entity}/{id}/actions/{action}` (và bản không có `id` cho action cấp entity) |
| Gom nhiều query trong một request | **Không thay.** Chấp nhận nhiều request hơn (HTTP/2 qua nginx); chỗ thật sự cần thì dùng endpoint gộp có chủ đích, ví dụ aggregate nhận nhiều spec |

OpenAPI cho phần entity vẫn sinh động từ metadata (`/metadata/openapi.json`, đã có) và được mở rộng theo các tham số trên.

### Thao tác custom (identity, admin, tenant, config, cron, policy, ...)

- Handler REST dựng bằng **huma v2** (có adapter cho chi, router metap đang dùng): kiểu Go của request/response chính là schema, và OpenAPI 3.1 được sinh ra từ đó. Gộp chung vào `/metadata/openapi.json` hoặc một tài liệu riêng (chốt ở P1).
- Logic nằm ở tầng service sẵn có (`pkg/identity`, `pkg/crud`, ...). Handler chỉ đọc input, gọi tầng service rồi ghi output. Phần kiểm tra quyền và input đang nằm trong handler gRPC được chuyển nguyên sang.
- **Quy ước URL** (chốt ở P1, sẽ ghi vào `08-cross-cutting`): tài nguyên theo danh từ số nhiều; hành động không phải CRUD thì đặt dưới tài nguyên dạng `POST .../{verb}`. Ví dụ:
  - `/admin/users`, `/admin/users/{id}/roles`, `/admin/users/{id}/mfa/reset`
  - `/me/mfa`, `/me/sessions/{id}`
  - `/admin/sso`, `/admin/support-grants`, `/admin/audit`, `/platform/tenants/{id}/suspend`

### Service gọi service

- **`HttpBackend`** trong metap thay cho `GrpcBackend`. Nó implement `crud.RecordBackend` qua `/api/{entity}`, chuyển tiếp token của người gọi, và đọc lỗi về lại `ServiceError` từ envelope chung. Các chỗ đang dùng `GrpcBackend`: `pkg/cronscheduler` (`workflow_transition`, `bulk_query_action`), `pkg/crud/record_backend.go`; ở waap là `m-zones/guardedbackend.go` (kiểm tra reference chéo service khi xoá zone) và gateway.
- Các lời gọi custom còn lại (rất ít) dùng client nhỏ viết tay, hoặc sinh bằng `oapi-codegen` từ OpenAPI khi số lượng đủ lớn.
- Deadline và huỷ dùng `context` kèm timeout mặc định của `pkg/kit/httpx`.

### Gateway: `pkg/apigateway` (thay `pkg/graphqlgateway`)

Gateway chung, mỏng, không biết entity nghiệp vụ nào:

- **Định tuyến:** reverse proxy (`httputil.ReverseProxy`) với bảng entity → upstream lấy từ metadata của từng upstream (giống cách gateway GraphQL hiện biết service nào phục vụ entity nào). Thao tác custom định tuyến theo tiền tố đường dẫn (`/admin/users*`, `/me/*` tới issuer, ...).
- **`expand` chéo service:** sau khi nhận response của service gốc, gateway gom các id reference theo entity đích, gọi `?ids=` một lần cho mỗi entity đích, rồi gắn kết quả vào. Như vậy có batch, tốt hơn GraphQL hiện tại (gọi theo từng id).
- **Không xác thực ở gateway:** service nào cũng tự kiểm token qua JWKS như hiện nay. Gateway chỉ chuyển tiếp header và cookie.
- Ở WAAP, nginx đứng trước vẫn giữ vai trò SPA và TLS.

## Giữ, bỏ

**Giữ:** `/auth/*` (login, MFA, SSO, signup, invitation, password), OAuth2 AS, JWKS/well-known, SCIM `/scim/v2`, health/metrics, attachment nhị phân, audit export, `/metadata/*`, REST entity CRUD.

**Bỏ:**
- `pkg/graphqlgateway`, `/graphql`, chế độ cookie-auth của gateway;
- `pkg/grpcapi` (server, client, status/trailer envelope), `pkg/grpcapi/pb`, `pkg/grpcapi/proto`, `gen.sh`;
- `GRPC_ENABLED` và các cổng gRPC trong compose của downstream.

## Ảnh hưởng tới downstream

- **platform-ui:** khoảng 28 file đang gọi GraphQL. Chúng chuyển sang `apiFetch`/`useApiQuery` có sẵn, kèm một lớp `records` client mới (list, get, create, update, delete, transition, action, `fields`, `expand`). `graphqlClient.ts`, `useGraphQLQuery`, `graphqlRecords.ts` và `graphqlNaming.ts` bị xoá sau cùng.
- **waap:**
  - `m-web` gọi qua platform-ui.
  - `m-gateway` (`waf-graphql-gateway`) được thay bằng `pkg/apigateway` cộng route riêng của WAF.
  - `c-config-distributor` đang đọc management qua `/graphql` (`management/management.go`), sẽ chuyển sang REST.
  - Read model `waf.security_events` (analytics) đang do gateway phục vụ: nó thành một upstream của `pkg/apigateway`, hoặc FE gọi thẳng `a-api`. Chốt ở P4.
- **metap-demo-jira/web:** 2 file đang dùng GraphQL.
- **metap-lowcode:** kiểm lại khi làm P3, vì có chỗ phụ thuộc vào gateway hoặc gRPC.

## Giai đoạn

Mỗi giai đoạn là một hoặc vài PR, chạy được và kiểm chứng được. GraphQL và gRPC chạy song song cho tới khi consumer cuối cùng chuyển xong. *(Thực tế: P1–P6 làm một lượt, không có giai đoạn chạy song song, xem "P1–P6: đã làm".)*

| | Việc | Xong khi |
|---|---|---|
| **P0 — Spike** | (1) Chuyển một nhóm identity nhỏ (Sessions, 3 RPC) sang huma, kiểm tra OpenAPI sinh ra và FE gọi được. (2) `pkg/apigateway` tối thiểu (proxy + bảng định tuyến) cộng `?expand=` chéo service trên `waf.scan_jobs?expand=zoneId` (scan job ở scanning-service, zone ở zones-service). | Hai mục trên chạy trên dev stack của waap; quy ước URL và cú pháp query được chốt |
| **P1 — REST entity đủ năng lực** | `fields`, `expand` (cùng service), `ids`, action route, OpenAPI mở rộng; quy ước URL ghi vào `08-cross-cutting` | Test tích hợp; OpenAPI bao phủ đủ |
| **P2 — Thao tác custom sang REST** | 63 RPC identity/admin cùng nhóm platform (config, cron, policy, user, dashboard, preference) chuyển sang handler huma, theo từng nhóm service | Mỗi nhóm: test tích hợp ngang với test gRPC cũ |
| **P3 — Service gọi service** | `HttpBackend`; cron-scheduler, waap `guardedbackend` | Không còn `GrpcBackend` được dùng |
| **P4 — Gateway** | `pkg/apigateway` đầy đủ; waap thay `waf-graphql-gateway`; `c-config-distributor` và read model analytics | Dev stack của waap chạy không cần `/graphql` |
| **P5 — FE** | platform-ui, m-web, metap-demo-jira/web sang REST | Không còn import GraphQL client |
| **P6 — Xoá** | Xoá `pkg/graphqlgateway`, `pkg/grpcapi`, proto, `gen.sh`, cấu hình gRPC trong compose; cập nhật CLAUDE.md và README các repo | Build và test sạch, không còn thư viện `graphql-go` và `grpc` trong `go.mod` |

## Đánh đổi chấp nhận

- **Mất gom nhiều query vào một request** (`graphqlFetch` gom mọi query phát ra trong cùng một vòng render). Một màn dashboard sẽ phát nhiều request hơn. Với tải management thì chấp nhận được; chỗ nào đo thấy chậm thì làm endpoint gộp có chủ đích.
- **Mất IDL trung lập ngôn ngữ (proto).** Service viết bằng ngôn ngữ khác sẽ dùng OpenAPI, kém chặt hơn proto nhưng đủ dùng. Hiện không có service nào như vậy.
- **Khối lượng chuyển đổi lớn nhưng mang tính cơ học:** hơn 70 RPC, khoảng 92 field GraphQL, khoảng 30 file FE. Logic không đổi chỗ.
- **Đảo ngược roadmap 90/95** (REST → GraphQL, 2026-09). Lý do đổi: các giả định lúc đó không còn đúng. Federation thật chưa bao giờ được xây; FE sinh type từ OpenAPI chứ không từ GraphQL; Go có sẵn REST CRUD; và chi phí ba lớp đã lộ rõ qua roadmap 113–115.

## Khi nào nên xem lại

Có trường hợp nào dưới đây thì xem lại quyết định, cho phần bị ảnh hưởng chứ không phải cho cả metap:

- metap phải phục vụ một luồng có QPS cao hoặc cần streaming. Hướng ưu tiên khi đó vẫn là tách luồng đó ra service riêng như edge và analytics của WAAP.
- Xuất hiện nhiều service đa ngôn ngữ cần hợp đồng chặt hơn OpenAPI.

## Câu hỏi còn mở (chốt ở P0/P1)

1. ~~Cú pháp query của list trên REST~~ → chốt: filter lồng/khoảng/`OR`/`NOT` đi qua `jql` trên query string, sort một field (xem `08-cross-cutting`, "Quy ước REST").
2. OpenAPI của thao tác custom: gộp vào `/metadata/openapi.json` hay tách tài liệu riêng.
3. Read model analytics của WAAP: upstream của gateway hay FE gọi thẳng `a-api`.
4. Chạy song song bao lâu, và phiên bản nào thì xoá hẳn (P6).

## P0: đã làm (2026-11-21)

Spike chứng minh được hai điểm rủi ro, đều chạy trên dev stack của WAAP.

**Thao tác custom bằng REST có kiểu (huma v2).**
- `pkg/httpserver/typedapi.go` dựng một API huma duy nhất trên chi router. OpenAPI 3.1 sinh từ kiểu Go, phục vụ ở `/metadata/api.json` (tách khỏi `/metadata/openapi.json` của entity; câu hỏi mở số 2 chốt theo hướng này).
- Lỗi của huma (input sai schema, body hỏng) và lỗi của handler đều ra cùng envelope `{"error":{code,message,fieldErrors,details}}`. Lỗi schema là 400 `validation_failed`, `fieldErrors` theo tên input.
- Xác thực là middleware dùng `authenticate()`, nên bearer, API key, cookie phiên kèm CSRF giữ nguyên hành vi.
- Nhóm Sessions đã chuyển: `GET /me/sessions`, `DELETE /me/sessions/{id}`, `POST /me/sessions/revoke-others`. Logic vẫn ở `pkg/identity`, handler chỉ đọc input, gọi xuống và định dạng output. Test tích hợp `TestSessionOperationsOverREST` phủ cả cookie và CSRF.

**Gateway và `expand` chéo service.**
- `pkg/apigateway` cùng binary `cmd/api-gateway` (`UPSTREAMS=name=url|prefix,...;...`): reverse proxy học bảng entity → service từ `/metadata/entities` của từng upstream bằng credential của người gọi; route không phải entity đi theo prefix dài nhất.
- `GET /api/{entity}[/{id}]?expand=f`: gom id theo entity đích, mỗi entity đích gọi một lần `GET /api/{target}?ids=` (lô 200), gắn kết quả vào `expanded.{f}`. Bản ghi không còn hoặc không được đọc thì ra `null`. Field không phải reference bị từ chối 400.
- REST list nhận `?ids=a,b,c` (dùng `CrudService.GetMany`, có kiểm quyền). `ids`, `expand`, `fields` là tham số dành riêng, không bị hiểu là filter.
- **Liên kết chéo service trong metadata:** `EntityField.RefersTo(entity)` cho field chuỗi chứa id của record mà service khác sở hữu. Không khoá ngoại, không kiểm entity đích có trong registry, schema vật lý không đổi (có test ở reconciler). Gateway dựa vào đó để biết `expand` sang đâu. WAAP khai báo cho `zoneId` của `waf.scan_jobs`, `waf.incidents`, `waf.security_events`.
- Chạy thật: đăng nhập và `/me/sessions` qua prefix tới zones-service; `waf.scan_jobs?expand=zoneId` (scanning-service) trả hostname của zone (zones-service) chỉ với một lời gọi sang zones; zone không tồn tại ra `null`; không có token thì service tự trả 401.

**Sửa kèm:** body lỗi 500 dự phòng của `pkg/kit/httpx` ghi nguyên chuỗi `"code":errcode.InternalError` (JSON không hợp lệ). REST trước đây làm rơi `details` của lỗi guard (metap#57); giờ `ErrorBody.Details` có ở mọi route.

**Chưa làm ở P0, chuyển sang P1:** `fields`; `expand` trong cùng service nếu không qua gateway; quy ước URL ghi vào `08-cross-cutting`; cú pháp filter lồng trên query string.

## P1–P6: đã làm (2026-11-21)

Chủ dự án chọn chuyển **một lượt**: mỗi nhóm dựng REST, đổi FE, xoá phần GraphQL/gRPC tương ứng ngay, không có giai đoạn
chạy song song. Câu hỏi mở số 4 vì vậy không còn.

**metap**
- **REST dùng backend đã bọc của service.** `AppState.Records` (từ `app.Options.WrapBackend`) và `AppState.Actions` giờ được
  dựng bất kể transport; mọi route `/api/{entity}` gọi `state.RecordBackend()`. Guard của waap (`GuardedZonesBackend`) vì thế
  áp lên REST.
- **Action khai báo qua REST:** `POST /api/{entity}/{id}/actions/{action}` (scope record) và `POST /api/{entity}/actions/{action}`
  (scope entity); body `{"input":{}}`, trả `{"data": ...}`; action không khai báo hoặc sai scope là 404 `action_not_found`.
- **`pkg/recordclient`** thay `GrpcBackend`: `crud.RecordBackend` + `crud.ActionInvoker` trên REST của service khác, chuyển tiếp
  token của người gọi (không có thì dùng service token), lỗi trả về đúng `*crud.ServiceError` (giữ `fieldErrors`, `details`).
  `cmd/cron-scheduler` dùng nó (`CRON_TARGET_URL` thay `CRON_TARGET_GRPC_ADDR`).
- **63 RPC identity/admin sang REST có kiểu.** Service chuyển thành Go thuần ở `pkg/adminapi` (kiểu request/response trong
  `types.go`, sinh một lần từ message protobuf rồi bảo trì tay; lỗi là `*crud.ServiceError`; caller lấy từ `WithCaller` hoặc
  `WithAuthenticator`). `pkg/httpserver/ops_admin.go` phục vụ chúng bằng huma, chỉ trên token issuer (`AppState.Identity`):
  `/me`, `/me/tenants`, `/me/mfa*`, `/tenant-users`, `/admin/users*`, `/admin/invitations`, `/admin/roles*`,
  `/admin/permission-resources`, `/admin/service-accounts`, `/admin/api-keys/{id}`, `/admin/scope-members`, `/admin/audit*`,
  `/admin/support-grants*`, `/admin/owner*`, `/admin/sso`, `/platform/tenants*`, `/platform/support-sessions`,
  `/platform/owner-recovery*`. Danh sách luôn là `[]`, không bao giờ `null`. Trường JSON trước đây là chuỗi (`diffJson`,
  `conditionJson`) của audit người dùng/vai trò và quyền nâng cao nay là JSON thật (`diff`, `condition`). SCIM chạy
  `UserAdminService` in-process, xác thực bằng đúng `authenticate()` của HTTP. Test gRPC cũ của các service này được port
  sang gọi thẳng `adminapi` (giữ luồng token thật: API key, session, support), thêm `TestAdminOperationsOverREST` cho tầng HTTP.
- **Nhóm platform** (policy, cron, config, preference, dashboard) đã có route REST sẵn từ trước; FE gọi thẳng.
- **Gateway:** `pkg/apigateway` gộp `/metadata/entities` và `/metadata/entity-access` của mọi upstream (FE trước đây phải tự
  gộp ở dev proxy, còn nginx production thì thiếu hẳn); `/metadata/entities/{entity}` đi tới service sở hữu. `apigateway.Main()`
  để sản phẩm dựng binary của mình.
- **Read model:** `app.Options.ExternalEntities` (đăng ký metadata, không reconcile, mọi lời gọi tới backend riêng qua
  `crud.RouteBackends`). Câu hỏi mở số 3 chốt: read model analytics của WAAP do alerting-service phục vụ như một entity bình
  thường, gateway không cần biết gì.
- `query.AggregateSpec` có tag JSON camelCase.
- **Xoá:** `pkg/grpcapi` (kể cả proto, pb), `pkg/graphqlgateway`, `app.Options.GRPCPort`; `graphql-go` không còn trong `go.mod`
  (`grpc`/`protobuf` chỉ còn là dependency gián tiếp của SDK cloud trong `pkg/secrets`).

**waap**
- `m-gateway` thành `api-gateway` (gọi `apigateway.Main()`, không có code WAF). Compose bỏ `GRPC_ENABLED`, cổng gRPC và service
  account của gateway; `UPSTREAMS` khai báo prefix cho zones-service.
- `GuardedZonesBackend` gọi scanning/alerting bằng `recordclient` (`SCANNING_URL`/`ALERTING_URL`); sibling không với tới được
  vẫn ra 503 `reference_check_unavailable`.
- `c-config-distributor/management` đọc zones bằng REST (`ZONES_URL` mặc định zones-service `:3000`).
- alerting-service phục vụ `waf.security_events` (`ExternalEntities` trên `m-analytics.Backend`).
- `m-web`: dev proxy và nginx chỉ còn trỏ tới gateway; `api/waf.ts` dùng client REST của platform-ui và `invokeAction`.

**platform-ui:** `api/records.ts` (list/infinite/trang đánh số/get/`ids`/aggregate/create/update/delete/transition/
`invokeAction`) thay `graphqlRecords.ts`; mọi module admin và account chuyển sang REST; `graphqlClient.ts`, `useGraphQLQuery`,
`graphqlRecords.ts`, `graphqlNaming.ts` đã xoá. Ngoại lệ có chủ đích: các trang control-plane của **metap-lowcode**
(`TenantsAdminPage`, `lowCodeControlPlane`) vẫn gọi gateway GraphQL riêng của hệ thống đó, qua client nhỏ tự chứa trong
`controlPlaneClient.ts`; đó là API của hệ thống khác, không phải của metap.

**Kiểm chứng:** metap `go build`/`go vet` sạch; toàn bộ test unit và integration qua (trừ test cần Redis/Vault thật, môi trường
chạy thiếu `REDIS_URL`/Vault, không liên quan). waap build/vet/test sạch. platform-ui và m-web: typecheck, oxlint, prettier sạch.
Chạy thật trên dev stack của waap qua gateway `:4000` với một tenant tạm (đã dọn): đăng nhập, `/me*`, `/tenant-users`, đọc và ghi
admin (mời, vai trò + quyền, chặn xoá vai trò đang dùng với `fieldErrors`, service account, config, preference), metadata gộp đủ
11 entity `waf.*`, CRUD zone, action `testOrigin`/`syncConfigState`/`correlate`, aggregate, `waf.security_events`,
`waf.scan_jobs?expand=zoneId` ra hostname của zone, xoá zone bị chặn 409 `record_referenced` khi còn scan job và được phép sau khi
xoá scan job. FE chưa kiểm qua trình duyệt (theo quy ước, chủ dự án tự kiểm).

**Chưa làm / ngoài phạm vi:**
- ~~`fields` và cú pháp filter lồng; quy ước URL~~ → xong, xem mục "Bổ sung" ngay dưới.
- **metap-demo-jira:** backend vẫn là Rust phụ thuộc `../metap/crates` đã xoá từ 2026-10-30, vốn không build được; 2 file FE dùng
  GraphQL chưa chuyển.
- **metap-lowcode:** Rust, cùng tình trạng; gateway GraphQL control-plane của nó giữ nguyên (xem ngoại lệ ở trên).
- control-plane (`c-config-distributor`) đã qua test unit nhưng chưa chạy lại end-to-end với Redis/edge sau khi đổi sang REST.

## Bổ sung (2026-11-21, sau P1–P6)

Làm nốt các mục "chưa làm" của P1–P6 và một lỗi lộ ra khi kiểm lại vai trò của gateway.

- **`?fields=a,b`** trên list, `?ids=` và get: `data` chỉ giữ các field đó, vỏ bản ghi giữ nguyên, field lạ là 400 `validation_failed`
  kèm `fieldErrors.fields`. Che field theo quyền chạy trước nên `fields` chỉ thu hẹp. Gateway biết `fields` đi cùng `expand`: nó hỏi
  service thêm cả các field sẽ expand (nếu không `?fields=name&expand=zoneId` mất id để expand, ra `null`), rồi bỏ chúng khỏi kết
  quả nếu người gọi không xin.
- **Filter lồng, sort:** không thêm cú pháp. JQL đã có `AND`/`OR`/`NOT`, ngoặc, so sánh, `~`, `IN`, `IS EMPTY`, `ORDER BY`; sort giữ một
  field vì keyset cursor gắn với đúng một khóa. Chốt trong `08-cross-cutting`.
- **OpenAPI cho route entity:** `/metadata/openapi.json` giờ mô tả 6 route generic (`/api/{entity}`, `/{id}`, `aggregate`,
  `transitions`, hai dạng `actions`) với đủ tham số, envelope `Record`/`Page`/`Error`. Entity là tham số đường dẫn, không có path theo
  từng entity; kiểu của `data` là `EntitySummary` của entity đó.
- **Quy ước URL** ghi vào `08-cross-cutting` ("Quy ước REST").
- **Lỗi lộ ra: rate limit sau gateway.** Limiter của service tính theo `RemoteAddr`; sau gateway (rồi nginx) mọi người dùng cùng một
  địa chỉ nên cả portal chung một bucket ~5 req/s. Sửa: `httpx.ClientIP` + `TRUSTED_PROXIES` (chỉ tin `X-Forwarded-For` từ proxy
  được liệt kê, đọc từ phải sang trái), service dùng nó cho rate limit, captcha đăng ký và IP của phiên; gateway tự giới hạn theo
  người gọi và chuyển địa chỉ người gọi cho các lời gọi riêng của nó (expand, metadata); nginx ghi đè `X-Forwarded-For` bằng
  `$remote_addr`; compose của waap đặt `TRUSTED_PROXIES`. Có test cho `ClientIP`, limiter sau proxy tin cậy và `fields`+`expand`.

## Chạy lại control-plane → edge sau khi đổi sang REST (2026-11-21)

Dựng stack thật (Postgres, Redis, zones-service, `waf-config-distributor`, `waf-edge`, một origin giả), tạo domain, zone, firewall rule
qua REST của zones-service (kể cả `syncConfigState` rồi `transitions/activate`: chỉ kích hoạt được khi `hasConfig`, thông báo guard
đọc được), cho distributor đọc zones qua REST (`ZONES_URL`) và edge nạp từ Redis.

Chạy đúng: resync đọc zone/rule qua `recordclient`, publish `waf:zone:<host>`; request tới host đó khớp rule trả 403, path khác đi tới
origin, host lạ trả 421; telemetry từ edge tới `/ingest/events` của distributor (`eventsSent` tăng; chặng tiếp theo tới analytics-api
không chạy trong lần này); xóa rule thì hết bị chặn sau một lần resync; chuyển zone sang `suspended` thì bị gỡ khỏi Redis và edge trả
421 (`resync found an orphaned rule-set, removing`); xóa zone khi scanning-service không chạy bị từ chối `reference_check_unavailable`
(đúng thiết kế).

**Hai lỗi thật lộ ra (cả hai có từ trước, không do bỏ GraphQL):**
1. **Rule thêm sau lần nạp đầu không bao giờ tới edge.** Edge bỏ qua zone có `configVersion` không đổi, mà không có gì tăng
   `Zone.configVersion` (docs nói "tự động", code không làm). Lần chứng minh ở roadmap 98 chỉ thử tạo mới trước khi edge nạp lần đầu nên
   không thấy. Sửa: `control-plane` gắn `configVersion` = hash nội dung rule-set (trừ `compiledAt`) khi publish
   (`ruleset.ContentVersion`); xem `docs/management/04-architecture-boundary.md`. Không dùng bộ đếm ở management vì nó bỏ sót rule áp
   cho cả tenant, sửa DB trực tiếp, và hai thay đổi đồng thời.
2. **`tenantId` trong rule-set do resync tạo ra luôn rỗng** (đường sự kiện thì có): REST không trả tenant trong bản ghi (GraphQL cũ thì
   có). `management.Client` giờ gắn tenant của phiên đăng nhập vào mỗi bản ghi. Edge không đọc field này nên chưa gây hỏng gì.

**Chưa kiểm:** đường sự kiện RabbitMQ (môi trường không có RabbitMQ; resync là đường đảm bảo hội tụ nên phần cốt lõi đã được chạy),
chặng distributor → analytics-api, và pillar DDoS / IP access list qua cùng pipeline (chỉ có unit test).
