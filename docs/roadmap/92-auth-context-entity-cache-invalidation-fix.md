## Phase 92: Đóng gap `AUTH_CONTEXT_ENTITY` cache-invalidation Phase 90 để lại (2026-09-22)

Trigger: chủ dự án — "fix nợ kỹ thuật" sau khi rà lại roadmap. Đây là gap đã ghi nhận rõ trong
`metap/CLAUDE.md`'s note kèm Phase 90 (bỏ REST entity CRUD): `invalidate_context_cache_if_auth_context_entity`
(tự động invalidate `ContextAttributesCache` khi record `AUTH_CONTEXT_ENTITY` được update) chỉ
từng tồn tại trong REST handler `routes::records::update_record` đã bị xoá — GraphQL/gRPC update
chưa bao giờ có side effect này, nên từ Phase 90 gap này trở thành "unconditional" (không transport
nào có, thay vì chỉ REST có).

### Rà soát nợ kỹ thuật khác cùng phiên, kết quả

- **`migrate_postgres.rs`'s 2 test còn trỏ bảng `records` đã xoá** (Phase 89 flag) — rà lại thấy
  **đã được fix từ trước** (commit `fb72c80`, "fix stale records-table e2e fixture", đã nằm trên
  `master` trước khi phiên này bắt đầu — Phase 89 doc bị stale, không phải nợ thật). Verify lại
  sống: cả 2 test pass trên Postgres thật.
- **`metap-reconciler::introspect()` nên re-derive mọi tín hiệu hội tụ từ `pg_catalog` hay giữ
  ledger** — vẫn là câu hỏi kiến trúc mở, chưa chốt hướng, không tự quyết trong phiên này (đúng
  convention "don't resolve unilaterally" đã áp dụng nhiều lần trong repo).

### Thiết kế fix (ở `metap-crud`, không phải per-transport)

Sửa tận gốc trong `CrudService` (`crates/metap-crud/src/crud_service.rs`/`crud_service/update.rs`)
thay vì thêm lại vào riêng REST — vì GraphQL/gRPC/REST-tương-lai đều đi qua đúng 1 `update()`
này, sửa 1 chỗ đóng gap cho mọi transport cùng lúc, đúng tinh thần "permission/validation không
được lệch giữa transport" đã có sẵn trong platform.

- `CrudService` thêm field opt-in `context_invalidation: Option<ContextInvalidationConfig>` —
  `None` qua `new()`/`with_audit()` (không đổi hành vi deployment hiện có), và 1 method mới
  **chainable** `with_context_invalidation(entity_name, cache)` — cố ý không làm constructor thứ
  3/4 cố định như `with_audit` vì đây là 2 opt-in độc lập, chain lên cả `new()` lẫn `with_audit()`
  đều được, tránh nổ tổ hợp constructor.
- `update()` gọi `invalidate_context_cache_if_configured` ngay sau commit (cùng vị trí gọi với
  `record_audit` đã có) — nếu entity vừa update khớp `entity_name` đã cấu hình, đọc `userId` từ
  **payload ghi đầy đủ, chưa mask theo quyền** (không phải từ `RecordDto` trả về đã mask — actor
  không có quyền đọc field `userId` vẫn phải kích hoạt invalidate đúng, không được âm thầm bỏ
  qua), rồi gọi `cache.invalidate(tenant_id, user_id)` — đúng `ContextAttributesCache` instance
  `AuthContext`/`resolve_request_context` đang đọc (caller truyền
  `AppState.context_attributes_cache.clone()` — rẻ, share chung `moka` store bên dưới).
- **Cố ý chưa wire vào `main.rs` của bất kỳ downstream nào** — hiện chưa app nào trong tổ chức
  thật sự set `AUTH_CONTEXT_ENTITY` (gap trước đây là "dormant everywhere it's wired"), nên phase
  này chỉ đóng gap ở tầng thư viện, không tự quyết định bật tính năng cho ai. App nào cấu hình
  `AUTH_CONTEXT_ENTITY` thật thì tự chain thêm `.with_context_invalidation(...)` khi build
  `CrudService` của mình.
- **Giữ đúng phạm vi REST handler cũ** — chỉ `update()` có side effect này, `delete()` một record
  `AUTH_CONTEXT_ENTITY` vẫn để cache cũ tồn tại tới hết TTL, y hệt hành vi trước Phase 90 (không
  phải gap mới phát sinh từ phase này).

### Verify

`cargo build/clippy(-D warnings)/fmt --check/test --workspace` sạch (0 fail, kể cả 2 benchmark
`sustained_concurrent_list_*` fail vì cần seed script ngoài — đã biết từ trước, không liên quan).
E2e mới (`crates/metap-crud/tests/crud_service_postgres.rs`,
`updating_the_configured_auth_context_entity_invalidates_only_the_affected_users_cache_entry`,
`--ignored`, Postgres thật): update record `AUTH_CONTEXT_ENTITY` → đúng cache entry của user bị
ảnh hưởng bị invalidate (đếm số lần fetch closure thật sự chạy lại, không đoán qua nội bộ cache),
cache entry của 1 user khác không liên quan hoàn toàn không bị đụng tới.

### Còn nợ / không làm trong phase này

- `delete()` không invalidate — như đã nêu, giữ đúng parity với REST handler cũ, không phải bug
  mới.
- Chưa wire feature này vào bất kỳ `main.rs` thật nào (`metap-demo-waf`, `metap-demo-jira`,
  `metap-lowcode`, `templates/metap-app`) — chờ app nào thật sự cần `AUTH_CONTEXT_ENTITY`.
- Câu hỏi kiến trúc `metap-reconciler::introspect()` (re-derive từ `pg_catalog` vs giữ ledger) vẫn
  mở, chưa chốt hướng.
