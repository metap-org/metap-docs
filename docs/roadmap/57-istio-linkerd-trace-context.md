## Phase 57: Chuẩn bị `metap-runtime`/`metap-grpc` cho tích hợp Istio/Linkerd — W3C Trace Context (2026-08-31)

Trigger: chủ dự án hỏi thẳng "http, grpc hỗ trợ các header của istio/linkerd chưa", sau đó xác nhận
mục tiêu — "muốn tích hợp được Istio trong tương lai, vì metap-lowcode sau này sẽ rất lớn". Không
phải yêu cầu deploy service mesh ngay, chỉ chuẩn bị codebase tương thích trước khi có nhu cầu thật.

**Rà soát trước khi lên plan** (đọc code, không đoán) — hiện tại không có gì tương thích chuẩn mesh:
`metap-runtime::request_id` chỉ đọc/echo header tự đặt tên `x-trace-id` (không phải chuẩn nào
Istio/Linkerd dùng), và `x-request-id` luôn tự sinh UUID mới thay vì đọc header có sẵn — Envoy
sidecar (Istio) tự set `x-request-id` ở ingress và cần app pass-through nguyên vẹn để nối log, app
cũ bỏ qua giá trị đó. Không đọc/forward B3 hay W3C Trace Context (`traceparent`). Điểm nghẽn duy
nhất mọi RPC outbound đi qua, `metap-grpc::client::GrpcBackend::signed_request`, chỉ gắn metadata
`authorization`, không gắn gì cho tracing. 3 caller thật của `metap_runtime::http_client::
default_client()` (`graphql-gateway`/`metap-jwks`/`cron-scheduler`) đều chạy lúc boot/background,
không nằm trong 1 incoming request nào — propagate ở đây hiện vô nghĩa.

Đi qua `EnterPlanMode`/`ExitPlanMode` trước khi code. **Chọn W3C Trace Context** (`traceparent`),
không phải B3 — chuẩn IETF thật, cả Istio (mặc định từ ~1.12+) và Linkerd đều hỗ trợ/forward, khớp
tự nhiên với field `trace_id`/`span_id` `tracing::Span` đã có sẵn, không cần thêm SDK OpenTelemetry
nặng (Envoy/linkerd-proxy tự tạo/export span ở tầng sidecar — app chỉ cần chain header đúng).

## Thiết kế và thực thi

- **Module mới `metap_runtime::trace_context`**: `TraceContext { trace_id, parent_span_id, span_id,
  sampled }`, `from_headers(&HeaderMap)` parse `traceparent` đến (format `00-{32hex}-{16hex}-{2hex}`,
  từ chối trace-id/parent-id toàn số 0 theo đúng spec) hoặc sinh trace gốc mới nếu thiếu/không hợp
  lệ, `to_traceparent_header()` (span_id hop này → parent cho hop sau). Ambient qua
  `tokio::task_local!` (`current()`/`scope()`) — không xuyên `tokio::spawn` boundary (ghi rõ giới
  hạn trong doc comment, chưa có handler nào trong codebase làm vậy nên chưa phải vấn đề thật). 7
  unit test.
- **`request_id::generate_request_ids`**: thêm parse `traceparent` bằng `trace_context::from_headers`
  rồi bọc `next.run(request)` trong `trace_context::scope`; sửa 1 chỗ thật để tương thích Envoy tốt
  hơn — `x-request-id` giờ đọc header đến nếu có + hợp lệ, chỉ sinh UUID mới khi thiếu (trước đó
  luôn sinh mới, phá pass-through mà Envoy/Istio ingress cần).
- **`trace::build`**: thêm field `trace_id`/`span_id` (từ `trace_context::current()`) vào span
  `http_request`, đổi tên field cũ thành `legacy_trace_id` (vẫn giữ nguyên `x-trace-id`/
  `RequestIds` — cộng thêm, không thay thế).
- **`metap_runtime::http_client::attach_trace_context`**: helper opt-in (không tự động) gắn
  `traceparent` vào 1 `reqwest::RequestBuilder` nếu đang trong `scope`. Không wire vào 3 caller hiện
  tại của `default_client()` vì cả 3 đều chạy boot-time/background, không có context nào để
  propagate — dựng sẵn cho lần đầu có caller thật nằm trong 1 request.
- **`metap-grpc::client::attach_traceparent`** (hàm tự do, không phải method) — gắn tự động vào
  `GrpcBackend::signed_request`, điểm nghẽn duy nhất mọi RPC outbound đi qua: có
  `trace_context::current()` thì gắn thêm metadata `traceparent`, không có thì bỏ qua, không lỗi.
  Sửa đúng 1 chỗ là mọi resolver GraphQL/route nào gọi ra upstream tự động có propagation, không
  cần sửa từng call site.

**Ngoài phạm vi** (chưa làm, chưa có tín hiệu thật cần): fallback đọc B3, gRPC phía server tự
`scope()` quanh xử lý RPC (`GrpcRecordService`'s handler hiện không gọi tiếp ra network nào khác —
chưa ai hưởng lợi), `tracestate` header (không cần thiết nếu không có vendor-specific state để
mang), tích hợp OpenTelemetry SDK/export span thật (Envoy/linkerd-proxy tự làm phần đó khi header
propagate đúng — `docker-compose.yml`'s observability profile mới chỉ có Grafana/Prometheus cho
metrics, không phải tracing), tự động hoá propagation cho `http_client` qua `reqwest-middleware`
(không thêm dependency mới khi 0 caller thật cần).

## Xác minh

`cargo build/test/clippy --workspace -D warnings/fmt --all --check` sạch toàn `metap` — 89 test
suite `ok` (224 test pass, 0 fail), không suite nào thêm mới (chỉ thêm test vào suite có sẵn của
`metap-runtime`/`metap-grpc`). `metap-runtime` giờ 11 module/28 test (từ 10 module/19 test ở Phase
56). `metap-grpc` +2 test (`attach_traceparent_is_a_no_op_outside_a_scope`/
`attach_traceparent_sets_metadata_inside_a_scope`). `metap-lowcode` build lại sạch (dùng
`metap-runtime`/`metap-grpc` qua `metap-lowcode-http`, không đổi gì trực tiếp, verify không vỡ).

Một lần build/test workspace bị gián đoạn giữa chừng bởi lỗi linker/temp-dir tạm thời (một tiến
trình ngoài phiên làm việc đã dọn bớt `target/` giữa lúc build — không liên quan tới thay đổi code
lần này); retry lại thành công, không phải regression.
