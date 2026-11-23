# Audit 08 — Application module của waap (`m-management/*`, 2026-11-23)

Phạm vi: lớp **APP** của management plane trong `metap-demo-waf` — các domain `zones`
(`guard`, `netcheck`, `presets`, `policy`, `versions`), `billing`, `scanning`, `alerting`,
`analytics` (đọc mã từng dòng các file logic, action và wiring). Đây là phần mà
[bản đồ 21](../../../metap-demo-waf/docs/architecture/08-cross-cutting/21-metap-vs-app-map.md) nói
chỉ được **cắm** vào metap; audit hỏi: chỗ cắm có giữ được các bảo đảm của metap không.
Phương pháp như audit 07: đọc mã, chưa khai thác thật, mỗi mục cần một test tái hiện trước khi sửa.

## Tóm tắt

Phần billing viết cẩn thận (kiểm quyền tường minh, không tin dữ liệu người gọi). Vấn đề tập trung
ở **ranh giới giữa action và metap**: metap bảo đảm "action không làm được nhiều hơn CRUD của
người gọi", nhưng bảo đảm đó chỉ đúng với việc đọc/ghi record — còn tác dụng phụ ra ngoài (gọi
mạng) thì không ai kiểm.

| # | Mức độ | Vấn đề | Trạng thái |
|---|---|---|---|
| 1 | **HIGH** | Webhook của `AlertPolicy` gọi URL do tenant nhập bằng `httpx.NewClient`, không phải `safehttp` → SSRF có phản hồi dạng oracle vào mạng nội bộ | **Đã fix** 2026-11-23 — `safehttp`, bắt buộc https, trả mã ngắn thay vì lỗi dial; test `notify_test.go` |
| 2 | MEDIUM | Route action của metap chỉ gọi `requireAuth`; action không đụng record (`resolveHostname`, `probeOrigin`) mở cho **mọi** thành viên đã đăng nhập, kể cả Viewer. waap không dùng `DeclareResource/CheckResource` ở đâu | **Đã fix** 2026-11-23 — chọn sửa ở metap: `EntityAction.Requires` (route kiểm quyền entity trước handler), waap khai báo `Requires: create` cho `resolveHostname`/`probeOrigin`; test e2e `TestDeclaredActionRequiresEntityPermission` |
| 3 | LOW | `testAlertPolicy`/`evaluateAlerts` gọi webhook **trước** khi ghi notification: người chỉ có quyền đọc policy kích được webhook dù việc ghi bị 403 | **Đã fix** 2026-11-23 — `ensureCanRecord` kiểm quyền tạo notification trước khi gọi webhook (test e2e chưa chạy: không có Postgres trong phiên) |
| 4 | LOW | `DecidePlanChange` áp thay đổi gói rồi mới đánh dấu request đã quyết định: hai quyết định đồng thời → gói đã đổi nhưng request ghi "rejected" | **Đã fix** 2026-11-23 — claim request (Update theo version đã đọc) trước khi áp gói; áp lỗi thì mở lại request. Còn lại đã xử lý 2026-11-23: trường `openRequestKey` (unique) giữ tenant của khách khi request đang chờ và rỗng khi đã quyết định, nên hai yêu cầu đồng thời chỉ một cái vào được (409); test đồng thời `TestConcurrentPlanRequestsAndDecisionsLetOneThrough` |
| 5 | LOW | Quota gói: fail-open khi không resolve được gói, kiểm-rồi-tạo không nguyên tử, và chỉ kiểm lúc `Create` (đổi `zoneId` của rule bằng `Update` vượt hạn mức từng zone) | **Một phần** 2026-11-23 — đã chặn đường `Update` đổi `zoneId` (test mới); fail-open giữ nguyên có chủ đích (test `TestPlanQuotasFollowWhatTheBillingDomainSays` ghim: billing lỗi không chặn onboarding), nâng log lên error; kiểm-rồi-tạo nay nguyên tử trong tiến trình: khoá theo phạm vi quota (tenant+entity, thêm zone cho rule) giữ qua đếm và tạo, test `TestConcurrentCreatesDoNotOvershootThePlan`. Giới hạn: khoá trong tiến trình, đúng với management một process; nhiều replica cần khoá advisory trong transaction tạo |
| 6 | LOW | `SubmitFindings` không nguyên tử/idempotent: tạo finding trước rồi mới chuyển trạng thái job; job đã `completed` → 500 sau khi finding đã ghi, retry sinh trùng; số finding mỗi lần gọi chỉ bị giới hạn bởi 1 MiB body | **Đã fix** 2026-11-23 — callback từ chối job không ở `queued`/`running` (409 `scan_not_running`) trước khi ghi gì, và giới hạn 500 finding/lần (413); lỗi giữa vòng vẫn có thể để lại một phần |
| 7 | INFO | `accountGuard` giữ nguyên `accountId` do client gửi khi không đọc được zone/zone không có account — cần test với role giới hạn theo account | **Đã fix** 2026-11-23 — `withZoneAccount` luôn bỏ `accountId` do client gửi trước, rồi mới chép từ zone (waap #73); test `TestRuleAccountIsTheZonesNotTheClients` |
| 8 | MEDIUM | `waf.security_events` (read model over Analytics) **không kiểm quyền nào**: route chỉ yêu cầu đăng nhập, Analytics lọc theo tenant, nên Viewer/Developer (không được cấp entity này) liệt kê và tổng hợp được mọi sự kiện của tenant (IP nguồn, đường dẫn). Kèm theo: entity không nằm trong registry của binary bootstrap, nên `waf.*` không bao giờ cấp được cho nó | Đã sửa 2026-11-23: `analytics.Backend` hỏi `CanReadEntity` trước mọi đọc (không nối dây thì từ chối hết); bootstrap đăng ký entity và cấp cho Viewer (persona tài liệu ghi Viewer đọc analytics); test `TestSecurityEventsNeedTheReadPermission` |
| 9 | HIGH | Treo pool kết nối ở lõi metap: `CrudService.List` giữ một giao dịch (một kết nối) rồi `hydrateRelatedDisplay` gọi `LoadSnapshot` cần kết nối thứ hai; số List đồng thời ≥ kích thước pool thì mọi kết nối bị giữ và đều chờ nhau tới khi request hết hạn. Lộ ra khi test ghi phiên bản chính sách chạy 6 thay đổi luật đồng thời | Đã sửa 2026-11-23 (metap): đọc quyền/snapshot của entity liên quan **trước** khi mở giao dịch; test `TestConcurrentListsWithRelatedDisplayDoNotExhaustTheConnectionPool` (12 List, pool 3) hết hạn trên mã cũ |
| 10 | LOW | Phiên bản chính sách: hai thay đổi cùng lúc đọc cùng "mới nhất" và ghi cùng số → lịch sử có hai "version 1" | Đã sửa 2026-11-23: `(zoneId, number)` unique, `RecordVersion` thử lại với số kế tiếp; test đồng thời đỏ trên mã cũ. Lưu ý triển khai: DB dev không được có sẵn cặp trùng, nếu không index không dựng được lúc boot |
| 11 | LOW | `matchCondition` không giới hạn độ sâu, số phép so sánh, kích thước danh sách hay độ dài regex ngoài trần 1 MiB của body; edge chia sẻ CPU giữa các tenant và chạy mọi luật của zone cho mọi request | Đã sửa 2026-11-23: tối đa 8 tầng, 50 phép so sánh, 200 giá trị, 1 KiB/giá trị, 512 byte/regex, 32 KiB tổng; zones từ chối lúc lưu, distributor bỏ luật vượt (cùng một bộ số, hai bản độc lập) |
| 12 | LOW | `correlate`: chống trùng bằng `strings.Contains(title, ip)` nên incident đang mở của `11.2.3.45` làm mất incident mới của `1.2.3.4`; `threshold < 1` mở incident cho mọi địa chỉ chỉ có một sự kiện; cửa sổ không giới hạn | Đã sửa 2026-11-23: so khớp ` from <ip> in `, cửa sổ 1 đến 1440 phút, ngưỡng tối thiểu 1 |
| 13 | INFO | `correlate` chỉ đọc 200 sự kiện mới nhất mỗi lượt: trong một đợt tấn công lớn số đếm bị cắt ở 200 (thang mức độ nghiêm trọng chỉ đạt "critical" đúng ở 200) và địa chỉ ồn nhất chiếm hết chỗ của các địa chỉ khác | **Mở.** Cách đúng là nhóm theo `(zoneId, sourceIp)` ngay ở Analytics qua API aggregate (đã có `Client.Aggregate`) thay vì đếm trong bộ nhớ |
| 14 | INFO | Persona: Developer có `update` trên `ScanFindings` nhưng quyền trường mặc định cho phép sửa mọi trường (kể cả `severity`), không chỉ trạng thái xử lý; `waf.*` đọc của Analyst tự mở rộng cho entity mới thêm về sau; snapshot phiên bản chính sách lấy "như người gọi thấy", nên chính sách cấp bản ghi làm snapshot thiếu và rollback sau đó xoá phần không thấy | **Mở**, thiết kế: cần grant theo trường trong `bootstrap.Grant` (chưa có) và snapshot bằng ngữ cảnh admin |

## Chi tiết

### 1. Webhook cảnh báo — SSRF (HIGH)

`alerting/notify.go: deliver` lấy `channels.webhook` (JSON tự do trên `AlertPolicy`, entity không
validate scheme hay đích — `alertnotification.go` chỉ khai báo enum kênh) rồi
`httpx.NewClient(deliveryTimeout).Post(url, ...)`. Client này theo redirect và dial bất kỳ địa chỉ
nào. Bản đồ 21 mục 4 đã ghi đúng quy tắc: *gọi địa chỉ khách nhập → `safehttp.NewClient`*; chỗ
duy nhất khác trong module làm đúng là `netcheck/outbound.go`.

Kẻ tấn công là tenant có quyền tạo/sửa `AlertPolicy` và gọi action `test` (hoặc chờ `evaluate`):
đặt `webhook` = `http://10.x.x.x:6379/`, `http://169.254.169.254/...`, một service nội bộ. Phản hồi
trả về trong `detail` (`webhook error: dial tcp 10.0.0.5:6379: connection refused` so với
`webhook 200`/`webhook 404`) → dò host/port nội bộ; thân POST là JSON do server dựng, đủ để gọi
các endpoint nội bộ không cần xác thực. Không đọc được thân phản hồi (khác audit 04 A#1) nhưng cùng
loại lỗi mà dự án đã xếp HIGH và đã có công cụ để sửa.

Sửa: `safehttp.NewClient` (tuỳ chọn allowlist host cho môi trường dev), validate URL `https` khi
lưu policy (guard cho `AlertPolicies`), không đưa `err.Error()` thô vào `detail` mà trả một mã
(`unreachable`/`refused_target`/`status_NNN`), bỏ theo redirect. Thêm test "webhook tới loopback
bị từ chối" tương tự test của `cronscheduler`.

### 2. Action không đụng record không có kiểm quyền nào (MEDIUM)

`pkg/httpserver/routes_actions.go: invokeDeclaredAction` chỉ `requireAuth`, kiểm action có khai
báo, rồi gọi handler. Phân quyền chỉ xảy ra khi handler tự đọc/ghi record bằng `rc`. Action
record-scope thì được quyền của `Get` bảo vệ phần nào; action **entity-scope** không đọc gì thì
không có gì chặn:

- `waf.zones.resolveHostname`: tra DNS (A/AAAA/CNAME) tên tuỳ ý.
- `waf.zones.probeOrigin`: một GET tới địa chỉ/cổng/`Host` tuỳ ý (chỉ địa chỉ công cộng — guard
  của `outbound.go` đúng) từ IP của nền tảng.

Bất kỳ ai đăng nhập — Viewer, service account — dùng được, chỉ chịu rate limit chung (5 req/s/IP,
burst 300): biến nền tảng thành máy quét cổng/phát request từ IP của mình, và các tenant dùng
chung danh tiếng IP đó. metap đã có đúng công cụ (roadmap 115 M6: `permission.DeclareResource`,
`PermissionService.CheckResource`, grant theo persona trong `m-bootstrap`) nhưng không file nào của
waap gọi chúng.

Sửa: khai báo resource (`waf.zones.probe`, ...) và `CheckResource` ở đầu handler hai action này
(và bất kỳ action nào về sau không đụng record); đồng thời sửa câu trong `CLAUDE.md`/doc của metap
"an action can never do more than the caller could with plain CRUD" thành "...đối với record;
tác dụng phụ ngoài record phải tự kiểm quyền". Hướng thay thế ở metap: để `EntityAction` khai
báo hành động policy bắt buộc (`RequiresAction`) và route kiểm trước khi gọi handler — an toàn
hơn vì không dựa vào việc từng handler nhớ gọi.

### 3. Tác dụng phụ trước kiểm quyền (LOW)

`testAlertPolicy` đọc policy (quyền `read`), gọi `deliver` (webhook thật), rồi mới
`recordNotification` (cần quyền `create` trên notification). Người chỉ có quyền đọc gọi `test`
nhận 403 ở bước cuối nhưng webhook đã bắn. `evaluateAlerts` cùng thứ tự. Sửa: kiểm quyền tạo
notification (hoặc ghi dòng `pending` trước) rồi mới gọi ra ngoài; cùng gốc với mục 2.

### 4. Quyết định đổi gói không nguyên tử (LOW)

`DecidePlanChange` (`billing/changes.go`): đọc request, kiểm `pending`, **`applyPlanChange`**, rồi
mới `Update` request với `req.Record.Version`. Hai admin của operator cùng thao tác: A từ chối
(Update xong, version +1), B duyệt: apply đổi subscription rồi Update bị `version_conflict` →
subscription đã đổi nhưng request ghi `rejected`. Sửa: "chiếm" request trước (Update status sang
`deciding`/giá trị cuối bằng version đã đọc), chỉ áp khi chiếm được; nếu áp lỗi thì hoàn lại. Cùng
kiểu: `RequestPlanChange` kiểm "đã có request chờ" rồi tạo — hai yêu cầu đồng thời đều qua (cần
ràng buộc unique một phần `(customerTenantId) WHERE status='pending'`).

### 5. Quota gói (LOW)

`guard/quota.go`: (a) `Resolve` lỗi → ghi log và **cho qua** (fail-open): lỗi tạm của billing =
vượt hạn mức; chọn có chủ đích hay không cần ghi lại, tối thiểu là metric/alert. (b) đếm rồi
`inner.Create` không khoá → N request đồng thời vượt hạn mức N−1 lần. (c) chỉ kiểm ở `Create`:
`Update` đổi `zoneId` của một firewall rule sang zone khác không đi qua quota từng zone.

### 6. Callback scanner (LOW)

`scanning/findings.go`: vòng `Create` từng finding rồi `Transition`. Job đã `completed` → transition
lỗi → `WriteInternalError` (500) sau khi finding đã ghi; scanner retry → trùng finding. Lỗi giữa
vòng để lại một phần. Không có `Idempotency-Key`/dedupe theo (job, endpoint, category). Callback
dùng token của một user thường nên scanner cần quyền tạo finding + transition job như người dùng;
nên có persona `scanner` tối thiểu (bootstrap). Route `/internal/*` không được nginx của portal
proxy (đã kiểm `m-web/nginx.conf`: chỉ các prefix công khai) — đúng.

### 7. `accountGuard` và `accountId` do client gửi (INFO)

`withZoneAccount` trả `data` nguyên vẹn khi `Get` zone lỗi hoặc zone không có `accountId` — nếu
body client có sẵn `accountId`, nó đi tiếp tới tầng metap. Phụ thuộc việc `CanCreateInScope` của
metap có kiểm `accountId` với role giới hạn theo account hay không. Chưa xác nhận: cần một test
tích hợp "role scoped theo account A tạo rule với accountId=B".

## Đã kiểm tra, không có vấn đề

- **billing**: mọi thao tác nhạy cảm kiểm `rc.TenantID == Operator && rc.IsAdmin()` (`IsAdmin` chỉ
  tính admin toàn tenant, role giới hạn theo scope không lọt); tenant của khách luôn lấy từ token
  (`rc.TenantID`), tên khách đọc từ registry thay vì nhận từ người gọi; đọc/ghi bằng `operatorRC`
  tổng hợp chỉ trong phạm vi plan/subscription/request; `listCustomerTenants` SQL tham số hoá,
  `LIMIT 500`, chỉ trả cho admin của operator.
- **netcheck**: kiểm chứng sở hữu domain chỉ cho `host` là apex hoặc tên con (`HasSuffix("."+apex)`),
  đọc tối đa 1 KiB, client ra ngoài qua guard địa chỉ công cộng (`outbound.go`), không theo
  redirect.
- **scanning**: `Run` và callback đều đi qua `state.Crud` với `rc` của người gọi (quyền
  create/transition vẫn áp), chuyển trạng thái qua workflow chứ không ghi `status` trực tiếp.
- **rollback**: dùng `state.Crud` (không qua guard) có chủ đích và ghi một version duy nhất;
  `applyPreset` đi qua backend có guard như thao tác của người dùng.
- **export/usage/validate**: đọc bằng `rc` người gọi nên bị lọc theo quyền; không có SQL trực tiếp
  trong module trừ hai câu `control.tenants` của billing (đã tham số hoá, và đã ghi ở mục 7 của
  bản đồ 21 là chỗ chưa sạch).
- **Route ngoài entity**: portal chỉ proxy các prefix công khai nên `/internal/*` không lộ.

## Đợt 2 (2026-11-23): các mục trước đây chưa kiểm

- **`m-bootstrap/personas.go`**: xem finding 8 và 14. Analyst không xoá/sửa được Zone và domain (chỉ
  có đọc cộng ghi trên luật, policy DDoS, danh sách IP); các action đòi `Requires: create` trên zone
  nên cũng ngoài tầm Analyst.
- **`versions/`**: xem finding 10 và 14; rollback không nguyên tử (đã ghi trong mã: chạy lại rollback tới cùng
  version thì hoàn tất).
- **`entities/*` — độ dài trường tự do**: chỉ `matchCondition` ảnh hưởng tới hiệu năng edge; đã giới hạn (finding 11).
  `note`, `description`, `channels` chỉ bị trần body 1 MiB, chấp nhận (không vào luồng nóng).
- **`alerting/correlate.go`**: xem finding 12 và 13.
- **`analytics/`**: xem finding 8; token người gọi được chuyển tiếp nguyên và a-api tự kiểm; a-api dựng
  SQL chỉ từ cột có trong allowlist, mọi giá trị là tham số, `tenant_id` luôn là điều kiện đầu.
- **`a-api`**: truy vấn ClickHouse tham số hoá, bộ lọc allowlist, keyset cursor kiểm tra định dạng, giới hạn trang.
- **Chưa kiểm sâu**: distributor (ngoài `compile` đã đọc ở finding 11), edge. Cần một đợt riêng.
