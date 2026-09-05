# Nâng cấp Frontend Platform — vượt khỏi demo API

- **Trạng thái:** done-partial — vòng dogfood thật đầu tiên (2026-09-05/06, xem "Scoping thật, lần
  đầu" bên dưới) đã đóng phần lớn 4 gap gốc; 1 gap thật còn lại (cron `targetConfig`), 1 gap chưa
  rõ còn cần không (bộ chuyển tenant)
- **Người đề xuất:** phản hồi từ việc tự test `apps/crm-fe` sau khi Phase 7 xong (2026-08-10)
- **Track sở hữu:** Frontend Platform (có thể kéo theo Backend Core)
- **Phase roadmap liên quan:** chưa gắn — cần scope trước khi gắn vào một phase cụ thể

## Vấn đề / động lực

`apps/crm-fe` hiện chỉ đủ để chứng minh API hoạt động (login, list/form generic, admin kit tối
giản từ Phase 15) — không đủ để thực sự dùng thử 4 entity mới của Phase 7 một cách có ý nghĩa.
Tự test cho thấy UI quá đơn giản, chưa lộ ra được các khả năng thật của platform (list view thứ
hai, workflow nhiều nhánh, field kiểu Reference/Money hiển thị ra sao).

**Đây không phải "chưa có trigger" như 4 ý ở mục dưới** — có trigger thật (dogfooding vừa lộ ra
gap), chỉ là *chưa được scope* để biết chính xác cần làm gì.

## Phạm vi

**Chưa chốt — cần một vòng scoping riêng trước khi implement.** Ghi lại ở đây các gap cụ thể đã
biết tính đến 2026-08-10, để vòng scoping đó có điểm bắt đầu thay vì từ số 0:

- ~~List view thứ hai (`ledger` của `accounting.journal`) hiện không gọi được qua API list~~ —
  **Đã xong (2026-08-22)**: `plan_list` (`crates/metap-query/src/query_planner.rs`) nhận thêm
  `ListInput.list_view: Option<String>`, route `GET /api/:entity?listView=<name>`
  (`crates/metap-http/src/routes/records.rs`) truyền qua tham số query cùng tên. Không truyền
  giữ nguyên hành vi cũ (`list_views.first()`); tên không tồn tại trả `400 unknown_list_view`
  (không âm thầm fallback về view mặc định — đổi view mà không báo lỗi dễ khiến FE hiển thị nhầm
  cột/filter mà không biết). 2 test e2e mới
  (`crates/metap-query/tests/query_planner_postgres.rs`), verify sống qua HTTP thật trên
  `accounting.journal`'s `ledger`. 3 gap còn lại (pagination admin UI kit, render field
  Reference/Money/Date, điều hướng entity trong demo app) cần nhìn qua browser thật để scope —
  ngoài khả năng tự verify của backend track.
- Admin UI kit (Phase 15) chưa có pagination trên list nào, `PolicyCondition`/cron
  `targetConfig` là raw-JSON thay vì structured builder, chưa có bộ chuyển tenant.
- Chưa rõ `GeneratedList`/`GeneratedForm` hiện render field `Reference`/`Money`/`Date` (3 kind
  mới toanh sau Phase 7) đẹp tới đâu — chưa từng nhìn qua browser thật với 4 entity mới.
- Chưa có cách điều hướng giữa các entity dễ dàng trong demo app (`EntitiesPage` hiện có nav
  tĩnh, chưa rõ có tự động liệt kê entity mới hay không).

**Vì sao ghi chú "phase này chắc sẽ nặng, kéo theo core":** ít nhất gap đầu tiên (list view
selection) đã xác nhận cần sửa `crates/metap-query`/`crates/metap-http`, không chỉ
`packages/platform-react`/`apps/crm-fe`. Nhiều khả năng còn phát sinh thêm khi scoping kỹ hơn
các gap còn lại — không giả định trước khi thực sự scope.

## Scoping thật, lần đầu (2026-09-05/06)

Brief này đứng yên ở `proposed` từ 2026-08-10 vì đúng như ghi ở trên — "chưa được scope", tức chưa
ai thực sự chạy `crm-fe` thật trong browser để đối chiếu 4 gap còn lại. Lần này làm thật (chạy
`crm-server` + `crm-fe` thật, seed `crm.customers`/`sales.orders` — entity `sales.orders` có đủ cả
3 kind `reference`/`date`/`money` cần soi), kết quả từng gạch đầu dòng ở "Phạm vi":

- **Pagination admin UI kit** — **hoá ra đã có sẵn** (`GeneratedList` dùng `@tanstack/react-virtual`
  + cursor `nextCursor`), không phải gap thật nữa — chắc đã được thêm ở một feature sau ngày viết
  brief này (`docs/features/00-index.md`'s 19-28, 2026-09-05) mà chưa ai quay lại đối chiếu.
- **`PolicyCondition` raw-JSON** — cũng **đã có structured builder** (`ConditionBuilder`,
  `AdvancedPoliciesPanel.tsx`) — không còn là gap. **Cron `targetConfig` thì vẫn còn raw-JSON
  `Textarea`** (`CronJobsAdminPage.tsx`) — gap thật duy nhất trong nhóm này còn sống.
- **Bộ chuyển tenant** — chưa xác nhận còn cần không: model hiện tại 1 JWT gắn với đúng 1 tenant,
  đổi tenant nghĩa là đăng nhập lại — chưa rõ đây có còn là nhu cầu thật hay là giả định cũ từ lúc
  viết brief. Để nguyên `proposed`, không tự quyết.
- **Render `Reference`/`Money`/`Date` trong `GeneratedList`** — **tìm ra 2 bug thật nghiêm trọng
  chưa ai từng thấy** (chính xác đúng điều brief này lo ngại — "chưa từng nhìn qua browser thật"):
  1. Dòng virtualized đè lên nhau (`ROW_HEIGHT` ước lượng 40px, dòng thật cao ~52px, không bao giờ
     đo lại) — sửa bằng `measureElement` của `@tanstack/react-virtual`.
  2. Cột dữ liệu lệch hẳn khỏi header, dồn hết sang trái — nguyên nhân là giới hạn thật của HTML
     `<table>`: một `<tr>`/`<td>` bị `position: absolute` (bắt buộc để virtualize) bị trình duyệt
     "fix up" bằng cách bọc riêng từng ô vào 1 bảng ẩn danh, cắt đứt khỏi lưới cột thật — có đặt
     `width` tường minh cũng không cứu được. Sửa tận gốc bằng cách chuyển hẳn sang CSS Grid
     (`role="table"/"row"/"cell"`) thay `<table>` thật.

  Nhân tiện tìm thêm 1 bug không nằm trong danh sách gốc: `WorkflowDiagram` render node đen kịt,
  không thấy chữ — do `metap-demo-crm/web/tailwind.config.cjs` (và `metap-demo-jira` y hệt) copy
  path từ `metap-demo-waf` sai độ sâu (`../../../platform-ui`, 3 cấp, trỏ ra ngoài `metap-org`,
  không tồn tại) nên Tailwind chưa từng quét được source `platform-ui` — class nào chỉ dùng trong
  đó (như `fill-primary`/`stroke-foreground`) không có CSS. Sửa lại đúng 2 cấp.

- **Điều hướng entity** — xác nhận đúng gap ("chỉ có 1 link 'Entities' quay về trang liệt kê, không
  có nav trực tiếp"). Sửa: `App.tsx`'s `navItems` build động từ `useEntities()`, mỗi entity 1 mục
  nav.

Tất cả các fix trên đã lên `platform-ui` (`GeneratedList.tsx`, `NavigationContext.ts`,
`resources.ts`) + `metap-demo-crm`/`metap-demo-jira` (`App.tsx`, `tailwind.config.cjs`), verify
`tsc`/`oxlint` sạch + dogfood sống qua browser thật (không phải chỉ đọc code).

**Còn lại thật sự cần làm, nếu muốn đóng hẳn brief này:**
1. Cron `targetConfig` → structured builder (như `ConditionBuilder` đã làm cho policy) — chưa
   làm, chưa ước lượng độ lớn.
2. Bộ chuyển tenant — cần hỏi lại có còn là nhu cầu thật không trước khi làm.

## Tiêu chí chấp nhận

Chưa có — sẽ điền khi feature này chuyển `proposed` → `approved`, sau một vòng scoping riêng
(xem "Phạm vi" ở trên).

## Ranh giới kiến trúc bị đụng tới

Chưa xác định đầy đủ — ít nhất gap list-view-selection đụng `metap-query`/`metap-http`
(`crates/metap-query/src/query_planner.rs`'s `plan_list`, route `/api/:entity`), có thể cần
ADR nếu đổi shape `ListInput`/response.

## Rủi ro / phụ thuộc

Không phụ thuộc Phase 7 (đã đóng). Track sở hữu chéo Frontend Platform + Backend Core — nếu
approve, cần sign-off cả hai track theo `docs/team-charter.md`.
