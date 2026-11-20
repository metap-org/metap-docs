# 116. Bỏ GraphQL và gRPC: metap chỉ nói HTTP/JSON

**Trạng thái:** đã chốt hướng 2026-11-21 (chủ dự án); chưa triển khai. Tài liệu này là quyết định và kế hoạch. Thay quyết định "Entity CRUD ứng dụng: GraphQL-only" (2026-09-21, [roadmap 90](90-remove-rest-entity-crud.md)) trong [09. ADR](../architectures/09-adr/00-index.md).

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

Mỗi giai đoạn là một hoặc vài PR, chạy được và kiểm chứng được. GraphQL và gRPC chạy song song cho tới khi consumer cuối cùng chuyển xong.

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

1. Cú pháp query của list trên REST: filter lồng (`all`/`any`), sort nhiều trường, JQL đi qua query string hay `POST /api/{entity}/search`.
2. OpenAPI của thao tác custom: gộp vào `/metadata/openapi.json` hay tách tài liệu riêng.
3. Read model analytics của WAAP: upstream của gateway hay FE gọi thẳng `a-api`.
4. Chạy song song bao lâu, và phiên bản nào thì xoá hẳn (P6).
