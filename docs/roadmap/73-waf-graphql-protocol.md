## Phase 73: `metap-demo-waf`'s FE→BE giao thức chuẩn hoá về GraphQL (2026-09-04)

Trigger: sau khi merge hết PR trong phiên và dọn nhánh local, chủ dự án yêu cầu — "check phần demo
waf, t muốn giao thức chuẩn gọi từ FE -> BE dùng graphql hết, có thể ngoại trừ phần shell, auth".

Khảo sát trước khi code: `data-plane/web` là REST 100% (`api/waf.ts` gọi thẳng `/api/:entity*`),
trong khi `metap`'s `graphql-gateway` (BFF gộp GraphQL từ nhiều service, xem Phase 61) đã tồn tại
và từng verify sống 2026-09-02 nhưng **chỉ 1 consumer thật** (`ZoneOverviewPage`, đã xoá cùng Phase
71's rewrite IA) — hạ tầng có sẵn nhưng không ai dùng. 2 câu hỏi phạm vi hỏi lại chủ dự án qua
`AskUserQuestion`, cả 2 chọn "Recommended":
1. 9 endpoint custom (không phải CRUD) — viết resolver GraphQL riêng thay vì để REST làm escape
   hatch.
2. Mutation (create/update/delete/transition) — cũng chuyển qua GraphQL, không chỉ query.

Việc này mở khoá phạm vi đầy đủ: 100% record/action traffic của portal qua `/graphql`, trừ
`/auth/*`/`/preferences/*` (đúng ngoại lệ chủ dự án nêu — session/CSRF cookie không phải thứ
GraphQL nên can thiệp) và `/metadata/*` (schema reflection, không phải business data — xem lý do
dưới).

## Phase 1/2 — extension point ở `metap` core (`metap-org/metap#11`)

`metap-graphql::build_schema` trước đây `.finish()` thẳng, không cho thêm field ngoài CRUD sinh từ
metadata. Tách `build_schema_parts` (dựng `Object` `Query`/`Mutation` với field CRUD generic,
KHÔNG `.finish()`) khỏi `build_schema` (wrapper cũ, finish ngay) — downstream có field riêng thì
gọi `build_schema_parts`, thêm `.field(...)` của mình, rồi mới `.register().finish()`. Cùng lý do
`metap-http::build_router` nhận `extra_routes`: field custom mang business-entity knowledge,
`metap-graphql` không được biết. `graphql-gateway`'s `schema_builder::build_with_extensions` làm
tương tự 1 tầng cao hơn (upstream discovery trước, rồi mới gọi `build_schema_parts` + extend).
Verify: e2e sống trên cả `metap-demo-crm`/`metap-demo-jira` xác nhận refactor không đổi behavior
schema generic.

## Phase 3 — `waf-graphql-gateway`, binary GraphQL riêng của WAF

`data-plane/graphql-gateway/` từ chỉ có `.env` (chạy binary generic của `metap`) thành 1 binary
Rust riêng dựng trên `metap-graphql-gateway` làm library, gọi `build_with_extensions` để thêm 8
field custom — mọi resolver là proxy mỏng (forward token caller, gọi thẳng REST endpoint đã có sẵn
ở Phase 71, trả JSON verbatim, không viết lại business logic):

| Field | Kiểu | REST gốc |
|---|---|---|
| `verifyZoneDns`/`testZoneOrigin`/`syncZoneConfigState` | Mutation | zones-service |
| `runScanJob` | Mutation | scanning-service |
| `testAlertPolicy`/`correlateIncidents`/`evaluateAlerts` | Mutation | alerting-service |
| `aggregate(entity, spec)` | Query | routed theo entity — Phase 70 chưa từng thêm `aggregate` vào `RecordBackend`/gRPC proto nên phải proxy REST như 7 field kia, không đi được đường gRPC generic |

`aggregate` là Query dù REST gốc là `POST` — về ngữ nghĩa vẫn là read (cùng cổng `AuthContext` REST
`list` dùng).

**Verify sống**: cả 3 service + gateway chạy thật trên Postgres/RabbitMQ, boot log xác nhận
`entities=9`. Gọi đủ 8 field qua `/graphql` bằng token đăng nhập thật (không phải service account)
— mọi field trả đúng kết quả REST endpoint gốc. `cargo build/clippy -D warnings/fmt --check` sạch.
Chi tiết đầy đủ ở `data-plane/graphql-gateway/README.md`'s "Đã verify sống".

## Phase 4 — `data-plane/web/src/api/waf.ts` chuyển transport

Ràng buộc: giữ nguyên tên/chữ ký mọi hàm export — 130+ call site ở `pages/*.tsx` (15 trang) không
được đổi gì, chỉ đổi phần thân file này.

Vấn đề kỹ thuật thật gặp phải: REST trả nguyên `data: {...}` JSON cho mọi field entity, còn GraphQL
bắt buộc chọn field tường minh trong selection set — mà `useRecords<T>`/`useRecord<T>` generic
theo `T`, không biết field nào ở call site (kiểu TS bị erase lúc runtime). Giải quyết bằng cách lấy
field list thật của entity qua `GET /metadata/entities/{entity}` (cố tình **để lại REST** — đây là
schema reflection tĩnh cache `staleTime: Infinity`, không phải business data đang thay đổi, cùng
nhóm ngoại lệ với `/auth/*`) rồi dựng selection set từ đó — `useRecords`/`useRecord` qua
`useEntity` (hook, cache chung với mọi consumer khác của platform-ui), còn 3 hàm mutation không
phải hook (`createRecord`/`updateRecord`/`transitionRecord`) qua `fetchEntityFields` (Promise cache
riêng, vì không gọi được hook ngoài React). Field `reference` cần sub-selection (`fieldName { id
}`, vì type GraphQL của nó là Object chứ không phải scalar) — `reshapeRecord` unwrap lại về đúng
UUID string phẳng REST từng trả.

9 field custom (7 action + `aggregate`) không cần field discovery — mỗi resolver trả nguyên JSON
response gốc của REST endpoint làm giá trị scalar `Json`, nên type `{data: ...}` cũ trong `waf.ts`
(`DnsVerifyResult`/`OriginTestResult`/...) không đổi 1 byte.

Phụ thuộc thêm ở `platform-ui` (`metap-org/platform-ui#3`): `graphqlNaming.ts` trước đó chỉ có
`listFieldName` (đủ cho `RelatedRecordsPanel`) — thêm `typeName`/`getFieldName`/
`createFieldName`/`updateFieldName`/`deleteFieldName`/`transitionFieldName` (mirror đầy đủ
`metap-graphql::naming`), và export module này ra `index.ts` (trước chỉ import relative được từ
trong chính package). Tiện thể sửa 2 doc comment cũ nói GraphQL gateway "chỉ query, mutation không
an toàn" — sai từ khi identity propagation được thêm 2026-09-02, `waf.ts` giờ là ví dụ mutation
CRUD đầy đủ đầu tiên qua đường này.

**Verify**: `tsc -b`/`oxlint`/`prettier --check`/`vite build` sạch trên `data-plane/web`. GraphQL
query/mutation shape (tên field, tên arg, selection set) đối chiếu trực tiếp với `metap-graphql`'s
`schema.rs`/`naming.rs` — cùng source sinh schema thật gateway đang chạy, không phải suy đoán.
**Chưa browser-test** — đúng "Frontend verification policy" của `metap`'s `CLAUDE.md` (không tự
browser-automation, viết code + typecheck/lint xong là báo cáo cho user tự kiểm).

## Còn lại

Không còn phase nào trong kế hoạch ban đầu (4/4 xong). User tự mở trình duyệt xác nhận là bước tiếp
theo hợp lý — chưa ai làm việc đó. `GeneratedList`/`RecordDetail` (nếu `metap-demo-waf` từng dùng
màn hình CRUD generic của `platform-ui`, xem mục "Raw records" ở Phase 71) vẫn ở REST theo đúng quy
ước hiện có của `platform-ui` — không nằm trong yêu cầu lần này, chỉ `api/waf.ts` mới đổi.
