# 107. Rà soát pattern lặp, chuẩn chung cho service tuỳ biến (metap + waap)

Trạng thái: **P0, P1.1 đến P1.4 xong** (2026-11-14); PR `metap-org/metap#44` và `metap-org/waap#33`. Còn lại: dọn layering của `infra`/`peripherals` (P1.4b), `metap-lowcode` chưa chuyển sang bộ kit mới, viết tài liệu dùng metap như framework (P2).
Trigger: chủ dự án thấy cách khai báo entity, helper, router/handler, xử lý error, bắn event/outbox chưa theo một chuẩn chung, và muốn có bộ kit để code custom ở từng service viết ra đúng tiêu chuẩn, rồi viết tài liệu kỹ thuật dùng metap như một framework.

## 1. Kết quả rà soát

Rà `metap` và `waap` (Go) theo các lớp: env/config, HTTP server và handler, gRPC, GraphQL, xử lý error, event/outbox, khai báo entity, import chéo giữa package. Những chỗ lệch chuẩn đáng kể:

- **HTTP:** không listener nào đặt timeout (slowloris); 20 handler đọc `r.Body` không giới hạn; edge buffer body không giới hạn; so sánh secret không constant-time; ba handler nuốt lỗi decode; mỗi plane tự viết `writeJSON`.
- **Error:** `crud.ServiceResult` (struct có `IsOk`) buộc mọi nơi gọi phải kiểm hai lần (`err` rồi `IsOk`), và lỗi nghiệp vụ lẫn lỗi hệ thống đi hai đường khác nhau.
- **Env:** giá trị không parse được bị bỏ qua im lặng; `Require` coi chuỗi rỗng là đã set; `infra.LoadConfig` và helper của control-plane đọc `os.Getenv` thẳng; 72 chỗ dùng alias `runtime.*`.
- **Entity:** bốn bộ helper `field/enumField/refField/boolPtr/strPtr` lệch tham số ở zones, scanning, alerting, analytics; hai cơ chế đăng ký (`init()` + `SubmitEntity` và danh sách `Options.Entities` thủ công, ghi hai lần).
- **Event/outbox:** topic ghép chuỗi ở 4 chỗ trong `workflow.go`, parse bằng suffix ở 3 consumer; payload là `map[string]any`; `_ = event.Ack()` nuốt lỗi.
- **Gateway:** engine GraphQL generic (khoảng 2,5k dòng) nằm trong waap dù không biết gì về WAF.

## 2. Đã làm

**P0, bảo mật nền HTTP** (`pkg/kit/httpx`, chỉ stdlib để edge/analytics dùng được): `NewServer`/`ApplyServerDefaults` (read-header 10s, read 60s, idle 120s, header 1 MiB), `DecodeJSON`/`DecodeJSONOrFail` (1 MiB, 413 `payload_too_large`), `Recover`, `MaxBody`, `SecretEqual`, `BearerToken`, `WriteJSON`/`WriteError`. `Recover` đặt ngay trong `RequestContext` để lỗi panic vẫn có `x-request-id`. Upload attachment giới hạn `ATTACHMENT_MAX_BYTES` (25 MiB), response webhook đọc tối đa 1 MiB, `S3ObjectStore.Get` từ chối object quá 256 MiB. Waap: edge giới hạn body `MAX_REQUEST_BODY_BYTES` (10 MiB, 413 không chạm origin), ingest của control-plane và analytics-api dùng cùng bộ kit.

**P1.1, error và env:**
- `CrudService`/`RecordBackend` trả `(T, error)`; lỗi nghiệp vụ là `*crud.ServiceError{Status, Code, Message, FieldErrors}`, lỗi thường là lỗi nội bộ. REST map một lần qua `httpserver.WriteCrudError`, gRPC mang `ServiceError` trong trailer và `GrpcBackend` khôi phục, GraphQL gateway map sang `extensions`. `ServiceResult`/`IsOk` không còn.
- Bẫy gặp khi chuyển: response nhúng `result.Data` giờ chỉ là map field chứ không còn là cả record; phải trả `RecordDto`. Bắt được bằng integration test (scanning `SubmitFindings`, zones `verifyDns`/`verifyOwnership`).
- `kit/env`: cảnh báo khi không parse được, `Require` coi rỗng là thiếu, thêm `env.List`; bỏ alias `runtime.*`; `infra.LoadConfig` và control-plane dùng `kit/env`.

**P1.2, khai báo entity:** `pkg/metadata/dsl.go` (`NewField/EnumField/RefField` + `.Require().Index().Sort().Search().Uniq()`, `Transition`, `Ptr[T]`). Một đường đăng ký: `init()` + `SubmitEntity`; `app.Main` reconcile entity đã đăng ký theo thứ tự phụ thuộc (`MetadataRegistry.Definitions()`), `Options.Entities` chỉ còn là override. Waap xoá bốn bộ helper và ba danh sách entity thủ công.

**P1.3, event/outbox:** `pkg/events` — builder và parser topic (`WorkflowTransitioned`, `Record`, `ParseTopic`), payload có kiểu và version (`Transitioned`, `RecordEvent`, `Envelope`, `Decode[T]`, trường `"v"`), `events.Emit` cho event tuỳ biến trong cùng transaction (topic được kiểm: có namespace, không wildcard, không trùng topic lifecycle). Consumer dùng `AckLogged`/`NackLogged`.

**P1.4, gateway:** `pkg/graphqlgateway` chuyển từ waap sang metap: `BuildSchema(registry, backends, WithPlatformBase, WithExtensions)`; sản phẩm chỉ giữ extension (`waap/management-plane/wafgateway`, 8 mutation WAF) và `main`. Test của metap dùng registry `shop.*` tự khai báo. Schema chỉ có backend read-only không còn lỗi "Mutation fields must be an object".

## 3. Quyết định và đánh đổi

- `ServiceError` đổi hẳn ở mọi nơi gọi (chủ dự án chọn "đổi hẳn" thay vì giữ shim), nên mọi repo phụ thuộc metap phải theo; `metap-lowcode` chưa theo (build lệch `go.mod`, để lúc làm lowcode).
- Payload event thêm trường `"v"`: consumer bỏ qua trường lạ nên không phá consumer cũ; đổi tên/xoá trường mới là bump version.
- `edge-plane` vẫn chỉ import `pkg/kit/*` (depguard); không kéo thêm gì từ metap.
- Không có thay đổi schema DB hay hành vi API nhìn thấy từ bên ngoài, ngoài mã lỗi 413 mới và log cảnh báo env.

## 4. Còn lại

- **P1.4b:** `infra` gom config + pool + outbox + eventbus, `peripherals` gom nhiều thứ không liên quan; import chéo `cronscheduler→oauthserver`, `workflow`/`cron→infra` (ghi outbox nên đi qua `events`). Cần feature brief trước khi đụng.
- `metap-lowcode`: chuyển sang `(T, error)`, `kit/env`, DSL, `events.Emit` (hiện dựng `infra.OutboxEvent` tay).
- Subscriber nghiệp vụ của control-plane vẫn tự parse `data.zoneId`.
- **P2:** tài liệu kỹ thuật dùng metap như framework (khai báo entity, route/handler, error, event, gateway) dựa trên bộ kit trên.
- Task 59 (S7 diễn tập migration WAF) vẫn mở.
