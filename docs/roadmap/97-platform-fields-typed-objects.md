## Phase 97: `platform_fields` — mọi output không phải scalar giờ là GraphQL object type thật (2026-09-26)

Trigger: sau Phase 95/96, chủ dự án hỏi "tiếp theo nên làm gì với core metap" — gợi ý model lại
GraphQL type đầy đủ cho `platform_fields` (thay vì `Json` scalar thô, đã ghi rõ là "còn nợ" trong
cả 2 phase trước), chủ dự án đồng ý làm ngay.

### Việc đã làm

Mọi output không phải scalar của `metap-graphql-http::platform_fields` giờ là 1 `Object` type có
tên thật, không còn `Json` scalar chung chung: `AdminUserSummary`, `CreateAdminUserResult`,
`TenantUserSummary`, `Policy`, `CronJob`, `CronJobRun`, `OAuthClient`, `DashboardConfig`,
`Preferences`, `PlatformConfigItem`, `TenantConfigItem`, `SetConfigResult`,
`SetTenantConfigResult`. Thêm 3 `InputObject` (`MatrixGrantInput`, `CronJobInput`,
`CronJobUpdateInput`) thay cho `Json!` argument của `syncPolicyMatrix`/`createCronJob`/
`updateCronJob`.

**Cơ chế**: mỗi type là 1 view mỏng trên đúng `serde_json::Value` mà mỗi resolver đã build sẵn từ
trước (không đổi logic build data) — `JsonHandle` (mới) đóng đúng vai trò
`metap-graphql/src/record_handle.rs`'s `RecordHandle` đã làm cho entity record: bọc JSON đã
serialize sẵn, mỗi field tự resolve bằng 1 lượt tra JSON-pointer (`json_field`, helper dựng 1
`Field` cho mỗi property). `add_platform_fields` đổi chữ ký, nhận/trả thêm `SchemaBuilder` để có
thể `.register()` các type mới cùng lúc thêm field — `SchemaHolder::build` (`lib.rs`) truyền/nhận
xuyên qua y hệt.

### Vẫn giữ `Json` cho field không có 1 shape cố định — có chủ đích, không phải sót

- `CronJob.triggerConfig`/`targetConfig`, `CronJobInput`/`CronJobUpdateInput`'s field cùng tên —
  shape đổi theo `triggerType`/`targetType`, resolver đã validate đầy đủ
  (`validate_trigger`/`validate_target_config`) — model GraphQL type ở đây chỉ trùng lặp validate
  đã có, không thay thế được.
- `Policy.condition` — cây điều kiện đệ quy (`PolicyCondition`), tương tự.
- `DashboardConfig.layout` — blob JSON tuỳ ý theo widget catalog của frontend (đúng thiết kế
  `metap-dashboards` từ đầu).
- `*ConfigItem.value`/`SetConfigResult.value`/`SetTenantConfigResult.value` — giá trị tuỳ theo
  từng config key (string/bool/number/object khác nhau).
- `explainPermission`'s kết quả, `cronJobWorkflowRun`'s kết quả — trace chẩn đoán/tiến độ step,
  giữ `Json`, không model (route REST cũ cũng chưa từng có schema chính thức cho 2 cái này).
- `public_schema`'s `publicConfig` — vẫn `Json` scalar, có chủ đích: schema này chưa có consumer
  thật nào (đã dò kỹ ở Phase 95), giữ tối giản.

### Verify

`cargo build/clippy(-D warnings)/fmt --check/test --workspace` sạch. **Mọi e2e test động tới field
này đều phải sửa query string** — thêm selection set thật (`{ id name }` thay vì gọi field trơn),
vì GraphQL từ chối validate 1 query thiếu selection set cho object type — bản thân việc sửa test
đã là 1 phép verify cho việc type hoá thật sự có hiệu lực, không phải chỉ đổi tên suông:
`crates/metap-http/tests/{platform_config_postgres,tenant_config_postgres,tenant_secret_postgres,
oauth2_authorization_server_postgres,http_server}.rs`,
`crates/metap-graphql-http/tests/{platform_fields_postgres,graphql_http_postgres}.rs` — tổng 42
test, tất cả pass sau khi sửa.

### Còn nợ / không làm trong phase này

- Chưa test qua browser thật (đúng chính sách repo) — cần rebuild + restart `jira-server`/
  `waf-*-service` để pickup schema mới nếu muốn xem qua GraphiQL "Docs" panel thật.
- `public_schema`'s `publicConfig` vẫn `Json` — để nguyên, không có consumer thật để justify.
