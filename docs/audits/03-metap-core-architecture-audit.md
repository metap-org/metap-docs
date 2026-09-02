> **Trạng thái (2026-09-02):** audit kiến trúc mới. **Finding #2 (HIGH, `ServiceTokenSource` retry
> loop) đã verify đúng và fix ngay trong ngày** — `crates/metap-grpc/src/client.rs`, đổi từ
> `sleep(REFRESH_INTERVAL)` cố định sang biến `next_delay` (giữ nguyên `RETRY_BACKOFF` khi lỗi thay
> vì cộng dồn thêm `REFRESH_INTERVAL`), `cargo build -p metap-grpc -p metap-graphql-gateway` sạch.
> Mọi finding còn lại (kể cả #1) **chưa fix/verify** — vẫn nguyên giá trị tham khảo. Khác audit
> trước
> (`02-full-codebase-audit.md`, review từng dòng code toàn bộ `crates/`/`apps/*`) — audit này tập
> trung vào **kiến trúc**: layering, ranh giới crate, backbone có thật sự "authn/authz/workflow/CRUD
> miễn phí" như tầm nhìn đã chốt không, doc CLAUDE.md có còn khớp code không, và ranh giới
> `metap`↔`metap-lowcode` có sạch không (đối chiếu lại làm rõ 2026-09-02: trục "entity định nghĩa
> bằng gì" độc lập với trục "deployment consume ra sao" — xem `docs/vision.md`'s section mới cùng
> ngày).

Phương pháp: 1 agent (Opus) đọc toàn bộ `metap/CLAUDE.md`, ADR liên quan, ~30 `Cargo.toml` +
cross-crate `use` để dựng dependency graph thật (không suy từ doc), rồi đọc sâu các crate được coi
là "backbone" (`metap-permission`/`metap-crud`/`metap-http`/`metap-control`/`metap-workflow`/
`metap-metadata`) cộng `graphql-gateway`/`metap-grpc` (vừa sửa cùng ngày). Không build (`cargo
build/tree/clippy` đều ghi vào `target/`, tránh phình theo đúng quy tắc 40GB của CLAUDE.md). Không
sửa code — chỉ report. Toàn bộ finding đối chiếu với working tree hiện tại (đang có ~20 file thay
đổi chưa commit từ phiên làm việc hôm nay), không phải trạng thái đã commit.

## Kết luận chung

**Backbone về cơ bản đáp ứng đúng tầm nhìn "platform core dùng chung được" — ranh giới
`metap`↔`metap-lowcode` sạch, không có coupling ngược.** Không crate nào trong `metap` phụ thuộc
ngược vào `metap-lowcode`; cả 3 crate lowcode đều phụ thuộc *vào* `metap` (`../../../metap/crates/*`),
không có chiều ngược lại. Không có tên business-entity nào lộ ra ngoài code test/doc-comment ở core
crate — claim "entity-agnostic" đứng vững.

Dependency graph không có cycle, layering nhìn chung hợp lý. Tenant scoping được enforce ở đúng
choke point (`Router::begin`'s `SET LOCAL search_path`, `ObjectStore` bắt buộc `tenant_id`,
`plan_list` luôn push `tenant_id = $n` không điều kiện) thay vì trông chờ kỷ luật ở call site.

Có **2 finding HIGH thật sự là rủi ro kiến trúc**, không phải chỉ là polish: **permission cấp
record (ABAC) không được enforce ở 2 route HTTP** (attachments + workflow-events), và **retry loop
của service-token tự refresh mới sửa hôm nay có bug khiến nó lặp lại đúng sự cố nó được viết ra để
fix**. Ngoài ra `metap/CLAUDE.md` đã lệch thực tế khá nhiều — 3 crate nguyên vẹn (~10% workspace)
không được nhắc tới, và vài claim cụ thể đã sai.

---

## 1. Vi phạm layering — sạch, có 1 lỗ hổng nhỏ cần vá

**Không tìm thấy vi phạm layering thật.** Grep toàn bộ core crate tìm business-entity identifier
chỉ ra test fixture/doc example. `metap_http::build_router` (`crates/metap-http/src/lib.rs:77-93`)
mount 13 nhóm route, tất cả entity-generic.

**LOW — `EntityDefinition.name` không được validate charset.** `crates/metap-metadata/src/compiler.rs:80-96`
validate `field.name` (regex + comment giải thích rõ lý do), `:290-304` validate `table_name`,
nhưng `entity.name` (do admin lowcode nhập tay) thì không — chảy thẳng vào
`metap_reconciler::table_name_for` (`crates/metap-reconciler/src/compile.rs:43-45`, chỉ
`.replace('.', "_")`) và `build_index_name`. Hiện tại an toàn nhờ **may mắn** (mọi identifier đều
được `quote_ident` trước khi vào SQL) chứ không phải do được validate ở đúng biên tin cậy như
`field.name`. Đề xuất: thêm `is_safe_entity_name` cùng chỗ với 2 check hiện có trong `validate()`.

---

## 2. Backbone có thật sự "authn/authz/workflow/CRUD miễn phí" không?

Về cơ bản có — `templates/metap-app/src/main.rs` là boot ~40 dòng thật (`bootstrap_platform` →
đăng ký entity → `build_router` → `serve::run`), CRUD/permission/workflow/optimistic-lock/outbox
đều có sẵn. Nhưng:

**MEDIUM — facade `metap` thiếu, downstream app phải đi vòng qua nó.**
`crates/metap/src/lib.rs:21-38` re-export 18 sub-crate nhưng **thiếu** `metap-reconciler`,
`metap-storage`, `metap-cron`, `metap-attachments`, `metap-dashboards`. Hệ quả thấy ở cả 2 consumer
thật: `../metap-demo-jira/Cargo.toml:14-16` phải tự thêm `metap-reconciler`/`metap-storage`/
`metap-outbox-publisher`; `../metap-demo-crm/Cargo.toml` phải tự thêm `metap-reconciler`/
`notification-worker`. Đi ngược đúng mục đích của facade (comment tự nói: "một dependency, một
import" — `crates/metap/src/lib.rs:2-4`) và CLAUDE.md dòng 47. Table-per-entity — capability đầu
bảng của platform — không với tới được từ facade, user `templates/metap-app` không dùng được nếu
không tự sửa `Cargo.toml`. Đề xuất: re-export nốt 5 crate còn thiếu (đều leaf/low-tier, không tạo
cycle — `metap-reconciler` chỉ phụ thuộc `metap-metadata`).

**MEDIUM — template hướng dẫn 1 dòng không compile được.**
`templates/metap-app/src/main.rs:58-61` bảo user "pass `metap::lowcode_http::router()` vào đây" —
nhưng facade **cố tình** không re-export lowcode (`crates/metap/src/lib.rs:11-19` nói rõ). Project
mới làm theo comment này sẽ lỗi compile. 2 gợi ý bên cạnh (`metap::graphql_http::router`,
`metap::grpc::serve`) thì đúng.

**LOW — bất đối xứng permission model mà app mới cần biết.** Check cấp entity là
**deny-by-default** (`crates/metap-permission/src/permission_service.rs:94-103`), nhưng field-level
masking và record-level condition lại **allow-by-default** khi chưa có policy nào. Field mới thêm
world-readable cho bất kỳ ai đọc được entity, cho tới khi có ai viết policy field. Hợp lý (mirror
đúng bản TS gốc) nhưng không hề có trong câu chuyện "authz miễn phí" của CLAUDE.md, và đúng kiểu lỗi
1 project mới dễ mắc.

**LOW — app mới toanh có 0 policy → mọi non-admin bị deny hết** cho tới khi gọi
`POST /admin/policies/seed-defaults` (`crates/metap-http/src/routes/admin.rs:478`). Không có trong
`main.rs` mẫu lẫn `templates/metap-app/README.md`'s flow như 1 bước bắt buộc.

---

## 3. Ranh giới crate

**Không có cycle** (verify từ manifest thật). Tier: `metap-runtime` (leaf) → `metap-infra`/
`metap-cache` → `metap-permission` → `metap-metadata` → `metap-query`/`metap-workflow`/
`metap-control` → `metap-crud` → `metap-http` → facade/`metap-app`.

**MEDIUM — `graphql-gateway` kéo cả `metap-http` chỉ để dùng 1 hàm middleware.**
`crates/graphql-gateway/src/server.rs:138` là chỗ duy nhất (ngoài test) dùng `metap_http::` — dù
doc comment đầu file (`:4-6`) tự nói "chỉ reuse phần standalone-safe". Nhưng `metap-http`'s
`Cargo.toml` kéo theo 14 metap crate + `sqlx`, `aws-sdk-s3`, `metap-cron`, `metap-dashboards`,
`metap-attachments` — không cái nào binary không-Postgres này dùng tới. `security_headers`
(`crates/metap-http/src/security_headers.rs:22`) là 1 `async fn` thuần không phụ thuộc `AppState`
— đúng hình dạng `metap-runtime` đã hấp thụ cho `rate_limit`/`trace`/`cors`/`request_id` 4 lần
trước đó. **Đây là đúng loại quyết định "có nên đưa vào metap-runtime" đã làm đúng 4 lần, bỏ sót
1 lần.** Đề xuất: chuyển `security_headers` sang `metap-runtime`, re-export lại từ `metap-http`, bỏ
`metap-http` khỏi dep của gateway.

**MEDIUM — `metap-permission` → `metap-cache` khiến `redis` thành hard-dep của gần như cả
workspace.** `crates/metap-permission/Cargo.toml:10` không điều kiện, `metap-cache`'s `redis` dep
cũng không có feature gate. Vì `metap-metadata` → `metap-permission`, **mọi** crate trong workspace
đều compile kèm Redis client — kể cả `metap-reconciler` và các consumer chưa từng cache gì.
`Cache` trait chỉ dùng sau `Option<Arc<dyn Cache>>`. Đề xuất: feature-gate `redis` ở `metap-cache`,
hoặc đảo hướng phụ thuộc (định nghĩa `Cache` trait tối giản ngay trong `metap-permission`, để
`metap-cache` implement nó).

**MEDIUM — `PostgresPolicyStore` nằm ở `metap-control`, không phải `metap-permission`.**
`crates/metap-control/src/lib.rs:535` export nó; `metap-permission` chỉ export trait `PolicyStore`.
Nghĩa là *tenancy control plane* lại sở hữu *permission persistence* — vì sao `metap::prelude` phải
re-export `metap_control::PostgresPolicyStore` cạnh `metap_permission::PermissionService`
(`crates/metap/src/lib.rs:50`). Có lý do chính đáng (impl cần `Router` để route theo tenant, và
`metap-permission` không được phép phụ thuộc `metap-control`) nhưng khiến ranh giới ngược với cách
CLAUDE.md mô tả — đây cũng chính là nguồn gốc 1 claim sai ở mục 4.

**LOW — `metap-metadata` → `metap-permission` là 1 layering inversion.**
`crates/metap-metadata/Cargo.toml:7`; nguyên nhân là `WorkflowTransition.guard: Option<PolicyCondition>`.
Hệ quả: `metap-permission` không bao giờ phụ thuộc được `metap-metadata` — nên việc validate "field
trong policy có thật trong entity không" về mặt cấu trúc là bất khả thi ở đúng chỗ nó nên nằm. Cần
biết trước khi ai đó thử làm.

**LOW — vài crate mỏng, có thể không cần tách riêng.** `metap-dashboards` (115 dòng, không phụ
thuộc metap crate nào), `metap-attachments` (209 dòng, tương tự), `metap-app` (97 dòng). Không sai
— đều sạch, đúng tiền lệ `metap-cron`'s single-table library — nhưng `metap-dashboards` đặc biệt là
1 CRUD wrapper 115 dòng cho 1 bảng, thêm cả 1 workspace member + 1 quyết định facade cho khá ít giá
trị.

---

## 4. Doc-vs-reality drift — đáng kể

**MEDIUM — 3 crate hoàn toàn không có trong CLAUDE.md.** `metap-attachments`, `metap-auth`,
`metap-dashboards` là workspace member thật (root `Cargo.toml`), là hard-dep của `metap-http`, và
`metap-auth` còn có mặt trong facade (`metap::tenant_auth`). Grep CLAUDE.md cho cả 3 tên: 0 kết
quả. Tương ứng phase thật đã ship (roadmap 25/27/33) — roadmap có track nhưng list-theo-crate ở
CLAUDE.md chưa bao giờ cập nhật theo. ~10% workspace vô hình với chính tài liệu được coi là bản đồ.
**`metap-auth` đáng chú ý nhất**: sở hữu OIDC + HTTP Basic — 2 trong 3 cách 1 request có thể auth —
CLAUDE.md dòng 152 ("JWT + live `user_roles` lookup") không hề nhắc Basic auth tồn tại.

**LOW — vài claim cụ thể sai, đối chiếu trực tiếp với code:**

| CLAUDE.md nói | Thực tế |
|---|---|
| `metap-permission` — "`PolicyStore` trait + `PostgresPolicyStore`" | `PostgresPolicyStore` nằm ở `metap-control`; `metap-permission` chỉ export trait. Lặp lại claim sai này 2 lần trong file. |
| `metap-http` — "`/api/:entity*`, `/metadata/*`, `/health`, `AuthContext`" | Thực ra 13 nhóm route: còn có auth/login/OIDC, admin users+policies, cron, dashboards, attachments, preferences, users, workflow-events, metrics. |
| `metap-storage` — "chưa wire vào route/feature nào" | Đã wire đầy đủ — `AppState.object_store` chạy xuyên `routes/attachments.rs`. |
| "cột `version` reserved cho optimistic locking" | Đã implement đầy đủ — `update.rs`/`delete.rs` đều `WHERE version = $n` → `409 version_conflict`. |
| Danh sách module `metap-runtime` | Thiếu `trace_context` (191 dòng, W3C propagation từ roadmap 57) và các item mới nhất (`env::flag_enabled`, `metap_grpc::optional_serve`). |

**LOW — ~40 tham chiếu `apps/crm-server`/`apps/jira-server` đã lỗi thời trong doc comment code**,
nằm trong 40 file khác nhau, 6 tháng sau khi tách repo. CLAUDE.md dòng 26 nói rõ
`crates/graphql-gateway/README.md` đã cập nhật, dòng 11-16 cho exemption rõ ràng cho path
`docs/...` — nhưng không có exemption nào cho `apps/...`, nên đây đọc như drift chưa fix chứ không
phải lựa chọn có ghi nhận. Tập trung ở các file 1 contributor mới đọc đầu tiên:
`crates/metap-http/src/lib.rs:571`, `crates/metap-http/src/state.rs` (4 chỗ),
`crates/metap-grpc/src/lib.rs:626-627`, `crates/metap-graphql/src/lib.rs:659`,
`crates/metap-control/src/router.rs:194`.

**LOW — 1 cross-reference nội bộ lỗi thời**: `crates/metap-http/src/error.rs:4-6` trỏ vào
`crates/metap-http/src/request_id.rs`, đã chuyển sang `metap-runtime` cùng đợt di chuyển các hàm mà
chính doc comment này đang mô tả.

---

## 5. Bề mặt multi-tenancy / bảo mật

**Nhìn chung mạnh.** Tenant scoping là cấu trúc, không phải quy ước: `Router::begin` dùng
`SET LOCAL` (transaction-scoped, có comment giải thích rõ vì sao session-level `SET` sẽ leak qua
pooled connection); `plan_list` luôn push `tenant_id = $1` trước mọi thứ khác, cap `limit`, filter
field allowlist từ list-view metadata, mọi value đều bind chứ không interpolate; role luôn đọc lại
từ `user_roles` mỗi request, không cache; Basic auth **từ chối** thay vì tự thay thế khi tenant
header không khớp user — đúng hướng, sai chỗ này sẽ là 1 lỗ hổng bypass auth cross-tenant thật sự
nghiêm trọng.

### **HIGH — permission cấp record (ABAC) không được enforce ở attachments và workflow-event history**

`CrudService::get` chạy 2 tầng check: entity-level rồi record-level
(`crud_service/get.rs:22-59`), `list` cũng push record policy vào SQL. **2 route chỉ chạy tầng
đầu:**

- `crates/metap-http/src/routes/workflow_events.rs:24-30` — chỉ `can_read_entity`, sau đó trả toàn
  bộ lịch sử transition cho bất kỳ `record_id` nào trong tenant.
- `crates/metap-http/src/routes/attachments.rs` — list (`:136-158`), download (`:202-253`), upload
  (`:63-69`), delete (`:269-275`) — đều chỉ check entity-level.

Hệ quả: trong app dùng record-level policy cho row-level security (đúng tính năng ABAC platform
quảng cáo — vd "chỉ đọc được issue của project mình sở hữu"), 1 caller bị `403` ở
`GET /api/jira.issues/{id}` vẫn `GET`/download được attachment, đọc full lịch sử state-change, và
xoá/thêm attachment cho chính record đó. `load_owned_attachment` chỉ cross-check *identity* của
record cha, không check *permission*.

Đáng chú ý: `hydrate_related_display` đã được fix đúng loại bug này trong 1 lần review 2026-08-22
(`crud_service.rs:150-154`: "display convenience không được lộ giá trị mà caller sẽ bị 403 nếu đọc
trực tiếp") — cùng lý do áp dụng y hệt ở đây, nhưng 2 route này ra đời trước/sau lần fix đó mà không
thừa hưởng nó. Đề xuất: cả 2 route đã có sẵn `state.crud` — gọi `CrudService::get(entity,
record_id, &context)` trước, propagate `ServiceResult::Err` của nó (có sẵn entity-level +
record-level + tenant + not-found trong 1 call).

### **HIGH — retry loop của `ServiceTokenSource` không thật sự retry sớm hơn**

`crates/metap-grpc/src/client.rs:87-104`:

```rust
loop {
    tokio::time::sleep(REFRESH_INTERVAL).await;   // 2400s
    match login_once(...).await {
        Ok(token) => background_current.store(...),
        Err(e) => {
            tracing::error!(... "retrying in {RETRY_BACKOFF:?}");
            tokio::time::sleep(RETRY_BACKOFF).await;   // 30s — rồi loop về đầu, sleep tiếp 2400s
        }
    }
}
```

Khi refresh lỗi, loop sleep 30s rồi **quay lại đầu vòng lặp và sleep thêm 2400s nữa** trước lần thử
tiếp theo — retry interval thật là 2430s, không phải 30s như log ghi. TTL token là 3600s, refresh
đầu ở 2400s: 1 lần lỗi tạm thời (upstream đang restart — đúng case comment `RETRY_BACKOFF` nhắc
tới) đẩy lần thử tiếp theo tới t≈4830s, quá xa mốc hết hạn. Token cache chết, mọi fallback trong
`pick_token` bắt đầu trả về JWT đã hết hạn.

Đây là **regression đúng sự cố code này được viết ra để fix** (comment ngay phía trên tự nói: "thay
thế 1 service JWT tĩnh mint 1 lần sau khi JWT đó hết hạn TTL 1h giữa lúc chạy và crash 1 caller lúc
boot"). Vì đây là **giao diện duy nhất cho 1 deployment microservice thật** theo đúng tầm nhìn đã
chốt, đây là đường outage cho cả gateway. Đề xuất: `continue` sau khi sleep backoff, hoặc restructure
thành `sleep(next_delay)` với `next_delay = RETRY_BACKOFF` khi lỗi, `REFRESH_INTERVAL` khi thành
công.

### **MEDIUM — `cron-scheduler` vẫn chạy pattern static-JWT đã từng gây outage**

`crates/cron-scheduler/src/main.rs:16-19` đọc `CRON_SERVICE_JWT` từ env, chỉ warn nếu thiếu — cùng
cái chết TTL-1h y hệt thiết kế cũ của gateway, và chính `metap-grpc/src/client.rs:19-21` cũng thừa
nhận đây là "ngoài phạm vi". `ServiceTokenSource` giờ đã là primitive tái dùng được trong
`metap-grpc` — đây là rủi ro production còn sống, đã biết nguyên nhân, đã biết cách fix, trong 1 ops
binary đang chạy thật, không phải giả định.

### **LOW — các note bề mặt khác**

- Rate limit key theo peer IP (`metap-runtime/src/rate_limit.rs:8-13`, có lý do chính đáng vì
  `X-Forwarded-For` giả mạo được) — nhưng đứng sau load balancer (kiến trúc deploy 07 đã plan) thì
  toàn bộ traffic dùng chung 1 bucket → cap burst 300 toàn cục, tự-DoS dễ dàng. Tradeoff có ghi
  trong code, chưa có trong hướng dẫn deploy của CLAUDE.md.
- `GET /metrics` public, không auth — lộ request count/latency theo route + RSS/CPU/fd process cho
  bất kỳ ai. Có chủ đích ("cùng convention với `/health`") nhưng `/health` không lộ gì tương đương.
- `GET /metadata/entities` trả full schema của **mọi** entity đã đăng ký cho bất kỳ user đã login
  nào, không filter theo `can_read_entity` per-entity. Chỉ lộ shape, không lộ tenant data — nhưng
  trong app nhiều entity thì lộ luôn cả sự tồn tại + tên field của entity mà caller không có quyền.
- `graphql-gateway` forward nguyên token caller xuống mọi upstream (đúng thiết kế đã chốt và có tài
  liệu tốt cho topology share-keypair) — nhưng không có gì trong code **enforce** việc mọi upstream
  thật sự share keypair đó. 1 deployment cấu hình sai sẽ âm thầm đưa bearer token của user cho 1
  service không verify được nó (fail closed) hoặc, tệ hơn, cho 1 domain trust khác.

---

## 6. Ranh giới `metap` ↔ `metap-lowcode` — sạch, không regression

Đối chiếu với tầm nhìn đã chốt lại 2026-09-02 (lowcode = cách *định nghĩa* entity, deployment shape
là trục độc lập):

- **Không có dependency ngược nào.** Không `Cargo.toml` nào trong `metap` nhắc tên crate lowcode
  nào. Cả 3 crate lowcode phụ thuộc *vào* `metap` — đúng chiều dự định.
- **Seam mà lowcode dùng thật sự generic**: `AppState.metadata: Arc<ArcSwap<MetadataRegistry>>`
  (hot-swap), `build_router`'s `extra_routes` param, `AppState.extra_openapi_paths`,
  `MetadataRegistry::merge_with` — không cái nào nhắc lowcode trong code, đều dùng được cho bất kỳ
  downstream binary nào. `CrudService` đọc registry đã swap 1 lần mỗi call để 1 lần publish giữa
  chừng không xé request đang chạy — đúng invariant cho model "entity lowcode vẫn chạy qua core
  primitive".
- **Exception có chủ đích ở CLAUDE.md dòng 40-44 vẫn đứng vững**: `provision_schema_tenant`/
  `provision_dedicated_db_tenant` ở lại `metap-control`, được `dev-tools provision-tenant` gọi
  (không phụ thuộc lowcode). Quyết định đúng — chuyển đi sẽ khiến `dev-tools` phụ thuộc ngược vào
  repo mới.
- **Coupling còn lại chỉ ở mức doc-comment**, không phải code: `metap-http/src/state.rs:34-41`,
  `metap-http/src/lib.rs:601-608`, `metap-app/src/lib.rs:215` đều nhắc tên `metap-lowcode` trong
  prose — hợp lý (trỏ tới 1 ví dụ thật của pattern mở rộng) chứ không phải vi phạm.
- 1 điểm ở cấp org-policy, không phải ranh giới: `metap-lowcode/README.md`/`docs/architecture.md`
  vẫn tiếng Anh — CLAUDE.md gốc đã ghi nhận đây là known gap "fix lần tới khi đụng vào doc đó".
  Không đổi từ lúc tách.

---

## Tóm tắt ưu tiên xử lý

| # | Mức độ | Vị trí | Vấn đề |
|---|---|---|---|
| 1 | **HIGH** | `metap-http/src/routes/attachments.rs`, `workflow_events.rs` | Record-level ABAC bị bỏ qua |
| 2 | **HIGH** | `metap-grpc/src/client.rs:87-104` | Retry loop của `ServiceTokenSource` sleep 2430s thay vì 30s — lặp lại đúng outage nó fix |
| 3 | MEDIUM | `cron-scheduler/src/main.rs:16-19` | Pattern static-JWT (`CRON_SERVICE_JWT`) còn sống, đã biết cách fix |
| 4 | MEDIUM | `metap/src/lib.rs:21-38` | Facade thiếu 5 crate — cả 2 demo app + template phải đi vòng (reconciler/storage không với tới) |
| 5 | MEDIUM | `templates/metap-app/src/main.rs:61` | Ghi `metap::lowcode_http::router()` — không tồn tại, không compile |
| 6 | MEDIUM | `graphql-gateway` deps | Kéo cả `metap-http` chỉ để dùng `security_headers` (nên ở `metap-runtime`) |
| 7 | MEDIUM | `metap-permission` → `metap-cache` | `redis` thành hard-dep transitively của cả workspace |
| 8 | MEDIUM | CLAUDE.md | `metap-attachments`/`metap-auth`/`metap-dashboards` hoàn toàn không được nhắc |
| 9 | LOW | CLAUDE.md | 5 claim sai cụ thể (vị trí PolicyStore, route metap-http, storage "chưa wire", version "reserved", module list metap-runtime) |
| 10 | LOW | `metap-control`/`metap-metadata` | `PostgresPolicyStore` sai chỗ theo doc; `metap-metadata → metap-permission` là inversion |
| 11 | LOW | `metap-permission` | Field/record permission allow-by-default trong khi entity permission deny-by-default (bất đối xứng chưa ghi tài liệu) |
| 12 | LOW | `metap-metadata/src/compiler.rs` | `entity.name` chưa validate charset (đang an toàn nhờ may mắn) |
| 13 | LOW | ~40 file doc-comment | Path `apps/crm-server`/`apps/jira-server` lỗi thời; `error.rs` trỏ file đã chuyển |
| 14 | LOW | rate_limit/metrics/metadata routes | Rate limit theo peer-IP sụp sau LB; `/metrics` public; `/metadata/entities` không filter theo quyền |

## Chưa verify được trong phạm vi lần này

- Mọi thứ cần DB/RabbitMQ/S3 thật — các e2e suite `#[ignore]` được đọc nhưng không chạy, nên
  idempotency của reconciler, fan-out của orchestrator, outbox drain, gateway e2e đều chưa confirm
  bằng thực nghiệm.
- Không compile thật — claim "template's `metap::lowcode_http` không compile" suy từ export list
  của facade, không phải từ 1 lần build thất bại thật (dù không mơ hồ).
- **Bản thân source `metap-lowcode` chưa được audit** — chỉ audit dependency edge trong `Cargo.toml`
  + layout repo, đủ cho câu hỏi ranh giới nhưng chưa xác nhận entity lowcode có thật sự đi qua
  `CrudService` lúc runtime hay không (cần đọc `metap-lowcode/crates/metap-lowcode-http/src/`).
- `metap-cron`'s workflow-automation layer (`TargetType::Steps`/`WaitEvent`, ~2500 dòng) chỉ đọc ở
  mức `lib.rs`/export — 1 subsystem lớn có thể tự có finding riêng.
- Working tree hiện có ~20 file thay đổi chưa commit (phiên làm việc hôm nay) — mọi finding đối
  chiếu với working tree, không phải trạng thái đã commit.
