# Crate dùng chung `metap-contrib` — plumbing xuyên cắt, "cách repo này viết code"

- **Trạng thái:** in-progress — crate đã tạo, 3 module đầu có test, **chưa** wire vào call site nào
  đang bị duplicate (đúng yêu cầu chủ dự án: "rà soát trước", di trú call site cũ là việc riêng)
- **Người đề xuất:** chủ dự án, 2026-08-31 (cùng lúc yêu cầu tách `metap-lowcode` thành mono-repo
  microservices — xem ghi chú "Liên quan" cuối file)
- **Track sở hữu:** Backend Core
- **Phase roadmap liên quan:** Phase 53 (`docs/roadmap/53-metap-contrib-and-lowcode-microservices-plan.md`)

## Vấn đề / động lực

`metap` (repo core) và `../metap-lowcode` (repo SaaS low-code, path dependency vào `metap`) đều tự
viết tay cùng vài loại boilerplate xuyên cắt (HTTP client, bearer-token parsing, đọc env var) mỗi
nơi một kiểu, không có chỗ chung nào để tham chiếu "đây là cách repo này làm việc đó" — rủi ro thật
đã xảy ra: 2 chỗ dựng `reqwest::Client` không set timeout (xem "Rà soát" dưới), một upstream treo có
thể làm hang vô hạn.

## Rà soát (bằng chứng thật, đọc code trước khi viết crate — không đoán)

**3 ứng viên xác nhận có duplicate thật, đã đưa vào `metap-contrib` (`crates/metap-contrib/src/`):**

| Module | Vấn đề tìm thấy | Bằng chứng (file:line) |
|---|---|---|
| `http_client` | `reqwest::Client::new()` không timeout — hang vô hạn nếu upstream treo | `graphql-gateway/src/schema_builder.rs:49`, `metap-jwks/src/client.rs:30` (không timeout) vs `cron-scheduler/src/main.rs:37` (tự set 30s, không chỗ nào tái dùng) |
| `bearer` | Cùng logic `strip_prefix("Bearer ")` viết tay 3 lần, 3 error type khác nhau | `metap-http/src/auth.rs:93-95` (axum), `graphql-gateway/src/server.rs:54-56` (axum thủ công), `metap-grpc/src/auth.rs:65-67` (tonic) |
| `env` | Idiom `env::var(X).ok().and_then(...).unwrap_or(default)` lặp | `metap-infra/src/config.rs` (~29 lần), `graphql-gateway/src/config.rs` (6 lần, độc lập, không tái dùng `AppConfig`) |

**Đã kiểm tra, KHÔNG đưa vào — đã tập trung tốt sẵn, không có duplicate thật:**
- gRPC client + service-JWT attach — chỉ 1 định nghĩa (`metap-grpc::client::GrpcBackend`),
  `graphql-gateway` tái dùng đúng.
- JWT decode — tập trung ở `metap_peripherals::decode_access_token`, dùng chung bởi
  `metap-http`/`metap-grpc`/`graphql-gateway`.
- Error response HTTP (`ApiError`/`IntoResponse`) — `metap-lowcode-http` tự import
  `metap_http::error::{internal_error_response, service_error_response}`, không có bản copy nào.
- `AuthContext`/`AdminContext` — định nghĩa 1 lần ở `metap-http/src/auth.rs`, mọi route ở
  `metap-lowcode-http`/`metap-control-http` import lại.
- List/pagination boilerplate ngoài `QueryPlanner` — không route nào tự parse query param tay.
- Request/response envelope JSON — nhất quán `{"data": ...}` qua `serde_json::json!` mọi nơi.

## Phạm vi

**Trong phạm vi (đã làm):**
- Tạo `crates/metap-contrib` (workspace member mới), 3 module: `http_client::{build, default_client}`,
  `bearer::parse_bearer`, `env::{env_or, require_env}` — mỗi hàm có unit test, `cargo clippy -D warnings` sạch.
- Framework-agnostic có chủ đích (`bearer::parse_bearer` không biết axum/tonic là gì) để cả
  `metap-http` (axum) lẫn `metap-grpc` (tonic) đều gọi được, tự map `None` sang error type riêng.

**Ngoài phạm vi (chưa làm, follow-up riêng — cần chốt trước khi làm):**
- Di trú `graphql-gateway`/`metap-jwks`/`cron-scheduler` sang dùng `http_client::default_client()`
  thay `reqwest::Client::new()` tay.
- Di trú `metap-http`/`graphql-gateway`/`metap-grpc` sang dùng `bearer::parse_bearer`.
- Di trú `graphql-gateway::config` sang dùng `env::env_or`/`env::require_env`.
- Bất kỳ module mới nào khác (vd generalize `GrpcBackend`'s channel+interceptor construction thành
  helper dùng chung, hay một cơ chế truyền `tenant_id` qua network boundary) — chỉ thêm khi có
  **caller thứ 2 thật** cần, không thêm trước (đúng nguyên tắc "không design cho tương lai giả
  định" — xem CLAUDE.md gốc của dự án).

## Tiêu chí chấp nhận

- `cargo build -p metap-contrib` và `cargo test -p metap-contrib` sạch — đạt (10/10 test pass).
- `cargo clippy -p metap-contrib -- -D warnings` sạch — đạt.
- `cargo build --workspace` ở `metap` sau khi thêm workspace member mới vẫn sạch — đạt.
- Không có call site cũ nào bị sửa trong lượt này (đúng yêu cầu rà soát trước) — đạt, verify bằng
  diff: chỉ file mới trong `crates/metap-contrib/`, không đụng `graphql-gateway`/`metap-http`/
  `metap-grpc`/`metap-infra`/`metap-jwks`.

## Ranh giới kiến trúc bị đụng tới

Không đụng layering hiện có (`docs/architectures/05-building-blocks.md`) — đây là một crate
plumbing ngang hàng `metap-infra`, không phải một layer mới. Không cần ADR riêng cho việc thêm
crate này (không phải quyết định kiến trúc gây tranh cãi) — nhưng phần "microservices cho
`metap-lowcode`" đi kèm yêu cầu ban đầu **có** đụng một ADR đã ghi (`09-adr.md` dòng 55, "Không
tách microservice cho hướng SaaS multi-tenant") — xem `docs/roadmap/53-metap-contrib-and-lowcode-microservices-plan.md` để biết phần đó đang ở trạng thái nào (chờ chủ dự án quyết định hướng, chưa
làm).

## Rủi ro / phụ thuộc

- Migrate call site cũ (`graphql-gateway`, `metap-jwks`, `cron-scheduler`, `metap-http`,
  `metap-grpc`) là hành vi thay đổi runtime thật (vd thêm timeout 30s cho request trước đây không
  có timeout) — dù nhỏ, vẫn nên làm thành 1 lượt riêng có review, không gộp lẫn vào lúc tạo crate.
- Hướng "mono-repo microservices" cho `metap-lowcode` đã được chọn (xem Phase 53,
  `../metap-lowcode/docs/architecture.md`) — khi T3/T4 (2 binary service mới) thật sự viết
  `main.rs`, đó sẽ là caller thứ 2/3 cho một helper "dựng `AppState` tối thiểu" — lúc đó tách vào
  `metap-contrib` là đúng lúc (chưa làm bây giờ, chỉ 1 caller thật hiện có).
