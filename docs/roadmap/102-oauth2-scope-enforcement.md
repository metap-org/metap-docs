## Phase 102: enforce OAuth2 `scope` vào RBAC/ABAC (2026-10-14)

Trigger: chủ dự án hỏi "tiếp theo roadmap nên làm gì", được gợi ý 3 hướng, chọn hướng 2 — nợ
bảo mật còn lại của Phase 88: `scope` của token OAuth2 chỉ *hiện diện* trong
`RequestContext.context_attributes`, không có gì enforce nó. Nghĩa là một client bên thứ ba được
admin duyệt consent với scope `read:widgets` vẫn làm được **mọi việc admin đó làm được** (đọc/ghi
mọi entity, tạo policy, tạo cron job...), vì `PermissionService::check_action` bỏ qua mọi policy
cho role `admin` và không ai nhìn `scope`.

### Quyết định thiết kế (chủ dự án giao cho phiên này tự chốt — đảo ngược được, ghi lại để rà)

1. **Scope chỉ thu hẹp, không bao giờ mở rộng.** Quyền thực của token = giao của (quyền RBAC/ABAC
   của user đứng sau) và (scope được cấp). Một token không scope (session login thường) không bị
   ảnh hưởng gì.
2. **Cú pháp**: `read:<entity>`, `write:<entity>` (= create + update), `delete:<entity>`,
   `transition:<entity>`; `<entity>` là tên entity chính xác hoặc `*`. Không khớp tiền tố
   (`read:waf.zones` không cho đọc `waf.zones.extra`). Token lạ ngoài cú pháp không cấp quyền gì.
   Scope rỗng = "không cấp gì", **không** phải "không giới hạn" (phân biệt qua việc claim có mặt hay
   không, không phải qua độ dài).
3. **Gate đặt trước admin bypass** trong `PermissionService::check_action` — đây là điểm hẹp duy
   nhất cả 5 `can_*_entity` (và `check_record_permission` của attachments) đều đi qua, nên mọi
   transport (GraphQL/gRPC/attachments) được phủ cùng lúc. Đặt sau admin bypass sẽ vô nghĩa:
   user admin duyệt client nào, client đó thành admin.
4. **Bề mặt admin cần scope `admin` tường minh** (`AdminContext`, `PlatformAdminContext`, mọi field
   `platform_fields` qua `caller()`/`require_platform_admin`, `explainPermission`). Lý do: các field
   này không có entity để gắn verb — policies/cron/users/config là nền tảng, không phải dữ liệu
   entity. Hệ quả có chủ đích: token scope hẹp cũng không đọc được `myPreferences`/`myDashboard`
   (an toàn theo mặc định; cấp `admin` nếu cần, hoặc sau này thêm scope mịn hơn nếu có nhu cầu).

### Lỗ hổng tìm thấy khi làm: gRPC bỏ rơi claim `scope`

`metap-grpc::auth::authenticate` chỉ lấy `tenant_id`/`sub`/`function_id` từ claims, **không bao giờ
đọc `scope`** — token OAuth2 qua gRPC (và qua `graphql-gateway`, vốn forward nguyên token của người
gọi sang upstream gRPC) là session đầy đủ. Chỉ gate ở `metap-http` sẽ để lại đúng đường vòng này.
Sửa bằng helper `RequestContext::apply_oauth_grant` (một chỗ, trong `metap-permission`) và cả
`metap-http`/`metap-grpc` đều gọi nó — transport nào quên gọi sẽ phục vụ token có scope như session
không giới hạn, nên doc comment của hàm nói thẳng điều này.

### Không làm / để lại

- Không thêm field vào `RequestContext` (48 điểm khởi tạo) — vẫn dùng `context_attributes`
  (`oauthScope`/`oauthClientId`) như Phase 88.
- Chưa enforce theo **field**/record cho scope — chỉ cấp entity+verb. Admin có token scope
  `read:waf.zones` vẫn thấy mọi field của `waf.zones` mà RBAC của họ cho thấy.
- `graphql-gateway` không tự enforce (không có roles, không có policy) — nó forward token và upstream
  gRPC enforce; một upstream cũ chưa có bản vá này vẫn phục vụ token scoped như session đầy đủ.
- Chưa có UI chọn scope/ánh xạ scope → mô tả cho consent screen (consent đã hiện scope thô).
- Còn nợ Phase 88: đã xong 2026-10-15 — `cron-scheduler` giờ gọi `metap_oauth_server::run_cleanup`
  (thư viện đã có + e2e từ `fb72c80`; bản đầu của dòng này ghi nhầm là chưa có gì, rồi từng chỉ
  "chưa ai gọi").
  `client_credentials` và consent screen cũng đã có code + test e2e từ trước.

### Xác minh

- Unit (`metap-permission`, 41 pass): 4 test cho cú pháp scope (verb↔action, khớp chính xác/`*`,
  scope rỗng/lạ không cấp gì, `admin` scope), 2 test `PermissionService` với store giả — admin có
  token scope hẹp bị chặn đúng action/entity ngoài scope, session thường giữ nguyên admin bypass.
- E2E trên Postgres 16 thật, chạy `--test-threads=1`:
  - `metap-http/tests/oauth2_authorization_server_postgres.rs` 11/11 — test mới: token OAuth2 do
    chính admin duyệt với scope `read:widgets` → `policies`/`myPreferences` bị 403; scope `admin`
    → qua; session thường của admin → qua (đối chứng).
  - `metap-grpc/tests/grpc_crud_postgres.rs` — thêm 3 assertion vào test lifecycle có sẵn: token
    scope `read:test.grpc_orders` của user admin `get` được, `create` bị `PermissionDenied`; scope
    `read:some.other_entity` thì `get` cũng bị `PermissionDenied`.
  - Các suite còn lại của `metap-http` (cookie_session, http_server, jwt_security, platform_config,
    tenant_config, tenant_secret), `metap-graphql-http`, `metap-grpc`, `metap-crud` đều pass.
- `cargo fmt`/`clippy --all-targets -D warnings` sạch cho `metap-permission`/`metap-grpc`/
  `metap-http`/`metap-graphql-http`.
- **Fail có sẵn, không liên quan thay đổi này** (xác nhận lỗi nằm ở môi trường/thiết kế test, không
  đụng code đã sửa): 2 benchmark `sustained_concurrent_list_*` của `crud_service_postgres.rs` (test
  "manual benchmark" mà `-- --include-ignored` gom nhầm, đã ghi ở Phase 11C), test cache của
  `rbac_abac_integration_postgres.rs` cần Redis/DragonflyDB (sandbox không có), 2 doctest
  `metap-grpc/src/serve.rs` không resolve được crate `metap`. Chạy song song (`--test-threads=4`)
  một số suite `metap-http`/`metap-crud` fail do dùng chung state DB — chạy tuần tự thì pass hết.
- **Đã kiểm 2026-10-14**: chạy build + clippy + test (gồm e2e trên Postgres thật) của `metap-lowcode`
  (41/41) và `metap-demo-waf` (`data-plane` 26/26, `control-plane` 28, `edge-plane` 43) với bản này —
  không lỗi nào do scope. Chạy lộ 3 lỗi có sẵn, không liên quan: fixture `table_name: "records"` trong
  `metap-lowcode` (+ test migrate legacy làm bẩn DB khi fail giữa chừng) và `result_large_err` của
  clippy 1.99 ở `scanning-service`; đã sửa ở PR riêng của 2 repo đó. `graphql-gateway` không đổi nên
  không có test riêng.
