## Phase 77: `metap-demo-waf` — gộp GraphQL request, workflow diagram tương tác, bản dịch tiếng Việt (2026-09-05)

Trigger: 3 yêu cầu liên tiếp trong 1 phiên làm việc trên WAF portal, ngay sau Phase 76's live-bugfix
session — (1) hỏi thẳng Dashboard có đang bắn nhiều request GraphQL song song không và có gộp được
không, (2) muốn workflow diagram (Phase 76's "Visualize workflow") hỗ trợ zoom/pan/kéo node giống
Jira, (3) nhận ra WAF chưa có mode eng/vi nào dù `platform-ui` đã có sẵn hạ tầng i18n.

## Đã thêm

- **GraphQL batch-request support** — `metap`'s `crates/graphql-gateway/src/server.rs` chuyển từ
  `GraphQLRequest`/`GraphQLResponse` sang `GraphQLBatchRequest` + `Schema::execute_batch`
  (`async-graphql` có sẵn — chấp nhận cả object `{query,variables}` đơn lẫn mảng trong cùng 1 POST
  body, `#[serde(untagged)]` nên trả về đúng dạng đã gửi) — backward-compatible 100% với mọi caller
  hiện tại (CRM/Jira/WAF), không cần đổi gì phía họ. `with_request_data` (fresh `DataLoader` mỗi
  request) áp cho từng phần tử của batch riêng lẻ, không share, giữ đúng ngữ nghĩa cũ.
  `platform-ui/src/api/graphqlClient.ts` thêm hàng đợi theo microtask: mọi `graphqlFetch` gọi
  trong cùng 1 tick (cùng `path`+`token`) được gộp thành 1 request mảng, response mảng trả về theo
  đúng thứ tự tách lại cho từng promise; 1 lệnh gọi lẻ vẫn gửi nguyên object đơn như trước — không
  đổi wire format khi không có gì để gộp. Kết quả: Dashboard's 6 `useAggregate` + 2 `useRecords`
  giờ chỉ còn 1 round-trip HTTP thay vì 8, cache/loading state từng ô vẫn độc lập (react-query giữ
  `queryKey` riêng, chỉ tầng network bên dưới được gộp).
- **`WorkflowDiagram` tương tác** (`platform-ui/src/workflow/WorkflowDiagram.tsx`, thêm trên nền
  Phase 76's SVG tĩnh) — zoom in/out (nút +/−/reset, cộng con lăn chuột — zoom giữ đúng điểm dưới
  con trỏ đứng yên, công thức "zoom to cursor" chuẩn), pan toàn canvas (kéo nền, Pointer Capture),
  kéo-thả từng node riêng lẻ (offset đè lên layout BFS-cột gốc từ Phase 76, không persist — reset
  về mặc định mỗi khi đổi entity/mở lại dialog, chưa có chỗ lưu layout ở backend). Thêm highlight
  khi hover: hover 1 node sáng node đó cộng mọi node nối trực tiếp qua 1 edge cộng các edge đó;
  hover 1 edge sáng 2 node đầu-cuối; phần còn lại mờ opacity xuống. Mỗi edge có thêm 1 đường "vô
  hình" (`stroke="transparent"`, dày 12px) chồng lên đường thật 1.5px để vùng bắt hover đủ rộng.
- **WAF i18n (en/vi)** — `platform-ui`: export thêm `i18n`/`resources` ra `src/index.ts`'s public
  API để app con merge translation riêng vào cùng 1 i18next instance thay vì tạo instance thứ 2;
  `LocaleSwitcher` thêm prop `compact` (bỏ label hiển thị, giữ `aria-label`) và được mount thẳng
  vào `AppShellLayout`'s header — mọi app dùng `AppShellLayout` (CRM/Jira/WAF) giờ tự có nút đổi
  ngôn ngữ, trước đó **không app nào thực sự render `LocaleSwitcher`** dù cả 3 đều đã bọc
  `LocaleProvider` từ trước. `metap-demo-waf/data-plane/web`: thêm `src/i18n/resources.ts` (~400
  key mỗi ngôn ngữ, namespace `waf.*` để không đụng key gốc của `platform-ui`) + `src/i18n/register.ts`
  (merge vào instance chung qua `i18n.addResourceBundle`, import 1 lần cho side-effect ở
  `main.tsx` trước khi render) — thay toàn bộ text hardcode tiếng Anh trong `App.tsx`,
  `components/primitives.tsx` (`StatusBadge` tra `waf.status.*` với fallback về giá trị gốc,
  `TimeSeries`), và 10 trang + 5 tab zone bằng `t("waf....")`. Tiện thể sửa 3 chỗ hack nối chuỗi
  kiểu `` `Zone ${action}d` ``/`` `Incident ${action}d` `` (ghép "d" vào tên action để giả quá khứ
  — không dịch được sang tiếng Việt) thành bảng tra `action → translation key` đúng nghĩa, ở
  `ZoneDetailPage`/`IncidentDetailPage`.

## Xác minh

`cargo build -p metap-graphql-gateway`/`cargo test -p metap-graphql-gateway` sạch (metap core);
build lại `waf-graphql-gateway` (wrapper binary của `metap-demo-waf`) xác nhận đổi ở core không
phá tương thích. `tsc -b`/`oxlint`/`prettier --check` sạch cho `platform-ui` và
`metap-demo-waf/data-plane/web`; build lại `metap-demo-crm/web` riêng để xác nhận đổi
`AppShellLayout` không phá app khác đang dùng chung thư viện. **Chưa browser-test** (đúng frontend
verification policy — để người dùng tự kiểm zoom/pan/kéo-node/hover/đổi ngôn ngữ trên browser
thật), verify ở đây dừng ở mức build/typecheck/lint tĩnh.
