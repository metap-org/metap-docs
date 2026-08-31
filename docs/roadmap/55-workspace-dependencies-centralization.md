## Phase 55: `[workspace.dependencies]` cho dependency dùng chung (2026-08-31)

Trước phase này, 29 crate trong workspace mỗi cái tự khai `version = "..."` cho mọi dependency
bên ngoài — không có version nào lệch nhau thật (verify bằng grep toàn bộ `crates/*/Cargo.toml`
trước khi sửa gì), nhưng update version sau này sẽ phải sửa N chỗ thay vì 1, và không có nơi nào
thể hiện rõ "đây là dependency chuẩn của cả workspace" so với "chỉ 1 crate cần thứ đặc thù này".

**Chốt dùng dependency nào ở `[workspace.dependencies]` bằng grep thật** (đếm số crate dùng mỗi
dependency, không đoán): 21 dependency được >= 2 crate dùng đưa vào — `anyhow`, `arc-swap`,
`async-graphql`, `async-trait`, `axum`, `base64`, `bytes`, `chrono`, `dotenvy`, `futures-util`,
`jsonwebtoken`, `moka`, `reqwest`, `secrecy`, `serde`, `serde_json`, `sqlx`, `tokio`, `tower-http`,
`tracing`, `uuid`. Dependency chỉ 1 crate dùng (`tracing-subscriber`/`lapin`/`argon2`/`ring`/
`aws-sdk-s3`/`lettre`/`tonic`/`prost`/...) **giữ nguyên local**, không đưa lên workspace.

**2 dạng khai báo, tuỳ dependency có feature set đồng nhất hay không** (verify bằng grep từng
biến thể trước khi quyết định dạng nào):
- Đồng nhất mọi nơi (hoặc superset an toàn) — full spec ở workspace, member chỉ cần
  `dep.workspace = true`: `anyhow`, `tracing`, `async-trait`, `arc-swap`, `base64`, `bytes`,
  `dotenvy`, `secrecy`, `jsonwebtoken`, `serde` (`["derive"]` mọi nơi), `serde_json` (1 chỗ pin
  cứng `"1.0.151"` ở `metap-peripherals`, chuẩn hoá về `"1"` như mọi nơi khác), `chrono` (10/12
  chỗ cần `["serde"]`, 2 chỗ không cần nhưng bật thêm không hại gì — dùng superset
  `["serde"]` cho cả 12), `uuid` (tương tự, superset `["v4", "serde"]` cho cả 24 chỗ dù
  nhiều chỗ chỉ cần 1 trong 2 feature), `moka` (`["future"]` mọi nơi), `reqwest`
  (`default-features = false, features = ["json", "rustls-tls"]` mọi nơi), `futures-util`
  (`default-features = false, features = ["std"]` mọi nơi).
- Feature set khác nhau thật giữa các crate — chỉ version/`default-features` ở workspace, mỗi
  member tự khai `features = [...]` riêng qua `dep = { workspace = true, features = [...] }`
  (Cargo: feature inherit từ workspace chỉ CỘNG thêm, không bớt được — nếu để superset ở workspace
  thì crate nhẹ như `graphql-gateway` (chỉ cần `postgres`/`runtime-tokio`/`tls-rustls`) sẽ bị ép
  compile luôn `uuid`/`chrono`/`json`/`macros`/`migrate` của những crate nặng hơn): `tokio` (14
  biến thể feature khác nhau/35 lần dùng), `sqlx` (nhiều biến thể — hầu hết dùng
  `runtime-tokio,tls-rustls,postgres,uuid,chrono,json` nhưng `db-migrate`/`dev-tools` cần
  `macros,migrate` thay vì `chrono,json`, `graphql-gateway`/`metap-graphql*`/`metap-grpc` chỉ cần
  3 feature cơ bản), `axum` (đa số trơn, `metap-http` cần thêm `["multipart"]`), `tower-http`
  (`["cors"]` vs `["cors", "trace"]`), `async-graphql` (trơn vs `["dataloader"]`).

Migrate cơ học qua script Python (không sed/regex thô để tránh sửa nhầm comment — chỉ 1 crate,
`metap-storage`, có comment trong `Cargo.toml`, verify trước khi chạy script là không đụng tới),
áp cho 28/29 crate (`crates/metap` — facade, chỉ có path dependency nội bộ — không đổi gì).

**Xác nhận**: `Cargo.lock` đã chỉ có đúng 1 version cho mỗi dependency centralize kể cả trước khi
sửa (verify bằng `grep` trong `Cargo.lock`) — nghĩa là centralize `[workspace.dependencies]` ở đây
chủ yếu là chuẩn hoá chỗ khai báo version/policy (update sau này chỉ sửa 1 chỗ, không phải N chỗ),
**không phải** cách giảm dung lượng `target/` — `target/` phình to là do artifact tích luỹ qua
nhiều lượt build/test/clippy trong 1 phiên làm việc, xử lý riêng bằng `cargo clean` (không liên
quan tới việc dependency có centralize hay không).
`cargo build/test/clippy --workspace -D warnings/fmt --all --check` sạch toàn `metap`; `metap-lowcode`/
`metap-demo-crm`/`metap-demo-jira` build lại sạch không cần sửa gì (không phải member của
workspace `metap`, path dependency vào từng crate cụ thể vẫn hoạt động y hệt — `workspace.dependencies`
chỉ áp trong nội bộ workspace gốc, không lan sang consumer ngoài).
