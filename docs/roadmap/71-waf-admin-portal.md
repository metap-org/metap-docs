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

**Cập nhật 2026-09-04**: build/test thật xong (phiên riêng, chủ dự án yêu cầu "bắt đầu viết test,
build và verify các thay đổi"). `cargo build/clippy -D warnings/test --workspace` cho `data-plane`
sạch (bắt đúng 1 lỗi clippy thật trong `scanning-service` — `useless_conversion`, sửa thành
`Value::Object(data.data)`). `zone_delete_guard` build/clippy sạch, generic bounds của
`run_resilient_consumer` không sao — 2 chỗ tôi nghi nhất hoá ra đúng ngay từ đầu.

Frontend: **48 lỗi `tsc -b` thật**, tất cả đã sửa — sạch cả `tsc -b`/`oxlint`/`prettier --check`/
`vite build` (1211 module, bundle 771KB) sau đó:
- 36/48 lỗi: mọi `toast({ title, variant: "success" })` sai hoàn toàn signature thật của
  `@metap/ui`'s `toast()` — hàm thật là `toast(message: string, { variant: "default" | "destructive" })`,
  không có variant `"success"`. Sửa cả 36 chỗ, map `"success"` → `"default"`.
- `OnboardingPage`'s `Stepper` dùng prop `state` không tồn tại (`StepperItem` thật dùng
  `variant`), và cấu trúc đúng là mỗi step 1 `StepperGroup` nối bằng `StepperConnector`, không
  phải gộp cả 4 step vào 1 group — viết lại đúng compound-component shape.
- `AnalyticsPage` index mảng cố định (`WINDOWS[1]`) làm fallback — `noUncheckedIndexedAccess` báo
  đúng vì kiểu vẫn là `T | undefined`; tách thành hằng số `DEFAULT_WINDOW` riêng.

Thêm test thuần (không cần DB/network) cho logic dùng ngay được: 19 test cho `compile.rs`
(`parse_match`/`compile_rule`/`compile_ddos`/`compile_zone`/`publishable`) ở `control-plane`, và
35 test cho `evaluate.rs`/`ratelimit.rs`/`clearance.rs` ở `edge-plane` (chi tiết ở Phase 72's mục
xác minh). Tổng: `cargo test --workspace` cả 4 workspace Rust trong `metap-org` đều xanh.

**Vẫn chưa verify**: chạy thật qua HTTP với Postgres sống (không có docker daemon trong môi trường
build), và test e2e `http_server.rs` của cả 3 service (đã có sẵn, `ignored`, cần
`DATABASE_URL`/Postgres thật).

## Còn lại (không thuộc phase này)

Xem Phase 72 (`72-control-edge-planes.md`) — `control-plane`/`edge-plane` giờ đã có code (cùng
2026-09-04, phiên tiếp theo cùng ngày), phần "Còn lại" ở đó thay cho ghi chú cũ ở đây.
