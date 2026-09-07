# Metadata low-code theo từng Tenant

- **Trạng thái:** approved — trigger chốt 2026-09-06, **thiết kế chốt 2026-09-07** (cả 2 câu hỏi mở
  đã đóng, xem "Thiết kế đã chốt" dưới + ADR mới trong `docs/architectures/09-adr/00-index.md`).
  Storage đã chốt (control-plane DB tập trung); `metap-lowcode-http` routing hoá ra không cần đổi
  gì (ghi chú cũ 2026-08-22 đã stale, xem "Rà soát" dưới). Sẵn sàng để code — chưa code.
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
- **Registry resolution — phần khó thật, ĐÃ CHỐT 2026-09-07** (xem "Thiết kế đã chốt" dưới + ADR).
  `AppState.metadata` hôm nay là MỘT `Arc<ArcSwap<MetadataRegistry>>` toàn cục — giữ nguyên,
  không đổi type; thêm `MetadataResolver` trait (`metap-metadata`) + field mới
  `metadata_resolver: Option<...>` trên `AppState`, `None` thì hành vi y hệt hôm nay. `metap-lowcode`
  implement resolver bằng cache theo tenant (cùng mẫu `RegistryCache`/`ContextAttributesCache` đã
  có 2 lần trong repo) — invalidate **explicit-trên-ghi là chính, TTL chỉ backstop**, khác
  `RegistryCache` (TTL-only) vì `publish`/`rollback`/`set_enabled` đã đi qua đúng 1 code path, gọi
  thẳng `.warm(tenant_id, registry)` bằng chính registry vừa merge (không rebuild lần 2) ngay tại đó
  hợp lý hơn chấp nhận độ trễ TTL.
- **Ghi chú stale đã sửa (2026-09-06)**: mục này lúc viết (2026-08-22) giả định
  `metap-lowcode-http` còn đọc thẳng `state.pool` cho *data* — không đúng nữa từ 2026-08-25 (mọi
  handler đã đi qua `Router::pool_for(caller's tenant)` qua `resolve_pool`, đúng loại gap Phase 16
  đã đóng 1 lần cho role lookup). Không liên quan trực tiếp: quyết định storage ở trên (metadata
  tập trung control-plane DB, không theo `Router`) nghĩa là *metadata* handler không cần đổi gì
  thêm ở khâu này — chỉ cần thêm `tenant_id` vào query/insert, vẫn dùng `state.pool` thẳng như
  bản chất "metadata tập trung" đã chọn, không cần `Router::begin`. `resolve_pool`'s cách route
  *data* theo tenant giữ nguyên không đổi.
- **Blast radius ra ngoài backend — ĐÃ CHỐT 2026-09-07.** `GET /metadata/openapi.json` (public,
  không `AuthContext`, không có tenant context nào để hỏi) **giữ nguyên vĩnh viễn: chỉ phản ánh
  `metadata_base`** (code-authored), không bao giờ trả entity low-code của bất kỳ tenant nào —
  quyết định sản phẩm chủ động, không phải giới hạn kỹ thuật (chốt trực tiếp bởi chủ dự án). Lý do
  chấp nhận được: pipeline codegen FE (`platform-ui generate:types`) chỉ đọc schema `EntitySummary`
  chung (`components.schemas`), chưa bao giờ đọc danh sách entity/paths cụ thể của một tenant —
  xem `metap/CLAUDE.md`'s "Metadata types stay generated, not hand-written". App nào thật sự cần
  schema đầy đủ (kể cả entity low-code riêng của tenant mình) dùng `GET /metadata/entities` — route
  này đã có `AuthContext` (tenant thật từ token) từ trước, chỉ cần đổi từ `state.metadata.load()`
  sang `state.metadata_for(context.tenant_id())`.

## Thiết kế đã chốt (2026-09-07)

Bản đầy đủ (lý do *tại sao*) nằm ở ADR mới trong `docs/architectures/09-adr/00-index.md`; mục này
là bản phác thảo mức code-signature để bắt tay vào code trực tiếp, không phải quyết định thêm.

**`metap-metadata`** (mới):
```rust
#[async_trait]
pub trait MetadataResolver: Send + Sync {
    async fn resolve(&self, tenant_id: Uuid) -> anyhow::Result<Arc<MetadataRegistry>>;
    async fn invalidate(&self, tenant_id: Uuid);
}
```

**`metap-http::AppState`** — thêm, không đổi field cũ:
```rust
pub metadata_resolver: Option<Arc<dyn MetadataResolver>>,  // None = mọi app hôm nay, không đổi gì

impl AppState {
    pub async fn metadata_for(&self, tenant_id: Uuid) -> anyhow::Result<Arc<MetadataRegistry>> {
        match &self.metadata_resolver {
            Some(resolver) => resolver.resolve(tenant_id).await,
            None => Ok(self.metadata.load_full()),
        }
    }
}
```
Đổi cơ học tại 9 call site hiện có (`metap-crud` x4, `metap-graphql-http` x3, `metap-http/routes/
metadata.rs`'s `list_entities`/`get_entity` x2) từ `state.metadata.load()`/`load_full()` sang
`state.metadata_for(tenant_id).await?` — tenant lấy từ `AuthContext`/`RequestContext` đã có sẵn ở
mọi call site đó, không cần cơ chế mới. `openapi_json` (public route) **không đổi** — tiếp tục đọc
thẳng `state.metadata.load()`, không bao giờ qua resolver (xem "Blast radius" trên).

`metap-graphql-http::SchemaHolder` (cache 1 `(Arc<MetadataRegistry>, Arc<Schema>)`, rebuild lazy
khi `Arc::ptr_eq` lệch) đổi thành cache theo `tenant_id` (`moka` hoặc `HashMap` trong `Mutex` tuỳ
tần suất tenant thay đổi thực tế) — cùng nguyên tắc, chỉ thêm chiều tenant.

**`metap-lowcode`** — implement `MetadataResolver`:
```rust
pub struct TenantMetadataResolver {
    pool: PgPool,
    metadata_base: Arc<MetadataRegistry>,
    cache: moka::future::Cache<Uuid, Arc<MetadataRegistry>>,  // TTL backstop 30s, giống RegistryCache
}

impl TenantMetadataResolver {
    /// Gọi từ `apply_registry` (publish/rollback/set_enabled) ngay sau khi merge — nạp thẳng
    /// registry vừa tính, không đợi lần đọc sau rebuild lại.
    pub async fn warm(&self, tenant_id: Uuid, registry: Arc<MetadataRegistry>) {
        self.cache.insert(tenant_id, registry).await;
    }
}

#[async_trait]
impl MetadataResolver for TenantMetadataResolver {
    async fn resolve(&self, tenant_id: Uuid) -> anyhow::Result<Arc<MetadataRegistry>> {
        self.cache
            .try_get_with(tenant_id, async {
                let extra = store::list_enabled_published_for_tenant(&self.pool, tenant_id).await?;
                Ok::<_, anyhow::Error>(Arc::new(self.metadata_base.merge_with(extra)?))
            })
            .await
            .map_err(|e| anyhow::anyhow!(e))
    }
    async fn invalidate(&self, tenant_id: Uuid) {
        self.cache.invalidate(&tenant_id).await;
    }
}
```

**`metap-lowcode::store`** — mọi hàm public thêm tham số `tenant_id: Uuid` đầu tiên, khoá chính
`low_code_entity_drafts`/`low_code_entity_versions` đổi `entity_name` → `(tenant_id, entity_name)`
(`crates/migrations/0010_low_code_entities.sql`, migration mới thêm cột — cơ học, không cần thiết
kế thêm, như "Rà soát" đã ghi). `list_enabled_published` → `list_enabled_published_for_tenant(pool,
tenant_id)`.

**HTTP layer** (`metap-lowcode-http`): mọi route publish/rollback/draft/... lấy `tenant_id` từ
`AuthContext`/`AdminContext` đã có (không phải request body) — cùng kỷ luật "tenant luôn từ token,
không bao giờ từ request" `metap-config`/`metap-control` đã áp dụng.

## Phạm vi

**Trong phạm vi:** storage (đã chốt 2026-09-06) + registry resolution + blast radius FE codegen
(cả 2 chốt 2026-09-07, xem "Thiết kế đã chốt" trên) — đủ để bắt tay code trực tiếp, không còn câu
hỏi thiết kế mở nào chặn việc code.

**Ngoài phạm vi:** đảo entity code-authored (`../metap-demo-crm/src/entities/*.rs`) sang theo
tenant — chỉ tầng DB-authored đổi.

## Tiêu chí chấp nhận

- 2 tenant khác nhau publish 2 entity low-code cùng tên nhưng khác field shape — cả hai đều list/
  get/create/update đúng theo shape riêng của tenant mình qua REST (`/api/:entity*`) lẫn GraphQL
  (`metap-graphql-http`), không tenant nào thấy field/entity của tenant kia.
- `GET /metadata/entities` (đã `AuthContext`) trả đúng danh sách entity (code-authored + low-code)
  của tenant gọi, không lẫn entity low-code của tenant khác.
- `GET /metadata/openapi.json` (public) không bao giờ chứa entity low-code của bất kỳ tenant nào,
  trước lẫn sau khi tính năng này ship — verify bằng cách publish 1 entity low-code cho 1 tenant và
  confirm response của route này không đổi.
- Publish/rollback cho tenant A xong, request **ngay sau đó** (không đợi TTL) của tenant A đã thấy
  registry mới — verify tính đúng của `.warm()` (invalidate-on-write là chính, không dựa vào TTL).
- Mọi app không set `metadata_resolver` (`../metap-demo-crm`, `../metap-demo-jira`,
  `../metap-demo-waf`) build/chạy y hệt trước khi có thay đổi này — verify bằng
  `cargo build --workspace`/e2e test hiện có ở cả 3 repo không cần sửa gì.

## Ranh giới kiến trúc bị đụng tới

`metap-metadata` (trait `MetadataResolver` mới), `metap-http::AppState` (field + method mới,
type cũ không đổi — đây là phần đụng tới `metap` core thật, dù chỉ ở chỗ resolution logic, không
phải business-entity knowledge nào), `metap-graphql-http::SchemaHolder` (thêm chiều tenant),
`metap-lowcode`/`metap-lowcode-http` (nơi thực thi `MetadataResolver` + storage theo tenant),
pipeline codegen frontend (`platform-ui generate:types` — không đổi gì, xác nhận ở "Blast radius"
trên). **ADR đã viết** (`docs/architectures/09-adr/00-index.md`, mục "Metadata registry resolution
trở thành pluggable-theo-tenant") — không còn là thay đổi giả định nền tảng chưa ghi lại lý do.

## Rủi ro / phụ thuộc

- **Trigger: đã có (2026-09-06).** SaaS pitch của `metap-lowcode` cần tenant tự tuỳ biến schema —
  chủ dự án xác nhận trực tiếp, không phải suy đoán.
- **Thiết kế đã chốt (2026-09-07)** — registry resolution và blast radius FE codegen không còn là
  rủi ro thiết kế mở; rủi ro còn lại là rủi ro *thực thi* bình thường (diff cơ học lớn ở
  `metap-lowcode::store`, cẩn thận không bỏ sót call site nào của `state.metadata.load()`).
- Giới hạn multi-instance đã biết trước, chấp nhận giống `RegistryCache`/`metap-config`'s tenant
  tier: 1 instance publish, instance khác chỉ thấy sau TTL của chính nó (30s) — không có invalidate
  xuyên instance. Ghi rõ trong code khi implement, không phải bug cần né.
