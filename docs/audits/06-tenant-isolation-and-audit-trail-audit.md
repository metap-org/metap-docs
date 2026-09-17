# Audit 06 — Hệ quả chưa đóng của per-tenant schema isolation, và bypass field-permission qua audit trail

> **Phạm vi (2026-09-17):** audit sweep có hệ thống (1 agent Opus), tập trung vào 2 vùng chưa từng
> được audit lần nào và đều vừa ship gần đây: **(a)** per-tenant schema isolation
> (`docs/features/35-per-tenant-schema-isolation.md`, ship 2026-09-11) và **(b)** crate `metap-audit`
> (audit 05, ship 2026-09-13). Chọn 2 vùng này vì audit 02/03/04 đều chạy *trước* khi cả hai tồn
> tại, còn audit 05 không phải sweep. Có rà thêm bề mặt sinh SQL (`metap-query`'s `aggregate`/`jql`)
> và tàn dư sau Phase 86 — cả hai **sạch**, ghi lại ở mục "Đã kiểm tra, không có vấn đề".
>
> **Cập nhật cùng ngày: cả 3 finding HIGH đã fix** (`metap` PR `claude/audit-06-fixes`), sau khi
> chủ dự án yêu cầu "fix hết". Mỗi fix có regression test **đã xác nhận fail trên code trước khi
> sửa** rồi mới pass — không chỉ pass suông. Với #1/#2 cố ý chọn hướng **bảo toàn hành vi đã được
> chốt trước đó** thay vì tự quyết câu hỏi sản phẩm còn treo (xem từng mục). #4 (backfill tenant
> scoping) vẫn treo — nó thuộc kế hoạch fix 3 phần đã ghi ở `metap-demo-waf/CLAUDE.md`, không
> thuộc đợt này.
>
> **Vòng 2 (cùng ngày):** chủ dự án phản hồi *"có mask, tuy nhiên giải mã được mask"* — đúng. Fix
> đầu của #3 xét readable theo **trạng thái hiện tại** của record, mà trạng thái đó **caller tự sửa
> được**, nên mask lật ngược lại được và để lộ cả giá trị quá khứ. Đã dựng lại sống, sửa tận gốc,
> và bản probe đầu tiên của chính tôi cũng đã **pass sai** trước khi phát hiện — xem mục #3.

## Tóm tắt

Ba finding HIGH. Hai trong số đó (#1, #2) **cùng một nguyên nhân gốc**: feature 35 đã đổi một bất
biến toàn hệ thống — *"`schema_name` của tenant `Schema`-strategy luôn là `"public"`"* — nhưng ít
nhất 3 chỗ khác trong codebase vẫn đang dựa vào bất biến cũ đó, trong đó có 2 chỗ dựa vào nó **như
một lập luận bảo mật/thiết kế được ghi thành doc**. Cả 2 đều **chưa nổ**, vì chưa tenant nào được
provision lại sau 2026-09-11 (mọi tenant `Schema` hiện tại vẫn `schema_name="public"`, xác nhận sống
2026-09-08) — chúng sẽ nổ ở **lần gọi `provision_schema_tenant` tiếp theo**.

| # | Mức độ | Vấn đề | Trạng thái |
|---|---|---|---|
| 1 | **HIGH** | `Router::pool_for` trả pool dùng chung, không set `search_path` — mọi tenant có schema riêng bị phục vụ sai schema. Không chỉ 3 call site `dev-tools` như doc khẳng định: `metap-lowcode` có 5 call site nữa, 1 trong đó nằm trên **mọi HTTP handler** | **Đã fix** — `Router::schema_pool` mới |
| 2 | **HIGH** | `POST /auth/login` không kèm `tenantId` tra `metadata.users`, nhưng `provision_schema_tenant` ghi admin mới vào `t_<uuid>.users` → **tenant vừa provision không đăng nhập được**. Đồng thời `users_email_unique` (audit 04 A#2 xác nhận là "đang chịu lực") bị clone thành unique *per-schema*, phá luôn tính toàn cục mà chính lập luận đó dựa vào | **Đã fix** — loại `users`/`user_roles` khỏi clone |
| 3 | **HIGH** | `GET /api/{entity}/{id}/audit-events` trả nguyên `diff` không mask field → **bypass field-level permission**: ai đọc được record thì đọc được giá trị mọi field trong toàn bộ lịch sử, kể cả field bị mask ở đường đọc thường | **Đã fix, 2 vòng** — mask qua chính `filter_readable_fields`; vòng 2 sau khi chủ dự án chỉ ra mask **lật được** bằng cách sửa field mà policy lấy làm điều kiện |
| 4 | MEDIUM | (Nhắc lại) Finding thứ 9 của `metap-demo-waf` — `backfill::run_batched_update` scope theo `tenant_id` sentinel → backfill boot-time chạm 0 dòng rồi vẫn `mark_completed` | **Vẫn treo** — thuộc kế hoạch fix riêng ở repo đó |
| 5 | LOW | Doc drift chịu lực: cả `CLAUDE.md` lẫn doc comment của `pool_for` khẳng định sai về hiện trạng, và chính khẳng định sai đó là lý do finding #1 chưa được fix | **Đã fix** |

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

**Hướng fix**: `pool_for` không thể set `SET LOCAL` (không có transaction) và cũng không nên `SET`
ở mức session trên pool dùng chung (đúng cái Bẫy #1 mà `begin()` cảnh báo). Hai hướng khả dĩ:
1. Trả `PoolConnection` đã `SET search_path` thay vì `PgPool` — đổi chữ ký, ép mọi caller giữ
   connection, nhưng đóng hẳn lỗ.
2. Giữ pool riêng theo schema (cùng pattern `dedicated_pools`, `moka` theo `schema_name`), mỗi pool
   có `after_connect()` set `search_path` — không đổi chữ ký caller.

→ **Đã chọn hướng 2 và đã làm** (hợp với call site hiện tại hơn: caller đang cần `PgPool` thật để
chạy DDL, và hướng 1 còn kéo theo vấn đề connection trả về pool vẫn mang search_path cũ). Chi tiết
ở mục "Đã fix" cuối file.

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

**Hướng fix**: câu hỏi sản phẩm "login đa tenant định danh người dùng bằng gì" vẫn **chưa chốt và
không tự chọn**. Nhưng không cần chốt nó mới đóng được lỗi này: fix đã làm chỉ **khôi phục đúng
model đã được chốt trước đó** (identity toàn cục, audit 04 A#2) bằng cách loại `users`/`user_roles`
khỏi clone — không mở rộng cũng không thu hẹp thiết kế. Nếu sau này chủ dự án quyết đổi sang model
identity per-tenant thật, đó là một thay đổi riêng, có chủ đích, không phải hệ quả phụ của
feature 35. Chi tiết ở mục "Đã fix" cuối file.

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
`filter_readable_fields` lên từng `diff` theo snapshot của caller) — **đã làm**. Mask ở đường ghi là
câu hỏi riêng (mask khi ghi thì mất luôn giá trị compliance của audit; không mask thì bảng vĩnh viễn
giữ secret) — **vẫn cần chủ dự án chốt, không tự quyết**. Chi tiết ở mục "Đã fix" cuối file.

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

---

## Đã fix (cùng ngày, `metap` PR `claude/audit-06-fixes`)

Ghi lại **hướng đã chọn và vì sao**, vì với #1/#2 tôi cố ý không tự quyết câu hỏi sản phẩm còn
treo — chỉ khôi phục lại đúng hành vi đã được chốt từ trước mà feature 35 vô tình phá.

**#1 — `Router::schema_pool` mới.** `pool_for` với strategy `Schema` giờ đi qua một pool cache theo
schema (`moka`, cùng shape/TTL với `dedicated_pools` sẵn có), dựng từ chính `connect_options()` của
pool dùng chung nên `Router::new` không phải nhận thêm tham số DSN. Mỗi connection của pool đó mang
`SET search_path` **mức session**. Mức session ở đây là đúng, dù `begin()` cảnh báo Bẫy #1 về đúng
việc này: khác biệt nằm ở chỗ mọi connection trong pool ấy chỉ phục vụ **một** schema duy nhất, nên
không có "request tiếp theo" nào để rò sang. `"public"` vẫn short-circuit trả thẳng pool dùng chung
— **không tenant nào đang tồn tại bị đổi hành vi, và không tốn thêm connection nào**.

Hai hướng đã cân nhắc và loại: (a) trả `PoolConnection` đã set search_path — đóng lỗ triệt để nhưng
đổi chữ ký của mọi caller xuyên 2 repo, và connection trả về pool vẫn mang search_path cũ nếu không
có `after_release`; (b) bắt caller tự qualify tên bảng — không khả thi vì caller là handler tổng
quát, không biết trước bảng nào.

**#2 — loại `users`/`user_roles` khỏi `TENANT_SCOPED_TABLES`.** Đây là lựa chọn bảo toàn: identity
của nền tảng này **cố ý toàn cục** (đó chính là lý do `users_email_unique` là global chứ không phải
`(tenant_id, email)` — audit 04 A#2 đã xác nhận cố ý). Không clone 2 bảng đó thì không mất gì về mặt
chức năng, vì `Router::begin` đã luôn đặt `metadata` trong `search_path` của mọi transaction tenant,
nên truy vấn không qualify vẫn resolve về bảng chung đúng như trước khi có schema riêng, vẫn lọc
theo cột `tenant_id` từng dòng.

Nhân tiện sửa luôn header sai của chính module `tenant_schema.rs`: nó khẳng định không clone thì
tenant non-`public` "không resolve được gì cả", coi việc clone là điều kiện cần để chạy. Sai —
`metadata` luôn nằm trong search_path. Việc clone mua **isolation vật lý** (đáng có), chứ không phải
thứ làm cho tenant chạy được.

**#3 — mask ở đường đọc.** `list_audit_events` lọc từng `diff` qua đúng
`PermissionSnapshot::filter_readable_fields` mà đường đọc thường (`row_to_dto_masked`) vẫn dùng —
cố ý **không** tự viết lại logic đánh giá field policy, vì viết lại lần hai đúng là cách hai đường
trôi lệch nhau. Hai chi tiết dễ sai đã xử lý: probe dựng từ record **hiện tại** (field policy có thể
điều kiện theo giá trị record, nên probe rỗng sẽ cho kết quả khác), và bù `null` cho mọi field đã
khai báo nhưng đang vắng trong record (nếu không, field readable mà đang null sẽ bị mask oan).

Quyết định ngữ nghĩa ban đầu — readable xét theo **trạng thái hiện tại** của record — **đã sai và
đã được sửa cùng ngày**, sau khi chủ dự án chỉ ra: *"có mask, tuy nhiên giải mã được mask"*. Tôi chỉ
cân nhắc chiều ngược lại (xét theo từng entry sẽ lọt field từng readable trong quá khứ) mà bỏ sót
điều quan trọng hơn: **chủ thể của phép kiểm tra là thứ caller tự dịch chuyển được**.

Kịch bản đã dựng lại sống, không phải suy đoán: policy cho `amount` readable **chỉ khi**
`resolution == "unlocked"`. Caller bị mask khỏi `amount`, nhưng `resolution` lại là field họ **có
quyền ghi hợp lệ**. Sửa `resolution` một phát là toàn bộ lịch sử `amount` mở lại:

```
resolution="locked"    → 0 dòng audit mang `amount`
caller tự sửa resolution="unlocked"
resolution="unlocked"  → 2 dòng lộ: {before:4242, after:9999} và {before:null, after:4242}
```

Nghiêm trọng ở chỗ nó trả lại **giá trị quá khứ** (`4242`) — thứ đường đọc thường không bao giờ trả
(đường đó chỉ phục vụ giá trị hiện tại `9999`). Tức là đúng nghĩa "giải mã được mask".

**Cách sửa, và vì sao không chọn cách chính xác hơn**: xét từng entry theo đúng trạng thái lịch sử
của nó mới là câu trả lời chính xác, nhưng **không khả thi** — `metadata.audit_trail_entries` chỉ
lưu `diff` từng entry, không hề lưu snapshot trạng thái đầy đủ; mà dựng lại trạng thái bằng cách
replay ngược các diff sẽ sai đúng vào lúc trail bị khuyết — điều mà thiết kế **cho phép** (ghi audit
là best-effort có chủ đích). Nên: field nào có read policy **phụ thuộc giá trị record**
(`record_state_dependent_read_fields`) thì bị loại khỏi lịch sử audit hoàn toàn, không đánh giá nữa.

Chấp nhận **over-mask có chủ đích**: field vừa có grant vô điều kiện vừa có grant có điều kiện cũng
bị loại luôn. Che nhầm lịch sử là chiều an toàn; để lọt mới là bug. Admin giữ nguyên quyền bypass.

Một bài học về chính cách tôi test: bản probe đầu tiên **pass mà không chứng minh được gì** — nó
dùng `PolicySubject::Context` nên điều kiện không bao giờ được đánh giá trên record, `amount` bị che
ở cả hai bước vì lý do hoàn toàn khác. Test chính thức vì vậy mang thêm một **control assertion**
khẳng định policy thật sự cấp `amount` trên đường đọc thường sau khi unlock.

Còn treo, không đổi: có nên mask ở **đường ghi** hay không vẫn là câu hỏi cho chủ dự án (mask khi ghi
thì mất giá trị compliance của audit; không mask thì bảng — vốn cố ý không bao giờ prune — giữ
secret vĩnh viễn). Lỗ phân quyền thì đã đóng bằng đường đọc.

**Một tồn dư cùng loại, đã cân nhắc và cố ý không tự sửa**: quyền đọc **mức record**
(`check_record_permission`) cũng xét theo trạng thái hiện tại, nên một record policy có điều kiện
cũng lật được y hệt. Tôi không áp cùng cách sửa ở đây vì nó khác về bản chất: lật được điều kiện mức
record nghĩa là caller **thật sự có quyền đọc record đó**, và chặn audit cho mọi record nằm dưới một
record policy có điều kiện sẽ vô hiệu hoá tính năng với hầu hết cấu hình ABAC thông thường ("đọc
record thuộc phòng ban mình"). Đây là câu hỏi sản phẩm, không phải chỗ tôi tự quyết — nêu ra để chủ
dự án chốt.

### Verify

- `cargo build --workspace`, `cargo clippy --workspace --all-targets -- -D warnings`, `cargo fmt`,
  `cargo test --workspace` (unit, 100/100 suite `ok`) — sạch.
- E2E **sống** trên Postgres 16 thật: `provisioning_postgres` 8/8, `router_postgres` 8/8,
  `crud_service_postgres` 18/20 (2 fail là 2 benchmark thủ công cần seed ngoài, đã biết từ Phase 87,
  không liên quan). Các crate còn lại (`metap-http`/`graphql`/`graphql-http`/`grpc`/`workflow`/
  `query`/`permission`) đều `ok`.
- **Mỗi fix có regression test đã xác nhận fail trước khi sửa**: test của #3 in ra đúng giá trị bị
  lọt (`amount: {before: 4242, after: 9999}`), test của #1 báo `pool_for` vẫn trả pool dùng chung.
- 2 nhóm test fail vì môi trường, **đã kiểm chứng là có sẵn chứ không phải do đợt sửa này** (chạy
  lại trên code đã stash cho kết quả y hệt): `vault_store.rs` và
  `metap-http/tests/tenant_secret_postgres.rs` — cả hai cần dev Vault qua Docker, mà Docker bị chặn
  trong môi trường này.
