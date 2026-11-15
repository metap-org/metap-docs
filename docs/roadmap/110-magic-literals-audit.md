# 110. Rà soát magic string và magic number trong metap và waap

**Trạng thái:** đợt 1 xong 2026-11-14 (metap#46, waap#36); đợt 2 xong 2026-11-15 (xem mục "Đợt 2"). Còn một số nhóm chưa làm, liệt kê ở cuối.

## Đã đếm (code không phải test)

- metap: mã lỗi viết tay lặp (`validation_failed` 26 lần, `record_not_found` 21, `unauthorized` 9, `forbidden` 9, ...), 47 chỗ `NewServiceError(404, ...)` dùng số thay vì `http.Status*`, `"admin"` 6 chỗ, tên policy action (`"read"`, `"create"`, ...) 60 chỗ, leeway JWT `20 * time.Second` lặp ở 3 nơi.
- waap: tên entity (`"waf.zones"` 43 lần, cả 10 entity ~170 lần), `"zoneId"` 50 lần, tên action 15 lần.

## Đã sửa (hành vi không đổi)

- `pkg/kit/errcode`: mọi mã lỗi một chỗ (stdlib-only nên dùng được cả ngoài metap). Thay 25 file.
- `ServiceError` dùng `http.Status*` ở mọi constructor.
- `permission.RoleAdmin`, `permission.PolicyActionRead|Create|Update|Delete|Transition`, `jwks.DefaultLeeway`.
- waap `model/`: tên entity, `FieldZoneID`/`FieldDomainID`, tên action; thay ~150 literal ở 25 file. `analytics.SecurityEvents` trỏ về `model.SecurityEvents`.
- Test unit và integration của cả hai repo vẫn xanh (trừ test cần Redis/Vault không có trong môi trường này).

## Một bẫy khi sửa hàng loạt

Thay `"zoneId"` bằng hằng số cũng đổi luôn struct tag `json:"zoneId"` thành `json:model.FieldZoneID` mà vẫn compile được (tag là chuỗi thô). Bắt được bằng grep sau khi thay, đã khôi phục. Lần sau dùng `go vet` (structtag) ngay sau khi thay hàng loạt.

## Đợt 2 (2026-11-15)

Chủ dự án chỉ ra `Format("2006-01-02T15:04:05.000Z07:00")` vẫn còn viết tay và yêu cầu một chỗ chuẩn cho các literal dùng chung.

- **`pkg/kit/standard`** (metap, chỉ dùng thư viện chuẩn nên cả `e-edge`/`a-api` dùng được): layout thời gian (`TimestampMillis`, `TimestampNanos`, `TimestampSeconds`, `NowRFC3339()`, `FormatMillis`), tên header và media type (`HeaderContentType`, `HeaderAuthorization`, `HeaderTraceparent`, `HeaderRequestID`, `HeaderCSRFToken`, `MediaTypeJSON`, `MediaTypeHTML`, `BearerPrefix`), tên schema Postgres (`SchemaMetadata`, `SchemaControl`, `SchemaPublic`) và khoá log slog (`LogError`, `LogTenantID`, `LogRunID`, `LogJobID`, ...). Trước đó cùng một layout thời gian có ba dạng khác nhau (mili giây, nano giây, RFC 3339) rải ở nhiều file.
- **`pkg/kit/errcode`** thêm mã còn thiếu (`EntityNotFound`, `ActionNotFound`, `NotTokenIssuer`, `InvalidTransition`, `InvalidRequestBody`, ...) và nhóm mã OAuth2 theo RFC 6749 (`OAuthInvalidClient`, `OAuthInvalidGrant`, ...). Mã riêng của waap nằm trong `model/` (`ErrScanAlreadyInProgress`, `ErrScannerUnavailable`, `ErrReferenceCheckUnavailable`).
- **waap `model/`**: `fields.go` (tên mọi field entity), `workflow.go` (trạng thái và action chuyển trạng thái của zone, scan job, finding, incident; trạng thái xác minh domain và định tuyến DNS). Định nghĩa entity, route, control-plane (`c-config-distributor` đọc zone/rule/policy theo tên field của management) và gateway dùng chung các hằng số này, nên gõ sai tên field trở thành lỗi biên dịch thay vì mất dữ liệu âm thầm.
- Thay hàng loạt chỉ ở những ngữ cảnh rõ nghĩa (truy cập record, danh sách field, khai báo entity, khoá slog, tên header). Hai lần thay nhầm bị bắt khi rà diff: cột `tenant_id`/`job_id` trong SQL bị đổi thành khoá log, và một chuỗi `Reason: "forbidden"` không phải mã lỗi. Đã sửa lại.
- Kiểm tra: `go vet` sạch, test unit và integration của cả hai repo xanh (trừ test cần Redis/Vault không có trong môi trường này).

## Chưa làm

- Khoá JSON của wire format (`"entity"`, `"status"`, `"action"`, `"createdAt"`, ...) trong gRPC/GraphQL; phần lớn là tên field của hợp đồng, chưa rõ có đáng gom không.
- Trạng thái của cron/workflow-run trong metap (`"waiting"`, `"running"`, `"failed"`, ~16 chỗ).
- Số cố định: giới hạn trang (`Limit: 100`, `50`, `200`), một số timeout lẻ; tên biến môi trường (~100 chuỗi `env.String("...")`), mỗi cái chỉ xuất hiện ở nơi đọc nó.
- Giá trị enum của field entity khi chỉ khai báo ở một chỗ (ví dụ `logged/challenged/blocked` của security event).
- Chưa bật linter bắt tái phát (ví dụ `goconst`); `.golangci.yml` hiện chỉ có depguard.
