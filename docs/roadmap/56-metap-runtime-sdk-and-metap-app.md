## Phase 56: `metap-runtime` trở thành SDK viết backend tuỳ biến thật + crate mới `metap-app` (2026-08-31)

Tiếp nối Phase 53's `metap-contrib` (đổi tên `metap-runtime` cùng ngày). Chủ dự án nêu rõ mục tiêu:
"mục tiêu là có runtime để viết backend tùy biến, và core metap cũng phụ thuộc vào runtime đó nữa"
— ngoài khai báo entity qua metadata, dev phải viết được route/handler thật, và `metap`-core (không
chỉ downstream app) cũng nên dùng `metap-runtime`.

**Vòng rà soát thứ 4 trong ngày cho `metap-runtime`** (sau 3 vòng ở Phase 53): tìm thêm
`rate_limit::build`/`trace::build` (kéo ra khỏi `metap-http::build_router`; `graphql-gateway` không
có rate-limit/tracing-span nào cả — gap tính năng thật, đóng luôn cùng lượt), `request_id`/
`request_context` (move nguyên vẹn, không caller ngoài nào tham chiếu tên cũ), `serve::run`
(`TcpListener::bind` + `axum::serve(...).with_graceful_shutdown(...)` + log "listening", lặp ở
4 `main.rs`: `metap-demo-crm`/`metap-demo-jira`/`templates/metap-app`/`graphql-gateway`).

**Rủi ro vòng lặp dependency tự phát hiện bởi chủ dự án** trước khi code: hàm "bootstrap AppState"
(pool + `Router` + `PermissionService` + JWT keypair, lặp thật ở 3 `main.rs` —
`metap-demo-crm`/`metap-demo-jira`/`templates/metap-app`) cần gọi `metap-control`/
`metap-permission`/`metap-cache`/`metap-infra` — nhưng 4 crate đó đã depend `metap-runtime`. Nhét
thẳng vào `metap-runtime` sẽ tạo vòng lặp thật (`metap-infra` → `metap-runtime` → `metap-infra`).
Đi qua `EnterPlanMode`/`ExitPlanMode` trước khi code — chốt **crate mới `metap-app`**, cùng tầng
facade `metap` (depend hết core crate cần thiết, không ai depend ngược — verify `cargo tree -p
metap-infra | grep metap-app` rỗng). `metap-app::bootstrap_platform(&AppConfig) -> PlatformParts`
thay đúng đoạn lặp 3 lần; entity registration/reconciliation/gRPC-notification-worker toggle/static
file serving vẫn ở lại từng `main.rs` — genuinely khác nhau, không phải boilerplate.

**Tài liệu hoá pattern "viết backend tuỳ biến"** (không viết abstraction mới — rà lại thấy mọi
primitive cần thiết đã có sẵn, generic, đúng): crate-level doc comment của `metap-app` liệt kê 5
gạch đầu dòng (custom router qua `extra_routes`, `Router::pool_for`, `AuthContext`/`AdminContext`,
`metap_infra::outbox::enqueue`, `EventBus::subscribe`), trỏ `metap-lowcode-http` làm ví dụ thật
đang chạy. `templates/metap-app/README.md` thêm mục ngắn cùng nội dung.

**Verify `templates/metap-app`**: Cargo.toml gốc dùng git-URL placeholder (`{{metap_git}}`/
`{{metap_rev}}` cho `cargo generate`), không build trực tiếp được trong repo — verify qua bản copy
sang scratchpad, thay placeholder bằng path dependency cục bộ, build/clippy/test thật. Phát hiện
thêm 1 bug có sẵn không liên quan tới lượt này: `tests/http_server.rs`'s `EntityField` literal
thiếu field `storage` (thêm vào struct từ trước, test chưa cập nhật) — fix luôn vì chặn verify.

`metap-runtime` giờ 10 module/19 test; `metap-app` mới, 1 hàm. 12 crate/binary phụ thuộc
`metap-runtime` (10 ở `metap` + 2 ở `metap-lowcode`). `cargo build/test (89 ok, 0 fail)/clippy -D
warnings/fmt --all --check` sạch toàn `metap` (từ sạch, `cargo clean` trước — dọn ~41GB `target/`
tích luỹ qua nhiều lượt build/test/clippy) và `metap-lowcode`; `metap-demo-crm`/`metap-demo-jira`
build lại sạch sau migrate; `graphql-gateway` build/clippy sạch sau khi thêm middleware mới.
