# Workflow visualize được / hướng BPM nhẹ

- **Trạng thái:** done (2026-09-02) — `WorkflowDiagram` (`platform-ui/src/workflow/WorkflowDiagram.tsx`),
  trigger "Visualize workflow" (`Dialog`) trong `WorkflowActionBar`, layout dùng chung
  (`workflow/layout.ts`), nút transition dùng chung (`workflow/TransitionButtons.tsx`). `pnpm
  typecheck`/`pnpm lint`/`pnpm format:check` sạch. Chưa tự verify bằng browser automation (đúng
  chính sách frontend verification — xem `../../CLAUDE.md`), chờ chủ dự án tự xem trên browser
  thật.
- **Người đề xuất:** ghi lại từ thảo luận kiến trúc, `docs/team-charter.md` ("Định hướng đang ghi
  nhận, chưa có trigger" #2). Trigger thật: chủ dự án xác nhận nhu cầu này luôn có sẵn ("user luôn
  cần xem quy trình khi áp dụng thay đổi state bản ghi"), rồi cho spec cụ thể (2026-09-02): bấm
  "visualize workflow" vẽ canvas step nào đến step nào, đánh dấu start/end/block, highlight state
  hiện tại của record, và hiện đúng button đổi state tương ứng.
- **Track sở hữu:** Frontend Platform (đọc metadata backend đã có sẵn, không cần core mới)
- **Phase roadmap liên quan:** chưa gắn phase

## Sửa lại 1 chỗ sai trong bản brief cũ

Bản `proposed` trước đó viết "hôm nay chỉ hiển thị qua metadata JSON thô hoặc list trạng
thái/action" — **sai, đã sai từ trước khi viết brief này**: `WorkflowActionBar`
(`platform-ui/src/workflow/WorkflowActionBar.tsx`) đã có sẵn 1 phần visualize — cột state (BFS
layout qua `computeLevels`/`groupByLevel`), badge phân biệt current/terminal/khác, và hàng nút
transition với tooltip lý do bị block. Cái thật sự thiếu, và là phần triển khai lần này: **cạnh nối
(edge) giữa các state** — bản cũ chỉ có cột đứng riêng lẻ, không vẽ được state nào chuyển sang state
nào.

## Vấn đề / động lực

Một entity có workflow (`EntityWorkflow`/`WorkflowTransition` — state, transition, guard) trước khi
brief này triển khai chỉ thấy được state hiện tại + nút hành động khả dụng — không thấy được toàn
bộ sơ đồ chuyển trạng thái (state nào nối với state nào, state nào là start/end, hành động nào đang
bị chặn với người dùng hiện tại).

**Không nhầm với việc build 1 workflow engine mới** — engine (`metap-cron`'s
`TargetType::Steps`/`WaitEvent`, Phase 17) đã xong từ trước. Đây thuần là 1 UI đọc metadata có sẵn
(`WorkflowTransition` đã có đủ `from`/`to`/`action`/`label`) và vẽ ra, không đụng backend.

## Phạm vi

**Trong phạm vi (đã làm):**
- `workflow/layout.ts` — tách `computeLevels`/`groupByLevel` (BFS column layout) ra khỏi
  `WorkflowActionBar` thành module dùng chung, để bar cột phẳng và canvas mới không lệch layout
  nhau.
- `workflow/WorkflowDiagram.tsx` — canvas SVG tự vẽ tay (không thêm dependency graph-rendering nào
  — xem mục "Rủi ro" cũ, giả định "cần thư viện mới" hoá ra không đúng vì đồ thị workflow trong repo
  đủ nhỏ để vẽ tay bằng SVG thuần):
  - Node = state, vị trí theo đúng cột BFS ở trên.
  - State bắt đầu (`workflow.initialState`): mũi tên nhỏ trỏ vào từ ngoài (kiểu ký hiệu UML state
    diagram).
  - State kết thúc (`workflow.terminalStates`): viền đôi (double ring).
  - State hiện tại của record (`currentState`): tô nền `primary`.
  - Cạnh (edge) = transition, có label = action, vẽ bezier cong kèm mũi tên; transition xuất phát
    từ state hiện tại mà bị chặn (`capabilities.transitions[].available === false`) vẽ nét đứt màu
    `destructive` kèm tooltip lý do (`reason`) — chỉ biết được trạng thái block cho các transition
    xuất phát từ state hiện tại, vì `capabilities` chỉ tính khả dụng cho state đó (đúng shape dữ
    liệu đã có, không mở rộng backend).
  - Legend giải thích 4 ký hiệu (start/current/end/blocked).
  - Hiện lại đúng hàng nút transition (dùng chung `TransitionButtons`, xem dưới) ngay dưới canvas —
    giữ nguyên style button-group hiện có thay vì đổi sang dropdown kiểu Jira (chủ dự án đồng ý cả
    2 hướng đều được, chọn giữ nguyên để không phá UX hiện tại).
- `workflow/TransitionButtons.tsx` — tách hàng nút transition (disabled/tooltip theo
  `capabilities.transitions`) ra khỏi `WorkflowActionBar` thành component dùng chung, để bar phẳng
  và canvas mới hiện đúng 1 logic nút, không hai nơi có thể lệch nhau.
- `WorkflowActionBar.tsx` — thêm nút "Visualize workflow" mở `Dialog` (`@metap/ui`) chứa
  `WorkflowDiagram`; bar cột phẳng + hàng nút cũ giữ nguyên, không thay thế (canvas là bổ sung sau
  1 trigger rõ ràng, không phải mặc định luôn hiện — đồ thị lớn sẽ chiếm nhiều chỗ hơn cột phẳng).
- i18n: `workflow.visualize`/`legendStart`/`legendCurrent`/`legendEnd`/`legendBlocked`/
  `currentStateActions` — cả `en`/`vi` (`platform-ui/src/i18n/resources.ts`).

**Ngoài phạm vi:**
- Không phải BPM editor cho phép kéo-thả định nghĩa workflow mới — chỉ visualize cái đã có, không
  edit qua UI này.
- Không visualize `metap-cron`'s `TargetType::Steps`/`WaitEvent` chain (đó là 1 tầng khác, activity
  chain, không phải state machine của entity) — nếu cần, nên là 1 brief riêng.
- Không tự động layout tránh giao cắt cạnh cho đồ thị phức tạp (nhiều state cùng cột, transition
  vòng lại cột trước) — dùng bezier cong đơn giản, chấp nhận giao cắt nhẹ ở đồ thị lớn; đủ cho quy
  mô workflow hiện có trong repo (vài state, "kanban 4 trạng thái" kiểu `jira.issues` là ví dụ lớn
  nhất).

## Tiêu chí chấp nhận

- Bấm "Visualize workflow" mở canvas vẽ đúng state/transition từ `EntityWorkflow` — verify bằng
  đọc code + `pnpm typecheck`/`pnpm lint`/`pnpm format:check` sạch (frontend verification policy:
  không tự test bằng browser automation, chờ chủ dự án xem trên browser thật để xác nhận UX).
- State bắt đầu/kết thúc/hiện tại có ký hiệu phân biệt rõ trên canvas.
- Transition bị chặn với người dùng hiện tại (từ `capabilities.transitions`) hiện khác màu/nét đứt,
  có tooltip lý do.
- Canvas hiện đúng nút transition khả dụng từ state hiện tại (dùng chung logic với bar phẳng, không
  viết lại).

## Ranh giới kiến trúc bị đụng tới

Không đụng backend/core — thuần đọc `GET /metadata/entities/{name}` đã public từ trước. Chỉ
`platform-ui` (repo riêng) — không đụng `design-system` (`Dialog`/`Badge`/`Tooltip` dùng nguyên,
không thêm primitive mới).

## Rủi ro / phụ thuộc

- Đồ thị workflow lớn (nhiều state/transition, nhiều cạnh giao cắt) chưa có layout tránh giao cắt
  thông minh — nếu tương lai có workflow phức tạp hơn nhiều so với hiện tại trong repo, có thể cần
  đổi sang 1 thư viện graph layout thật (vd dagre) thay vì BFS-column tự viết tay.
