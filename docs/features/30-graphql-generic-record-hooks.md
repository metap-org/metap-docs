# `@metap/platform-ui` — bộ hook CRUD generic qua GraphQL (adapter dữ liệu cho UI tuỳ biến)

- **Trạng thái:** done (2026-09-05) — `platform-ui/src/api/graphqlRecords.ts` mới (7 export:
  `useGraphQLRecords`/`useGraphQLRecord`/`useGraphQLAggregate`/`createGraphQLRecord`/
  `updateGraphQLRecord`/`deleteGraphQLRecord`/`transitionGraphQLRecord` + `useInvalidateGraphQLRecords`,
  types `GraphQLRecord`/`GraphQLListResponse`/`GraphQLSingleResponse`/`AggregateSpec`/`AggregateRow`).
  `metap-demo-waf/data-plane/web/src/api/waf.ts` đã dogfood: xoá hết phần generic cục bộ, re-export
  lại đúng tên cũ (`useRecords`/`useRecord`/`useAggregate`/4 mutation) từ `@metap/platform-ui` —
  **0 trong 130+ call site ở `pages/*.tsx` phải đổi**, đúng phương án rẻ nhất đã ghi trong "Rủi ro".
  `tsc`/`tsc -b`/oxlint/prettier/`vite build` sạch ở cả 2 repo. Chưa commit/push — chờ xác nhận.
- **Người đề xuất:** thảo luận trực tiếp trong phiên làm việc — câu hỏi "low-code không tạo được
  layout tuỳ biến như WAF thì sao", dẫn tới ý: tách phần *fetch/gọi API* (thuần logic, entity-
  agnostic) ra khỏi phần *UI generated*, để 1 app muốn tự viết layout tuỳ biến vẫn dùng lại được
  tầng data mà không phải tự viết lại từ đầu.
- **Track sở hữu:** Frontend Platform (đọc/sửa chéo sang `metap-demo-waf` để dogfood)
- **Phase roadmap liên quan:** không thuộc phase nào

## Vấn đề / động lực

`platform-ui` đã có đúng loại "adapter thuần logic" này cho REST — `useApiQuery`/`useApiMutation`/
`useApiInfiniteQuery` — nhận `path`/`queryKey`, không biết gì về layout, entity nào cũng gọi được.
Nhưng khi `metap-demo-waf` chuyển sang GraphQL (Phase 73), nó phải **tự viết lại y hệt tầng đó**
trong `metap-demo-waf/data-plane/web/src/api/waf.ts`: `useRecords`/`useRecord`/`useAggregate`/
`createRecord`/`updateRecord`/`deleteRecord`/`transitionRecord` — đọc lại code xác nhận **cả 7 hàm
đều nhận `entity: string` làm tham số, tự build query GraphQL từ metadata (`useEntity`), không
hardcode field/entity nào của WAF** — về bản chất đã là 1 bộ hook tổng quát, chỉ đang mắc kẹt trong
1 app thay vì sống ở `platform-ui` để app khác dùng lại.

`graphqlClient.ts`'s doc comment tự ghi nhận điều này: *"metap-demo-waf's data-plane/web/src/
api/waf.ts is a full CRUD example"* — tức file đó vốn đã được coi là bằng chứng khả thi, chỉ chưa
ai tách ra.

**Đây là câu trả lời cho tension "low-code không tạo layout tuỳ biến được"**: low-code (và cả code
thường) vẫn luôn cần viết tay JSX cho 1 layout tuỳ biến — nhưng nếu tầng *lấy/ghi dữ liệu* generic
có sẵn, phần phải viết tay chỉ còn đúng layout, không phải viết lại cả tầng gọi API mỗi lần muốn 1
UI không theo khuôn `GeneratedList`/`GeneratedForm`.

## Phạm vi

**Trong phạm vi:**
- Thêm `platform-ui/src/api/graphqlRecords.ts`, build trên `useGraphQLQuery`/`graphqlFetch`/
  `graphqlNaming.ts` đã có sẵn (không viết lại phần transport/batching/naming — chỉ thêm đúng lớp
  "generic CRUD hook" đang thiếu):
  - `GraphQLRecord<T>`, `GraphQLListResponse<T>`, `GraphQLSingleResponse<T>` — đổi tên từ
    `WafRecord`/`ListResponse`/`SingleResponse` (bản chất là `RecordDto` generic, comment gốc của
    WAF đã tự ghi "Mirrors metap's RecordDto").
  - `recordSelection`/`reshapeRecord` (nội bộ, không export) — xây selection set GraphQL từ field
    list, undo lại thành `data` bag.
  - `useGraphQLRecords<T>(entity, filters?, limit?, enabled?, path?)`,
    `useGraphQLRecord<T>(entity, id, path?)` — `path` thêm mới, default `"/graphql"` (WAF hard-code
    hằng số này; app khác có thể route khác).
  - `AggregateSpec`/`AggregateRow` (type mới hoàn toàn ở `platform-ui`, chưa từng có REST-side
    tương đương ở đây dù backend đã generic từ Phase 70/75) + `useGraphQLAggregate(entity, spec,
    enabled?, path?)`.
  - `createGraphQLRecord`/`updateGraphQLRecord`/`deleteGraphQLRecord`/`transitionGraphQLRecord` —
    hàm imperative (không phải hook), y hệt chữ ký gốc + `path?` cuối cùng.
  - `useInvalidateGraphQLRecords()` — generic, key `["graphql-records"]`/`["graphql-record"]`/
    `["graphql-aggregate"]` (khác tiền tố `"waf-"` gốc, và khác `["records", ...]` REST đang dùng ở
    `GeneratedList`/`useApiInfiniteQuery`, để 1 app có cả 2 transport cùng lúc — như WAF's
    `/records/*` escape hatch REST — không đụng cache nhau).
- **Dogfood ngay**: `metap-demo-waf/data-plane/web/src/api/waf.ts` đổi 7 hàm generic trên sang
  import từ `@metap/platform-ui` thay vì tự định nghĩa — chỉ giữ lại phần thật sự đặc thù WAF:
  `ENTITIES` map, `ZoneData`/`Zone` type, `daysAgo`, và 6 custom action
  (`verifyDns`/`testOrigin`/`syncConfigState`/`runScanJob`/`testAlertPolicy`/`correlateIncidents`/
  `evaluateAlerts`).
- Đây là bằng chứng thật cho câu hỏi gốc: xác nhận adapter tổng quát đủ để 1 app tự viết layout
  tuỳ biến (như `ZoneDetailPage`) vẫn dùng lại được tầng data, không phải viết riêng.

**Ngoài phạm vi:**
- Không đổi `useGraphQLQuery`/`graphqlFetch`/`graphqlNaming.ts` — dùng nguyên trạng.
- Không tạo UI mới nào — feature này thuần tầng data.
- Không đổi 6 custom action của WAF — chúng thật sự đặc thù nghiệp vụ, không tổng quát hoá được.
- Không đổi cách `GeneratedList`/`GeneratedForm`/`RecordDetail` hoạt động (vẫn REST, không liên
  quan) — 2 tầng data (REST cho generated UI, GraphQL cho UI tuỳ biến) sống song song, không cần
  hợp nhất trong phạm vi này.

## Tiêu chí chấp nhận

- `platform-ui`: `tsc`/`oxlint`/`prettier --check` sạch. 7 hàm mới export từ `src/index.ts`.
- `metap-demo-waf/data-plane/web/src/api/waf.ts` sau khi đổi: không còn định nghĩa
  `useRecords`/`useRecord`/`useAggregate`/`createRecord`/`updateRecord`/`deleteRecord`/
  `transitionRecord`/`recordSelection`/`reshapeRecord`/`fetchEntityFields` cục bộ — import thẳng
  từ `@metap/platform-ui`, đổi tên gọi tại ~130 call site nếu tên hàm đổi (hoặc re-export cùng tên
  cũ ngay trong `waf.ts` để 0 call site phải sửa — quyết định lúc code, ưu tiên rẻ nhất).
- Không đổi hành vi/giao diện nào — hành vi mạng (request GraphQL gửi đi) giống hệt trước khi tách.
- `metap-demo-waf/data-plane/web`: `tsc -b`/oxlint/prettier/`vite build` sạch.

## Ranh giới kiến trúc bị đụng tới

`platform-ui/src/api/graphqlRecords.ts` (mới) + `metap-demo-waf/data-plane/web/src/api/waf.ts`
(giảm mạnh, chỉ còn phần đặc thù WAF). Đúng rule "UI+logic dùng lại được → platform-ui" — đây là
logic thuần (không JSX), vẫn đúng chỗ platform-ui vì đã tổng quát hoá theo entity, giống hệt
`useApiQuery`/`useApiMutation` đã có.

## Rủi ro / phụ thuộc

Đổi tên hàm (`useRecords` → `useGraphQLRecords` chẳng hạn) chạm ~130 call site trong
`metap-demo-waf` theo chính comment gốc của file đó ("130+ call sites... needed zero changes" khi
đổi REST→GraphQL trước đây) — nên cân nhắc giữ nguyên tên cũ qua re-export tại `waf.ts` thay vì bắt
130 chỗ đổi tên, trừ khi muốn tên nhất quán với REST-side ngay từ đầu. Không phụ thuộc feature
khác; độc lập hoàn toàn với #26-29.
