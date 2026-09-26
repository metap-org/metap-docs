## Phase 96: `get`/`{entity}List` GraphQL error thiếu `extensions` — tìm sống lúc điều tra 2 test fail (2026-09-26)

Trigger: sau Phase 95, verify sống phát hiện `crates/metap-http/tests/http_server.rs` có 2 test
fail (`auth_context_entity_enriches_org_scoped_policies_and_supports_explicit_cache_invalidation`,
`full_http_lifecycle_over_a_real_server_and_a_real_jwt`) — lúc đó ghi nhận là "pre-existing, không
đụng, ngoài phạm vi Phase 95". Chủ dự án yêu cầu quay lại điều tra và sửa cả 2.

### Phát hiện: 2 test fail chỉ là 1 bug duy nhất, không phải 2 bug riêng

Audit 04 finding B#2 (2026-09-03) đã chuyển mọi resolver sang dùng `service_result_to_gql` —
helper build đúng shape `extensions.code`/`status`/`fieldErrors` cho lỗi GraphQL, thay vì string
phẳng `"{status}: {message}"` cũ. Nhưng field `get` (`Query.{camel}(id)`) và `{entity}List` **bị
bỏ sót** lúc đó — cả 2 vẫn tự build lỗi bằng `format!("{status}: {}", message.unwrap_or(error))`
ngay tại chỗ, không có `extensions` gì cả.

Hệ quả: `full_http_lifecycle_over_a_real_server_and_a_real_jwt`'s assertion
`errors[0].extensions.status == 404` fail — **không phải vì record chưa xoá xong**, mà vì
`extensions` không tồn tại nên đọc ra `null`, không phải `404`. Chẩn đoán ban đầu ("có thể là bug
delete thật") ghi trong Phase 95's doc là sai — sau khi sửa field `get`, cả 2 test tự pass mà không
đụng gì tới logic delete.

### Fix

Tách phần build `extensions` ra khỏi `service_result_to_gql` (`crates/metap-graphql/src/
schema.rs`) thành hàm riêng `service_error_to_gql(status, error, message, field_errors) ->
GqlError`, rồi gọi hàm này ở nhánh lỗi của `get`/`list`. `service_result_to_gql` chính nó gọi lại
hàm này cho nhánh `Err`, hành vi không đổi cho mọi caller cũ (create/update/delete/transition,
federation's `entity_resolver`). `list`'s nhánh `Ok` vẫn phải tự match tay (không gọi
`service_result_to_gql` trực tiếp được) vì nó cần cả `page`, thứ hàm kia bỏ qua — chỉ nhánh `Err`
đổi.

### Verify

`cargo build/clippy(-D warnings)/fmt --check/test --workspace` sạch. Thêm test mới
(`crates/metap-graphql/tests/graphql_schema_postgres.rs`'s
`get_and_list_permission_errors_carry_the_same_extensions_shape_mutations_do`) — permission-denied
qua cả `get` và `list` đều phải có `extensions.code`/`status`, y hệt mutation. Chạy lại toàn bộ
`graphql_schema_postgres.rs` (5 test), `graphql_http_postgres.rs`, `platform_fields_postgres.rs`,
và `metap-http/tests/http_server.rs` (4 test, cả 2 test từng fail giờ pass) — tất cả pass, không
regression nào khác.

### Còn nợ / không làm trong phase này

- Không model lại error shape cho REST (REST route đã xoá hết những gì liên quan, `metap-http`'s
  còn lại — `/auth/*`, attachments, oauth2 protocol — dùng error shape riêng của chúng, không đụng
  tới `service_result_to_gql`).
