## Phase 83: Ngừng demo/phát triển `metap-demo-crm` (2026-09-11)

Note, không code. Trigger: chủ dự án yêu cầu "gạch crm khỏi tài liệu, không demo nó nữa" — lý do
nêu rõ: chưa nắm nghiệp vụ CRM đủ để dẫn dắt việc mở rộng app này.

Quyết định: `metap-demo-crm` (repo riêng từ Phase 51) không còn là app demo/downstream được chọn
để minh hoạ hay verify việc mới nữa. `metap-demo-jira` và `metap-demo-waf` vẫn là 2 app demo đang
dùng.

Phạm vi áp dụng — chỉ cập nhật các chỗ đang **mô tả/giới thiệu** `metap-demo-crm` như một app demo
đang hoạt động, thêm ghi chú deprecated cạnh, không sửa đè nội dung lịch sử:
- `../CLAUDE.md` (root `metap-org`): dòng mô tả repo trong bảng cấu trúc + 1 đoạn ghi chú riêng.
- `../../metap/CLAUDE.md`: thêm đoạn ghi chú ngay sau "No example apps in this repo".

Cố ý **không đụng tới**: các đoạn kỹ thuật/lịch sử ở `metap/CLAUDE.md`, `metap-lowcode/docs/`,
`metap-demo-waf/CLAUDE.md`, `platform-ui/docs/audits/` dẫn `metap-demo-crm` làm ví dụ/bằng chứng
cho quyết định kiến trúc hoặc bug đã tìm ra (Phase 79/80/82, `metap-lowcode`'s Phase T1-T5 kiến
trúc doc, ...) — những đoạn đó vẫn đúng về mặt lịch sử/kỹ thuật, rewrite sẽ vi phạm đúng quy ước
"không rewrite lịch sử" đã ghi trong `../CLAUDE.md`. Cũng không đụng tới code/config (`Cargo.toml`
path dependency, `docker-compose*.yml`, `.cargo/config.toml`) — `metap-demo-crm` vẫn build/chạy
được nếu cần dùng lại sau này, chỉ là không còn được chọn làm ví dụ mặc định.

Chưa làm, ngoài phạm vi phiên này: sửa README/CLAUDE.md của chính `metap-demo-crm`/`metap-demo-jira`
(2 repo đó không nằm trong scope truy cập của phiên này) — nếu cần đánh dấu deprecated ngay trong
chính repo `metap-demo-crm`, đó là việc riêng cần add repo đó vào scope trước.
