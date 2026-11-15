# 112. Shell sidebar, list gọn, audit log phân trang

**Trạng thái:** làm xong 2026-11-15 trên nhánh `claude/metap-docs-features-g1zjqn` (metap, design-system, platform-ui, waap), cùng đợt với [111](111-list-paging-search-widgets.md). Frontend chưa kiểm tra trên trình duyệt (theo quy ước FE: người dùng tự xem).

## Vấn đề (chủ dự án nêu)

- Topbar chứa toàn bộ menu: hết chỗ khi sản phẩm có hơn vài mục; muốn sidebar trái kiểu Cloudflare.
- Màn list tốn diện tích: hàng ~52px, giới hạn `max-w-7xl`, tiêu đề to.
- Audit không phân trang và tốn chỗ: API trả cả lịch sử (tối đa 200) một lần, mỗi sự kiện chiếm 3–5 dòng. Tab audit của zone gọi 1 request cho mỗi bản ghi con rồi gộp ở trình duyệt, và mất lịch sử của con đã xoá.

## Shell và list

- design-system: `Sidebar` (`SidebarHeader/Content/Footer/Group/Item`), thu gọn thành cột icon (nhãn còn cho screen reader, `title` khi hover, chữ viết tắt khi không có icon); `SidebarItem asChild` để routing vẫn là của consumer.
- platform-ui `AppShellLayout`: sidebar trái (trạng thái thu gọn nhớ trong `localStorage`), sheet trái trên mobile, topbar 48px chỉ còn chọn tenant + menu tài khoản (email, role, ngôn ngữ, đăng xuất). `ShellNavItem` thêm `group`, `icon`; `NavigationAdapter.pathname` (tuỳ chọn) để đánh dấu mục đang mở.
- `GeneratedList`: bỏ `max-w-7xl`, tiêu đề `text-lg`, ô `py-1.5` (hàng ~40px), header 36px, nút trên hàng 28px, khung cuộn cao theo layout mới.
- waap: menu chia nhóm Tổng quan / Bảo vệ / Vận hành / Quản trị kèm icon; bỏ "Raw records" (trang chọn entity `/records`) và "Entities (Low-code)" khỏi menu và route. Route `/records/:entity/...` vẫn giữ vì nút "xem" của list dẫn tới đó.

## Audit

- `EntityAuditConfig.Parent`: tên một field reference (có `refEntity`) trỏ tới bản ghi sở hữu, vd. `zoneId`; `Validate` từ chối field không phải reference. Mỗi entry audit ghi thêm `parent_entity`/`parent_id` (migration `0039_audit_trail_parent.sql`, cột nullable trên bảng partition cha, index `(tenant_id, parent_entity, parent_id, occurred_at)`).
- `audit.Store.Query` thay `ListForRecord`: lịch sử của một bản ghi, tuỳ chọn cả con (`ChildEntities`), lọc theo entity/hành động/người/khoảng thời gian, keyset `(occurred_at, id)`.
- `GET /api/{entity}/{id}/audit-events?includeChildren=true&entities=&actions=&actor=&from=&to=&cursor=&limit=` trả `{data, page: {nextCursor}}` (mặc định 50, tối đa 200). Con chỉ gồm entity khai báo `Audit.Parent` trỏ về entity này và người gọi có quyền đọc cấp entity; diff của mỗi entry vẫn được che theo quyền đọc field trên bản ghi của chính entry đó.
- Lịch sử cũ: `dev-tools backfill-audit-parents <entity.json>...` điền parent từ cột reference hiện tại của bản ghi (chỉ DB dùng chung; bản ghi đã bị xoá cứng thì giữ NULL). Đã chạy trên DB dev: 4 entry của `waf.ddos_policies`.
- platform-ui `AuditLog`: bảng gọn (thời gian tương đối, người, hành động, đối tượng, các trường đổi), bấm dòng để xem diff Trường | Trước | Sau, lọc hành động/người/đối tượng, "Tải thêm". `AuditTrail` dùng nó; `useAuditFeed` là hook phân trang.
- waap: DdosPolicy, FirewallRule, IpAccessList khai báo `Audit.Parent: zoneId`; `ZoneAuditTab` còn 1 request phân trang (trước: 3 list + 1 request mỗi bản ghi con).
- Test: `TestQueryIncludesChildrenPagesAndFilters`, `TestBackfillParentsFillsFromTheRecordsParentColumn` (audit), `TestAuditParentHistoryIncludesChildrenAndPages` (crud, gồm con đã xoá và cursor sai), `TestAuditParentMustBeAReferenceWithRefEntity` (metadata).

## Còn lại

- ScanJob/Incident có `zoneId` dạng chuỗi (khác service), không phải reference, nên chưa vào lịch sử zone.
- `AuditTrailList`/`useAuditEventsForRecords` (gộp ở client) vẫn export cho app cũ, waap không còn dùng.
- Bộ lọc khoảng thời gian đã có ở API, chưa có ô chọn trên `AuditLog`.
- Các trang waap khác còn tự dựng layout riêng (độ dày chưa đồng bộ với `GeneratedList`).
