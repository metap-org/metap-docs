## Phase 63: Audit `platform-ui` (login/phân quyền + `WorkflowDiagram`) và sửa 20 finding (2026-09-02 → 03)

Trigger: chủ dự án dùng thử `WorkflowDiagram` (vừa ship ở `docs/features/10-workflow-visualize.md`)
trên browser thật và báo "**mũi tên bị ghi đè rất xấu**", kèm yêu cầu audit rộng hơn: "audit
`platform-ui` nhé, đặc biệt liên quan đến luồng login/phân quyền, và workflow visualize mới thêm".

Báo cáo đầy đủ ở `platform-ui/docs/audits/02-auth-permission-workflow-diagram-audit.md` (20
finding, có bảng ưu tiên + số liệu verify + hướng xử lý từng mục). File này chỉ ghi phần đáng nhớ ở
mức phase.

## Vì sao `typecheck`/`lint`/`format:check` sạch mà vẫn ship ra thứ hỏng

Đây là bài học chính của phase. Feature workflow visualize được đánh dấu done với đúng 3 lệnh đó
sạch — nhưng **không lệnh nào nói được SVG có vẽ ra hình đúng hay không**. Toàn bộ 3 lỗi HIGH đều ở
tầng hình học, nơi toolchain FE mù hoàn toàn.

Cách audit làm để không lặp lại: **tính lại toạ độ bằng số từ chính công thức trong source** — port
`edgeGeometry` sang script, lấy mẫu 201 điểm trên mỗi đường Bézier, đếm điểm rơi vào trong hộp
node. Không vi phạm "Frontend verification policy" (không dùng browser automation), mà vẫn cho
bằng chứng định lượng. Dùng workflow `Zone` có thật ở `metap-demo-waf` chứ không dựng case cho đủ
xấu.

Kết quả đo trên chính workflow đó (4 state — nhỏ nhất sản phẩm đang có):

| | Trước | Sau |
|---|---|---|
| Cạnh `resume` bị node che | **93%** | **0%** |
| Nhãn chồng nhau | có (chồng 8px) | không (cách ≥13px) |

## Giả định bị bác bỏ

`docs/features/10-workflow-visualize.md` từng ghi trong "Ngoài phạm vi": *"không tự động layout
tránh giao cắt cạnh cho đồ thị phức tạp — chấp nhận giao cắt nhẹ ở đồ thị lớn; đủ cho quy mô
workflow hiện có"*. **Sai.** Vấn đề không phải "giao cắt nhẹ ở đồ thị lớn" mà là **cạnh bị xoá hẳn
ở đồ thị nhỏ nhất**: chỉ cần một cặp `active → paused` / `paused → active` (bất kỳ workflow nào cho
quay lại trạng thái trước — rất phổ biến) là đã vỡ. Rủi ro "sau này workflow phức tạp hơn thì cần
dagre" cũng thành hiện thực sớm hơn dự đoán rất nhiều.

Quyết định khi sửa: **vá tầng vẽ cạnh tự viết, không kéo `dagre`**. `layout.ts` (BFS-column) đang
chia sẻ với badge row của `WorkflowActionBar`, đổi thuật toán xếp cột sẽ lan sang đó; mà mọi lỗi
đều ở tầng **vẽ cạnh**, không phải ở việc xếp cột — vị trí node vốn hợp lý.

## Nhóm A — tầng vẽ cạnh (`platform-ui`)

Thay đổi cốt lõi: **tách render thành 3 pass** (mọi path → mọi nhãn → mọi node). SVG không có
`z-index`, thứ tự vẽ *chính là* thứ tự chồng — gộp path + nền nhãn vào 1 lượt trước node là nguyên
nhân gốc. Kèm: bỏ rect nền đục đổi sang halo chữ (`paintOrder="stroke"`); `edgeGeometry()` chia 4
nhánh routing (self-loop vòng lên trên / cùng cột bow sang phải / cột liền kề thẳng qua gutter /
còn lại thả xuống row gutter) để không đoạn nào chạy trong hộp node; `key` đổi sang
`${from}|${to}|${action}` vì backend `find_transition` khớp theo `(action, from)` nên `action`
**không** unique toàn workflow; state không reachable vào cột phụ thay vì bị bỏ im lặng.

**Chính bước verify bằng số bắt được 2 lỗi trong bản fix**, trước khi báo xong: `Math.abs(spread)`
làm cặp cạnh song song `±0.5` triệt tiêu thành cùng một đường; và cặp cạnh **ngược chiều cùng cột**
vẫn trùng khít vì gom nhóm theo cặp *có thứ tự* (`a→b`) thay vì cặp không thứ tự.

## Nhóm B + C — login/phân quyền, đụng cả `metap`

Thứ tự bắt buộc: B2 (backend) trước B1, vì trước đó FE không có dữ liệu nào để gate nút Delete.

- **`RecordCapabilities` thêm `can_delete`** (`metap-crud`). Kèm một sửa không hiển nhiên:
  `get.rs`/`get_many.rs` phải thêm `EntityAction::Delete` vào `enrich_record_for_actions` — hàm này
  chỉ resolve relation field mà các action được liệt kê cần, nên thiếu nó thì một delete policy
  điều kiện trên relation field sẽ đánh giá trên dữ liệu chưa enrich và trả sai, im lặng. Thêm
  `#[serde(default)]` vì `metap_grpc::client` parse thẳng response service khác vào struct này —
  field bắt buộc mới sẽ làm **fail cứng cả `get`** với upstream build từ code cũ.
- **`GET /auth/me` trả thêm `email`** (`metap-http` + `metap_peripherals::find_user_by_id`, scope
  theo tenant). Trước đó FE muốn biết email của chính mình phải kéo **toàn bộ user list của
  tenant** rồi tìm client-side, chạy ở mọi trang, refetch mỗi lần focus cửa sổ.
- **`openapi.rs` thêm `validator`/`setFields`** vào schema `WorkflowTransition` — drift thật so với
  `entity.rs`, đúng loại mà chính doc comment của file đó cảnh báo. **Chưa regenerate
  `generated-types.ts`**: bước đó cần một backend đang chạy, mà `metap` không có app chạy được và 2
  repo demo không trong scope session — việc còn lại cho chủ dự án.
- **`platform-ui`**: gate Edit/Delete theo capabilities đã có sẵn trong cùng response (disable +
  tooltip lý do, không ẩn); OIDC callback xoá token khỏi URL (`history.replaceState`) và
  `navigate(..., { replace: true })` nên không còn `#token=<JWT>` trong history để bấm Back về, kèm
  xử lý callback lỗi thay vì treo mãi ở "Signing you in…"; `AdminOnly` bọc 4 trang admin **bên
  ngoài** thân component cũ nên non-admin không bắn request `/admin/*` nào; `roles: []` không còn bị
  hiểu ngược thành "không ai xem được".

## Phát hiện ngoài lề, đã sửa

`cargo clippy --workspace --all-targets` **vỡ sẵn** trên branch từ commit `bd2c05e`: commit đó thêm
field `computed` vào `EntityField` nhưng bỏ sót 2 bench fixture (`metap-query`,
`metap-reconciler`). `cargo build --workspace` không bắt vì bench không build mặc định. Đã thêm
`computed: None` — không liên quan audit, sửa vì nó chặn tín hiệu clippy sạch.

## Trạng thái

19/20 finding đã sửa. Còn nửa sau của A10: thống nhất tooltip lý-do-bị-chặn trên canvas (hiện vẫn
là `<title>` native của SVG) sang `Tooltip` của `@metap/ui` — cố ý để riêng vì bọc
`TooltipTrigger asChild` quanh một `<g>` SVG có rủi ro về positioning mà chính sách frontend
verification không cho tự kiểm bằng browser.

Verify: `metap` — `cargo build/clippy --all-targets/test --workspace` sạch cả 3 (e2e `--ignored`
chưa chạy, cần Postgres/RabbitMQ). `platform-ui` — `typecheck`/`lint`/`format:check` sạch, lint về
đúng baseline 6 warning có sẵn, 0 warning ở file đụng tới. Lưu ý cho session sau: phải
`pnpm build` bên `design-system` trước, không thì typecheck đỏ hàng loạt `Cannot find module
'@metap/ui'` (consumer resolve qua `dist/`, không phải `src/`). Không chạy browser — những thứ cần
mắt (halo chữ dày mỏng, tooltip nút disable, trang admin của non-admin, luồng OIDC thật) giao lại
cho chủ dự án.
