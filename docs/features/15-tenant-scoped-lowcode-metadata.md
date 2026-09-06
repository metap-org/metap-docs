# Metadata low-code theo từng Tenant

- **Trạng thái:** approved — trigger chốt 2026-09-06. Storage đã chốt (control-plane DB tập
  trung); `metap-lowcode-http` routing hoá ra không cần đổi gì (ghi chú cũ 2026-08-22 đã stale, xem
  "Rà soát" dưới). Còn 2 câu hỏi thật chưa trả lời: **registry resolution** (cache/invalidate theo
  tenant — phần khó thật) và **blast radius FE codegen** (`GET /metadata/openapi.json` theo
  tenant). Chưa code.
- **Người đề xuất:** ghi lại 2026-08-22 từ thảo luận sau khi Phase 18 xong,
  `docs/team-charter.md` ("Định hướng đang ghi nhận, chưa có trigger" #8) — chốt hướng dài hạn cho
  Phase 11C's "quy tắc cô lập schema cấp Tenant" (Phase 11, `docs/roadmap.md`); **kích hoạt
  2026-09-06** — chủ dự án chốt trực tiếp "nhất định phải làm với `metap-lowcode`" (SaaS pitch cần
  mỗi tenant tự tuỳ biến schema, không chỉ dữ liệu riêng).
- **Track sở hữu:** Backend Core (nhưng **thực thi ở `../metap-lowcode`**, không phải `metap`
  core — đúng ranh giới đã có từ Phase 52: `metap-lowcode` là nơi mọi khả năng DB-authored/
  low-code sống, `metap` core không biết khái niệm "tenant tự định nghĩa entity")
- **Phase roadmap liên quan:** không thuộc phase nào

## Vấn đề / động lực

Hôm nay mọi tenant dùng chung 1 tập entity DB-authored (low-code) toàn cục. Ý này: tenant tự định
nghĩa entity/field **riêng của mình**, khác shape với tenant khác — hướng dài hạn nếu SaaS pitch
của `metap-lowcode` cần "mỗi tenant tự tuỳ biến schema", không chỉ tự nhập dữ liệu riêng (tầng dữ
liệu — bảng `records` JSONB — đã cho mỗi tenant dữ liệu hoàn toàn riêng rồi, đây là về tầng
**shape**).

## Rà soát trước khi thiết kế (đã làm 2026-08-22, không suy đoán)

- **Chỉ tầng DB-authored (`metap-lowcode`) đổi, code-authored giữ nguyên global.** Quyết định
  Phase A (`docs/low-code-metadata-storage-design.md`) đã tự đóng khung đúng ranh giới này — không
  cần đảo entity code-authored, chỉ mở rộng cơ chế `merge_with` (base cố định + extra) đã có, đổi
  "extra" từ 1 tập entity toàn cục thành 1 tập theo từng tenant.
- **Storage — CHỐT 2026-09-06**: tập trung ở control-plane DB (`DATABASE_URL` chung hôm nay dùng
  cho `control.tenants`), **không** đi theo `Router::begin(tenant_id)` sang DB riêng của
  `DedicatedDb` tenant. Lý do chọn: tách bạch rõ "nơi định nghĩa schema" (control-plane, tập
  trung) khỏi "nơi lưu data" (theo tenant, có thể DB riêng) — đơn giản hơn hẳn phương án còn lại
  (metadata sống cùng DB data của tenant), không cần đổi `metap-lowcode-http`'s cách lấy pool
  (`state.pool` thẳng, không qua `Router`) để đọc/ghi *metadata* (đọc/ghi *data* của record theo
  entity đó vẫn qua `Router::pool_for` như hôm nay, không đổi — 2 việc khác nhau). Cụ thể:
  `low_code_entity_drafts`/`low_code_entity_versions`
  (`crates/migrations/0010_low_code_entities.sql`) thêm cột `tenant_id`, khoá chính đổi từ
  `entity_name` sang `(tenant_id, entity_name)` — kéo theo sửa mọi hàm public của
  `metap-lowcode::store` (draft/publish/rollback/list/export/import), diff cơ học không nhỏ.
- **Registry resolution — phần khó thật.** `AppState.metadata` hôm nay là MỘT
  `Arc<ArcSwap<MetadataRegistry>>` toàn cục. Theo tenant nghĩa là mỗi tenant cần registry riêng —
  build tươi mỗi request quá đắt (1 lần merge phải validate lại toàn bộ field/list-view của mọi
  entity tenant đó). Hướng hợp lý hơn: cache theo tenant (cùng mẫu `RegistryCache`/
  `ContextAttributesCache` đã có 2 lần trong repo) — nhưng invalidate nên **explicit-trên-ghi là
  chính, TTL chỉ backstop**, khác `ContextAttributesCache` (TTL là chính) vì `publish`/`rollback`
  đã đi qua đúng 1 code path, gọi `.invalidate(tenant_id)` ngay tại đó hợp lý hơn chấp nhận độ trễ
  TTL.
- **Ghi chú stale đã sửa (2026-09-06)**: mục này lúc viết (2026-08-22) giả định
  `metap-lowcode-http` còn đọc thẳng `state.pool` cho *data* — không đúng nữa từ 2026-08-25 (mọi
  handler đã đi qua `Router::pool_for(caller's tenant)` qua `resolve_pool`, đúng loại gap Phase 16
  đã đóng 1 lần cho role lookup). Không liên quan trực tiếp: quyết định storage ở trên (metadata
  tập trung control-plane DB, không theo `Router`) nghĩa là *metadata* handler không cần đổi gì
  thêm ở khâu này — chỉ cần thêm `tenant_id` vào query/insert, vẫn dùng `state.pool` thẳng như
  bản chất "metadata tập trung" đã chọn, không cần `Router::begin`. `resolve_pool`'s cách route
  *data* theo tenant giữ nguyên không đổi.
- **Blast radius ra ngoài backend**: `GET /metadata/openapi.json` hôm nay là 1 schema toàn cục,
  public — pipeline codegen FE (`pnpm generate:types`) giả định đúng 1 schema duy nhất. Theo tenant
  nghĩa là endpoint này cần biết "hỏi cho tenant nào", và bước codegen (chạy lúc dev) cần câu trả
  lời riêng — chưa nghĩ tới.

## Phạm vi

**Trong phạm vi:** đã kích hoạt (2026-09-06) — storage đã chốt (xem "Rà soát" trên). Còn thiếu
thiết kế chi tiết cho registry resolution (cache/invalidate theo tenant) và blast radius FE
codegen trước khi viết spec code-level đầy đủ; migration DB (`0010_low_code_entities.sql` thêm
`tenant_id`) và sửa hàm public của `metap-lowcode::store` là việc cơ học, không cần thiết kế
thêm.

**Ngoài phạm vi:** đảo entity code-authored (`../metap-demo-crm/src/entities/*.rs`) sang theo
tenant — chỉ tầng DB-authored đổi.

## Tiêu chí chấp nhận

<Chưa xác định.>

## Ranh giới kiến trúc bị đụng tới

`metap-lowcode`/`metap-lowcode-http` (storage — đã chốt + registry resolution — chưa chốt),
`metap-http::AppState` (registry theo tenant thay vì global — đây là phần đụng tới `metap` core
thật, dù chỉ ở chỗ `AppState.metadata`'s type/resolution logic, không phải business-entity
knowledge nào), pipeline codegen frontend (`platform-ui generate:types`). Cần ADR trước khi code
phần registry resolution — đây là thay đổi giả định nền tảng ("1 registry toàn cục"), không phải
tính năng cộng thêm; storage (đã chốt) không cần ADR vì không đổi giả định nền tảng nào, chỉ thêm
cột.

## Rủi ro / phụ thuộc

- **Trigger: đã có (2026-09-06).** SaaS pitch của `metap-lowcode` cần tenant tự tuỳ biến schema —
  chủ dự án xác nhận trực tiếp, không phải suy đoán.
- Registry resolution (cache theo tenant, invalidate-on-write) vẫn là phần rủi ro kỹ thuật cao
  nhất, **chưa chốt** — cần thiết kế cẩn thận trước khi code, không phải mở rộng đơn giản từ cơ
  chế cache có sẵn (`RegistryCache`/`ContextAttributesCache`).
- Blast radius FE codegen (`GET /metadata/openapi.json` theo tenant) cũng chưa chốt.
