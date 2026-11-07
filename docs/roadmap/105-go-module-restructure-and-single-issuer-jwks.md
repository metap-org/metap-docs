## Phase 105: cấu trúc lại module Go và JWKS một nơi cấp token (2026-11-07)

Trigger: ba repo Go (`metap`, `metap-demo-waf`, `metap-lowcode`) cùng chạm vào cách tổ chức module và cách
các service xác thực lẫn nhau. Làm ở hai repo song song: `metap` PR #41 và `metap-demo-waf` PR #28, cùng một
nhánh `claude/jwks-single-issuer`. Chủ dự án chốt hướng; phase này ghi lại để tài liệu khớp với code.

### Đã làm ở `metap`

| # | Thay đổi | Chi tiết |
|---|---|---|
| M1 | Module Go ra thư mục gốc | `go/` không còn. `go.mod` ở gốc, module path `github.com/metap-org/metap` (khớp remote nên `go get` được, repo khác không cần `replace` chỉ vì đường dẫn). Thư viện ở `pkg/<tên>`, binary ở `cmd/*`. Mọi chỗ cũ ghi `go/<pkg>` đọc là `pkg/<pkg>`, chạy lệnh Go từ thư mục gốc |
| M2 | `migrations/` là một Go package | `migrations.FS` qua `go:embed`; `db-migrate` áp bộ nhúng sẵn trừ khi có `MIGRATIONS_DIR` |
| M3 | `pkg/testkit` | Mỗi test một database mới, sao từ template đã migrate. Test tích hợp mang `//go:build integration`, nên `go test ./...` chỉ chạy unit. Thiếu Postgres hoặc Redis thì test **fail, không skip**. Testkit áp lại các `ALTER DATABASE ... SET` (như `search_path` của migration 0028) vì `CREATE DATABASE ... TEMPLATE` không sao chép chúng |
| M4 | Mock và test | mockery v3 (`.mockery.yml`), `crud.TxBeginner`/`PermissionChecker` và `crudmock.RecordBackend` xuất ra cho repo khác dùng; mọi `t.Fatal`/`t.Error` đổi sang testify |
| M5 | `pkg/kit/{env,shutdown,httpx}` | Tiện ích chỉ dùng thư viện chuẩn, cho code ngoài management plane. depguard (`.golangci.yml`) từ chối mọi import ngoài thư viện chuẩn ở đây; `pkg/runtime` giữ lại bằng alias. `Serve` trả lỗi listener ngay, không đợi tín hiệu |
| M6 | `pkg/app` | Khung khởi động service (config, router, permission, reconcile, CRUD, gRPC tuỳ chọn, HTTP) với các `Module` route tuỳ chọn |
| M7 | Tách phụ thuộc nặng | `pkg/secrets` (kho secret cloud), `pkg/storage/s3store` (S3) ra khỏi `control` và `storage`, nên `httpserver` không còn nạp AWS SDK. Nhóm route tuỳ chọn (`oauth2routes`, `cronroutes`, `dashboardroutes`, `attachmentroutes`) thành package riêng; `BuildRouter` nhận `Mounts` |
| M8 | JWKS một nơi cấp token | `pkg/jwks`: `KeySource` (issuer dùng `SharedKeyStore` trong bộ nhớ, service khác dùng `JwksClient` trỏ về `/.well-known/jwks.json` của issuer), `LoadKeyDir`/`WatchKeyDir` đọc thư mục khoá `JWKS_KEYS_DIR` (`<kid>.key` + `signing-kid`), `HasKid` để verifier phân biệt token EdDSA/JWKS với token RSA cũ |
| — | Sửa | Bỏ `peripherals.Reconcile` (nhắm bảng `records` đã xoá ở migration 0033, ghi một WARN mỗi trường được index ở mỗi lần boot); testkit mặc định cổng Postgres của compose (5433) |

Xoay khoá không cần khởi động lại: thêm khoá mới (`dev-tools gen-jwks-key <dir>`, đã công bố nhưng chưa ký),
đợi mọi verifier làm mới cache, trỏ `signing-kid` sang khoá mới, đợi token dài hạn nhất của khoá cũ hết hạn
rồi xoá file `.key` cũ.

### Đã làm ở `metap-demo-waf`

- Module mỗi plane ra khỏi thư mục `go/`: `data-plane/go.mod`, `control-plane/go.mod`, `edge-plane/go.mod`, module
  path `github.com/metap-org/waap/<plane>`. `edge-plane` chỉ import `pkg/kit/*` của metap (depguard bắt buộc),
  thay cho mã env, shutdown, serve tự viết.
- Ba `main.go` của data-plane dùng `pkg/app`, bỏ khoảng 150 dòng sao chép mỗi file.
- **`zones-service` là nơi duy nhất cấp token**; `scanning-service`, `alerting-service` và
  `waf-graphql-gateway` chỉ verify bằng `JWKS_URL`, không giữ khoá riêng. Mọi đăng nhập đi vào `:3000`.
  Trước đó ba service data-plane cùng giữ một khoá riêng (xem `CLAUDE.md` của `metap-demo-waf`).
- `GuardedZonesBackend` chuyển tiếp **token của chính người gọi** cho bước kiểm tra xoá zone giữa các service
  thay vì tự ký token mới, và từ chối xoá (503) khi không có token của người gọi. Gateway nhận
  `UPSTREAM_N_REST_URL` rõ ràng thay vì suy ra từ URL đăng nhập.
- Hợp đồng control-plane → edge-plane (định dạng Redis) được cố định bằng golden file; test cả phía sinh và phía
  nhận cùng fail khi lệch. `control-plane/ports` có seam và mock cho đồng bộ, resync (không bao giờ xoá sạch
  edge khi data-plane không liên lạc được) và ingest.
- Sửa hai lỗi bảo mật: `CLEARANCE_SECRET` mặc định của edge sinh từ timestamp, nay dùng `crypto/rand`; so sánh
  ingest token dùng `!=`, nay so sánh hằng thời gian.
- CI dựng theo nhánh `metap` cùng tên khi có, để thay đổi trải qua hai repo qua được CI trước khi `metap` merge.

### Ghi chú

- Topology token vẫn là **một nơi cấp**, chưa phải nơi cấp tách riêng (issuer vẫn là một service nghiệp vụ,
  `zones-service`). Tách ra một dịch vụ định danh riêng là quyết định sau.
- `pkg/grpcapi/pb/metap_crud.pb.go` được sửa tay byte độ dài `go_package` vì máy không có `protoc`; sinh lại ở lần
  đổi `.proto` kế tiếp.
- Sibling repo vẫn import theo module path cũ cần đổi sang `github.com/metap-org/metap/pkg/...`.

### Verify

- Hai PR đã merge. Chưa tự chạy lại bộ test của hai repo cho phase này; mục này ghi theo mô tả commit.
- Theo mô tả của PR #28: kiểm trên compose stack việc xoá zone chéo service bằng token của người gọi, và xoay
  khoá trực tiếp (công bố, chuyển ký, gỡ khoá cũ).
