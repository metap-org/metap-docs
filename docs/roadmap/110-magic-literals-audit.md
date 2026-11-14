# 110. Rà soát magic string và magic number trong metap và waap

**Trạng thái:** đợt 1 xong 2026-11-14 (metap#46, waap#36). Còn một số nhóm chưa làm, liệt kê bên dưới.

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

## Chưa làm

- Trạng thái và tên workflow action của waap (`"queued"`, `"running"`, `"open"`, `"run"/"start"/"fail"` của scan job...) vẫn là literal; nên khai báo cạnh entity có workflow đó.
- Tên field entity ngoài `zoneId` (`"hostname"`, `"sourceIp"`, `"enabled"`, `"priority"`, ...) và tên cột JSON trong payload còn lặp nhiều; gom theo entity khi sửa tới.
- Khóa JSON trong gateway/gRPC (`"entity"`, `"data"`, `"status"`, ~200 chỗ) phần lớn là tên field của wire format, chưa rõ có đáng gom không.
- Số cố định chưa đặt tên: `Limit: 200` (cron scheduler), `Limit: 30` (gateway), vài timeout lẻ. Phần lớn timeout lớn đã là hằng số có tên.
- Chưa bật linter bắt tái phát (ví dụ `goconst`); `.golangci.yml` hiện chỉ có depguard, và bật thêm linter là việc riêng.
