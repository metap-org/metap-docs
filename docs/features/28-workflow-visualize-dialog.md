# `WorkflowVisualizeDialog` ở `@metap/platform-ui` — dọn 2 chỗ lặp Dialog+`WorkflowDiagram`

- **Trạng thái:** proposed
- **Người đề xuất:** `platform-ui/docs/audits/03-waf-demo-component-placement-audit.md` (2026-09-05),
  finding #7.
- **Track sở hữu:** Frontend Platform (đọc/sửa chéo sang `metap-demo-waf`)
- **Phase roadmap liên quan:** không thuộc phase nào

## Vấn đề / động lực

`metap-demo-waf/data-plane/web/src/pages/ZoneDetailPage.tsx:191-213` và
`IncidentDetailPage.tsx:148-170` lặp verbatim cùng 1 khối: `Dialog > DialogTrigger(asChild, Button
"workflow.visualize") > DialogContent(max-w-3xl) > DialogHeader > DialogTitle >
WorkflowDiagram(...)`, chỉ khác props truyền cho `WorkflowDiagram`. Cả 2 đã import
`WorkflowDiagram` đúng chỗ từ `@metap/platform-ui` — chỉ lớp Dialog bọc ngoài là lặp. Mức ưu tiên
thấp hơn #27 (chỉ 2 chỗ, đúng ngưỡng tối thiểu "đã lặp lại thật"), gộp vào đợt xử lý này vì cùng
audit, cùng track, tránh phải quay lại sửa `RecordDetail`-adjacent code 2 lần.

## Phạm vi

**Trong phạm vi:**
- 1 component mới `WorkflowVisualizeDialog` ở `platform-ui/src/workflow/` (cạnh `WorkflowDiagram`
  đã có), nhận đúng props `WorkflowDiagram` cần cộng `label` (text nút trigger, i18n do caller
  truyền vào — không hardcode key `workflow.visualize` của app nào).
- Áp dụng vào `ZoneDetailPage.tsx`/`IncidentDetailPage.tsx`.

**Ngoài phạm vi:** không đổi `WorkflowDiagram` bản thân nó.

## Tiêu chí chấp nhận

- 2 call site giữ nguyên hành vi (trigger mở dialog, `max-w-3xl`, tiêu đề, diagram render đúng).
- `platform-ui`: `tsc`/`oxlint`/`prettier` sạch. `metap-demo-waf/data-plane/web`: `tsc`/`oxlint`/
  `prettier`/`vite build` sạch.

## Ranh giới kiến trúc bị đụng tới

`platform-ui/src/workflow/WorkflowVisualizeDialog.tsx` (mới) + 2 file trong
`metap-demo-waf/data-plane/web/src/pages/`.

## Rủi ro / phụ thuộc

Không phụ thuộc feature khác — độc lập với #26/#27, có thể làm song song.
