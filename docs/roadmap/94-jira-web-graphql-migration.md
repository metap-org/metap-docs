## Phase 94: `metap-demo-jira/web` chuyển sang GraphQL — phát hiện thêm 1 app hỏng + 1 bug backend (2026-09-25)

Trigger: ngay sau Phase 93 (`platform-ui`'s generated UI), rà sâu thêm phát hiện `metap-demo-jira`
có **1 frontend hoàn toàn riêng** (`web/`, 10 file trang hand-written — Backlog/Board/Dashboard/
Sprint Report/Advanced Search/Logwork Report/Issue Detail + panel con) không dùng `GeneratedList`/
`GeneratedForm` của `platform-ui`, mà tự viết REST call thẳng `/api/jira.*` — cũng gãy y hệt WAF
portal, cùng nguyên nhân (Phase 90 xoá REST entity CRUD).

### Việc đã làm — 10 file, chuyển hết sang `@metap/platform-ui`'s GraphQL hooks

`BacklogPage`/`BoardPage`/`AdvancedSearchPage`/`DashboardPage`/`SprintReportPage`/`IssuePanels`
(AssigneePicker/WorklogsPanel/WatchersPanel)/`IssueDetailPage` (SubtasksPanel/CommentsPanel/
IssueLinksPanel)/`CustomizableDashboardPage`/`LogworkReportPage` — mọi `useApiQuery`/
`useApiMutation`/`apiFetch` thao tác entity record (`jira.projects`/`.sprints`/`.issues`/
`.comments`/`.watchers`/`.worklogs`/`.issue_links`) chuyển sang `useGraphQLRecords`/
`useGraphQLRecord`/`create|update|delete|transitionGraphQLRecord` (`platform-ui`, đã mở rộng ở
Phase 93). **Giữ nguyên REST** cho 2 route Phase 90 cố ý chừa lại — `AttachmentsPanel`
(`/api/jira.issues/:id/attachments*`) và `SprintReportPage`'s burndown
(`/api/jira.issues/:id/workflow-events`) — và cho `/dashboards/me`/`/dashboards/tenant-default`
(`CustomizableDashboardPage`, route `metap-config`/`metap-dashboards`, không đụng gì bởi Phase 90).

Mở rộng thêm `platform-ui`'s `useGraphQLRecords` (Phase 93 chưa có) để nhận `sort`/`jql` — cần cho
`CommentsPanel` (sort theo `-createdAt`) và 3 trang dùng JQL (xem dưới).

### Bug backend thật tìm ra, không phải chỉ vấn đề FE

3 trang (`AdvancedSearchPage`, `LogworkReportPage`, `CustomizableDashboardPage`'s stat tile) dùng
`?jql=` (`metap-query::jql`, ngôn ngữ query nhỏ REST đã hỗ trợ từ lâu qua `ListInput.jql`).
`metap-graphql/src/list_input.rs`'s `list_input_from_args` đã đọc `jql` từ args **từ lúc file
này được viết** — nhưng `schema.rs`'s `add_query_fields` **chưa từng khai báo** `jql` là argument
hợp lệ của field `{entity}List` (chỉ có `filter`/`sort`/`cursor`/`limit`/`listView`). GraphQL
validation từ chối bất kỳ argument nào field không khai báo ("Unknown argument") — nghĩa là `jql`
**chưa bao giờ dùng được qua GraphQL**, âm thầm, từ lúc GraphQL entity CRUD tồn tại, chỉ lộ ra khi
di chuyển 3 trang này khỏi REST. Sửa 1 dòng (`crates/metap-graphql/src/schema.rs`, thêm
`.argument(InputValue::new("jql", TypeRef::named(TypeRef::STRING)))`) — `metap-graphql-gateway` tự
động thừa hưởng fix vì dùng chung `build_schema_parts`, không có logic khai báo argument riêng.

### Gap hạ tầng thêm, không liên quan Phase 90 nhưng tìm ra cùng lúc

`web/vite.config.ts`'s dev-server proxy **chưa từng có `/graphql`** (app này chưa từng dùng GraphQL
trước đây) — đã thêm. Đồng thời phát hiện **`/dashboards` cũng thiếu từ trước**, không liên quan gì
Phase 90 — nghĩa là `CustomizableDashboardPage` chưa bao giờ chạm được backend của nó qua `pnpm
dev`, một bug độc lập tồn tại từ lâu, tiện tay sửa luôn vì cùng file.

### Verify

`tsc -b`/`oxlint`/`prettier --check` sạch cho `web/` (không đụng file có drift format sẵn có từ
trước). `metap` core: `cargo build/clippy(-D warnings)/fmt --check/test --workspace` sạch sau fix
`jql`. Verify sống toàn diện qua `jira-server` thật (Postgres thật): tạo project → issue → filter
theo reference field (`project`) → tìm đúng; comment tạo 2 dòng, list `sort: "-createdAt"` → đúng
thứ tự; watcher tạo/xoá; transition (`start` → `in_progress`); `jql` filter chạy không lỗi
validation; `AssigneePicker`'s pattern lấy version-rồi-update chạy đúng; xoá record đang bị tham
chiếu trả đúng lỗi `record_referenced` kèm `fieldErrors` (bonus verify, không cố ý tạo tình huống
này nhưng gặp thật lúc dọn dữ liệu test).

**Chưa verify qua browser thật** — đúng chính sách repo.

### Còn nợ / không làm trong phase này

- Chưa test qua browser thật.
- Vài record test (1 project, 1 issue, 2 comment) còn sót lại trong Postgres dev cục bộ do server
  bị tắt giữa chừng lúc dọn dẹp — vô hại (dev DB local), không dọn thêm trong phiên này.
- `useInfiniteGraphQLRecords` (Phase 93) chưa nhận `jql` — không cần cho phase này (mọi trang ở
  đây dùng list phẳng qua `useGraphQLRecords`, không infinite-scroll), bổ sung khi có nhu cầu thật.
