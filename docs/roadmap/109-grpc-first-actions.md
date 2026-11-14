# 109. gRPC làm chuẩn cho API trong service, ra ngoài qua GraphQL

**Trạng thái:** xong cho 3 service waap và gateway, 2026-11-14 (metap-org/metap#46, metap-org/waap#36). Chưa có đo hiệu năng, chưa có e2e qua gateway thật.

## Quyết định (chủ dự án, 2026-11-14)

- gRPC là transport chuẩn cho mọi API viết trong service. Ra ngoài (portal, bên thứ ba) đi qua GraphQL gateway.
- REST chỉ cho ngoại lệ bắt buộc, và **contract REST vẫn được giữ** để service nào cần vẫn gọi được.
- `e-edge` được dùng REST (là reverse proxy HTTP, và là module chỉ dùng thư viện chuẩn). `a-api` cũng vậy.

## Hiện trạng trước khi làm

CRUD đã đi gRPC (`RecordService`) và gateway đã gom các service thành một GraphQL; REST CRUD đã gỡ từ 2026-09-21. Còn lệch ở các thao tác không phải CRUD của waap (`verify-dns`, `test-origin`, `sync-config-state`, `run`, `correlate`, `evaluate`, `test`): đó là route REST, gateway gọi bằng `ForwardPost` qua HTTP.

## Thiết kế: action khai báo trong metadata

- `EntityDefinition.Actions []EntityAction{Name, Label, Description, Scope}`; `Scope` là `record` (cần id) hoặc `entity`. Hợp lệ hoá trong `metadata.Validate` (tên camelCase, không trùng, scope đúng).
- `crud.ActionRegistry` giữ handler theo `(entity, action)`; `app.Options.Actions` đăng ký, boot từ chối nếu entity khai báo action mà không có handler.
- `ActionService.Invoke` là một RPC chung cho mọi entity và action (payload là `Struct`, cùng thiết kế với `RecordService`), nên thêm action không phải sinh lại proto. Server kiểm tra cặp (entity, action) có được khai báo và đúng scope trước khi gọi handler.
- **Quyền:** action không có cơ chế quyền riêng. Handler phải đọc/ghi qua `RecordBackend` với `RequestContext` của người gọi, nên action không bao giờ làm được nhiều hơn CRUD thường. Đây là ràng buộc về cách viết handler, chưa có cơ chế kỹ thuật ép buộc.
- `grpcapi.GrpcBackend` cài `crud.ActionInvoker`; token là token của người gọi (forwarded), như các RPC khác.
- Gateway tự sinh mutation `<action><TypeName>` (ví dụ `verifyDnsWafZones(id, input)`), kèm lỗi có `extensions.code/status`. Product giữ tên mutation cũ qua `graphqlgateway.InvokeAction`.
- REST giữ nguyên hành vi: mỗi action viết một hàm lõi, `httpserver.RecordActionRoute` (REST) và handler action (gRPC) cùng gọi hàm đó.

## Đã làm ở waap

9 route chuyển thành action: `waf.zones` (`verifyDns`, `testOrigin`, `syncConfigState`), `waf.domains` (`verifyDns`), `waf.scan_jobs` (`run`), `waf.alert_policies` (`test`, `evaluate`), `waf.incidents` (`correlate`). Gateway gọi chúng qua gRPC; tên mutation cũ của portal còn nguyên (bọc mỏng). `UPSTREAM_N_REST_URL` chỉ còn cần cho các field platform admin.

## Ngoại lệ REST (giữ)

`/auth/*`, OAuth2 và JWKS, health và metrics, attachments (upload/download nhị phân), webhook nhận từ ngoài, `POST /internal/scan-jobs/{id}/findings` (scanner gọi ngược), `/ingest/events` (edge gửi lên control), toàn bộ `e-edge` và `a-api`.

## Chưa làm / rủi ro

- Test: unit (registry, validate, mutation sinh ra bằng backend giả) và integration gRPC thật qua `GrpcBackend.Invoke`; **chưa có e2e từ portal qua gateway tới service thật**, và chưa chạy lại e2e portal sau khi đổi.
- Các route `/internal/...` mà cron/job gọi bằng REST vẫn giữ REST; chưa đổi người gọi sang gRPC.
- Đóng cổng REST công khai của service (chỉ mở gRPC nội bộ) chưa làm: theo quyết định ở trên REST vẫn được phép tồn tại.
- Mất kiểu ở hợp đồng (payload `Struct`). Nếu cần hợp đồng có kiểu cho action nào đó thì viết proto riêng cho action đó.
