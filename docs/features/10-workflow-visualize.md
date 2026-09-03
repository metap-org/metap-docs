# Workflow visualize được / hướng BPM nhẹ

- **Trạng thái:** done (2026-09-02), **sau một vòng sửa lại tầng vẽ cạnh** — `WorkflowDiagram`
  (`platform-ui/src/workflow/WorkflowDiagram.tsx`), trigger "Visualize workflow" (`Dialog`) trong
  `WorkflowActionBar`, layout dùng chung `workflow/layout.ts`, nút transition dùng chung
  `workflow/TransitionButtons.tsx`.
  - **Lần ship đầu không đạt.** Chủ dự án xem trên browser thật và báo mũi tên bị vẽ đè lên nhau
    rất xấu — đúng cái mà chính sách frontend verification để dành cho bước này, và là thứ
    `typecheck`/`lint`/`format:check` sạch hoàn toàn không nói được gì. Audit
    (`platform-ui/docs/audits/02-auth-permission-workflow-diagram-audit.md`) verify bằng số, tìm ra
    3 lỗi HIGH: rect nền nhãn màu đục vẽ đè lên arrow của cạnh trước (A1), node vẽ sau cạnh +
    control point của cạnh lùi nằm trong hộp node → **93% một cạnh bị che** (A2), self-loop mất
    hoàn toàn (A3); cộng 4 MEDIUM (cạnh nhảy cột xuyên node, 2 transition cùng cặp `(from,to)` vẽ
    trùng khít, `key={transition.action}` trùng key vì backend khớp theo `(action, from)`, state
    không reachable bị bỏ im lặng).
  - **Đã sửa xong** cùng ngày: tách render thành 3 pass (mọi path → mọi nhãn → mọi node, vì SVG
    không có `z-index`), halo chữ thay rect đục, `edgeGeometry()` chia 4 nhánh routing để không
    đoạn nào chạy trong hộp node, fan cạnh song song theo cặp **không thứ tự**. Verify lại bằng
    cùng phương pháp đã dùng để tìm lỗi (lấy 201 mẫu mỗi đường Bézier, đếm điểm rơi vào hộp node)
    trên 3 workflow: **93% → 0% bị che**, không nhãn nào chồng nhau. Chi tiết ở mục "Đã sửa gì" của
    audit.
  - Còn lại chưa làm: thống nhất tooltip lý-do-bị-chặn (hiện là `<title>` native của SVG, trong khi
    `TransitionButtons` ngay cạnh dùng `Tooltip` của `@metap/ui`) — audit A10, đáng làm riêng.
  - Vẫn **chưa tự verify bằng browser automation** (đúng chính sách frontend verification — xem
    `../../CLAUDE.md`). Số liệu chứng minh không cạnh nào bị che và không nhãn nào chồng, nhưng
    thẩm mỹ (halo dày mỏng, bow/dip thoáng chưa, self-loop cân chưa) vẫn cần chủ dự án nhìn.
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
- ~~Không tự động layout tránh giao cắt cạnh cho đồ thị phức tạp (nhiều state cùng cột, transition
  vòng lại cột trước) — dùng bezier cong đơn giản, chấp nhận giao cắt nhẹ ở đồ thị lớn; đủ cho quy
  mô workflow hiện có trong repo (vài state, "kanban 4 trạng thái" kiểu `jira.issues` là ví dụ lớn
  nhất).~~ **Giả định này sai, đã bác bỏ 2026-09-02.** Audit tính lại toạ độ bằng số cho workflow
  `Zone` (`metap-demo-waf`, **4 state** — tức đúng "quy mô hiện có" mà dòng trên cho là đủ) cho
  thấy vấn đề không phải "giao cắt nhẹ ở đồ thị lớn" mà là **cạnh bị xoá hẳn ở đồ thị nhỏ nhất**:
  chỉ cần một cặp `active → paused` / `paused → active` (bất kỳ workflow nào cho phép quay lại
  trạng thái trước — rất phổ biến) là nhãn cạnh sau đã xoá mất thân mũi tên cạnh trước. Đây là
  thiếu sót thật ở tầng vẽ cạnh, không phải giới hạn chấp nhận được của phạm vi.

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

- ~~Đồ thị workflow lớn (nhiều state/transition, nhiều cạnh giao cắt) chưa có layout tránh giao cắt
  thông minh — nếu tương lai có workflow phức tạp hơn nhiều so với hiện tại trong repo, có thể cần
  đổi sang 1 thư viện graph layout thật (vd dagre) thay vì BFS-column tự viết tay.~~ Rủi ro này đã
  **thành hiện thực ngay ở workflow 4 state**, sớm hơn nhiều so với dự đoán ("tương lai có workflow
  phức tạp hơn"). Cần quyết định trước khi sửa: vá tầng vẽ cạnh tự viết (đưa control point ra ngoài
  hộp node, offset cạnh trùng cặp, halo chữ thay rect đục, case riêng cho self-loop) hay đổi hẳn
  sang graph layout library thật. Chi tiết từng lỗi + hướng xử lý đề xuất ở
  `platform-ui/docs/audits/02-auth-permission-workflow-diagram-audit.md` phần A.
- Bài học về quy trình: `pnpm typecheck`/`lint`/`format:check` sạch **không nói gì** về việc SVG có
  vẽ ra hình đúng hay không. Với feature thuần render hình học, cần một bước verify riêng — tính
  lại toạ độ bằng số từ chính công thức trong code (cách audit đã dùng, không vi phạm chính sách
  không-dùng-browser-automation) — chứ không chỉ dựa vào toolchain FE báo sạch.
