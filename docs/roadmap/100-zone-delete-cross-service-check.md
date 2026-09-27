## Phase 100: `zone_delete_guard`'s cross-service reference check, đóng gap Phase 99 để lại (2026-09-27)

Trigger: ngay sau Phase 99 (3/4 guard đã sửa, guard thứ 4 cờ lại là quyết định kiến trúc), chủ dự
án chốt hướng dứt khoát: "port hết sang graphql" — không dừng ở việc cờ lại, làm luôn.

### Vấn đề cụ thể cần giải

Bản REST cũ của `zone_delete_guard` gọi `GET {scanning,alerting}-service/api/{entity}` với header
auth forward nguyên vẹn từ request gốc — cả 2 phần đó giờ đều không còn đúng: route REST đã xoá,
và bản thân middleware cũng unreachable (Phase 99). Port lại cần trả lời 2 câu hỏi kiến trúc, không
chỉ đổi REST→gRPC:

1. **Xác thực bằng gì** — service-account (login cố định) sẽ chỉ thấy đúng 1 tenant (sai cho mọi
   tenant khác, vì 3 service này là `Schema`-strategy, nhiều tenant chung 1 bảng vật lý).
2. **Boot của `zones-service` có phụ thuộc `scanning-service`/`alerting-service` đã chạy chưa** —
   nếu có, phá nguyên tắc "3 service độc lập triển khai" chính tài liệu app này đặt ra.

### Giải pháp: mint token cùng identity, kết nối lười + tự phục hồi

**Xác thực — không dùng service-account, mint token mới cùng `tenant_id`/`user_id` với request
gốc.** Cả 3 WAF service dùng chung 1 signing key (JWKS Ed25519 hoặc RSA tĩnh, tuỳ triển khai) —
`zones-service` vốn đã giữ đúng key đó (`state.token_signer`/`state.jwt_encoding_key_pem`, dùng để
tự publish JWKS). `GuardedZonesBackend` giờ giữ 2 field này (không giữ cả `AppState`, chỉ đúng 2 cái
cần cho việc mint), tự dispatch y hệt `AppState::mint_token` (JWKS khi cấu hình, RSA tĩnh khi
không). Token mint ra gán vào `RequestContext::forwarded_bearer_token` — đúng cơ chế
`GrpcBackend::signed_request` đã ưu tiên sẵn (giống hệt cách `waf-graphql-gateway` forward token
thật của caller), chỉ khác là mint mới thay vì forward nguyên bản (vì `zones-service` chưa bao giờ
thấy JWT gốc của caller, gRPC server-side không giữ lại raw token). Bên nhận (`scanning-service`/
`alerting-service`) tự giải mã token này độc lập, ra đúng role/tenant thật của caller — kiểm tra
chạy đúng quyền thật, không phải danh nghĩa 1 service chung.

**Không phụ thuộc boot-time — kết nối lười, chỉ cache khi thành công.** `CrossServiceLink` (mới)
không dial gRPC lúc `zones-service` khởi động — lần xoá `waf.zones` đầu tiên mới thử kết nối; thành
công thì cache dùng lại, thất bại thì **không cache** (lần xoá kế tiếp tự thử lại, không cần
restart). Nhờ vậy `zones-service` khởi động hoàn toàn không cần 2 service kia đã chạy — giữ đúng
tính chất "độc lập triển khai". Cái giá: pha nào 2 service kia đang down, mọi lệnh xoá Zone bị chặn
`503 reference_check_unavailable` — y hệt triết lý guard REST cũ ("không xác minh được = chặn, thà
chặn nhầm còn hơn để lọt 1 bản ghi mồ côi").

### Việc đã làm

- `guarded_backend.rs`: thêm `CrossServiceLink` (lazy-connect, cache khi thành công), hàm
  `mint_token` (dispatch JWKS/RSA nội bộ), `zone_referenced_by` (mint token → check
  `waf.scan_jobs` ở scanning, `waf.incidents`+`waf.security_events` ở alerting). `delete()` cho
  `waf.zones` giờ gọi hàm này trước khi giao cho `inner.delete`.
- `main.rs`: thêm `SCANNING_GRPC_ADDR`/`ALERTING_GRPC_ADDR` (mặc định `:3011`/`:3021`, đúng port
  gRPC chuẩn của 2 service kia), truyền `state.token_signer`/`state.jwt_encoding_key_pem` vào
  `GuardedZonesBackend::new` — không cần credential/service-account mới nào cả.
- Test mới trong `zone_domain_guard_postgres.rs`:
  `guarded_backend_delete_blocks_when_cross_service_check_is_unreachable` — trỏ cả 2 addr vào cổng
  không ai nghe (`127.0.0.1:1`), xác nhận `503 reference_check_unavailable` và **zone không bị xoá
  thật** (query DB trực tiếp xác nhận).

### Phát hiện phụ khi verify sống: `scanning-service`'s `.env` cục bộ tắt `GRPC_ENABLED`

Không phải bug code — `.env` (gitignored, không commit) của `scanning-service` trong máy này đang
`GRPC_ENABLED=false` (giá trị mặc định của chính `.env.example`, cả 3 service đều mặc định `false`
— cờ opt-in, không phải bug), nên suốt các phase 98/99 gateway luôn báo `scanning: degraded`. Bật
`true` để verify sống phase này — giải thích luôn thắc mắc "chưa điều tra" mà Phase 98 để lại; đây
không phải regression, chỉ là 1 cờ opt-in địa phương chưa bật.

### Verify sống — đầy đủ, qua gateway thật

Với `zones-service` + `scanning-service` (GRPC_ENABLED=true) + `alerting-service` +
`waf-graphql-gateway` chạy thật:

1. Tạo Zone A (không tham chiếu gì) → `deleteWafZones` → **thành công**. Log xác nhận
   `GrpcRecordService` ở cả `scanning-service` (`List waf.scan_jobs`) và `alerting-service` (`List
   waf.incidents`, `List waf.security_events`) đều nhận đúng request trước khi `zones-service` tự
   xoá.
2. Tạo Zone B, tạo `ScanJob` trỏ `zoneId` vào Zone B (qua `createWafScanJobs`, cũng qua gateway) →
   `deleteWafZones(Zone B)` → **bị chặn**, lỗi GraphQL đúng shape cũ:
   `{"code":"record_referenced","status":409}`, message `"This zone is still referenced by
   \"waf.scan_jobs\" and cannot be deleted."`.
3. Xoá `ScanJob` đó → thử xoá lại Zone B → **thành công** — xác nhận check là động (theo dữ liệu
   thật tại thời điểm gọi), không phải cache sai hay false-positive vĩnh viễn.

`cargo build/clippy(-D warnings)/fmt --check/test --workspace` sạch. Dọn dữ liệu test sau khi
verify, tắt hết service.

### Còn nợ

- Không còn gap kiến trúc nào ở `zone_delete_guard` — cả 4 guard cũ giờ đều chạy đúng qua đường
  gRPC/GraphQL duy nhất còn tồn tại.
- Chưa có test tự động (`#[ignore]` e2e) cho chính kịch bản "bị chặn đúng vì có tham chiếu thật" —
  test mới thêm ở Phase này chỉ phủ nhánh "unreachable → chặn", không phủ nhánh "reachable nhưng có
  tham chiếu thật → chặn" (nhánh đó chỉ mới verify sống thủ công ở trên, cần cả 3 service +
  Postgres cùng lúc, phức tạp hơn 1 test `#[ignore]` đơn giản — có thể làm sau nếu cần coverage tự
  động cho chính xác nhánh này).
