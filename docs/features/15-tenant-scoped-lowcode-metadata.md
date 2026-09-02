# Metadata low-code theo từng Tenant

- **Trạng thái:** proposed — chưa có trigger
- **Người đề xuất:** ghi lại 2026-08-22 từ thảo luận sau khi Phase 18 xong,
  `docs/team-charter.md` ("Định hướng đang ghi nhận, chưa có trigger" #8) — chốt hướng dài hạn cho
  Phase 11C's "quy tắc cô lập schema cấp Tenant" (Phase 11, `docs/roadmap.md`)
- **Track sở hữu:** Backend Core
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
- **Storage**: `low_code_entity_drafts`/`low_code_entity_versions`
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
- **`metap-lowcode-http` hôm nay dùng thẳng `state.pool`, không qua `Router`** — đúng cho thế giới
  global hôm nay. Nếu metadata theo tenant, tái hiện đúng loại gap Phase 16 đã đóng 1 lần cho role
  lookup — mọi handler cần đi qua `Router::begin(tenant_id)`, và câu hỏi thiết kế chưa trả lời:
  bảng metadata theo tenant nên sống ở đâu (cùng chỗ `Router` route dữ liệu tenant đó, hay tập
  trung 1 bảng ở control-plane DB kèm cột `tenant_id`).
- **Blast radius ra ngoài backend**: `GET /metadata/openapi.json` hôm nay là 1 schema toàn cục,
  public — pipeline codegen FE (`pnpm generate:types`) giả định đúng 1 schema duy nhất. Theo tenant
  nghĩa là endpoint này cần biết "hỏi cho tenant nào", và bước codegen (chạy lúc dev) cần câu trả
  lời riêng — chưa nghĩ tới.

## Phạm vi

**Trong phạm vi (nếu được kích hoạt):** chưa chốt — 4 điểm rà soát ở trên là điều kiện tiên quyết
phải giải trước khi viết spec chi tiết.

**Ngoài phạm vi:** đảo entity code-authored (`../metap-demo-crm/src/entities/*.rs`) sang theo
tenant — chỉ tầng DB-authored đổi.

## Tiêu chí chấp nhận

<Chưa xác định.>

## Ranh giới kiến trúc bị đụng tới

`metap-lowcode`/`metap-lowcode-http` (storage + registry resolution), `metap-http::AppState`
(registry theo tenant thay vì global), pipeline codegen frontend (`platform-ui generate:types`).
Cần ADR khi có trigger — đây là thay đổi giả định nền tảng ("1 registry toàn cục"), không phải
tính năng cộng thêm.

## Rủi ro / phụ thuộc

- **Trigger: chưa có.** Chưa tenant nào trong repo cần entity/field khác shape so với tenant khác.
- Registry resolution (cache theo tenant, invalidate-on-write) là phần rủi ro kỹ thuật cao nhất —
  cần thiết kế cẩn thận trước khi code, không phải mở rộng đơn giản từ cơ chế cache có sẵn.
