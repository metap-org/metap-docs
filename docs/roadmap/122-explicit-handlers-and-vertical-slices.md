# 122. Handler viết tường minh và vertical slice cho `internal/` (kế hoạch)

Chủ dự án yêu cầu (2026-11-24), sau roadmap 120: handler của action nên viết **tường minh (boilerplate) để dễ tuỳ biến**, chia layer
(hoặc vertical slice, mỗi slice đủ layer của một web app) để dễ hiểu và unit test đủ, rồi **áp cùng khuôn đó cho `internal/` của metap**
để mọi thứ nhất quán. Audit chuyển sang ghi **nguyên tử** bằng event (outbox). Tài liệu này là kế hoạch và danh sách rủi ro; quyết định
kỹ thuật của phần đã làm nằm ở roadmap 120 §2.8.

**Trạng thái (2026-11-24): P0 (lõi `pkg/action`), P0b (audit nguyên tử) và P2 (bốn slice mẫu) đã làm; P1, P3, P4 chưa làm.**

## 1. Hiện trạng đo từ mã (`internal/`, commit 3ca65ae)

- `internal/admin`: 11 service, khoảng 85 operation, `types.go` 2.910 dòng; mỗi method lặp khung `caller` → gọi `identity.*(ctx, pool, …)`
  → ánh xạ lỗi → ghi audit. Tầng dữ liệu nhận thẳng `*pgxpool.Pool`, không có interface, nên 11 trên 13 file test là test cần Postgres.
- Chỉ 4 file mở transaction (config, identity, mfa, owner). Audit của `admin` ghi sau thao tác, best-effort.
- Quyền truy cập khai báo ở `operationAccess` (`ops_admin.go`); có lưới an toàn sẵn: `routes.golden`, `api.golden.json`,
  `TestRouteInventory`, job `integration` của CI.

## 2. Thiết kế đích

**Hai lớp handler** (đã làm, roadmap 120 §2.8): tường minh `action.Custom` + `a.Decode`/`a.LoadRecord`/`a.Step`/`a.Audit`/`a.Emit`;
đường tắt `action.Record`/`action.Entity` cho ca đơn giản. `action.Unit` là lõi không phụ thuộc entity, để thao tác quản trị dùng được.

**Vertical slice** cho mỗi nhóm thao tác, ví dụ `internal/admin/apikeys/`:

```
types.go     DTO vào/ra (chỉ struct)
rules.go     luật thuần, chỉ stdlib: validate, quyết định, tính diff   → unit test bảng, không DB
store.go     interface (port), khai báo ở phía dùng
pgstore.go   adapter Postgres                                          → integration test
service.go   handler tường minh: caller → decode → rules → store → Step → Audit
register.go  mức truy cập + route + tên action, một chỗ
```

Test của `service.go` dùng store trong bộ nhớ và `actiontest.Sink`; một bộ **contract test** chạy trên cả memory store và `pgstore` để
hai bản không lệch nhau. depguard ép lớp: `rules` không import pgx/huma, `service` không import pgx, chỉ `pgstore` import pgx,
`internal/httpapi` chỉ gọi service. Cùng cách chia với `zones/` của waap (docs 20/21).

## 3. Thứ tự làm (mỗi bước một PR xanh; di chuyển trước, đổi logic sau)

| Bước | Nội dung | Trạng thái |
|---|---|---|
| P0 | Lõi `pkg/action`: lớp tường minh, `Unit`, `Emit` chỉ trong `Step`, step không lồng | **Xong** (metap `2bb8b36`) |
| P0b | Audit nguyên tử: `audit.Enqueue` trong transaction của `CrudService` và `Step`; `audit.Drain`/`RunDrain` idempotent; wiring `app.Run` và `cmd/outbox-publisher` | **Xong** (metap `ec1d462`) |
| P1 | waap: chuyển handler cần tuỳ biến sang `Custom` (19 handler hiện vẫn chạy nguyên); đẩy cặp nhánh trùng tên với metap | Chưa |
| P2 | Slice mẫu trong metap: `preferences`, `sessions`, `apikeys`, `mfa` (package con `internal/admin/<slice>/`, dùng `internal/admin/kernel`); tài liệu `docs/guides/02-admin-slice-template.md`; luật depguard `slice-rules-pure` và `slice-service-no-driver`; contract test chạy trên memstore và pgstore | **Xong** (xem §6) |
| P3 | Slice có transaction: `config`, `credentials`, `sso`, `owner`, `support`, `tenant`, `role`/`policy`, cuối cùng `identity`. Trước đó tách `identity.*` khỏi pool để nhận `DBTX` | **Nền đã xong** (2026-11-24): `tenancy.SharedTx(ctx)` và `kernel.IdentityDB` (một câu lệnh identity chạy trên transaction của unit trong ctx nếu tenant dùng DB chung, không thì trên pool); `apikeys`, `sessions`, `preferences` dùng nó; test pool một connection. Các slice còn lại và audit nguyên tử của chúng chưa chuyển, xem §7 |
| P4 | `signin`, `authctx`, SCIM: để sau khi P2/P3 chứng minh khuôn (gần transport: cookie, redirect, state machine) | Chưa |

## 4. Rủi ro và cách xử lý

1. **Deadlock pool** (lớn nhất). `admin` gọi `identity.*(ctx, s.d.cfg.Shared, …)` xin connection mới; đặt trong `Step` (đang giữ một
   transaction) là đúng họ lỗi đã sửa ở metap #88/#89. Xử lý: store nhận `DBTX` lấy từ `ctx` (như `crud` với unit ctx); mỗi slice có `Step`
   kèm test pool nhỏ; lint chặn `pool.Acquire/Query` trong thân `Step`. Tốn công nhất của P3.
2. **Đổi hành vi bảo mật mà không biết** (`Durable`, `guardOwner`, admin cuối cùng, khoá MFA). Chỉ di chuyển code, không sửa logic trong
   cùng PR; golden OpenAPI và route giữ nguyên từng byte; mỗi slice thêm test từ chối cho từng mức truy cập.
3. **Hai bản store lệch nhau**: contract test dùng chung.
4. **Audit**: đã chuyển sang nguyên tử bằng outbox (roadmap 120 §2.8). Đổi giá: nhất quán sau cùng (độ trễ poll mặc định 500 ms), thứ tự
   chuỗi theo thứ tự drain. `internal/admin` và `ops_sessions` vẫn ghi kiểu cũ cho đến khi slice của chúng được chuyển.
5. **Boilerplate phình ra**: handler chỉ điều phối (khoảng 30 dòng), nghiệp vụ ở `rules`, dữ liệu ở `store`.
6. **Bề mặt thay đổi lớn** (khoảng 11.000 dòng): mỗi slice một PR, từ nhỏ đến lớn; không đổi tên getter kiểu gRPC trong đợt này.
7. **Phá chữ ký `pkg/action`**: chấp nhận được (chưa phát hành ổn định); chữ ký `Record`/`Entity` giữ nguyên nên waap vẫn build.
8. **Hai lớp decode**: huma đã decode input của thao tác quản trị, `action` decode input của entity action; slice quản trị chỉ dùng lõi
   `Step`/`Audit`, không dùng `Decode` của action.

## 5. Ghi chú kiểm chứng P0/P0b (2026-11-24)

- `go test ./...` xanh; `go test -tags integration ./...` xanh trừ Redis và Vault (môi trường này không có; CI có).
- Quét `METAP_TEST_POOL_MAX=1` lại sau P0b: không còn HANG của sản phẩm. Một HANG ở `pkg/tenancy` là lỗi của chính test (đọc "bên ngoài"
  bằng pool khi unit đang giữ connection duy nhất), đã sửa bằng connection riêng. Các FAIL còn lại ở pool=1 là giới hạn đã biết
  ("reconcile cần pool ít nhất 2") hoặc test cố ý lấy hai connection (`TestAUnitOnlyCapturesItsOwnTenant`).
- Test mới: lớp tường minh và `Unit` (`pkg/action`), audit trong cùng transaction với event (`sink_postgres_test.go`), drain idempotent,
  thứ tự theo tenant, dòng hỏng bị đặt riêng (`pkg/audit/drain_e2e_test.go`), publisher bỏ qua topic audit.

## 6. Kết quả P2 (2026-11-24)

- **Bốn slice**, mỗi slice có `types`, `rules`, `store`, `pgstore`, `service`, memstore, contract test và test của service. OpenAPI
  (`api.golden.json`) và `routes.golden` **không đổi một byte**; toàn bộ `internal/admin/...` và `internal/httpapi` vẫn xanh.
- **`internal/admin/kernel`** (caller, lỗi, `Durable`, `ParseID`, `Auditor`) là phần dùng chung; `admin` ủy quyền sang đó nên chỉ còn một định
  nghĩa cho `WithCaller`, `identityErr`, ghi audit và thông điệp từ chối của support session.
- **Tìm thấy và gỡ một bản cài đặt trùng**: tính năng sessions có hai bản, `admin.AccountService.ListMySessions...` (không route nào dùng, chỉ test
  gọi) và `internal/httpapi/ops_sessions.go` (tầng transport tự gọi `identity.*`, vi phạm phân tầng của roadmap 118). Nay chỉ còn slice
  `sessions`, và `ops_sessions.go` chỉ decode và gọi nó.
- **Contract test bắt một giả định sai** ngay lần chạy đầu (`preferences`: mặc định là `en`, không phải rỗng).
- Phần của `admin` bị gỡ: `PreferencesService`, `AccountService.{Sessions,Mfa}`, `UserAdminService.{CreateApiKey,ListApiKeys,RevokeApiKey}` và các DTO
  tương ứng. Còn ở `admin`: `CreateServiceAccount`/`ListServiceAccounts` (dính tới role và người dùng), `ResetMfa` (cần guard chủ sở hữu).
- **Chưa kiểm chứng ở môi trường này**: `golangci-lint` (build bằng Go 1.25, không đọc được `go.mod` 1.26), nên hai luật depguard mới chưa được
  chạy thật; CI sẽ chạy. Nếu glob không khớp, luật chỉ im lặng không bắt gì (không làm hỏng build).

## 7. P3: nền đã làm, và điều phải quyết trước khi chuyển slice (2026-11-24)

**Đã làm.** `pkg/auth/identity` vốn đã nhận `DBTX`; vấn đề nằm ở chỗ `internal/admin` truyền vào pool. Nay `tenancy.SharedTx(ctx)` trả transaction của unit
trong ctx khi tenant dùng DB chung (chiến lược Schema; bảng identity nằm ở đó), và `kernel.IdentityDB` là `identity.DBTX` chọn lúc gọi: transaction của
unit nếu có, pool nếu không. Một thay đổi identity trong `Step` vì vậy chung connection và chung commit với dữ liệu của tenant, và pool một connection không
bị deadlock (test `TestIdentityStatementsJoinTheUnitOfWorkOnAPoolOfOne`). Ngoài unit thì hành vi y hệt trước, nên ba slice đã có (`apikeys`, `sessions`,
`preferences`) được nối vào mà không đổi gì quan sát được.

**Giới hạn cố hữu.** Tenant có database riêng (`DedicatedDb`) thì dữ liệu identity ở DB chung, không thể cùng transaction với unit của tenant: identity
vẫn đi qua pool (connection khác, transaction khác). Atomic chỉ có với tenant Schema; với tenant Dedicated là hai transaction nối bằng thứ tự và audit sau cùng.
`SharedTx` là chính transaction của unit chứ không phải savepoint: một câu lệnh lỗi (vi phạm unique mà code định bắt) làm hỏng cả unit, nên code mong đợi lỗi
như vậy (`isUniqueViolation` trong `owner.go`, `ensureIdentity`) phải tự mở savepoint trước khi chuyển.

**Chưa làm, cần quyết.** Chuyển một slice sang audit nguyên tử nghĩa là thay `kernel.Auditor.Record` (ghi sau, dạng `ActionUpdate` với cặp before/after) bằng entry
xếp hàng trong `Step`. `Unit.AuditEntry` hiện chỉ tạo entry dạng `ActionCustom` với `detail`; trail của admin hôm nay dùng `ActionUpdate` và cặp before/after, và UI đọc
dạng đó. Hai hướng: (a) cho `Unit` nhận entry dựng sẵn, giữ nguyên dạng audit hiện có (đổi `pkg/action`, không đổi trail); (b) chuyển admin sang dạng `Custom`
(đổi trail và UI). Đề xuất (a).

## 8. P3: audit nguyên tử trên DB chung, slice mẫu `apikeys` (2026-11-24)

Quyết định (chủ dự án chốt (a)): giữ nguyên dạng audit mà trail và UI đang đọc (cập nhật với cặp before/after), và ghi nó nguyên tử với thay đổi.

- **Audit store là một, ở DB chung**; `audit.RunDrain` đã drain outbox của DB chung lẫn của từng DB riêng vào đó. Nên thao tác identity ghi entry vào **outbox của DB
  chung, trong cùng transaction với thay đổi identity**, cho mọi tenant: không cần cơ chế event mới, và tenant có DB riêng cũng atomic cho phần này.
- `tenancy.Router.BeginSharedUnit` mở unit trên DB chung bất kể chiến lược của tenant (không bao giờ mở DB riêng của tenant). `action.NewSharedUnit` +
  `SharedSink` chạy `Step` trong đó; `Unit.AuditPrepared` nhận entry dựng sẵn; `kernel.AuditEntryOf` dựng đúng entry mà `WriteAudit` ghi.
- `apikeys.Service.WithSink(action.NewSink(router))`: `Create` và `Revoke` là một `Step`, khoá và entry cùng commit hoặc cùng rollback. Không sink thì giữ
  đường cũ ("đổi rồi ghi best-effort"). Test: sink trong bộ nhớ; tích hợp trên pool một connection cho cả tenant dùng DB chung lẫn tenant có DB riêng
  (secret của DB riêng trỏ vào hư không, vì unit chung không bao giờ mở nó), kèm một thay đổi hỏng sau khi ghi khoá thì cả khoá lẫn entry biến mất.
- Còn lại (cùng khuôn): `mfa`, `sessions`, `preferences` rồi `config`, `credentials`, `sso`, `owner`, `support`, `tenant`, `role`/`policy`, `identity`.
  Phần ghi vào DB riêng của tenant (`user_roles`) trong cùng thao tác vẫn là transaction thứ hai, nhất quán sau cùng.
