# Audit `metap-demo-waf` — component đặt đúng chỗ theo rule design-system/platform-ui chưa

- **Trạng thái:** done (2026-09-05) — audit đầy đủ tại
  `../../../platform-ui/docs/audits/03-waf-demo-component-placement-audit.md`. Quét 23/23 file
  `.ts`/`.tsx` dưới `data-plane/web/src/`, không sample. Kết quả: **8 finding thật** — 5 nhóm 1
  (nên chuyển `@metap/ui`: `PageHeader`/`EmptyState`/`SectionCard`/`StatTile`/`TimeSeries`, đều
  đã lặp lại 3-12 chỗ), 2 nhóm 2 rõ ràng (nên chuyển `@metap/platform-ui`: pattern
  busy+try/catch/toast lặp 18 lần/9 file, `WorkflowDiagram` dialog wrapper lặp 2 lần) + 1 nhóm 2
  mức ưu tiên thấp (`shortDate`/`dayLabel`, utility thuần), 1 nhóm 3 có ghi chú 2 chiều
  (`StatusBadge` giữ nguyên vì data là từ vựng WAF, nhưng lộ ra 1 gap thật ở
  `platform-ui/src/field/FieldValue.tsx:75-77`'s enum rendering không map tone theo giá trị).
  **Nhóm 4 (vi phạm import trực tiếp): 0 finding** — xác nhận sạch hoàn toàn. Chưa sửa code nào
  (đúng phạm vi) — feature nào làm tiếp, theo thứ tự nào, chờ chủ dự án quyết định.
- **Người đề xuất:** yêu cầu trực tiếp trong phiên làm việc — sau khi lên 6 feature brief từ
  `docs/frontend-checklist.md` (19-24), kiểm tra chéo xem app downstream mới nhất
  (`metap-demo-waf/data-plane/web`) có đang tuân thủ đúng ranh giới kiến trúc "UI → design-system
  (`@metap/ui`), UI+logic → platform-ui (`@metap/platform-ui`)" hay không.
- **Track sở hữu:** Frontend Platform (đọc chéo sang App/Entity-track's repo: `metap-demo-waf`)
- **Phase roadmap liên quan:** không thuộc phase nào — cùng tinh thần audit chủ động như Phase 59
  ("Rà soát atom lạc chỗ ở `platform-ui`")

## Vấn đề / động lực

Rule kiến trúc "không component UI tổng quát nào được viết riêng trong 1 app, mọi atom phải sống ở
`@metap/ui`, mọi composition UI+logic dùng lại được phải sống ở `@metap/platform-ui`" đã bị vi phạm
trong quá khứ ít nhất 1 lần (ghi lại ở `platform-ui/README.md`'s "Nguyên tắc kiến trúc", trích dẫn
ở root `CLAUDE.md`) và từng cần 1 đợt audit chủ động để dọn lại (Phase 59, "Rà soát atom lạc chỗ ở
`platform-ui` + thêm `FileUpload`").

`metap-demo-waf/data-plane/web` là app downstream mới nhất (build từ 2026-09-01), có IA riêng
10-module zone-centric (Zone/Incident/Finding/Alerting/Analytics...) — nhiều khả năng cùng lúc vừa
tự viết UI atom cục bộ cho nhu cầu riêng của WAF (badge trạng thái Zone, biểu đồ nhỏ ở tab
Analytics, timeline sự kiện...) thay vì tái dùng/đóng góp ngược, vừa có thể đã import thẳng 1 thư
viện ngoài (icon set khác, CSS module riêng) mà không qua `@metap/ui`. `docs/frontend-checklist.md`
(rà soát 2026-09-04) tự ghi rõ: **chưa từng audit bất kỳ consumer app cụ thể nào** ("Không có
consumer app nào... checkout cục bộ lúc rà soát này — mọi mục kiểm tra ở tầng thư viện"), nên đây
là lỗ hổng audit thật, không phải suy đoán.

## Phạm vi

**Trong phạm vi:**
- Quét toàn bộ `.tsx`/`.ts` dưới `metap-demo-waf/data-plane/web/src/` (không sample — liệt kê rõ
  đã xem hết bao nhiêu file trong báo cáo cuối).
- Với mỗi component/đoạn JSX đáng chú ý, phân vào 1 trong 4 nhóm:
  1. **Nên chuyển `@metap/ui`** — thuần trình bày, không biết business/entity cụ thể, có khả năng
     dùng lại ở app khác (ví dụ 1 kiểu badge trạng thái tổng quát, không riêng "Zone status").
  2. **Nên chuyển `@metap/platform-ui`** — có cả UI lẫn logic (gọi API, biết field/entity cụ thể)
     nhưng bản thân logic đó không đặc thù riêng WAF (ví dụ 1 cách hiển thị related-record chung
     chung, không phải "Zone" cụ thể).
  3. **Giữ nguyên tại app** — đặc thù nghiệp vụ WAF thật, không tổng quát hoá được có ý nghĩa (ví
     dụ biểu đồ DDoS-specific, layout riêng của 1 zone tab).
  4. **Vi phạm import trực tiếp** — dùng thẳng 1 thư viện UI ngoài (icon lib khác, CSS-in-JS khác,
     component Radix trần không qua `@metap/ui`'s wrapper) thay vì qua `@metap/ui`.
- Với nhóm 1 và 2: ghi rõ đã lặp lại **>= 2 chỗ thật** hay mới có **1 chỗ dùng duy nhất** — theo
  đúng kỷ luật "không trừu tượng hoá sớm" (root `CLAUDE.md`: "Three similar lines is better than a
  premature abstraction"), 1 chỗ dùng chưa chắc đáng chuyển ngay, chỉ ghi nhận để tự quyết định
  sau khi đọc audit, không tự động đề xuất chuyển mọi thứ tìm thấy.
- Sản phẩm cuối: 1 file audit tại `platform-ui/docs/audits/03-<slug>.md` (tiếp đúng số thứ tự sau
  `01-frontend-performance-audit.md`/`02-auth-permission-workflow-diagram-audit.md`), liệt kê
  từng finding theo `file:dòng`.

**Ngoài phạm vi:**
- Tự động sửa/di chuyển code — audit trước, sửa là 1 hoặc nhiều PR riêng sau khi thống nhất danh
  sách với chủ dự án.
- Audit `control-plane`/`edge-plane` của `metap-demo-waf` — không có frontend, không liên quan.
- Audit `metap-demo-crm`/`metap-demo-jira`'s web — khác trigger, có thể làm sau nếu cần, không gộp
  vào đây (giữ đúng 1 audit = 1 phạm vi rõ ràng).

## Tiêu chí chấp nhận

- File audit liệt kê **từng finding cụ thể theo `file:dòng`**, không mô tả chung chung kiểu "có vài
  chỗ chưa chuẩn".
- Mỗi finding có đúng 1 trong 4 phân loại ở "Phạm vi", kèm lý do ngắn.
- Báo cáo nói rõ đã quét bao nhiêu file `.tsx`/`.ts` trên tổng số file thật của
  `data-plane/web/src` (đếm bằng `find`/`git ls-files`, không phải ước lượng) — chứng minh là audit
  đầy đủ, không phải xem vài file rồi suy diễn.
- Không có thay đổi code nào trong phạm vi feature này — chỉ tài liệu audit, đúng cách Phase 59 và
  audit 02 đã làm (tài liệu hoá xong mới quyết định sửa).
- Nếu số finding đủ lớn/đủ tranh cãi (ví dụ > 10 finding nhóm 1/2), cân nhắc 1 vòng verify độc lập
  kiểu Phase 41 (audit + fork verify riêng từng finding trước khi tin) — quyết định lúc audit dựa
  trên số lượng thật, không cam kết trước.

## Ranh giới kiến trúc bị đụng tới

Đọc chéo 3 repo: `metap-demo-waf/data-plane/web` (đối tượng audit), `platform-ui/src` +
`design-system/src` (nơi so sánh "đã có atom/component tương đương chưa"). Chỉ đọc, không sửa —
không đụng ranh giới runtime nào. Không cần ADR (audit không phải quyết định kiến trúc, chỉ là tài
liệu hoá hiện trạng).

## Rủi ro / phụ thuộc

Ranh giới "đặc thù nghiệp vụ thật, không tổng quát hoá được" (nhóm 3) đôi khi chủ quan — nếu có
finding gây tranh cãi giữa "nên chuyển" và "giữ nguyên", audit nên nêu rõ 2 chiều lập luận thay vì
tự quyết định 1 mình, để chủ dự án chốt cuối cùng (đúng tinh thần "báo lại khi nghi ngờ" thay vì
âm thầm quyết định thay). Không phụ thuộc phase/feature nào khác; độc lập hoàn toàn với feature
19-24 (những feature đó sửa `platform-ui` sẵn có, audit này tìm thêm việc mới từ 1 hướng khác).
