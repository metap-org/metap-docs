## Phase 75: `aggregate` lên thành capability generic của `RecordBackend` — gRPC + GraphQL (2026-09-04)

Trigger: frontend `metap-demo-waf` báo lỗi `Unknown field "aggregate" on type "Query"` khi mở
Dashboard. Rà lại: Phase 70 chỉ đưa `aggregate` vào REST (`CrudService::aggregate`, `POST
/api/{entity}/aggregate`) — chưa từng vào `RecordBackend` trait (`metap-crud`), nên gRPC
(`GrpcBackend`/`GrpcRecordService`) và GraphQL đơn-service (`metap-graphql`'s dynamic schema
builder) đều không có, kể cả cho một app chỉ mount `metap-graphql-http` trực tiếp, không riêng gì
qua gateway.

**Lưu ý quan trọng phát hiện sau khi đã làm xong phần này**: `metap-demo-waf` hoá ra **đã có sẵn**
1 field `aggregate` khác — 1 resolver custom trong `waf-graphql-gateway` (binary riêng của WAF,
Phase 73) forward thẳng sang `POST /api/{entity}/aggregate` REST. Phiên này ban đầu không biết
điều đó (kiểm tra thư mục `graphql-gateway/` không kỹ, chỉ thấy `.env`, bỏ sót `Cargo.toml`/
`src/main.rs`) nên đã làm phần dưới đây như một gap thật của core — **gap ở core THẬT SỰ đúng**
(gRPC/GraphQL đơn-service vẫn thiếu `aggregate` bất kể WAF), nhưng lỗi 500 ban đầu người dùng thấy
lại do nguyên nhân khác hẳn: `docker-compose` chạy nhầm sang binary `graphql-gateway` generic của
`metap` core thay vì `waf-graphql-gateway` — xem Phase 76's mục sửa lỗi đó. Hai field `aggregate`
(field cũ REST-forward trong WAF binary, field mới generic ở core) từng đụng nhau (`async-graphql`
panic "Field `aggregate` already exists" khi chạy đúng binary WAF lần đầu) — xử lý bằng cách xoá
bản REST-forward trong `waf-graphql-gateway`, giữ bản generic ở core (nhất quán hơn, không cần
hardcode entity→upstream trong 1 app demo, cùng đường gRPC mọi field khác đã dùng).

## Đã làm

- **`metap_query::AggregateSpec`** (kiểu wire thô mới — `metrics`/`bucket` vẫn là string chưa
  parse) + `into_input()` — gom logic parse (`AggregateMetric::parse`/`TimeBucket::parse`, default
  `count`/`DEFAULT_GROUPS`) dùng chung cho cả 3 transport thay vì mỗi nơi tự lặp lại. REST
  (`metap-http/routes/records.rs`) đổi sang dùng thẳng `AggregateSpec` làm body, bỏ struct
  `AggregateBody` cũ trùng lặp.
- **`RecordBackend::aggregate`** (`metap-crud`) — method mới trong trait, implement cho cả 3:
  `CrudService` (in-process, parse lỗi trả `ServiceResult::err_with_message(400, ...)` chứ không
  phải `Err` — đúng convention mọi method khác trong impl này đã theo), `GrpcBackend` (client, RPC
  mới), `CompositeBackend` (route theo entity name, y hệt `list`/`get`/...).
- **RPC `Aggregate`** mới trong `metap_crud.proto` (`AggregateRequest{entity_name, spec:
  Struct}` → `AggregateResponse{rows: repeated Struct}`) — server (`GrpcRecordService`) deserialize
  `Struct` → `AggregateSpec` → `into_input()` → gọi `CrudService::aggregate` sẵn có, lỗi parse map
  `Status::invalid_argument`.
- **`Query.aggregate(entity: String!, spec: Json!): Json`** trong `metap-graphql` — field tĩnh
  (không per-entity, khác mọi field khác) thêm 1 lần trong `build_schema_parts` sau vòng lặp
  entity, trả bọc `{"data": [...]}` khớp đúng envelope REST đã trả để 2 transport không lệch shape.

## Xác minh

`cargo test` toàn bộ crate chạm tới (`metap-query`/`metap-crud`/`metap-grpc`/`metap-graphql`/
`graphql-gateway`/`metap-http`) sạch, kể cả e2e Postgres thật — cùng 3 fail
`platform_config_postgres.rs` đã xác nhận có sẵn trước, không liên quan. Verify sống qua đúng
binary WAF thật (`waf-graphql-gateway`, sau khi sửa Phase 76's bug chạy nhầm binary): query
`aggregate` trả data đúng (`{"data":[{"count":2,"group":"suspended"}]}` cho `waf.zones` group theo
`status`).

## Chưa làm

Field-level masking cho `aggregate` — như Phase 70 đã ghi, chưa có tương đương (cần readable-field
set thread vào planner), vẫn giữ nguyên hiện trạng đó, không phải việc phase này làm thêm.
