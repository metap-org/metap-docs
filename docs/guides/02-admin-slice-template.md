# Khuôn slice cho `internal/admin` (roadmap 122, P2)

Một slice là một nhóm thao tác quản trị viết đủ các lớp trong một package con `internal/admin/<slice>/`, để đọc từ trên xuống, test từng
lớp riêng, và sửa một chỗ khi cần tuỳ biến. Bốn slice đầu tiên làm mẫu: `preferences`, `sessions`, `apikeys`, `mfa`. Muốn thêm slice mới,
sao chép slice nhỏ nhất (`preferences`) rồi đổi tên.

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

- `rules.go` không import `pgx` và `net/http`.
- `service.go` không import `pgx`: chỉ chạm DB qua `Store`.
- Tầng ứng dụng không import transport (`httpapi`, huma, router): luật có từ roadmap 118.

## Contract test: vì sao có

Memstore và pgstore là hai bản cài đặt của cùng một port; nếu chỉ test service trên memstore thì hai bản có thể lệch mà không ai biết.
Contract là một bộ ca chạy trên cả hai. Nó đã bắt một giả định sai ngay ở slice đầu: store thật trả `en` (locale mặc định) khi chưa lưu gì,
còn memstore ban đầu trả rỗng.

## Điều slice hiện chưa làm (có chủ đích)

- **Audit vẫn ghi kiểu cũ** (sau thao tác, best-effort, qua `kernel.Auditor`). Chuyển sang nguyên tử (cùng transaction, qua outbox, roadmap 120
  §2.8) cần store nhận `DBTX` lấy từ `ctx` thay vì pool cố định; đó là P3. Vì audit đi qua port `Auditor`, đổi sau này không đụng service.
- **Store dùng pool trực tiếp**, chưa dùng `Step`. Slice nào cần nhiều ghi trong một transaction (P3) sẽ dùng `action.Unit.Step`.
- `mfa` chỉ gồm phần của chính người dùng; admin đặt lại MFA của người khác (`ResetMfa`) ở lại service người dùng vì cần guard chủ sở hữu và
  identity dùng chung. `RegenerateRecoveryCodes` hiện **không** audit (như trước khi chuyển); có nên audit hay không là câu hỏi riêng.
