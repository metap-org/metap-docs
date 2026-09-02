## Phase 62: `RelatedView` metadata + `RelatedRecordsPanel` — thay `ZoneOverviewPage` viết tay bằng khai báo (2026-09-01)

Trigger: sau khi build `ZoneOverviewPage.tsx` (Phase 61's T6), chủ dự án hỏi thẳng "query không tự
parse từ entity à mà phải code ở service?" — xác nhận đúng tinh thần low-code platform: hạn chế
code tay, cần 1 component generic đọc từ metadata, giống hệt `GeneratedList`/`RecordDetail` đã
generic cho list/detail 1 entity.

**Research trước khi lên plan** (không đoán, đọc code thật): `metap-dashboards` chỉ là kho lưu
JSON blob opaque (`DashboardConfig { layout: serde_json::Value }`, không có khái niệm widget/
entity nào). Widget catalog của `jira-fe`'s `CustomizableDashboardPage.tsx` hard-code cứng vào
`jira.issues` (`BarChartWidgetConfig`/`StatTileWidgetConfig` không có field `entity`), không tái
dùng được. `metap-metadata`'s `EntityDefinition` chưa có khái niệm quan hệ ngược (chỉ `Reference`
xuôi con→cha). `metap-graphql`'s schema sinh tự động chỉ 1 chiều (không tự sinh field cha→con).
Không có gì sẵn để tái dùng — cần khái niệm metadata mới thật.

## Thiết kế

**`RelatedView`** (`metap-metadata/src/entity.rs`) — struct mới: `name`/`label`/`entity`/
`filterField`/`fields`/`limit`. Mô tả "hiển thị record liên quan từ entity khác, lọc theo field X
= id record hiện tại".

**Phát hiện quan trọng lúc thiết kế, tự sửa trước khi code** (không phải lỗi ai chỉ ra): định
thêm `related_views: Vec<RelatedView>` thẳng vào `EntityDefinition` — grep thử mới thấy
`EntityDefinition { ... }` được construct bằng struct literal (không dùng `..Default::default()`)
ở **~50 file trên toàn bộ org** (mọi `*_entity.rs`, mọi test fixture, ở cả `metap`/`metap-lowcode`/
`metap-demo-crm`/`metap-demo-jira`/`metap-demo-waf`). Thêm field bắt buộc kiểu đó sẽ phá compile
tất cả 50 file cho 1 capability chỉ 1 entity dùng. **Đổi thiết kế**: `RelatedView` đăng ký riêng
qua macro `submit_related_views!` — đúng cơ chế `inventory` mà `submit_entity!` đã dùng, nhưng là
1 submission type khác (`RelatedViewsFactory`), lưu vào 1 map riêng trong `MetadataRegistry`
(`related_views: HashMap<String, Vec<RelatedView>>`), không đụng `EntityDefinition` — 0 file nào
trong ~50 file kia cần sửa. `MetadataRegistry::register_all_submitted()` gom cả 2 loại
submission trong 1 lời gọi, giữ đúng lời hứa "khai báo 1 chỗ, tự động phát hiện" của
`submit_entity!`.

**Cố tình không validate cross-entity** (`compiler.rs::validate` không đụng tới
`entity`/`filterField`/`fields`) — đúng tiền lệ `FirewallRule.matchCondition`. Lý do: validate
cần `EntityDefinition` đầy đủ của entity đích trong CÙNG registry — đúng vấn đề route-leak Phase
61 đã gặp. Hệ quả thật: `fields` phải liệt kê tay (không suy ra "toàn bộ field" được vì không có
metadata đích để đọc) — không phải thiếu sót, là hệ quả không tránh được của việc giữ an toàn
cross-service.

**`RelatedRecordsPanel`** (`platform-ui/src/detail/RelatedRecordsPanel.tsx`, mới) — nhận
`id`/`relatedViews`, tự build **1 query GraphQL gộp** (1 fragment/related view, alias theo
`name`, field name tự tính qua `graphqlNaming.ts`'s `listFieldName` — mirror chính xác
`metap-graphql/src/naming.rs`'s thuật toán, ghi rõ comment "đổi 1 bên phải đổi bên kia"), gửi qua
`useGraphQLQuery` (thêm ở Phase 61's T6). Wire vào `RecordDetail.tsx`: entity có
`relatedViews.length > 0` thì tự hiện panel — **0 code thêm cho entity mới khai `related_views`**.

**Consumer chứng minh**: `zone_entity.rs` khai `zone_related_views()` (DdosPolicy/FirewallRule/
ScanJob/Incident — đúng nội dung `ZoneOverviewPage` từng hard-code). Sau khi verify
`RecordDetail` tự hiển thị đúng qua route có sẵn `/records/waf.zones/:id` — **xoá hẳn
`ZoneOverviewPage.tsx` + route `/zones/:zoneId/overview` + link riêng ở `EntitiesPage`** — đúng
minh chứng "trang React viết tay → khai báo metadata".

## Xác minh

`metap-metadata`: 43 unit test pass (4 test mới cho `submit_related_views!`/
`register_all_submitted`/`get_related_views`, gồm test xác nhận related view KHÔNG cần entity
đích registered). `cargo build/clippy -D warnings/fmt --check` sạch toàn `metap` — xác nhận
0 trong ~50 file `EntityDefinition { ... }` nào cần sửa (đúng thiết kế). `platform-ui`:
`typecheck`/`lint`/`format:check` sạch, `pnpm generate:types` chạy thật (trỏ `zones-service`
đang chạy) xác nhận `relatedViews` xuất hiện đúng trong `generated-types.ts`.
`metap-demo-waf/data-plane`: `cargo build/clippy/fmt` sạch cả 3 service, `web/`'s `tsc -b`/
`oxlint`/`prettier --check` sạch sau khi xoá `ZoneOverviewPage`.

**Verify sống qua HTTP thật** (không phải suy luận từ đọc code): `GET /metadata/entities/waf.zones`
trả đúng `relatedViews` (4 entry). Đúng query mà `RelatedRecordsPanel` tự build — gộp 4 related
view từ cả 3 service trong 1 request — chạy qua cả `graphql-gateway` trực tiếp lẫn qua `web/`'s
dev-server `/graphql` proxy, trả đúng dữ liệu thật. Không tự browser-test (đúng frontend
verification policy).
