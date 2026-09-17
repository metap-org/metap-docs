## Phase 87: Sửa e2e fixture còn trỏ vào `records` sau Phase 86, phát hiện thêm 2 bug thật (2026-09-16)

Bối cảnh: Phase 86 xoá hẳn bảng `records` chung nhưng để lại "Known gap, chưa làm trong phiên đó"
— bộ e2e test (`cargo test --workspace -- --ignored`) của `metap-crud`/`metap-graphql`/
`metap-graphql-http`/`metap-graphql-gateway`/`metap-grpc`/`metap-http`/`metap-workflow` vẫn còn
fixture `table_name: "records".to_string()` và SQL thô thao tác trực tiếp bảng vật lý đã bị xoá.
`.github/workflows/ci.yml` không chạy `--ignored` nên việc này không chặn CI, nhưng bất kỳ ai chạy
bộ e2e thật sau khi migration 0033 áp dụng sẽ gặp lỗi ngay. Phiên này làm đúng phần còn lại đó, và
verify **sống** (không chỉ compile) — dựng Postgres 16 native trong môi trường (không có Docker),
chạy từng crate e2e tuần tự (`--test-threads=1`, tránh 1 flake có sẵn không liên quan trong cách
harness dựng server song song mỗi test).

### Sửa fixture (7 file test + 1 benchmark)

Trỏ lại mỗi `table_name: "records"` fixture sang bảng dedicated riêng do chính file đó tự tạo
(`CREATE TABLE IF NOT EXISTS`, đúng pattern `metap-query/tests/query_planner_postgres.rs` đã dùng
từ Phase 86) — `metap-crud`, `metap-graphql`, `metap-graphql-http`, `metap-graphql-gateway`,
`metap-grpc` (2 file), `metap-http` (nhiều file), `metap-workflow`, và
`metap-query/benches/plan_list_bench.rs`.

Xoá hẳn (không port) mảng test trong `crud_service_postgres.rs` dựng cảnh nhiều entity chia sẻ 1
bảng qua cột discriminator `entity` (`unique_field_entity()`/`ensure_sku_unique_index()`, test
naming convention cũ `uniq_records_<entity>_<field>`) — hành vi này không còn xảy ra được nữa sau
Phase 86, và test sibling còn lại (naming convention `uniq_<table>_<field>`) đã cover đủ.

Tiện thể sửa 1 assertion đã stale từ trước: test logout trong `metap-http` còn assert xoá 2
cookie, trong khi Phase 64 (session-absolute-max-age cookie) đã thêm cookie thứ 3 từ lâu — không
liên quan Phase 86 nhưng bị phát hiện khi chạy lại bộ e2e sống lần đầu sau một thời gian.

2 test benchmark thủ công (`sustained_concurrent_list_against_a_real_multi_entity_abac_workflow`,
`sustained_concurrent_list_across_many_tenants_at_ten_million_rows`) được sửa fixture cho
compile-valid nhưng **vẫn fail đúng như thiết kế** — chúng cần 1 script seed dữ liệu ngoài (out-of-
band) riêng, không phải việc đổi tên bảng có thể thay thế. Không sửa script đó trong phiên này.

### 2 bug thật phát hiện khi verify sống, sửa luôn tại gốc

Không phải lỗi do đổi fixture — cả 2 đã tồn tại từ trước, chỉ là chưa từng bị exercise bởi 1 lần
chạy e2e sống kể từ Phase 86:

1. **`AUTH_CONTEXT_ENTITY` hỏng hoàn toàn cho mọi caller (REST lẫn gRPC), không chỉ test.**
   `metap-peripherals::fetch_context_attributes` vẫn query bảng `records` đã xoá qua cột
   discriminator `entity = $2`. Sửa: nhận `table_name` và query thẳng bảng đó, không cần
   discriminator nữa (mọi entity giờ có bảng riêng). `metap_control::resolve_request_context`
   resolve tên entity đã cấu hình (`AppState.auth_context_entity`/`AuthConfig.auth_context_entity`
   — vẫn là tên entity, không đổi shape) sang `table_name` qua tham số `MetadataRegistry` mới;
   `metap-grpc::AuthConfig`/`OptionalServeConfig` được thêm field `metadata` để cấp registry đó,
   vì đường auth của gRPC trước đây không có quyền truy cập registry. Tính năng này hiện đang
   "ngủ" ở mọi nơi đã wire nó (3 service của `metap-demo-waf` forward `state.auth_context_entity`,
   nhưng chưa nơi nào thực sự set nó thành `Some(...)`) — không phải outage đang xảy ra, nhưng sẽ
   hỏng ngay lập tức nếu ai đó bật tính năng này lên.
2. **Fixture field kiểu `Reference` (`test.children.parentId`, v.v.) cần cột vật lý thật + sync
   trigger** để khớp với những gì `metap_reconciler::compile()`/`execute()` build trong production
   — `metap_metadata::field_has_real_column` nói bất kỳ field `Reference` nào có `ref_entity` luôn
   có cột thật, vô điều kiện (nhánh shared-vs-dedicated từng gate việc này đã bị xoá cùng bảng
   `records` ở Phase 86). Thiếu cột này, query `"parentId" = $N::uuid` trong
   `find_referencing_records` sẽ âm thầm không match gì cả — delete guard sẽ cho phép xoá 1 parent
   đang bị tham chiếu, đúng dạng lỗi "guard không thực sự guard" mà `metap-demo-waf/CLAUDE.md`'s
   phát hiện thứ 8 đã mô tả. Sửa: fixture tự tạo cùng shape function/trigger mà
   `executor::build_sync_trigger_sql` sinh ra (không kéo `metap-reconciler` làm dev-dependency,
   theo đúng convention có sẵn của file này cho fixture unique-index).

### Verify

`cargo build`/`clippy --all-targets -- -D warnings`/`test --workspace` (unit) sạch. Toàn bộ e2e
suite của 7 crate trên (`cargo test -p <crate> -- --ignored --test-threads=1`) chạy sống chống lại
Postgres thật, pass hết — trừ đúng 2 test benchmark thủ công fail vì thiếu seed ngoài (như thiết
kế). `metap-workflow`/`metap-http`/`metap-graphql`/`metap-graphql-gateway`/`metap-grpc` cũng được
chạy lại full để xác nhận không có regression ngoài phạm vi sửa.

Sau khi PR mở, CI báo `Rust — dependency vulnerability audit` đỏ — không phải do PR này:
`master` tự nó đã đỏ cùng lỗi này từ trước (RUSTSEC-2026-0285, rustls TLS 1.3 handshake-boundary
bug, publish 2026-09-14). Port fix luôn vào PR này thay vì đợi 1 PR riêng: `cargo update -p rustls
--precise 0.23.45` theo đúng "Solution" của advisory, verify lại bằng `cargo audit` (không còn
báo lỗi) và full build/clippy/test lại 1 lần nữa.

### Còn lại

Chưa cập nhật `metap/CLAUDE.md`'s đoạn "Known gap, not fixed in this pass" (viết trong phiên Phase
86) — xoá đoạn đó/thay bằng ghi chú đã fix, trỏ về entry này.
