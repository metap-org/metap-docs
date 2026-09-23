## Phase 93: `platform-ui`'s generated UI chuyển sang GraphQL — REST entity CRUD đã hỏng thật (2026-09-24)

Trigger: rà lại roadmap sau Phase 90-92, chủ dự án hỏi "phần FE platform-ui đã update theo GraphQL
chưa hay vẫn REST" — kiểm tra sống lộ ra **`GeneratedList`/`GeneratedForm`/`RecordDetail`/
`WorkflowActionBar` (repo `../platform-ui`) vẫn gọi REST `/api/:entity*`, route đã bị xoá hoàn
toàn khỏi `metap` core từ Phase 90**. Xác nhận sống: `curl http://localhost:3000/api/waf.zones`
trên `metap-demo-waf`'s `zones-service` (đang chạy thật) → `404`; `ZonesPage.tsx`/
`ZoneDetailPage.tsx` (WAF portal thật) dùng đúng 3 component đó — nghĩa là UI xem/sửa Zone trên
portal WAF đang gãy thật sự, dù health-check server vẫn xanh (health check không test entity
CRUD). Đây là gap nghiêm trọng hơn các nhóm REST khác còn treo (`/admin/*`...) vì nó chặn đường
CRUD chính (không phải phụ), và không ai phát hiện trước đó vì verify Phase 90 chỉ chạy qua
Rust integration test (GraphQL/gRPC), chưa từng chạy qua chính FE dùng chung này.

### Có sẵn 1 nửa, chỉ chưa được dùng

`platform-ui/src/api/graphqlRecords.ts` — bộ hook GraphQL generic (list/get/create/update/delete/
transition/aggregate) đã tồn tại từ trước (rút ra từ `metap-demo-waf`'s `waf.ts`), nhưng đúng theo
doc-comment gốc của chính nó: *"`GeneratedList` and friends are unaffected — they stay on REST"* —
viết trước Phase 90, lúc đó đúng (REST còn sống), giờ stale. Chưa từng được wire vào 3 component
generated UI chính.

### Việc đã làm

Chuyển `GeneratedList.tsx`/`GeneratedForm.tsx`/`RecordDetail.tsx`/`WorkflowActionBar.tsx` sang
dùng `graphqlRecords.ts` làm data layer duy nhất cho entity record — không còn lựa chọn REST/
GraphQL, chỉ 1 đường. Mở rộng `graphqlRecords.ts` để đủ thay REST hoàn toàn:

- `recordSelection`/`reshapeRecord` chọn thêm `capabilities` (envelope field có sẵn ở backend,
  trước đó không được select) và, với field `reference` có `refDisplayField`, chọn kèm field hiển
  thị để dựng lại `relatedDisplay` (batch reference label REST từng trả qua
  `RecordDto.related_display`) — verify sống: `project { id name }` trả đúng
  `{id, name: "..."}` từ `jira-server` thật.
- `useInfiniteGraphQLRecords`/`fetchAllGraphQLRecords` (mới) — cursor-pagination + "export all"
  thay `useApiInfiniteQuery`/vòng lặp REST. GraphQL list field (`{entity}List`) đã nhận đủ
  `filter`/`sort`/`limit`/`cursor` từ phía backend (`metap-graphql`) từ trước, chỉ chưa ai truyền
  `sort`/`cursor` qua phía FE.
- `GraphQLError` (mới, `api/graphqlClient.ts`) — mirror đúng shape `ApiError` REST
  (`code`/`status`/`fieldErrors`), đọc từ `extensions` GraphQL server trả (`metap-graphql`'s
  `service_result_to_gql`) — verify sống: lỗi validation thật trả đúng
  `extensions.fieldErrors`. Mọi chỗ check `error.code === "record_referenced"`/`error.fieldErrors`
  chuyển sang `instanceof GraphQLError` mà không đổi logic hiển thị.

### Đánh đổi có chủ đích

`GeneratedForm`'s update mutation **mất optimistic update** (REST bản cũ có, GraphQL bản mới
không) — cache của `useGraphQLQuery` giữ nguyên shape response thô từ GraphQL (`select` chỉ
reshape lúc đọc, không ghi lại vào cache), nên sửa tay cache thô lúc optimistic sẽ phải undo hết
`reshapeRecord` rồi tự dựng lại đúng shape server trả — rủi ro cao hơn giá trị UX 1 nhịp instant-
update mang lại. Thay bằng invalidate-sau-khi-thành-công, cùng tradeoff
`useInvalidateGraphQLRecords`'s doc-comment gốc đã chấp nhận cho `GeneratedList` ("stale 1 nhịp
còn hơn sửa cache thủ công dễ vỡ").

### Verify

`pnpm typecheck`/`lint`/`format:check` sạch trong `platform-ui` (không đụng 8 file khác đang có
prettier drift sẵn có từ trước — xác nhận qua `git stash` không phải do phiên này gây ra). Verify
sống qua `jira-server` thật (Postgres thật, không mock):
- List query đúng shape mới (`sort`/`cursor`/`capabilities` trong selection) → chạy sạch, trả
  đúng `nextCursor`/`hasMore`.
- Tạo record thật (`JiraProjects`), tạo tiếp record tham chiếu tới nó (`JiraIssues.project`), query
  list với `project { id name }` → đúng cả `id` lẫn display name.
- `capabilities` xác nhận **`null` trên `list`, có giá trị thật trên `get`** — hành vi backend
  đúng như kỳ vọng (tính capability mỗi record tốn kém, chỉ tính khi lấy 1 record); `GeneratedList`
  không đọc field này nên không ảnh hưởng.
- Lỗi validation thật (thiếu field bắt buộc) trả đúng `extensions.fieldErrors`.

**Chưa verify qua browser thật** — đúng chính sách repo (không tự verify FE bằng browser
automation, viết code + typecheck/lint rồi bàn giao). Chủ dự án cần tự mở `metap-demo-waf`'s
portal (hoặc app khác dùng `platform-ui`) trên browser để xác nhận UI thật render đúng trước khi
coi việc này hoàn toàn xong.

### Còn nợ / không làm trong phase này

- Chưa test qua browser thật (như trên).
- `useGraphQLRecords` (bản list phẳng, không infinite-scroll — dùng cho dashboard/widget) chưa
  được thêm `sort` param như bản infinite mới — không cần cho phiên này, có thể bổ sung khi có nhu
  cầu thật.
- Các app khác dùng `platform-ui` (`metap-demo-waf`, `metap-demo-crm` deprecated) cần tự
  `pnpm install`/rebuild để nhận thay đổi này (thư viện `link:` cục bộ, không phải version
  publish) — chưa chủ động đi rebuild/verify từng app tiêu thụ trong phiên này ngoài
  `metap-demo-waf`'s dev stack đang chạy sẵn (dùng `link:` nên tự nhận code mới ngay, không cần
  bước riêng).
