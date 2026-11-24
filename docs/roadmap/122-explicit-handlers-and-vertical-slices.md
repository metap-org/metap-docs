# 122. Handler viết tường minh và vertical slice cho `internal/` (kế hoạch)

Chủ dự án yêu cầu (2026-11-24), sau roadmap 120: handler của action nên viết **tường minh (boilerplate) để dễ tuỳ biến**, chia layer
(hoặc vertical slice, mỗi slice đủ layer của một web app) để dễ hiểu và unit test đủ, rồi **áp cùng khuôn đó cho `internal/` của metap**
để mọi thứ nhất quán. Audit chuyển sang ghi **nguyên tử** bằng event (outbox). Tài liệu này là kế hoạch và danh sách rủi ro; quyết định
kỹ thuật của phần đã làm nằm ở roadmap 120 §2.8.

**Trạng thái (2026-11-24): P0 (lõi `pkg/action`) và P0b (audit nguyên tử) đã làm, đã push; P1 đến P4 chưa làm.**

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
| P2 | Slice mẫu trong metap: `preferences`, `sessions`, `apikey`, `mfa`; tài liệu slice mẫu; luật depguard; contract test | Chưa |
| P3 | Slice có transaction: `config`, `credentials`, `sso`, `owner`, `support`, `tenant`, `role`/`policy`, cuối cùng `identity`. Trước đó tách `identity.*` khỏi pool để nhận `DBTX` | Chưa |
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
