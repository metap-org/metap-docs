# Khuôn slice cho `internal/admin` (roadmap 122, P2 và P3)

Một slice là một nhóm thao tác quản trị viết đủ các lớp trong một package con `internal/admin/<slice>/`, để đọc từ trên xuống, test từng
lớp riêng, và sửa một chỗ khi cần tuỳ biến. Bốn slice đầu tiên làm mẫu: `preferences`, `sessions`, `apikeys`, `mfa`; P3 đưa toàn bộ phần còn lại vào khuôn (`trail`, `support`, `settings`, `tenants`, `roles`, `policies`,
`credentials`, `ssoconfig`, `owner`, `users`, `account`), nên package `admin` chỉ còn nối dây (`wiring.go`). Muốn thêm slice mới, sao chép slice nhỏ nhất (`preferences`) rồi đổi tên.

## Các file và việc của từng file

| File | Việc | Được import | Test |
|---|---|---|---|
| `types.go` | DTO vào/ra, chỉ struct. **Tên và tag giữ đúng như OpenAPI đang hiển thị** (huma lấy tên schema từ tên kiểu Go), nên golden `api.golden.json` không đổi | stdlib | không cần |
| `rules.go` | Quyết định thuần: validate, tính hạn, dựng view, chọn cái giữ lại. Không I/O | stdlib, kiểu miền (`identity`) | **unit, theo bảng** |
| `store.go` | Port: interface nhỏ mà service cần, khai báo ở phía dùng. Ghi rõ lỗi nào mỗi hàm trả | `context`, `uuid`, kiểu miền | qua contract test |
| `pgstore.go` | Adapter Postgres của port; bọc `identity.*` hoặc SQL | `pgx`, `identity` | **integration** (contract) |
| `service.go` | Chỉ điều phối: ai gọi (`kernel`), quy tắc, store, ghi audit. Handler khoảng 30 dòng | `kernel`, port, `rules` | **unit**, store trong bộ nhớ |
| `memstore_test.go` | Store trong bộ nhớ cho test; đóng vai luôn phần còn lại của hệ thống (người, session tồn tại) | | |
| `contract_test.go` | `runStoreContract`: những gì **mọi** Store phải làm. Chạy trên memstore (unit) và pgstore (integration) | | |
| `pgstore_postgres_test.go` | `//go:build integration`: chạy contract trên Postgres thật | | |

Mức truy cập và route **không** nằm trong slice: chúng ở `operationAccess` (`internal/httpapi/ops_admin.go`), một bảng duy nhất, có test.
`internal/httpapi` chỉ decode, gọi service, bọc kết quả.

## Dùng `kernel` (`internal/admin/kernel`)

`Authenticated(ctx)`, `At(ctx, level)`, `Durable(ctx, store.IsSupportSession)` (cho thao tác tạo quyền truy cập sống lâu hơn khoảnh khắc: người,
khoá, role, đặt lại mật khẩu/MFA, nhà cung cấp đăng nhập; từ chối vendor support session), `Err`, `IdentityErr`, `ParseID`, và `Auditor`
(`StoreAuditor` trong production, spy trong test). `kernel` không import `admin`, nên slice và các service cũ của `admin` dùng chung mà
không vòng.

## Luật (depguard, `.golangci.yml`)

- `rules.go` không import `pgx` và `net/http` (nói mã HTTP qua `kernel.StatusBadRequest`, `StatusConflict`, ...).
- `service.go` không import `pgx`: chỉ chạm DB qua `Store`.
- Tầng ứng dụng không import transport (`httpapi`, huma, router): luật có từ roadmap 118.

## Contract test: vì sao có

Memstore và pgstore là hai bản cài đặt của cùng một port; nếu chỉ test service trên memstore thì hai bản có thể lệch mà không ai biết.
Contract là một bộ ca chạy trên cả hai. Nó đã bắt một giả định sai ngay ở slice đầu: store thật trả `en` (locale mặc định) khi chưa lưu gì,
còn memstore ban đầu trả rỗng.

## Những điều P3 thêm vào khuôn

- **Slice không import nhau.** Cái cần ở slice khác là một interface nhỏ nhận trong `New` (`users.Owners`, `ssoconfig.OwnerOf`), do `admin/wiring.go` thoả mãn. Kiểu dùng chung thì để ở `kernel`
  và `admin` giữ bí danh (`type Empty = kernel.Empty`), để tên schema OpenAPI không đổi.
- **Audit nguyên tử là mặc định.** `service` gọi `deps.Change`, `ChangeNew`, `ChangeInTenant` hoặc `ChangeInTenantNew`; có `Sink` thì thay đổi và entry cùng một transaction, không thì "đổi rồi ghi".
  Store nhận `ctx` và (với DB chung) dùng `kernel.IdentityDB` nên tham gia transaction của unit.
- **Trong một change: không mở DB riêng của tenant, không gọi mạng, không ghi cache trước commit** (§9). Đặt phần đó trước hoặc sau change; ví dụ `owner.CompleteRecovery` gán role admin *trước* change.
- **Test của `service` dùng fake viết tay** có `auditSpy` (implements `audit.Store`) và `kernel.Deps{IsSupport: ...}` để khỏi chạm DB. Fake không có transaction: kiểm "không ghi audit khi lỗi", còn rollback thật
  để test tích hợp.
- **Slice có nhiều chỗ chạm identity** (như `users`) vẫn một `Store` rộng, chia theo nhóm trong file; nếu ngày nào quá to thì tách interface theo nhóm như `signin` (`People`, `Sessions`, ...).

## Điều slice hiện chưa làm (có chủ đích)

- Slice P3 không có memstore và contract test dùng chung (chỉ bốn slice P2 có); các test tích hợp cũ là lưới an toàn cho `pgstore`.
- Tenant `DedicatedDb`: dữ liệu identity ở DB chung nên không cùng transaction với phần ghi `user_roles` của tenant; hai transaction nối bằng thứ tự (roadmap 122 §7).
- `mfa` chỉ gồm phần của chính người dùng; admin đặt lại MFA của người khác là `users.ResetMfa`. `RegenerateRecoveryCodes` được audit (chủ dự án chốt 2026-11-24): entry nói là đã đổi, không bao giờ chứa mã.
