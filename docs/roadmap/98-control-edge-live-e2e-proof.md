## Phase 98: chứng minh sống pipeline `control-plane` → `edge-plane`, tìm ra 2 bug thật đang chặn đường (2026-09-27)

Trigger: chủ dự án hỏi "tiếp theo làm gì", được gợi ý 2 hướng — (1) chứng minh sống pipeline
`control-plane`/`edge-plane` (gap lớn nhất còn treo của `metap-demo-waf`, ghi rõ trong
`../CLAUDE.md`'s "Còn không covered" từ Phase 72), (2) chốt hướng kiến trúc cho "ledger drift"
pattern ở `metap-reconciler` (chỉ là gợi ý, chưa làm). Chủ dự án chọn "1 làm đi, 2 gợi ý" — phase
này là kết quả của (1).

### Phát hiện: `control-plane` đã hỏng hoàn toàn từ 2026-09-21, chưa ai biết

Việc đầu tiên khi bắt tay vào chứng minh pipeline sống: đọc lại `control-plane/waf-config-
distributor/src/dataplane.rs` để hiểu cách nó gọi `data-plane`. Hoá ra nó gọi REST
`GET/POST /api/{entity}...` — route này đã bị `metap` core xoá hẳn từ 2026-09-21 (bỏ REST entity
CRUD, GraphQL-only). Vì `control-plane` không có test e2e sống nào của riêng nó (chỉ unit test
thuần logic, đã ghi rõ trong `../CLAUDE.md` từ Phase 72), không ai từng chạy lại pipeline này kể từ
lúc REST bị xoá — bug nằm im 6 ngày không bị phát hiện. Đây chính là lý do "chứng minh sống" là
việc đáng làm nhất còn lại: không phải chỉ thiếu bằng chứng, mà pipeline thật sự đang gãy.

**Sửa**: viết lại `dataplane.rs` theo đúng pattern GraphQL đã dùng ở Phase 93/94
(`platform-ui`/`metap-demo-jira/web`) — naming convention camelCase hand-mirror từ
`metap-graphql/src/naming.rs` (`waf.zones` → `wafZones`/`wafZonesList`), field selection tĩnh (biết
trước entity nào cần field gì, không cần fetch metadata động như phía TypeScript). `Record` struct
giữ nguyên shape (`id`/`status`/`data: Map`) nên `compile.rs` (và toàn bộ 28 unit test của nó)
không phải sửa gì.

### Phát hiện thứ 2: URL đúng không phải port của từng service, mà là gateway

Sửa xong GraphQL, chạy thật vẫn `404 Not Found` ở `POST /graphql`. Hoá ra `zones-service`/
`alerting-service` **không tự mount `metap-graphql-http::router()`** — mỗi service chỉ có gRPC
(`GRPC_ENABLED`) để `waf-graphql-gateway` (port 4000) tổng hợp lại; bản thân service không có
`/graphql` nào để gọi trực tiếp. `ZONES_URL`/`ALERTING_URL` của `control-plane` phải trỏ vào chính
gateway đó, không phải port riêng của từng service. Đã sửa default (`config.rs`, `.env.example`)
từ `:3000`/`:3020` sang `:4000` (cùng 1 URL cho cả 2, vì gateway tổng hợp cả 3 service vào 1
schema) — ghi rõ lý do trong `dataplane.rs`'s module doc comment để không ai lặp lại nhầm lẫn này.

### Phát hiện thứ 3: cả 3 service data-plane không build được

Khởi động `zones-service`/`scanning-service`/`alerting-service` để test sống thì cả 3 build lỗi
`E0560: struct RouteGroups has no field named 'dashboards'` — di sản của Phase 95 (xoá hẳn nhóm
route `dashboards` khỏi `RouteGroups`, chuyển sang GraphQL `platform_fields`) mà main.rs của cả 3
service chưa cập nhật theo (comment cũ vẫn nói "cron/tenant_config stay on" — mô tả sai, 2 nhóm đó
cũng không còn là `RouteGroups` toggle nữa, đã chuyển hết sang GraphQL). Cùng lớp bug với
`metap-demo-jira/src/main.rs`'s `tenant_config` field ở phiên trước — downstream binary không tự
cập nhật theo signature change của core. Sửa cả 3 file (bỏ field `dashboards: false`, sửa lại
comment cho đúng thực tế).

### Verify sống — lần đầu tiên trong lịch sử repo này

Dựng đủ stack thật: `zones-service` + `scanning-service` + `alerting-service` + `waf-graphql-
gateway` (data-plane), `waf-config-distributor` (control-plane, resync 15s cho nhanh), `waf-edge`
(edge-plane), tất cả trỏ Postgres/RabbitMQ/DragonflyDB container sẵn có (`metap-postgres-1`/
`metap-rabbitmq-1`/`metap-dragonfly-1`).

1. Tạo `Domain` (mượn record `example.com` sẵn có) + `Zone` (`e2e-verify.example.com`, set thẳng
   `domainId`/`verificationStatus`/`hasConfig` — bỏ qua middleware REST-only tự động điền các field
   này, vì mục tiêu là test pipeline compile/publish/enforce, không phải UX onboarding) qua GraphQL
   thật trên gateway, `transitionWafZones(action: "activate")` để đưa zone về `active`.
2. Tạo `FirewallRule` chặn path chứa `/e2e-blocked`.
3. `waf-config-distributor` tự resync (log: `published rule-set hostname="e2e-verify.example.com"
   rules=1`) — xác nhận trực tiếp bằng `redis-cli GET waf:zone:e2e-verify.example.com`, đúng JSON
   đã compile.
4. `waf-edge` tự load lại rule-set (`zones=1` trong `/__edge/health`).
5. `curl -H "Host: e2e-verify.example.com" .../e2e-blocked/admin` → **403 Request blocked** (đúng
   rule vừa tạo). `curl` path khác → **502 Origin unreachable** (origin giả không tồn tại, nhưng
   quan trọng là *không* bị block — xác nhận WAF chỉ chặn đúng path được cấu hình, không chặn nhầm).
6. Telemetry vòng lên: `/__edge/health`'s `eventsSent` tăng lên 1; query lại
   `wafSecurityEventsList` trên gateway thấy đúng record mới
   (`action: "blocked"`, `requestPath: "/e2e-blocked/admin"`, `triggeredById` đúng id rule) — chứng
   minh nhánh `edge → control-plane ingest → alerting-service` (Phase 72's quyết định "option 2",
   ghi lại là "quyết định trong phiên, có thể đảo ngược") cũng hoạt động đúng thật.
7. Dọn dẹp: xoá `FirewallRule`/`Zone`/`SecurityEvent` vừa tạo (không đụng dữ liệu có sẵn của tenant
   `9de4259e-...`). Resync sau đó tự phát hiện zone mồ côi và unpublish khỏi Redis đúng như thiết
   kế (`resync found an orphaned rule-set, removing`) — chứng minh luôn nhánh xoá/hội tụ, không chỉ
   nhánh tạo mới.

`cargo build/clippy(-D warnings)/fmt --check/test --workspace` sạch ở cả 3 workspace
(`data-plane`, `control-plane`, `edge-plane`) sau mọi thay đổi.

### Còn nợ / không làm trong phase này

- Chỉ test pillar WAF/Firewall (path-block). Chưa test sống pillar DDoS (`CompiledDdos`/rate
  limit) hay `IpAccessList` qua cùng pipeline — logic compile của cả 2 đã có unit test (28 test cũ
  của `compile.rs`), chỉ chưa có 1 lượt chạy sống riêng như FirewallRule vừa làm.
- `zone_domain_guard`/`zone_config_guard` (middleware REST tự điền `domainId`/`hasConfig`) không
  chạy khi tạo record qua gRPC/GraphQL-gateway (middleware chỉ gắn ở tầng axum REST của
  `zones-service`, không phải ở `CrudService`) — test này né bằng cách set field thẳng tay. Đây là
  gap thật, có thể ảnh hưởng Customer Portal thật (portal cũng gọi qua GraphQL gateway giống hệt
  test này) — **chưa xác nhận Customer Portal có đi qua đường nào khác để middleware này vẫn chạy
  hay không, cần điều tra riêng, không giả định**.
- `scanning-service`'s gRPC upstream vẫn "degraded" trên gateway trong lần chạy này (không ảnh
  hưởng test — không đụng tới entity nào của `scanning-service`) — chưa điều tra tại sao gRPC port
  3011 không nghe, có thể chỉ là timing lúc gateway boot trước khi scanning-service sẵn sàng.
- Chưa dựng lại thành 1 test tự động (`#[ignore]` e2e trong `control-plane`/`edge-plane`) — lần
  chạy này hoàn toàn thủ công (curl + đọc log). Biến thành test tự động là việc làm thêm hợp lý
  tiếp theo nếu muốn pipeline này được verify lại mỗi lần có thay đổi, không chỉ 1 lần.
