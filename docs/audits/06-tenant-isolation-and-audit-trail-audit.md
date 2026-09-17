# Audit 06 — Hệ quả chưa đóng của per-tenant schema isolation, và bypass field-permission qua audit trail

> **Phạm vi (2026-09-17):** audit sweep có hệ thống (1 agent Opus), tập trung vào 2 vùng chưa từng
> được audit lần nào và đều vừa ship gần đây: **(a)** per-tenant schema isolation
> (`docs/features/35-per-tenant-schema-isolation.md`, ship 2026-09-11) và **(b)** crate `metap-audit`
> (audit 05, ship 2026-09-13). Chọn 2 vùng này vì audit 02/03/04 đều chạy *trước* khi cả hai tồn
> tại, còn audit 05 không phải sweep. Có rà thêm bề mặt sinh SQL (`metap-query`'s `aggregate`/`jql`)
> và tàn dư sau Phase 86 — cả hai **sạch**, ghi lại ở mục "Đã kiểm tra, không có vấn đề".
> **Chưa code gì cả** — đúng shape của `04-*.md`/`05-*.md`.

## Tóm tắt

Ba finding HIGH. Hai trong số đó (#1, #2) **cùng một nguyên nhân gốc**: feature 35 đã đổi một bất
biến toàn hệ thống — *"`schema_name` của tenant `Schema`-strategy luôn là `"public"`"* — nhưng ít
nhất 3 chỗ khác trong codebase vẫn đang dựa vào bất biến cũ đó, trong đó có 2 chỗ dựa vào nó **như
một lập luận bảo mật/thiết kế được ghi thành doc**. Cả 2 đều **chưa nổ**, vì chưa tenant nào được
provision lại sau 2026-09-11 (mọi tenant `Schema` hiện tại vẫn `schema_name="public"`, xác nhận sống
2026-09-08) — chúng sẽ nổ ở **lần gọi `provision_schema_tenant` tiếp theo**.

| # | Mức độ | Vấn đề |
|---|---|---|
| 1 | **HIGH** | `Router::pool_for` trả pool dùng chung, không set `search_path` — mọi tenant có schema riêng bị phục vụ sai schema. Không chỉ 3 call site `dev-tools` như doc khẳng định: `metap-lowcode` có 5 call site nữa, 1 trong đó nằm trên **mọi HTTP handler** |
| 2 | **HIGH** | `POST /auth/login` không kèm `tenantId` tra `metadata.users`, nhưng `provision_schema_tenant` ghi admin mới vào `t_<uuid>.users` → **tenant vừa provision không đăng nhập được**. Đồng thời `users_email_unique` (audit 04 A#2 xác nhận là "đang chịu lực") bị clone thành unique *per-schema*, phá luôn tính toàn cục mà chính lập luận đó dựa vào |
| 3 | **HIGH** | `GET /api/{entity}/{id}/audit-events` trả nguyên `diff` không mask field → **bypass field-level permission**: ai đọc được record thì đọc được giá trị mọi field trong toàn bộ lịch sử, kể cả field bị mask ở đường đọc thường |
| 4 | MEDIUM | (Nhắc lại, chưa fix) Finding thứ 9 của `metap-demo-waf` — `backfill::run_batched_update` scope theo `tenant_id` sentinel → backfill boot-time chạm 0 dòng rồi vẫn `mark_completed` |
| 5 | LOW | Doc drift chịu lực: cả `CLAUDE.md` lẫn doc comment của `pool_for` khẳng định sai về hiện trạng, và chính khẳng định sai đó là lý do finding #1 chưa được fix |

---

## 1. HIGH — `Router::pool_for` bỏ qua schema của tenant

**Bằng chứng** (`crates/metap-control/src/router.rs`):

```rust
pub async fn pool_for(&self, tenant: TenantId) -> anyhow::Result<PgPool> {
    match self.resolve(tenant).await? {
        TenantStrategy::Schema { schema_name } => {
            validate_schema_name(&schema_name)?;   // ← validate rồi VỨT ĐI
            Ok(self.shared_pool.clone())            // ← không SET search_path
        }
```

`schema_name` được validate xong rồi không dùng vào việc gì — đây là code smell đủ rõ để review
bắt được. Pool trả về mang `search_path` mặc định của database (`public, metadata, control`, do
`0028_metadata_schema.sql:37` đặt), nên mọi bảng per-tenant clone trong `t_<uuid>` là **không bao
giờ với tới được** qua đường này.

**Doc comment của chính hàm này tự biện minh bằng một tiền đề nay đã sai**:

> "real per-tenant schema isolation isn't built yet (`provisioning.rs`'s doc comment: `schema_name`
> is always `"public"` in practice)"

Nhưng `crates/metap-control/src/provisioning.rs:56` bây giờ là:

```rust
let schema_name = format!("t_{}", tenant_id.simple());
```

**Blast radius — doc nói sai chỗ quan trọng nhất.** `metap/CLAUDE.md` khẳng định:

> "its only 3 callers are `dev-tools` CLI paths, never the real HTTP/`CrudService` path"

Sai. Khẳng định đó chỉ đúng *trong repo `metap`*. Thực tế còn 5 call site nữa ở `metap-lowcode`:

| Call site | Tính chất |
|---|---|
| `crates/presenter/src/lib.rs:87` (`resolve_pool`) | **Mọi HTTP handler của low-code đều đi qua** — doc của chính file đó viết: "every handler resolves `Router::pool_for(caller's tenant)` (`resolve_pool`) before…" |
| `services/reconciler-orchestrator/src/lib.rs:170/194/287` | DDL fleet-wide cho entity low-code đã publish |

**Hệ quả thật**: với tenant có schema riêng, `CrudService` (qua `Router::begin`, set `search_path`
đúng) đọc/ghi `t_<uuid>.*`, trong khi mọi handler low-code và orchestrator (qua `pool_for`) đọc/ghi
`metadata.*`/`public.*`. **Cùng một tenant, dữ liệu tách đôi theo đường truy cập** — im lặng, không
lỗi. Với orchestrator thì DDL được áp lên bảng sai schema.

Mức độ chính xác cần ghi rõ để không thổi phồng: đây **chưa phải rò rỉ cross-tenant trực tiếp**, vì
các query vẫn còn filter `tenant_id` ở tầng cột. Nhưng toàn bộ lý do tồn tại của feature 35 —
isolation ở tầng schema — bị vô hiệu im lặng trên các đường đó, và việc dữ liệu tách đôi là lỗi
đúng nghĩa.

**Hướng fix đề xuất** (chưa làm, chờ chốt): `pool_for` không thể set `SET LOCAL` (không có
transaction) và cũng không nên `SET` ở mức session trên pool dùng chung (đúng cái Bẫy #1 mà
`begin()` cảnh báo). Hai hướng thật sự khả dĩ, cần chủ dự án chọn:
1. Trả `PoolConnection` đã `SET search_path` thay vì `PgPool` — đổi chữ ký, ép mọi caller giữ
   connection, nhưng đóng hẳn lỗ.
2. Giữ pool riêng theo schema (cùng pattern `dedicated_pools`, `moka` theo `schema_name`), mỗi pool
   có `options().after_connect()` set `search_path` — không đổi chữ ký caller.

Hướng 2 hợp với call site hiện tại hơn (caller đang cần `PgPool` thật để chạy DDL). Không tự chọn.

---

## 2. HIGH — Tenant provision sau feature 35 không đăng nhập được; unique email toàn cục bị phá

Hai mặt của cùng một thay đổi.

**(a) Không đăng nhập được.** `crates/metap-http/src/routes/auth.rs:111-126`:

```rust
let verify_result = match body.tenant_id {
    Some(tenant_id) => { let mut tx = state.router.begin(tenant_id.into())... }  // search_path đúng
    None => local.verify(&state.pool, &body.email, &body.password).await,        // pool chung
};
```

`verify_credentials` (`metap-peripherals/src/auth.rs:152`) dùng tên bảng **không qualify**:
`SELECT id, tenant_id, email, password_hash FROM users WHERE email = $1` — nên nhánh `None` resolve
theo `search_path` mặc định, tức `metadata.users`.

Trong khi đó `provision_schema_tenant` (`provisioning.rs:64-68`) ghi admin mới vào schema riêng:

```rust
sqlx::query(&format!("SET search_path TO \"{schema_name}\", metadata, control"))...
let user = metap_peripherals::create_user(&mut *conn, tenant_id, admin_email, admin_password).await?;
```

→ row nằm ở `t_<uuid>.users`, nhánh `None` không bao giờ thấy → trả `401 invalid_credentials`.

Đây không phải lỗi phụ: audit 04 A#2 (2026-09-13) đã **xác nhận cố ý** giữ unique email toàn cục
với lý do ghi thành văn — *"ràng buộc này đang chịu lực cho `POST /auth/login` (không có tenant
picker)"*. Không có tenant picker nghĩa là nhánh `None` **là đường mặc định**. Vậy tenant provision
theo feature 35 chỉ login được nếu client tự truyền `tenantId` — thứ mà UI hiện không có.

**(b) Unique email toàn cục bị phá.** `tenant_schema.rs` clone bảng bằng
`CREATE TABLE ... (LIKE ... INCLUDING CONSTRAINTS INCLUDING INDEXES)`, và `("metadata", "users")`
nằm trong `TENANT_SCOPED_TABLES`. `INCLUDING INDEXES` copy `users_email_unique` thành một index
**riêng của từng bảng** → hai tenant hoàn toàn có thể cùng giữ `admin@foo.com` ở schema riêng của
mình. Tính toàn cục mà A#2 dựa vào không còn.

**Đáng chú ý về trình tự**: feature 35 ship 2026-09-11, A#2 được xác nhận "cố ý, không đổi"
2026-09-13 — tức lúc xác nhận thì tiền đề đã bị phá 2 ngày trước rồi, không ai để ý. Đây đúng là
loại tương tác chéo giữa 2 feature mà audit sinh ra để bắt.

**Hướng fix**: phụ thuộc câu hỏi sản phẩm chưa chốt — login đa tenant định danh người dùng bằng gì.
Không tự chọn. Tối thiểu, cần chốt trước khi có tenant thật nào được provision lại.

---

## 3. HIGH — `/audit-events` bypass field-level permission

**Bằng chứng**: `crates/metap-crud/src/crud_service/audit_events.rs` —
`list_audit_events` chỉ kiểm tra record-level (ABAC):

```rust
self.check_record_permission(entity_name, record_id, EntityAction::Read, context).await?
...
let events = store.list_for_record(tenant_id, entity_name, record_id).await?;
Ok(ServiceResult::ok(events))   // ← trả thẳng, không mask field
```

Đối chứng — đường đọc thường **có** mask (`crud_service/helpers.rs:430`):

```rust
let filtered_data = snapshot.filter_readable_fields(context, &row.data);
```

Còn phía ghi (`create.rs:128`) đưa nguyên payload vào diff:
`diff: metap_audit::diff_json_objects(&JsonObject::new(), &data)` — mọi field, không lọc.

Route đã mount thật: `crates/metap-http/src/lib.rs:154` →
`GET /api/{entity}/{record_id}/audit-events`.

**Hệ quả thật**: với entity bật `audit.enabled = true`, người dùng bị field policy chặn đọc field X
ở `GET /api/{entity}/{id}` vẫn lấy được **toàn bộ lịch sử giá trị của X** (cả before lẫn after, mọi
lần đổi) qua audit-events. Đây đúng cùng class lỗi với finding #1 của audit 03 (ABAC bị bỏ qua ở
`attachments`/`workflow_events`) — và mỉa mai là doc comment của chính hàm này viện dẫn đúng bản fix
đó làm tiền lệ: họ port đúng tầng *record-level*, nhưng bỏ sót tầng *field-level* mà đường đọc chính
vẫn đang áp.

Điều này cũng **nâng cấp mức độ** cho mục còn treo của audit 05 ("mask field nhạy cảm trước khi ghi
audit — chưa làm"): trước đây là vệ sinh dữ liệu, giờ là lỗ phân quyền đang sống trên route thật.
Cộng thêm ràng buộc "bảng audit cố ý không bao giờ prune", một giá trị nhạy cảm ghi vào là nằm đó
vĩnh viễn.

**Hướng fix**: mask ở đường đọc là bắt buộc và đủ để đóng lỗ phân quyền (áp
`filter_readable_fields` lên từng `diff` theo snapshot của caller). Mask ở đường ghi là câu hỏi riêng
(mask khi ghi thì mất luôn giá trị compliance của audit; không mask thì bảng vĩnh viễn giữ secret) —
cần chủ dự án chốt, không tự quyết.

---

## 4. MEDIUM — (nhắc lại) backfill scope theo tenant sentinel, vẫn chưa fix

Đã ghi đầy đủ ở `metap-demo-waf/CLAUDE.md` (finding thứ 9) và mục 3 của kế hoạch fix ở đó. Xác nhận
lại trong lần audit này là **vẫn còn nguyên**: `crates/metap-reconciler/src/backfill.rs:65` vẫn
`WHERE t.tenant_id = $2`, và cả 3 service của `metap-demo-waf` vẫn reconcile lúc boot bằng
`PLATFORM_TENANT_ID` (`Uuid::nil()`). Đưa vào đây để audit index có một chỗ theo dõi trạng thái
thống nhất, không phải phát hiện mới.

## 5. LOW — Doc drift chịu lực

Không phải "doc hơi cũ" — đây là 2 khẳng định sai đang **được dùng làm lý do không fix** finding #1:

| Chỗ | Khẳng định | Thực tế |
|---|---|---|
| `metap/CLAUDE.md` (bullet `metap-control`) | "its only 3 callers are `dev-tools` CLI paths, never the real HTTP/`CrudService` path" | Còn 5 call site ở `metap-lowcode`, 1 nằm trên mọi HTTP handler |
| `router.rs`, doc comment `pool_for` | "real per-tenant schema isolation isn't built yet… `schema_name` is always `public` in practice" | `provisioning.rs:56` sinh `t_<uuid>` từ 2026-09-11 |

Ngoài ra, mục "Metadata-driven records" trong `metap/CLAUDE.md` mở đầu bằng *"By default, business
records live in one generic `records` table"* ở thì hiện tại, phải đọc hết 2 đoạn nữa mới gặp đính
chính "2 đoạn trên mô tả trạng thái trước thay đổi này". Đúng convention "không rewrite lịch sử",
nhưng với người/agent đọc lướt thì đây là cái bẫy — đề xuất chuyển 2 đoạn đó vào block trích dẫn
lịch sử có nhãn rõ ràng thay vì để ở thì hiện tại. Câu *"A dedicated table always lives in
`entities`"* trong cùng mục cũng đã sai từ Phase 82 (giờ là `waf`/`crm`/`jira`).

---

## Đã kiểm tra, không có vấn đề

Ghi lại để lần audit sau không tốn công rà lại:

- **SQL injection qua tên field/entity**: `aggregate.rs:277` (`format!("\"{}\"", field.name)`) và các
  điểm nội suy tương tự là **an toàn** — `metap-metadata::compiler::validate` ép
  `^[A-Za-z][A-Za-z0-9_]*$` cho mọi field name, và `MetadataRegistry::register` (`registry.rs:155`)
  là chokepoint duy nhất, luôn gọi `validate`. Loader YAML (feature 33) cố ý không tự validate mà
  đẩy về `register`, nên không mở đường vòng. Đây là phòng thủ đã làm đúng — quan trọng vì entity
  low-code lấy tên field từ input của admin.
- **Tàn dư bảng `records` sau Phase 86/87**: sạch. Chỉ còn `migrate_generic_to_dedicated` +
  `dev-tools migrate-to-dedicated-table` (escape hatch cố ý giữ) và fixture test của chính nó.
- **`validate_schema_name`**: whitelist `^t_[a-z0-9]+$` + `public` là chặt, và `create_tenant_schema`
  gọi lại nó trước khi nội suy vào DDL. Riêng việc `pool_for` gọi rồi bỏ kết quả là finding #1, không
  phải lỗi của hàm này.
