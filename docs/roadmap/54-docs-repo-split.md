## Phase 54: Tách `docs/` ra repo riêng `metap-docs` (2026-08-31)

Tiếp nối 3 lần tách trước (frontend library — Phase 47, demo app — Phase 51, low-code
control-plane — Phase 52) — cùng lý do nền: `metap` chỉ nên là library workspace thuần, không
mang theo thứ không phải code. `docs/` (80 file tracked, ~996KB: `architectures/`, `roadmap/`,
`features/`, `audits/`, cộng các doc rời `vision.md`/`why.md`/`team-charter.md`/`agile-process.md`/
`CONTRIBUTING.md`/`frontend-checklist.md`/`multi-tenant-platform-design.md`/
`modular-spi-architecture.md`/`low-code-platform-v1.md`/`low-code-metadata-storage-design.md`)
chuyển nguyên trạng sang `../metap-docs` — copy qua `git ls-files docs` (không dùng `find`, tránh
dính file không track), giữ nguyên cấu trúc `docs/` làm subfolder trong repo mới (không flatten
lên root) — mọi liên kết chéo *nội bộ* giữa các file trong `docs/` (vd `docs/features/README.md`'s
link tới `07-split-lowcode-saas-crates.md`, `docs/roadmap.md`'s link tới `roadmap/NN-*.md`) không
đổi gì, vì chúng vốn đã tương đối trong chính thư mục `docs/`.

**Khác biệt lớn nhất so với 3 lần tách trước — không có compiler bắt lỗi tham chiếu hỏng.** Khảo
sát trước khi tách: 150 chỗ trong doc-comment code (`crates/*.rs`, 84 file), 24 chỗ trong
`CLAUDE.md`, 10 chỗ trong `README.md` (ở `metap`) trích dẫn đường dẫn `docs/...` — tách xong,
những trích dẫn này **không sửa lại**, theo quyết định chủ dự án (2 câu hỏi rõ ràng trước khi
làm): giữ nguyên tinh thần "không viết lại lịch sử" đã áp dụng cho `docs/roadmap/*.md` ở các phase
trước, mở rộng sang cả doc-comment code — sửa hàng loạt 184 chỗ qua script có rủi ro đụng nhầm
comment không liên quan, đổi lại lợi ích nhỏ (comment cũ vẫn đọc hiểu được, chỉ là trỏ sang repo
khác). Thay vào đó: 1 ghi chú rõ ràng ở đầu `CLAUDE.md`/`README.md` của `metap`, nói `docs/...`
trong bất kỳ comment/text nào của repo này giờ có nghĩa `../metap-docs/docs/...`.

**Phạm vi chốt trước khi làm** (2 câu hỏi rõ ràng, không đoán): chỉ `docs/` của `metap` — không
gộp `docs/` riêng của `metap-lowcode`/`metap-demo-crm`/`metap-demo-jira` (mỗi repo giữ nguyên
`docs/` của chính nó, đúng cách mỗi repo tự giữ `README.md`/`CLAUDE.md` riêng).

Không có tooling/CI nào phụ thuộc chức năng vào `docs/` tồn tại trong `metap` (`grep` xác nhận:
mọi chỗ `.github/workflows/*.yml` nhắc "docs/" chỉ là comment trích dẫn, không phải step đọc file
thật) — xoá `docs/` khỏi `metap` không làm CI vỡ.
