## Phase 86: Loại bỏ hoàn toàn bảng `records` chung, bắt buộc table-per-entity (2026-09-14)

Bối cảnh: chủ dự án yêu cầu "loại bỏ hoàn toàn records, ưu tiên entity lưu riêng". `records`
(schema `metadata` từ Phase 82's migration 0030) là bảng JSONB chung ban đầu mọi entity dùng mặc
định trước khi table-per-entity tồn tại. Kể từ Phase 21 (`metap-demo-jira`), Phase 36
(`crm.customers`), Phase 79 (toàn bộ 9 entity WAF), và bản vá gần nhất trong `metap-lowcode`
(entity low-code mới publish tự động có bảng riêng, không còn hardcode `"records"`), **không còn
entity thật nào trong org này còn dùng bảng chung này** — `metap/CLAUDE.md` đã tự ghi chú đây là
"a distinct, not-yet-scheduled future direction". Phiên này lên lịch và làm luôn, kể cả DROP bảng
vật lý (xác nhận qua AskUserQuestion, chấp nhận rủi ro cho entity legacy chưa migrate — không
query được Postgres thật để xác nhận rỗng trong phiên).

### Đụng 2 repo, không chỉ `metap`

Khảo sát phát hiện `metap-lowcode`'s `LowCodeEntityDefinition::default_table_name()`
(`crates/domain/src/definition.rs`) và `presenter/routes/draft.rs`'s `save_draft` đều hardcode
`table_name: "records"` làm placeholder cho draft chưa publish — không phải data thật (luôn bị
`application::publish::resolve_table_name` ghi đè trước khi publish/reconcile), nhưng
`LowCodeEntityDefinition::validate_shape()` gọi thẳng `metap_metadata::validate()` — cùng hàm
compiler validation phiên này siết lại để từ chối `"records"`. Vì 2 repo build qua Cargo `path`
dependency độc lập (không version-lock), thứ tự merge bắt buộc: **`metap-lowcode` (PR #3, đổi
placeholder sang `UNPUBLISHED_TABLE_NAME = "metadata.lowcode_unpublished_placeholder"`) merge
trước → `metap` merge sau**. PR đó cũng tiện tay fix 2 lỗi build có sẵn không liên quan (phát hiện
khi rebuild against `metap` mới nhất): `EntityDefinition` thiếu field `audit` chưa set, và
`metap_control::build_secret_store`'s signature đã đổi từ `&AppConfig` sang `&SecretStoreConfig`
(Phase 85 side effect chưa được `metap-lowcode` theo kịp).

### `metap` core

- `crates/metap-metadata/src/compiler.rs`: `table_name_ok` bỏ hẳn nhánh `entity.table_name ==
  "records"` — chỉ còn chấp nhận `^[a-z][a-z0-9_]*\.[a-z][a-z0-9_]*$`. 2 test regression mới:
  `generic_records_table_is_no_longer_a_valid_table_name`,
  `a_bare_table_name_with_no_schema_is_rejected`.
- **Xoá hẳn code path xử lý shared-table** (không chỉ làm nó unreachable qua validation) — đây là
  phần việc lớn nhất, vì `is_dedicated(entity)`/`dedicated_table: bool` là discriminator rải khắp
  cả CRUD lẫn query planning, không gọn trong 1 file:
  - `metap-crud/src/crud_service/helpers.rs`: xoá `is_dedicated()`, gộp `RECORD_COLUMNS`/
    `RECORD_COLUMNS_DEDICATED` thành 1 hằng số, gộp `row_to_dto`/`row_to_dto_dedicated` thành 1
    hàm (tham số `entity_name` luôn truyền vào, không đọc từ cột `entity` nữa vì cột đó không còn
    tồn tại ở bất kỳ bảng nào). `unique_violation`'s prefix chỉ còn `uniq_<table>_`. Ripple qua cả
    5 submodule (`create.rs`/`list.rs`/`update.rs`/`delete.rs`/`transition.rs`) — mỗi file bỏ
    nhánh `if dedicated {...} else {...}` sinh SQL, chỉ giữ lại nhánh dedicated.
  - `metap-query/src/query_planner.rs` + `aggregate.rs` + `jql/codegen.rs`: bỏ tham số
    `dedicated_table`/nhánh `entity = $n` filter khỏi `sort_field_expression`/`value_expression`/
    `resolve_field`/`compile_expr`/`parse_and_compile_jql` — API nội bộ các hàm này đổi chữ ký
    (bỏ 1 tham số `bool`), không phải public API ngoài crate.
  - `metap-graphql-gateway/src/schema_builder.rs`: placeholder `table_name` (gateway không có
    Postgres pool riêng để query, chỉ cần qua được validation) đổi từ `"records"` sang
    `"metadata.gateway_unused_placeholder"`.
  - `metap-control/src/tenant_schema.rs`: **bug thật tự phát hiện** — `TENANT_SCOPED_TABLES`
    (Phase per-tenant-schema-isolation, phiên trước) vẫn liệt kê `("metadata", "records")`; nếu
    không xoá, `create_tenant_schema` sẽ `CREATE TABLE (LIKE metadata.records ...)` cho một bảng
    sắp không còn tồn tại ngay khi có tenant `Schema`-strategy mới được provision sau khi migration
    0033 chạy. Xoá khỏi danh sách, thêm ghi chú vào đoạn "Excluded, deliberately" giải thích lý do
    khác với 2 exclusion còn lại (`tenant_hostnames`/`outbox_events` bị loại vì không nên copy,
    `records` bị loại vì không còn tồn tại).
  - `metap-reconciler::migrate_generic_to_dedicated` + `dev-tools migrate-to-dedicated-table`:
    **giữ nguyên, cố ý không xoá** — vẫn là công cụ hợp lệ duy nhất để bất kỳ ai còn data cũ trên
    `records` (ví dụ low-code entity legacy chưa migrate) chạy trước khi migration DROP áp dụng
    lên môi trường của họ.
- Fixture test (`metap-metadata::registry.rs`/`compiler/tests.rs`/`openapi.rs`,
  `metap-crud::validation.rs`, `metap-workflow::lib.rs`, `metap-app::entities_yaml.rs`,
  `metap-reconciler::orchestrator/tests.rs`, `metap-query::aggregate/tests.rs`/`jql.rs`,
  `metap-query::tests/query_planner_postgres.rs`) đổi `table_name: "records"` sang dạng dedicated
  hợp lệ. `query_planner_postgres.rs` (e2e thật, có insert/query trực tiếp vào bảng vật lý) được xử
  lý đầy đủ nhất — tự `CREATE TABLE IF NOT EXISTS entities.test_widgets` trong `setup()` (crate
  này không phụ thuộc `metap-reconciler` nên viết DDL tay, theo đúng shape bảng dedicated thật),
  không còn đụng `records` ở bất kỳ câu SQL nào.
- Migration mới `crates/migrations/0033_drop_records_table.sql`: `DROP TABLE IF EXISTS
  metadata.records` — destructive, có comment đầu file nhắc rõ bước vận hành cần làm trước khi áp
  dụng lên môi trường có data thật (`dev-tools migrate-to-dedicated-table` cho từng entity legacy).
  Không có FK nào trỏ vào `records` (đã grep xác nhận), `DROP TABLE` tự dọn PK/index của chính nó.

### Verify

`cargo build --workspace`, `cargo clippy --workspace --all-targets -- -D warnings`, `cargo test
--workspace` (unit) đều sạch sau khi sửa hết fixture — không cần Postgres thật cho phần này.

### Known gap, chưa làm trong phiên này

**Bộ e2e test (`cargo test --workspace -- --ignored`)** của `metap-crud`/`metap-graphql`/
`metap-graphql-http`/`metap-graphql-gateway`/`metap-grpc`/`metap-http`/`metap-workflow` vẫn còn
fixture `table_name: "records".to_string()` và (đặc biệt `metap-crud/tests/
crud_service_postgres.rs`) SQL thô thao tác trực tiếp bảng `records` vật lý, kể cả 1 mảng test
riêng dựng cảnh nhiều entity cùng chia sẻ 1 bảng qua cột discriminator `entity` — chính hành vi
phiên này vừa xoá khỏi code thật, nên mảng test đó giờ test một kịch bản không thể xảy ra nữa.
`.github/workflows/ci.yml` chỉ chạy `cargo test --workspace` (không `--ignored`), nên việc này
**không chặn CI** — nhưng lần tới ai chạy bộ e2e thật (cần Postgres) sau khi migration 0033 đã áp
dụng, các test này sẽ fail vì bảng không còn tồn tại. Cần 1 phiên riêng: trỏ lại từng fixture vào
`CREATE TABLE IF NOT EXISTS` bảng dedicated của riêng nó (theo đúng pattern
`query_planner_postgres.rs` đã áp dụng trong phiên này), và xoá hẳn (không port) mảng test
shared-table-specific trong `crud_service_postgres.rs`.
