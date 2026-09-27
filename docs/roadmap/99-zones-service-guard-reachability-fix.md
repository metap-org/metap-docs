## Phase 99: `zones-service`'s create/update/delete guards đã hoàn toàn unreachable — sửa 3/4, chốt gap còn lại (2026-09-27)

Trigger: sau Phase 98 (chứng minh sống pipeline `control-plane`/`edge-plane`), chủ dự án hỏi tiếp
"tiếp theo làm gì", được gợi ý 2 hướng — (1) điều tra gap `zone_domain_guard`/`zone_config_guard`
đã ghi trong Phase 98's "Còn nợ" (middleware REST-only có thể không chạy qua đường GraphQL/gRPC
thật), (2) chốt hướng kiến trúc "ledger drift" ở `metap-reconciler` (chỉ gợi ý). Chủ dự án chọn "1
được làm xem" — phase này là kết quả điều tra + sửa của (1).

### Phát hiện: cả 4 guard đều là dead code — bug thật đang chặn nút "Create Zone" của portal thật

Đọc lại `zones-service/src/routes.rs`: cả 4 guard (`zone_domain_guard`,
`firewall_rule_match_condition_guard`, `ip_access_list_value_guard`, `zone_delete_guard`) đều là
axum middleware gắn thẳng vào literal REST path (`/api/waf.zones`, `POST /api/waf.firewall_rules`,
...). Route này đã bị `metap` core xoá hẳn từ 2026-09-21 (bỏ REST entity CRUD) — và
**`zones-service` chưa bao giờ tự mount `metap-graphql-http::router()`** (chỉ có gRPC,
`GRPC_ENABLED`, để `waf-graphql-gateway` tổng hợp lại). Nghĩa là kể từ lúc REST bị xoá, đường tạo/
sửa/xoá `waf.zones`/`waf.firewall_rules`/`waf.ip_access_lists` **duy nhất** còn lại là gRPC — và
không guard nào trong 4 cái từng chạm vào đường đó cả.

Hậu quả rõ nhất, xác nhận sống: nút "Create Zone" ở trang Onboarding thật (`web/src/pages/
OnboardingPage.tsx`) gọi `createRecord(ENTITIES.zones, {...})` **không bao giờ tự truyền
`domainId`** — comment ngay trong code ghi rõ "domainId is assigned server-side (`routes::
zone_domain_guard`)". Vì guard đó chưa từng chạy qua GraphQL/gRPC, mọi lần tạo Zone thật từ portal
đều nhận `validation_failed: domainId required` — tái hiện được ngay lập tức bằng 1 lệnh `curl`
GraphQL thủ công trước khi sửa gì.

### Thiết kế sửa

`state.crud` (kiểu cụ thể `Arc<CrudService>`) được gRPC serve trực tiếp — không có chỗ nào để chèn
logic guard vào đường gRPC cả, vì `metap-grpc::GrpcRecordService`/`OptionalServeConfig` hard-code
kiểu `Arc<CrudService>` thay vì dùng seam `RecordBackend` mà `metap-graphql` đã dùng từ lâu cho
đúng mục đích này (decoration/remote dispatch).

**Sửa ở `metap` core trước** (`crates/metap-grpc/src/{service,serve}.rs`): đổi
`GrpcRecordService::new`/`OptionalServeConfig.crud` từ `Arc<CrudService>` sang
`Arc<dyn RecordBackend>` — coercion tự động ở mọi call site cũ (`CrudService` đã impl
`RecordBackend` sẵn), không phải sửa gì ở `metap-demo-jira`/`metap-lowcode`/2 service WAF còn lại
(build lại cả 2 repo, xác nhận không đổi). Kèm 1 sửa nhỏ: `aggregate` RPC handler trước đó tự
convert `AggregateSpec` → `AggregateInput` rồi gọi thẳng `CrudService::aggregate` (inherent
method, nhận `&AggregateInput`) — giờ gọi qua trait (`&AggregateSpec`), bỏ bước convert thừa.

**`zones-service` mới**: module `guarded_backend.rs`, `GuardedZonesBackend` bọc
`Arc<dyn RecordBackend>`, port lại đúng logic của 3/4 guard vào `create`/`update`:
- `waf.zones` create: resolve/tạo `Domain` theo apex của `hostname`, chèn `domainId` trước khi gọi
  `inner.create` — y hệt `zone_domain_guard` cũ, chỉ đọc/ghi thẳng `JsonObject` thay vì parse body
  axum.
- `waf.firewall_rules` create/update: validate `matchCondition` qua
  `crate::match_condition::is_valid_match_condition` (logic gốc, không đổi).
- `waf.ip_access_lists` create/update: validate `value` qua `crate::ip_format::is_valid_ip_or_cidr`
  (logic gốc, không đổi).

`main.rs` bọc `state.crud` bằng `GuardedZonesBackend` trước khi đưa vào `metap::grpc::optional_serve`
— đây là **transport duy nhất còn thật sự nhận mutation** nên là chỗ đúng để chèn guard. Xoá hẳn
4 `.layer(axum::middleware::from_fn...)` cũ (dead code, xoá hẳn theo đúng convention "delete rather
than deprecate" của cả tổ chức), xoá luôn 3/4 hàm guard trong `routes.rs` (logic đã chuyển sang
`guarded_backend.rs`).

### Gap có chủ đích, không đóng trong phase này: `zone_delete_guard`

Guard thứ 4 (kiểm tra cross-service trước khi xoá Zone — chặn nếu `scanning-service`/
`alerting-service` vẫn còn `ScanJob`/`Incident`/`SecurityEvent` trỏ vào zone đó) **không port**.
Bản REST cũ tự nó cũng đã hỏng kép: vừa unreachable (REST guard), vừa gọi
`GET {scanning,alerting}-service/api/{entity}` — cũng đã bị xoá. Port đúng cách cần
`zones-service` tự dial gRPC của 2 service kia bằng 1 service-account login riêng — 1 phụ thuộc
boot-time thật giữa 3 service mà chính tài liệu app này mô tả là "độc lập triển khai" (`../CLAUDE.md`'s
bảng 3-plane/3-service). Đây là quyết định kiến trúc/triển khai, không phải fix 1 dòng — cờ lại
thay vì đoán, theo đúng convention "không tự quyết architecture, cờ lên" đã ghi trong
`../CLAUDE.md`'s "Notable open questions". **Hậu quả tạm thời**: xoá 1 Zone còn tham chiếu từ
service khác hiện không bị chặn, không báo lỗi — orphan âm thầm, là 1 regression thật so với hành
vi REST cũ (dù bản REST cũ cũng đã unreachable, ít nhất về mặt thiết kế nó từng đúng).

### Phát hiện phụ — cùng ngày, đóng luôn theo yêu cầu chủ dự án ("làm đi")

`cargo test --workspace -- --ignored` phát hiện cả 3 `data-plane` service (`zones-service`,
`scanning-service`, `alerting-service`) đều có `tests/http_server.rs` (bản smoke-test gốc copy từ
`metap-http`, byte-identical cả 3) vẫn gọi REST `/api/{entity}` trực tiếp — fail `404` thay vì
`401`/`201` mong đợi. Cùng gốc rễ với mọi phát hiện REST-removal khác trong 2 phase gần đây, khác
file/phạm vi — ban đầu cờ lại chưa sửa, sau đó chủ dự án yêu cầu làm luôn cùng ngày.

**Sửa**: port cả 3 file (byte-identical, sửa 1 lần rồi copy) theo đúng pattern
`metap-http/tests/http_server.rs` đã tự làm ở Phase 90 — mount `metap::graphql_http::router(&state,
...)` làm `extra_routes` của `build_router`, gọi `/graphql` (`createTestTasks`/`testTasks` thay vì
`/api/test.tasks`) thay vì REST. Test tự nó vẫn dựng 1 axum server thật + JWT RSA thật (khác các
guard test ở trên gọi thẳng `RecordBackend` trong process) — vì mục đích của file này đúng là
"chứng minh axum + `AuthContext` + `CrudService` chạy đúng qua HTTP thật", không phải test riêng
1 guard cụ thể. Cả 3 test pass qua Postgres thật.

### Verify

- Viết lại 3 file test cũ (`zone_domain_guard_postgres.rs`, `firewall_rule_match_condition_guard_postgres.rs`,
  `ip_access_list_value_guard_postgres.rs`) — trước đây dựng cả 1 axum server thật + mint JWT RSA để
  test qua REST, giờ gọi thẳng `GuardedZonesBackend`'s `RecordBackend` method trong process, dùng
  `RequestContext` dựng tay (role `admin` bypass permission check qua `is_admin()`, không cần JWT/
  `user_roles` nào) — đơn giản hơn hẳn bản cũ. 4 test qua Postgres thật, tất cả pass.
- `cargo build/clippy(-D warnings)/fmt --check/test --workspace` sạch cho cả `metap` (core change)
  và `data-plane` (toàn bộ, gồm cả `metap-demo-jira`/`metap-lowcode` build lại xác nhận không đổi).
- Verify sống thật qua gateway: tạo `Zone` **không truyền `domainId`** (đúng payload portal thật
  gửi) qua `createWafZones` GraphQL mutation thật (zones-service + scanning-service +
  alerting-service + waf-graphql-gateway chạy thật) — thành công, `domainId` tự resolve ra 1
  `Domain` mới tạo cho apex đúng. Dọn dữ liệu test sau đó.

### Còn nợ

- `zone_delete_guard`'s cross-service check — xem mục "Gap có chủ đích" ở trên, cần chủ dự án chốt
  hướng kết nối 3 service. (3 `http_server.rs` test đã đóng cùng ngày, xem mục "Phát hiện phụ".)
- Chưa test sống pillar DDoS/IpAccessList qua cùng cơ chế `GuardedZonesBackend` (chỉ Zone +
  FirewallRule + IpAccessList validate được test trực tiếp; DDoS create không đi qua guard nào cả
  nên không cần, nhưng chưa xác nhận sống qua portal thật).
