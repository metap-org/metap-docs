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
| 4 | LOW | `DecidePlanChange` áp thay đổi gói rồi mới đánh dấu request đã quyết định: hai quyết định đồng thời → gói đã đổi nhưng request ghi "rejected" | **Đã fix** 2026-11-23 — claim request (Update theo version đã đọc) trước khi áp gói; áp lỗi thì mở lại request. Còn lại: chưa có ràng buộc unique một phần cho "một request chờ" mỗi tenant |
| 5 | LOW | Quota gói: fail-open khi không resolve được gói, kiểm-rồi-tạo không nguyên tử, và chỉ kiểm lúc `Create` (đổi `zoneId` của rule bằng `Update` vượt hạn mức từng zone) | **Một phần** 2026-11-23 — đã chặn đường `Update` đổi `zoneId` (test mới); fail-open giữ nguyên có chủ đích (test `TestPlanQuotasFollowWhatTheBillingDomainSays` ghim: billing lỗi không chặn onboarding), nâng log lên error; kiểm-rồi-tạo vẫn không nguyên tử |
| 6 | LOW | `SubmitFindings` không nguyên tử/idempotent: tạo finding trước rồi mới chuyển trạng thái job; job đã `completed` → 500 sau khi finding đã ghi, retry sinh trùng; số finding mỗi lần gọi chỉ bị giới hạn bởi 1 MiB body | **Đã fix** 2026-11-23 — callback từ chối job không ở `queued`/`running` (409 `scan_not_running`) trước khi ghi gì, và giới hạn 500 finding/lần (413); lỗi giữa vòng vẫn có thể để lại một phần |
| 7 | INFO | `accountGuard` giữ nguyên `accountId` do client gửi khi không đọc được zone/zone không có account — cần test với role giới hạn theo account | **Đã fix** 2026-11-23 — `withZoneAccount` luôn bỏ `accountId` do client gửi trước, rồi mới chép từ zone (waap #73); test `TestRuleAccountIsTheZonesNotTheClients` |

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

## Chưa kiểm

`m-bootstrap/personas.go` (grant từng persona — nơi quyết định Viewer/Developer thật sự làm được
gì; mục 2 phụ thuộc vào đây), `versions/` (đúng đắn của snapshot/rollback), `entities/*` (mức
`maxLength` của trường văn bản tự do: `note`, `description`, `channels`), `alerting/correlate.go`
(giới hạn khối lượng sweep), `analytics/` (token chuyển tiếp và lọc theo zone), control plane
(distributor), edge và `a-api`.
