## Phase 59: Rà soát atom lạc chỗ ở `platform-ui` + thêm `FileUpload` (2026-09-01)

Trigger: chủ dự án hỏi "quay lại `design-system`/`platform-ui` thì cần làm thêm gì" trong lúc chưa
có người review nào (mọi component ở `component-status.md` vẫn "Đã review: Chưa" tính tới thời
điểm này) — quyết định: chưa có reviewer thì cứ tiếp tục build component/checklist, không chờ.
Rà soát chủ động (đọc code, không đoán) tìm tiếp cùng loại gap Phase 48 từng dọn (`TagsField`/
`MultiFieldSelect`/`Input`+`Chip` viết tay lẽ ra phải ở `@metap/ui`) — tìm được 2 việc.

## 1. `BarChart` chuyển từ `platform-ui` sang `design-system`

`platform-ui/src/charts/BarChart.tsx` đọc code thấy hoàn toàn generic — không biết `jira.issues`
hay entity nào, chỉ nhận `{label, value, color?}`, render inline SVG (cố ý không dùng lib chart —
xem doc-comment component), đã tự đọc token màu qua `hsl(var(--...)))` giống mọi component
`@metap/ui` khác. Đúng là 1 UI atom bị để nhầm ở tầng business-screen.

Chuyển nguyên trạng thành `design-system`'s `src/components/bar-chart/` (component + 8 test +
story), chỉ đổi 1 chỗ: nhận `className` qua `cn()` (khớp convention chung) thay vì `style` cố
định. `platform-ui/src/charts/` xoá hẳn (thư mục rỗng theo). 2 consumer thật
(`apps/jira-fe`'s `DashboardPage.tsx`/`CustomizableDashboardPage.tsx`) đổi import từ
`@metap/platform-ui` sang `@metap/ui`, sửa luôn 1 doc-comment cũ trỏ sai chỗ.

## 2. `FileUpload` — component mới, thay `<input type="file">` trần

`apps/jira-fe`'s `IssueDetailPage.tsx` (attachment panel, consumer thật duy nhất của
`metap-attachments`'s route generic) dùng `<input type="file">` không style gì. Build
`FileUpload` mới (`design-system/src/components/file-upload/`, 12 test) — click-to-browse +
drag-and-drop, danh sách file đã chọn kèm nút xoá, tự đọc token/convention chung (không phải
danh mục chuẩn gốc trong `readme.md`, cùng dạng "mở rộng" như `TagsInput`/`BarChart`).

**Phạm vi validate cố tình giới hạn ở HTML-level** — chủ dự án hỏi thêm về validate file (size,
"file khoá kiểu cert", content, validate Excel) khi chốt yêu cầu; quyết định: chỉ enforce
`accept` (MIME/extension allow-list) và `maxSize` (byte ceiling) — đúng 2 thứ 1
`<input type="file" accept=...>` gốc đã diễn đạt được, không đọc nội dung file (không verify
cert/chữ ký, không parse schema Excel, không quét virus). Lý do: cùng ranh giới `Form`/`FormField`
đã lập từ đầu (`readme.md`'s "Spec chi tiết") — "lib lo UI + hiển thị lỗi, app lo business rule
validate"; đọc nội dung file để verify format cụ thể (X.509, OOXML spreadsheet schema...) sẽ trói
1 UI atom generic vào 1 định dạng nghiệp vụ cụ thể, phá đúng nguyên tắc phân tầng
(`design-system` không phụ thuộc app-specific parsing). Việc đó thuộc về app (parse sau
`onFilesChange`) hoặc backend (`metap-attachments`'s upload endpoint).

**1 phát hiện thật lúc viết test**: `accept` re-check ở JS chỉ thật sự có tác dụng qua
drag-and-drop — native file picker của trình duyệt (qua click) đã tự lọc theo `accept` từ trước
khi JS kịp chạy; `userEvent.upload` trong test mô phỏng đúng hành vi đó (không "chọn" được file
sai type qua input khi `accept` đã set). Ghi lại trong doc-comment + comment ngay tại test tương
ứng, không phải bug — xác nhận đúng như thiết kế.

Retrofit vào `apps/jira-fe`'s `IssueDetailPage.tsx`'s `AttachmentsPanel`: bỏ `fileInputRef`/reset
tay `.value = ""` (component tự làm việc đó nội bộ), giữ nguyên hành vi cũ (1 file, upload ngay
khi chọn, `uploadError` hiển thị lỗi upload thật — khác với lỗi validate HTML-level của
`FileUpload` tự hiển thị).

## Xác minh

`design-system`: `build:types`/`lint`/`pnpm build` sạch; `pnpm test:run` 44 file/349 test pass
(12 test mới cho `FileUpload`, 8 cho `BarChart`). `platform-ui`: `typecheck`/`lint`/`format:check`
sạch. `apps/jira-fe`'s `web/`: `tsc -b` (build thật) sạch, `oxlint` không thêm warning mới (các
warning còn lại đều pre-existing, không liên quan) — đã `pnpm build` lại `design-system` trước
khi verify vì 2 repo consumer resolve `@metap/ui` qua `dist/` (symlink `link:../design-system`),
không phải `src/`.

Nợ hạ tầng phát hiện phụ, không thuộc phạm vi phase này: `prettier --check` ở `apps/jira-fe`'s
`web/` báo lệch format sẵn có ở ~13 file (không phải do lần sửa này — nhiều file trong đó không
hề bị đụng tới) — cùng loại drift với `cargo fmt` chưa chạy đã gặp ở phía Rust cùng ngày, để
nguyên, chưa dọn.
