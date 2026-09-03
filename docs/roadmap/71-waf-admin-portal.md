## Phase 71: WAF Customer Portal thật + 4 nhóm endpoint "phải tự code" (2026-09-03)

Trigger: chủ dự án hỏi "ngoài ra còn phần nào rất to để implement không" → đếm code thật cho thấy
`metap-demo-waf` mới là **portal CRUD generic + docs rất kỹ**: `web/src` đúng 5 file
(`App`/`main`/`index.css` + `LoginPage` + `EntitiesPage`), nav là 1 mục cho mỗi entity `waf.*`,
mọi màn hình là `GeneratedList`/`GeneratedForm`. Trong khi `docs/07-portal-features.md` spec
sitemap 10 module, và `docs/13-screen-api-map.md` tự liệt ~1/3 tính năng **không phải CRUD**. Yêu
cầu: implement luôn portal admin, không hỏi quyết định, không verify/build/test.

## Backend — đúng những chỗ generic CRUD không sinh được

`docs/13-screen-api-map.md` đã liệt sẵn danh sách này; phase này làm 4/6 nhóm đầu.

**`zones-service/src/routes.rs`**
- `POST /api/waf.zones/{id}/verify-dns` — xác minh sở hữu domain qua TXT `_waf-verify.<hostname>`
  (so với `verificationToken` của chính zone) + kiểm tra `dnsRoutingStatus` (CNAME có trỏ về edge
  chưa, **không** gate activation). Dùng DoH qua HTTP (`DOH_RESOLVER_URL`, mặc định `dns.google`)
  thay vì thư viện DNS: service đã có HTTP client, không cần dependency mới hay egress UDP, đổi
  resolver là đổi env.
- `POST /api/waf.zones/{id}/test-origin` — probe origin 1 lần, `redirect::Policy::none()`.
- `POST /api/waf.zones/{id}/sync-config-state` — **tính lại `hasConfig`** từ số
  `DdosPolicy`/`FirewallRule` thật của zone. Phát hiện lúc code: `hasConfig` được `zone_entity.rs`
  mô tả là "app layer tự flip" nhưng **chưa ai implement** (grep toàn repo: chỉ có đúng 1 dòng
  comment) — nghĩa là guard `activate` (cần `hasConfig == true` + `verificationStatus ==
  "verified"`) **chưa bao giờ pass được**, không zone nào rời được `pending`. Endpoint idempotent,
  bỏ qua write khi giá trị đã đúng để không bump `version` mỗi lần gọi.
- `zone_delete_guard` (middleware) — chặn xoá zone khi service **khác** còn record trỏ tới, đúng
  gap đã bàn trước khi code: `find_referencing_record` quét `MetadataRegistry` của **process đang
  chạy**, mà từ lúc tách pillar `zones-service` chỉ đăng ký 3 entity của nó, nên xoá zone âm thầm
  bỏ rơi `waf.scan_jobs`/`waf.incidents`/`waf.security_events`. **Là middleware chứ không phải
  route override**: `metap-http` đăng ký `GET`/`PATCH`/`DELETE` chung trên `/api/{entity}/{id}`,
  nên đăng ký tĩnh `/api/waf.zones/{id}` chỉ với `DELETE` sẽ thắng path match cho cả 3 method và
  biến `GET`/`PATCH` zone thành 405 (axum khớp path trước, method sau, không fallthrough).
  Forward luôn header `authorization` **và** `cookie` của người gọi sang service anh em — 3 service
  chung keypair (bài học Phase 61 T6) nên token thật của caller dùng được, không cần service
  account, và lệnh kiểm tra chạy đúng quyền của caller. Không kiểm tra được (service kia sập) thì
  **chặn** chứ không cho qua.
- `GET /internal/health/deep` — cho operator biết vì sao 1 lệnh xoá vừa trả 503.

**`alerting-service/src/routes.rs`**
- `POST /internal/incidents/correlate` — luật tĩnh v1 đúng như `docs/02-domain-model.md` chốt: cùng
  `zoneId` + cùng `sourceIp` trong cửa sổ 15 phút, >= 5 event. Chống trùng bằng "đã có incident
  `open` của zone này mà title chứa IP đó chưa" (Incident không có field `sourceIp` — nó correlate
  event chứ không copy) → chạy lại nhiều lần vẫn idempotent, quan trọng vì đây là job có lịch.
- `POST /internal/alerts/evaluate` — đếm **theo từng zone, không cộng dồn** (đúng ghi chú
  `docs/08-module-detail-specs.md` module 8), ghi `AlertNotification` cho cả lần gửi hỏng.
- `POST /api/waf.alert_policies/{id}/test` — chạy đúng đường delivery thật.
- `channels` là JSON tự do: `{"webhook": url}` POST thật; `{"email": ...}` **chỉ log** kèm nói rõ
  chưa có mail transport — thành thật hơn là `deliveryStatus: "sent"` không ai kiểm chứng được.

**`scanning-service/src/routes.rs`**
- `POST /api/waf.scan_jobs/{id}/run` — bàn giao cho `SCANNER_URL` nếu có; **không có scanner thì
  job vẫn queue và nói thẳng là chưa ai nhặt**. Portal không chạy DAST (đúng đính chính
  2026-08-30 của `docs/13-screen-api-map.md`).
- `POST /internal/scan-jobs/{id}/findings` — callback cho scanner: tạo `ScanFinding` **trước** rồi
  mới complete job (ngược lại sẽ có khoảnh khắc job `completed` mà findings rỗng). Đi qua
  transition (`start`/`complete`/`fail`) chứ không ghi thẳng `status`.

## Frontend — IA theo zone, không phải theo bảng

`web/src` từ 5 file lên ~17: `api/waf.ts` (1 chỗ duy nhất biết tên entity + đường dẫn endpoint),
`components/primitives.tsx` (`StatTile`/`StatusBadge`/`SectionCard`/`TimeSeries` SVG thuần, không
thêm thư viện chart), và 12 màn hình:

| Màn hình | Module (docs `07`) |
|---|---|
| `DashboardPage` | 6 — tổng quan, 5 stat tile + 4 chart, **toàn bộ từ aggregate API** |
| `OnboardingPage` | 1 — wizard 4 bước: tạo zone → verify DNS (+test origin) → protection mặc định → activate |
| `ZonesPage` / `ZoneDetailPage` (5 tab) | 2/3/4 — Overview / DDoS / Firewall rules / Scans / Events |
| `IncidentsPage` / `IncidentDetailPage` | 7 |
| `FindingsPage` | 5 |
| `AlertingPage` | 8 |
| `AnalyticsPage` | 6 đầy đủ (chọn window/zone, top source IP) |
| `SettingsPage` | 10 — editor mỏng trên `/admin/config` của `metap-config` |

Quyết định đáng ghi:
- **Màn hình CRUD generic cũ không bị xoá**, dồn vào 1 mục nav `Raw records` (admin) — vẫn là cách
  nhanh nhất để soi field mà màn hình sản phẩm chưa hiện.
- Firewall rule **đổi thứ tự bằng cách hoán vị `priority` của 2 rule**, không đánh số lại cả list:
  2 write thay vì N, và không có khoảnh khắc 2 rule cùng priority.
- `matchCondition` để **JSON thô** kèm ghi chú: grammar (dùng lại `PolicyCondition` hay tự định
  nghĩa cho `uri.path`/`header.x`) vẫn là câu hỏi mở trong `docs/02-domain-model.md` — builder trực
  quan dựng trên grammar sai thì vứt.
- `verificationToken` sinh ở **client** (`crypto.randomUUID()`) vì `metap` không có hook
  "before create" per-entity, và thêm hook đó vào core cho 1 field của 1 app đúng là loại
  business-entity knowledge mà `metap-*` không được mang. Chỗ đúng về lâu dài là `computed` field.

## Xác minh

**Không có.** Chủ dự án yêu cầu rõ trong phiên này: "không verify code, không cần build, không cần
test, chỉ code" — sẽ tự kéo nhánh về kiểm tra. Nên toàn bộ Phase 70 + 71 đang ở trạng thái **chưa
compile lần nào**: chưa `cargo build`, chưa `tsc`, chưa `oxlint`, chưa chạy thật. Danh sách chỗ dễ
sai nhất khi verify: (1) `plan_aggregate`'s SQL + `to_jsonb`, (2) `zone_delete_guard` có thực sự
bắt đúng path và không chặn nhầm request khác, (3) prop của component `@metap/ui` dùng trong 12
màn hình mới (`Select`/`Toggle`/`Stepper`/`Dialog` — viết theo type export, chưa render lần nào),
(4) `Cargo.toml` mới thêm `reqwest`/`chrono`/`uuid` cho cả 3 service.

## Còn lại (không thuộc phase này)

`control-plane` và `edge-plane` vẫn **0 dòng code** — phần khiến sản phẩm là một WAF thật (chặn
được request) chưa tồn tại. Đường tới hạn nếu muốn demo end-to-end: `control-plane` (subscribe
outbox → compile rule-set → Redis) rồi `edge-plane` tối thiểu 1 loại rule. Cộng: Admin Portal
backend (`Plan`/`Subscription`), Traefik/reverse-proxy thật cho production (routing hiện chỉ chạy
trong Vite dev-server), và telemetry edge→up vẫn là quyết định kiến trúc chưa chốt
(`docs/04-architecture-boundary.md`).
