## Phase 65: Sửa 5 finding audit 04 (SSRF, CSRF token endpoint, error fidelity, coverage, trace) (2026-09-03)

Trigger: audit 04 (`docs/audits/04-auth-protocols-gateway-audit.md`, chạy cùng ngày) ra 17 finding
và chưa fix cái nào. Chủ dự án chọn xử lý theo lô: đầu tiên A#1 + B#4 + B#5, sau đó B#2 + A#4.

Báo cáo đầy đủ + trạng thái từng finding ở `docs/audits/00-index.md`. File này chỉ ghi phần đáng
nhớ ở mức phase.

## A#1 (HIGH) — SSRF có phản hồi qua cron webhook

`target_config` của một cron job `webhook` (url/method/headers/body) do **tenant admin** viết, và
response body quay lại chính admin đó qua `GET /admin/cron-jobs/{id}/runs`. Trước bản vá, URL đi
thẳng vào `reqwest` — đọc được cloud metadata (`169.254.169.254` → IAM credential), chạm được mọi
service trong VPC, và tự đặt được header tuỳ ý khi làm việc đó.

Module mới `crates/cron-scheduler/src/executor/ssrf_guard.rs`. Ba điểm đáng nhớ hơn bản thân danh
sách chặn:

1. **IPv4-mapped IPv6 là bypass một dòng nếu quên.** `::ffff:169.254.169.254` không khớp bất kỳ
   luật IPv6 nào nếu chỉ kiểm tra `Ipv6Addr`; phải unmap về IPv4 rồi mới xét.
2. **Redirect đi vòng qua toàn bộ check.** Guard chỉ nhìn thấy URL đầu tiên. Một host hợp lệ trả
   `302 Location: http://169.254.169.254/` là xong. Phải có `reqwest::Client` riêng cho webhook với
   `redirect::Policy::none()` — không phải tuỳ chọn, là một nửa của bản vá.
3. **Giới hạn còn lại được ghi rõ chứ không giấu**: DNS rebinding chưa đóng (guard resolve xong,
   `reqwest` resolve lại lúc connect). Đóng đúng cách cần pin IP đã validate mà vẫn giữ Host/SNI —
   `reqwest` không hỗ trợ trực tiếp. Chấp nhận có ý thức: cần DNS server của kẻ tấn công + thắng
   một cuộc đua với job chạy theo lịch cron, khác xa "gõ `http://169.254.169.254/` vào một ô form".

Tiện thể vá 1 panic có sẵn: `truncate()` cắt response theo **byte offset**, một response UTF-8 có
ký tự nhiều byte rơi đúng byte 500/2000 là panic cả task executor.

## B#4 (MEDIUM) — cookie/CSRF ship với 0 test

Cơ chế cookie session + CSRF ship ngày 2026-09-03 (Phase 64) mà `metap_session`/`x-csrf-token`
không xuất hiện trong bất kỳ test nào toàn workspace. Tách `requires_csrf_check`/`csrf_matches`
thành hàm thuần → 6 unit test không cần DB, cộng `tests/cookie_session_postgres.rs` (7 e2e).

Lỗ nhỏ tìm được **khi viết test**, không phải khi đọc code: cookie rỗng + header rỗng trước đó so
sánh bằng nhau và **pass**.

## B#5 (LOW) — chẩn đoán ban đầu của audit sai, đã đính chính

Audit viết "`attach_trace_context` không có caller nào → sửa rẻ: gọi ở 3 call site". **Sai.** Gọi
thêm ở 3 chỗ đó vẫn là no-op, vì cả 3 chạy **ngoài** mọi request scope (`cron-scheduler` consume từ
RabbitMQ, gateway fetch metadata lúc boot, `metap-jwks` refresh nền) — `trace_context::current()`
trả `None` ở cả ba. Lỗ hổng thật là **không có gì tạo trace context ngoài request HTTP**.

Fix thật: `dispatch::execute` mở root trace context mới cho mỗi job run, rồi mới gắn
`attach_trace_context` vào 3 callback REST. Gateway metadata-fetch và JWKS refresh **cố ý để
nguyên** — thao tác boot/nền một lần, một root trace cho mỗi lần đó gần như không có giá trị vận
hành. Đính chính được ghi thẳng vào file audit, không sửa lặng lẽ.

## B#2 (MEDIUM) — error mất thông tin qua mỗi hop

`tonic::Status` chỉ mang `Code` + text, nên `ServiceResult::Err` mất gần hết khi qua gRPC: `400` và
`422` cùng gộp vào `InvalidArgument`, `error` code bị bỏ, và `field_errors` bị nén thành **một con
số đếm** trong câu message. Qua `graphql-gateway` thì đó là tất cả những gì client nhận được — một
form biết mình sai 3 chỗ nhưng không bao giờ biết là chỗ nào, dù nền tảng sinh ra chi tiết đó từ
entity metadata miễn phí.

Envelope đầy đủ giờ đi trong `Status::details` dạng JSON, client dựng lại nguyên vẹn. Mapping theo
`Code` giữ làm fallback cho peer không nói envelope này (build cũ, hoặc lỗi do sidecar/proxy sinh)
— additive cả hai chiều. GraphQL chuyển sang `extensions` (`code`/`status`/`fieldErrors`) thay vì
prefix số vào message. Dùng JSON chứ không `google.rpc.BadRequest`: envelope đã đúng shape REST body
đang dùng, một biểu diễn cho cả 3 transport thay vì thêm một proto phải giữ đồng bộ.

## A#4 (MEDIUM) — `GET /auth/token` phát credential nhưng miễn CSRF

Endpoint này mint một Bearer JWT mang danh tính đầy đủ của caller. Miễn CSRF vì là `GET` — nhưng
luật "safe method" nói về **thay đổi trạng thái**, còn phát credential là loại rủi ro khác. Trước
bản vá, thứ duy nhất chặn một trang lạ đọc token đó là CORS: đúng ở hiện tại, nhưng khiến endpoint
nhạy cảm nhất phụ thuộc hoàn toàn vào một env var được set đúng và mọi origin trong allowlist
không dính XSS.

`cookies::credential_issuing_request_allowed` mirror đúng thứ tự ưu tiên của `AuthContext`: Bearer
thắng và không bao giờ bị gate. Phía FE, `apiFetch` đổi sang gắn CSRF header cho **mọi** request
thay vì lọc theo method — gửi thừa trên GET không tốn gì, mà tránh phải nuôi danh sách "GET nào
đặc biệt" ở hai nơi; từ nay backend gate thêm endpoint nào cũng không cần sửa FE.

## Xác minh

`cargo test --workspace` 90/90 suite ok; `cargo clippy -D warnings` + `cargo fmt --check` sạch;
`platform-ui` typecheck sạch, lint về đúng baseline cũ. **25 e2e test (`--ignored`) chưa chạy** —
môi trường phiên này không có Docker/Postgres; cần chạy `cargo test --workspace -- --ignored` với
dev Postgres trước khi merge.

Disk quota hết 2 lần giữa phiên (linker `Bus error`, `No space left on device` — không phải máy
hỏng), `cargo clean` giải phóng ~30GB mỗi lần. Cùng triệu chứng Phase 64 đã gặp.

## Còn treo

12/17 finding chưa xử lý. Hai cái cần chốt hướng trước vì đụng thiết kế: **B#1** (gateway
aggregator tĩnh, fail-closed toàn phần) và **A#2** (`users_email_unique` unique toàn cục thay vì
`(tenant_id, email)`). **B#7** (SchemaLimits hardcode) gộp vào
`docs/features/18-config-tiers-db-backed.md` thay vì sửa lẻ.
