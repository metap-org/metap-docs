# Workflow guard — condition builder có cấu trúc (tái dùng `ConditionBuilder`)

- **Trạng thái:** done (2026-09-05) — `platform-ui` commit `078e967`. `ConditionBuilder`/
  `ConditionNodeEditor`/`AttributePicker`/`ValueEditor`'s `entity` prop nới sang
  `Pick<EntitySummary, "fields">` để tái dùng được cho draft low-code (không phải `EntitySummary`
  thật) — tái dùng 100%, không viết UI mới.
- **Người đề xuất:** rà soát `docs/frontend-checklist.md` (2026-09-04), mục 3 "Workflow Builder"
  > Condition builder.
- **Track sở hữu:** Frontend Platform
- **Phase roadmap liên quan:** chưa gắn phase

## Vấn đề / động lực

Guard của workflow transition hiện là JSON thô trong 1 `Textarea` (`guardText`, parse qua
`JSON.parse` — `admin/LowCodeEntitiesAdminPage.tsx:579-611, 730-732`). Theo `metap-metadata`,
`WorkflowTransition.guard` có kiểu `Option<PolicyCondition>` — **cùng type hệt** với điều kiện ABAC
mà Phase 48 đã xây UI builder có cấu trúc thật cho: `admin/policies/ConditionBuilder.tsx` +
`admin/policies/ConditionNodeEditor.tsx`, thao tác trên `PolicyCondition | null` qua props
`value`/`onChange` — **đã là component tổng quát, không hard-code riêng cho policy** (đọc code xác
nhận: nhận `entity: EntitySummary` + `subject: "context" | "record"`, không có gì buộc chặt vào
ngữ cảnh policy). Vì vậy đây gần như thuần wiring/tái dùng, không phải xây UI builder mới từ đầu —
đúng tinh thần "gap có sẵn atom/component nhưng chưa wiring" mà chính checklist gợi ý là rẻ nhất.

## Phạm vi

**Trong phạm vi:**
- Thay `Textarea` JSON thô cho guard ở `TransitionRowEditor`
  (`admin/LowCodeEntitiesAdminPage.tsx`) bằng `ConditionBuilder` đã có.
- Nếu `ConditionBuilder`/`ConditionNodeEditor` hiện đang sống trong `src/admin/policies/` và có
  import/giả định riêng cho ngữ cảnh policy (cần đọc kỹ trước khi kết luận là 100% tái dùng được
  nguyên trạng) — tách phần lõi (không phụ thuộc policy) ra 1 chỗ dùng chung (ví dụ
  `platform-ui/src/shared/conditionBuilder/`), `admin/policies` và workflow builder cùng import từ
  đó.
- Guard rỗng vẫn là 1 giá trị hợp lệ (transition không điều kiện) — giữ đúng hành vi hiện tại.

**Ngoài phạm vi:**
- Execution history/detail/retry/cancel — gap khác của Workflow Builder, cần khái niệm backend
  mới (workflow-execution record), không phải feature này.
- Đổi format `PolicyCondition` hay guard evaluation logic ở backend — feature này thuần FE.

## Tiêu chí chấp nhận

- `TransitionRowEditor` render `ConditionBuilder` thay `Textarea` cho guard, UI có cấu trúc
  field/operator/value giống hệt trải nghiệm ở `PoliciesAdminPage`'s advanced panel.
- Tạo transition không guard → lưu/publish thành công, hành vi giống hệt trước (không guard).
- Tạo transition có guard qua builder → JSON gửi lên backend đúng shape `PolicyCondition` backend
  đã chấp nhận trước đây (test bằng cách publish rồi kiểm response `200`, không đổi format).
- Sửa 1 workflow đã có guard dạng JSON cũ (từ trước khi có builder) → builder load đúng, hiển thị
  lại được điều kiện đó (không mất dữ liệu khi migrate từ Textarea sang builder).
- Tái dùng thật `ConditionBuilder`/`ConditionNodeEditor` — không viết lại UI tương đương từ đầu;
  nếu vì lý do kỹ thuật không tái dùng được, ghi rõ lý do cụ thể trong PR thay vì âm thầm viết mới.

## Ranh giới kiến trúc bị đụng tới

`platform-ui/src/admin/LowCodeEntitiesAdminPage.tsx`, và có thể tách 1 phần code từ
`platform-ui/src/admin/policies/*` ra vị trí dùng chung nếu cần (xem "Phạm vi"). Thuần
UI+logic composition, đúng chỗ `platform-ui`, không đụng `design-system` (không cần atom mới —
`ConditionBuilder` đã build trên atom `@metap/ui` có sẵn: `Button`, và bên trong
`ConditionNodeEditor` dùng `TreeItem`). Không cần ADR.

## Rủi ro / phụ thuộc

Cần đọc kỹ `ConditionNodeEditor.tsx` trước khi ước lượng effort thật — nếu nó giả định ngữ cảnh
`subject: "context" | "record"` theo nghĩa hẹp của policy (ví dụ field/operator list chỉ hợp lệ
cho attribute của request context/record trong policy check, không khớp ngữ cảnh guard của
workflow transition), việc tách dùng chung sẽ tốn công hơn dự kiến ban đầu — cần xác nhận trước
khi approve, không giả định trước khi đọc code.
