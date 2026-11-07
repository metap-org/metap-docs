## Phase 104: workflow theo người gọi, quyền theo từng action, tách nhiệm vụ (2026-11-07)

Trigger: rà workflow engine của `metap-go` cho luồng duyệt thay đổi policy của sản phẩm WAAP
(`metap-demo-waf`, `docs/requirements/03-business-analysis/02-policy-change.md`). Chủ dự án nêu nhu cầu:
**cùng một luồng, mỗi người dùng theo quyền thấy luồng khác nhau**; khách self-service tự làm chủ, không
cần ai duyệt; khách doanh nghiệp và admin của NCC có phân chia quyền, trách nhiệm rõ ràng. Rà xong thấy
khung đã có (capabilities trả transition theo người gọi, guard là `PolicyCondition`) nhưng thiếu 7 điểm
W1–W7; chủ dự án chốt "Metap phải sửa thì update luôn".

### Đã làm

| # | Thiếu sót | Sửa |
|---|---|---|
| W1 | Quyền `transition` tính chung cho mọi action của entity: không thể cho developer "gửi duyệt" mà không cho "duyệt" | Policy trên `transition:<action>` (`permission.TransitionActionPolicy`) chỉ cấp hoặc cấm action đó; `transition` không kèm action vẫn phủ mọi action như cũ. Hai tập được đánh giá cùng nhau, cấm tường minh ở tập nào cũng thắng. Kiểm ở đường ghi (`PermissionService.CanTransitionAction`) và trong capabilities (`PermissionSnapshot.CanTransitionAction`) |
| W2 | Guard chỉ đọc trường của record; ngữ cảnh người gọi chỉ dùng được ở vế giá trị, nên không viết được "người gọi có vai trò X" | Tiền tố `$ctx.` cho thuộc tính: `$ctx.roles contains "approver"`, `$ctx.approvalRequired eq true`, đọc mọi context attribute. Trong record policy, lá `$ctx.` là hằng số theo request nên được tính trước thành `true`/`false` trong SQL |
| W3 | Không có cách để cùng một entity, account self-service bỏ qua bước duyệt còn account doanh nghiệp bắt buộc duyệt | Không cần cơ chế riêng: `$ctx.<attribute>` đọc chế độ của account từ context attribute (cơ chế `ContextAttributes` đã có). Workflow khai báo hai đường từ `draft` (`publish` có guard `$ctx.approvalRequired neq true`; `submit` → `approve` có guard `$ctx.approvalRequired eq true`); capabilities tự hiện đúng đường cho từng người |
| W4 | Người duyệt khác người gửi phải tự dựng bằng `setFields`, dễ quên, và client có thể sửa trường đó | `$workflow.actors.<action>`: người thực hiện action đó gần nhất trên record, đọc từ `workflow_events` (bảng chỉ ghi thêm, client không sửa được). Chỉ truy vấn khi có guard dùng, gộp một truy vấn cho cả lô ở `GetMany`. Compiler chỉ nhận `$workflow.actors.<action có thật>`; record policy từ chối `$workflow.` |
| W6 | `reason` của transition không khả dụng là biểu thức kỹ thuật (`condition failed: …`) | `guardMessage` / `validatorMessage` trên `WorkflowTransition` thay thông điệp đó trong lỗi và trong capabilities. Compiler từ chối message khi không có guard/validator tương ứng |
| — | Capabilities bỏ qua policy ở mức entity cho update, delete, transition: người không có policy vẫn thấy `available: true` rồi bấm thì bị 403 | Capabilities áp thêm policy mức entity (từ snapshot đã tải, không thêm truy vấn), nên không bao giờ đề nghị một hành động mà đường ghi sẽ từ chối |

Không làm ở phase này:

- **W5 duyệt nhiều người** (N người duyệt, theo nhóm): cần entity Approval riêng và phép đếm trong guard;
  để khi có sản phẩm cần.
- **W7 thao tác của NCC có người duyệt**: không cần code mới. Dựng bằng chính các khả năng trên trong
  tenant của NCC (`PLATFORM_TENANT_ID`): thao tác mạnh tạo yêu cầu, `$workflow.actors` bảo đảm người
  duyệt khác người yêu cầu.

### Ghi chú

- Admin vẫn bỏ qua policy ở mức entity và record như trước, nhưng **guard vẫn chạy với admin**: admin
  không tự duyệt được thay đổi của chính mình nếu guard cấm.
- `transition:<action>` là tên action của policy, khác ngữ pháp OAuth2 scope `transition:<entity>` (thu
  hẹp token theo entity).
- Tên field không thể bắt đầu bằng `$` (compiler), nên `$ctx.` và `$workflow.` không trùng field thật.

### Verify

- Unit test cho từng package: đọc `$ctx`, bỏ qua thuộc tính có tiền tố khi gom relation field, ma trận
  quyền theo action (cấp riêng, cấp chung, cấm tường minh, admin, mặc định cấm), gập `$ctx` trong SQL và
  từ chối `$workflow.`, `guardMessage`/`validatorMessage`, validation của compiler.
- E2E `TestOneWorkflowShowsEachCallerOnlyTheTransitionsTheyCanTake` trên PostgreSQL 16 thật: một workflow
  phục vụ account self-service (tự phát hành), account doanh nghiệp (phải gửi duyệt; người gửi có cả vai
  trò approver vẫn không tự duyệt được; người không có `transition:approve` không thấy và bị 403; approver
  khác duyệt được), và vai trò không có policy transition nào không thấy transition nào.
- Toàn bộ test suite `metap-go` xanh trên Postgres; `metap-lowcode` và WAAP (data-plane, control-plane)
  build, vet, test pass với cây này.
