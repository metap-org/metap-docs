# Config trong database, phân tầng operator / platform-global / per-tenant

- **Trạng thái:** **lát 1 done 2026-09-03** (`metap-config` + `platform_configs` +
  `/platform/config`, đóng audit 04 A#7 — xem `../roadmap/66-platform-config-tiers.md`). Lát 2
  (`tenant_configs` + theme) và lát 3 (secret + nới `FORBIDDEN_HEADERS`) chưa bắt đầu.
- **Người đề xuất:** chủ dự án, 2026-09-03 ("những config như này có thể set trong database, đầu
  API admin/config hay gì đó - per tenant, ngoài ra thì với lowcode thì có thêm mức config-global")
- **Track sở hữu:** HTTP-API Surface (bề mặt `/admin/*`) + Backend Core (tầng đọc/cache)
- **Phase roadmap liên quan:** Phase 66 (lát 1)

## Vấn đề / động lực

Ý xuất phát khi vá audit 04 A#1: guard SSRF cho cron webhook được cấu hình bằng 2 env var mới
(`CRON_WEBHOOK_ALLOWED_HOSTS`, `CRON_WEBHOOK_ALLOW_PRIVATE_TARGETS`), và đặt câu hỏi rất hợp lý:
sao không để trong DB, có API `/admin/config`, chỉnh per-tenant?

**Cảnh báo quan trọng nhất, phải chốt trước mọi thứ khác: đúng hai config đó là ví dụ *không*
được phép per-tenant.** Toàn bộ giá trị của bản vá A#1 nằm ở chỗ danh sách host là **operator
kiểm soát**. Nếu tenant admin sửa được `CRON_WEBHOOK_ALLOW_PRIVATE_TARGETS`, họ tự bật lại đúng
lỗ hổng vừa vá — kẻ tấn công không cần khai thác gì cả, chỉ cần đổi một ô cấu hình. Điều này
không làm ý tưởng sai; nó chỉ ra rằng **tầng của một config là thuộc tính của chính config đó,
không phải của người đang hỏi** — và đó là phần thiết kế phải làm cho đúng trước khi viết dòng
code nào.

Điểm thứ hai đáng nói: **pattern này đã tồn tại sẵn trong repo, hai lần**, nên đây không phải
xây mới từ đầu:

| Bảng | Tầng | Ai sửa |
|---|---|---|
| `tenant_auth_configs` (`0019`) | per-tenant, `jsonb config` | tenant admin |
| `dashboard_configs` (`0022`) | **2 tầng**: tenant default (`owner_user_id IS NULL`) ← personal override | admin / chính user |
| `user_preferences` (`0007`) | per-user | chính user |

`dashboard_configs`'s `get_effective_dashboard` đã làm đúng cái resolution nhiều tầng mà đề xuất
này cần. Nên câu hỏi thật không phải "có làm được không" mà là **"có nên gộp chúng thành một bề
mặt chung, và những config nào ngoài kia được phép nhập hội"**.

## Khảo sát thật: 47 env var hiện có thuộc tầng nào

Đếm bằng grep trên `crates/` (không suy đoán). Kết quả quan trọng và hơi phản trực giác:
**gần như toàn bộ env var hiện tại là config *triển khai*, không phải config *tenant*.**

**Tầng 0 — không thể nằm trong DB (con gà quả trứng).** Cần chúng để *đến được* DB hoặc để verify
được token trước khi đọc bất cứ thứ gì: `DATABASE_URL`, `OUTBOX_DATABASE_URL`, `RABBITMQ_URL`,
`REDIS_URL`, `AUTH_JWT_{PRIVATE,PUBLIC}_KEY_PATH`, `VAULT_*`, `AWS_SECRETS_*`,
`GCP_SECRETS_PROJECT_ID`, `HOST`, `PORT`, `NODE_ENV`. Nhóm này đóng, không bàn thêm.

**Tầng 1 — operator-only. Có thể vào DB, nhưng tenant admin *không bao giờ* được sửa.** Sửa được
là thủng bảo mật, không phải "cấu hình sai":
- `CRON_WEBHOOK_ALLOWED_HOSTS`, `CRON_WEBHOOK_ALLOW_PRIVATE_TARGETS` — bật lại SSRF (audit 04 A#1).
- `CORS_ORIGINS` — thêm một origin là gỡ luôn lớp phòng thủ duy nhất mà audit 04 A#4 dựa vào
  trước khi được vá, và vẫn là lớp bảo vệ cho mọi endpoint khác.
- `CRON_SERVICE_EMAIL`/`_PASSWORD`/`CRON_LOGIN_URL`/`CRON_TARGET_BASE_URL` — credential và đích
  callback của service account.
- `GRPC_ENABLED`/`GRPC_PORT` — mở cổng mạng.

**Tầng 2 — platform-global, `platform_admin` sửa được, hợp lý để vào DB.** Đây là nhóm *có giá trị
nhất* và đang bị bỏ quên nhất, vì phần lớn hiện **hardcode trong code chứ không phải env**:
- `SchemaLimits` (depth 10 / complexity 1000) — hardcode ở `graphql-gateway/src/schema_builder.rs:143`,
  chính là audit 04 **A#7**, và doc comment của `Default` tự nói "starting guardrails, not a
  permanent tuning".
- Rate limit (200ms / burst 300) — hardcode 2 chỗ: `metap-http/src/lib.rs:73`,
  `graphql-gateway/src/server.rs:131`.
- `TOKEN_TTL_SECONDS = 3600` — hardcode ở `metap-http/src/routes/auth.rs:21`.
- Từ env: `POLICY_CACHE_TTL_SECONDS`, `AUTH_CONTEXT_CACHE_TTL_SECONDS`, `CRON_TICK_MS`,
  `CRON_BATCH_SIZE`, `OUTBOX_POLL_MS`, `OUTBOX_BATCH_SIZE`.

**Tầng 3 — thật sự per-tenant.** Bản đầu của brief này viết "gần như rỗng" và đề xuất hoãn
`tenant_configs` lại. **Sai, và sai vì đo nhầm thước**: khảo sát chỉ grep 47 env var đang có, mà
env var thì theo định nghĩa là thứ operator set lúc deploy — không đời nào tìm ra nhu cầu
per-tenant trong đó. Chủ dự án chỉ ra hai ca cụ thể, cả hai đều là feature tương lai nên không
thể xuất hiện trong grep đó:

- **Secret cho webhook gọi third-party.** Tenant khai một webhook đến Stripe/Slack/API nội bộ của
  chính họ thì phải kèm credential. Đây là per-tenant theo đúng nghĩa đen — secret của tenant A
  không được để tenant B thấy, và operator cũng không nên phải set tay một env var cho mỗi tenant.
- **Theme.** Màu, logo, tên hiển thị theo tenant.

Hai ca này **khác nhau về bản chất** chứ không chỉ khác giá trị, và đó là phần thiết kế phải làm
cho đúng — xem 2 mục ngay dưới.

## Secret không phải config — hai loại value, không phải một

Nhét credential third-party vào cột `jsonb` rồi trả về qua `GET /admin/config` là tạo ra đúng ba
vấn đề cùng lúc: giá trị nằm plaintext trong DB backup, lọt vào log ở bất kỳ chỗ nào log request/
response, và ai đọc được config là đọc được credential.

Repo **đã có sẵn hạ tầng đúng cho việc này**, đang dùng cho DSN của tenant `DedicatedDb`: trait
`SecretStore` (`crates/metap-control/src/secret_store.rs`) với 4 impl — `EnvStore`, `VaultStore`,
`AwsSecretsManagerStore`, `GcpSecretManagerStore` — và `build_secret_store(&AppConfig)` chọn backend
từ env lúc boot. `control.tenants.dsn_secret_ref` lưu **tham chiếu**, giá trị thật nằm ở Vault/AWS/
GCP. Đó chính xác là hình dạng webhook secret cần.

Nên `tenant_configs` phải có hai loại value, phân biệt bằng schema chứ không bằng quy ước đặt tên:

```
{"theme": {"primaryColor": "#0af"}}                  → plain, đọc/ghi bình thường
{"webhookAuth": {"secretRef": "tenant/<id>/stripe"}} → chỉ lưu ref, giá trị ở SecretStore
```

Ba ràng buộc bắt buộc, không phải tuỳ chọn:
1. **`GET` không bao giờ trả giá trị secret**, chỉ trả `secretRef` (và có/không tồn tại). Ghi được,
   đọc không được — semantics chuẩn cho credential.
2. **`secretRef` phải được server tự gắn tiền tố `tenant_id`**, tenant không được tự đặt chuỗi ref
   tuỳ ý — nếu không, tenant A đọc secret của tenant B chỉ bằng cách đoán ref. Đây đúng kỷ luật
   `S3ObjectStore` đã áp: `s3.rs:69` build `{tenant_id}/{key}` **bên trong**, không call site nào
   dựng được key chạm sang tenant khác.
3. `SecretStore` hiện **chỉ có một method** `db_credentials(&self, dsn_secret_ref) -> DbCreds`, và
   cả 4 impl đều hardcode payload `{"dsn": "..."}`. Phải thêm một method secret tổng quát
   (`get_secret(ref) -> SecretString`) — không lớn, nhưng là thay đổi trait có 4 impl, phải tính
   vào phạm vi chứ không phát hiện giữa chừng.

## Va chạm trực tiếp với bản vá SSRF A#1 — đã kiểm chứng trong code

Đây là điểm quan trọng nhất và không hiển nhiên. `executor/ssrf_guard.rs` vừa thêm hôm nay
**chặn thẳng header `authorization`** (`FORBIDDEN_HEADERS`, gọi ở `webhook.rs:65`). Mà một webhook
gọi third-party thì gần như luôn cần đúng header đó. **Tính năng secret cho webhook, làm nguyên
trạng, sẽ bị chính bản vá A#1 chặn.**

Không phải lỗi của bản vá — lý do cấm vẫn đúng: header + đích nội bộ = giả mạo credential vào
service nội bộ. Nhưng lý do đó chỉ còn hiệu lực khi đích **có thể** là nội bộ; guard đã chặn riêng
việc đó bằng kiểm tra IP rồi. Với một đích đã qua kiểm tra IP/allowlist, `Authorization` chỉ có thể
mang secret **của chính tenant đó** đến một host **do chính tenant đó chọn** — chuyện bình thường.

Hướng gỡ đề xuất (cần chốt khi làm, không tự quyết bây giờ): **cấm `Authorization` dạng literal,
cho phép dạng `secretRef`**. Tenant không gõ được credential thành text vào config; họ trỏ tới một
secret đã lưu, và executor resolve nó qua `SecretStore` ngay trước khi gửi. Như vậy vừa mở đúng ca
dùng thật, vừa giữ nguyên tính chất mà A#1 cần: giá trị header không đến từ nội dung request, và
không bao giờ đi ngược vào `cron_job_runs` để tenant admin đọc lại. `cookie`/`proxy-authorization`
thì giữ cấm tuyệt đối — không có ca dùng hợp lệ nào.

Kèm theo: cần test khẳng định response body của webhook **không** chứa secret vừa gửi đi, vì
`cron_job_runs` là thứ tenant admin đọc được.

## Theme có đường đọc riêng, không đi chung với các config khác

Theme phải hiển thị **trên màn hình login** — lúc đó browser chưa có session, chưa biết tenant nào.
Nghĩa là nó cần một endpoint **public, không auth**, resolve tenant theo hostname/subdomain, khác
hoàn toàn `GET /admin/config` (đã auth, đã biết tenant từ JWT).

Hệ quả thiết kế: không thể có đúng một endpoint đọc `tenant_configs`. Phải có allowlist tường minh
những khoá **được phép public** (màu, logo, tên hiển thị) và endpoint public chỉ bao giờ trả đúng
tập đó — mọi khoá khác, kể cả khoá plain không phải secret, không bao giờ lọt ra đường này. Cùng
tinh thần "allowlist trong code" ở trên: một cột `is_public` trong DB thì chính nó lại là thứ cần
được bảo vệ.

## Mức "config-global" cho low-code

Đúng như chủ dự án nêu, low-code cần thêm một tầng — và lý do rất cụ thể: trong `metap-lowcode`,
`low_code_entity_versions`/`low_code_entity_drafts` **không có `tenant_id`**. Định nghĩa entity
low-code hôm nay là **global toàn deployment**, không phải per-tenant (xem
`docs/features/15-tenant-scoped-lowcode-metadata.md` — việc tenant-scope hoá chính nó đã là một
brief riêng, chưa làm). Nên config đi kèm định nghĩa low-code thừa hưởng đúng tính chất đó: nó
phải sống ở tầng global, rồi mới cho tenant override bên trên.

Kết quả: chuỗi resolve 3 mức, đúng hình dạng `get_effective_dashboard` đã chạy được với 2 mức:

```
mặc định trong code  ←  platform-global (DB)  ←  per-tenant (DB)  [←  per-user, chỉ nơi có nghĩa]
```

Mỗi tầng chỉ ghi đè khoá nó thực sự khai báo; khoá không khai báo thì rơi xuống tầng dưới. Không
tầng nào được phép ghi đè một khoá thuộc **Tầng 1** — đó là bất biến phải cưỡng chế bằng code
(danh sách khoá tenant-writable là allowlist tường minh trong Rust, không phải cột trong DB —
một cột trong DB thì chính nó lại cần một config để bảo vệ, quay vòng vô hạn).

## Phạm vi

**Trong phạm vi:**
- Một `ConfigKey` registry typed trong Rust: mỗi khoá khai báo tầng cho phép (`Operator` /
  `PlatformGlobal` / `Tenant`), kiểu dữ liệu, giá trị mặc định, validation, **và có phải secret
  hay không, có được đọc public hay không** — 5 thuộc tính, không phải 3.
- Bảng `platform_configs` (global) + `tenant_configs` (per-tenant), cùng shape `jsonb` như
  `tenant_auth_configs` đã dùng.
- `GET/PUT /admin/config` (tenant admin, chỉ thấy/sửa được khoá `Tenant`),
  `GET/PUT /platform/config` (`PlatformAdminContext`, khoá `PlatformGlobal`), và một endpoint
  **public** chỉ trả tập khoá được đánh dấu public, resolve tenant theo hostname (cho theme ở màn
  hình login).
- `SecretStore::get_secret(ref) -> SecretString` — method mới trên trait, 4 impl phải cài.
- Cache đọc: cùng cơ chế đã có (`metap-cache` cho `PermissionService`, `RegistryCache`,
  `ArcSwap` cho `MetadataRegistry`) — **không** query DB mỗi request. Secret **không** cache chung
  đường với config thường.
- Gỡ va chạm với A#1: cho phép `Authorization` dạng `secretRef` trong webhook header, giữ cấm dạng
  literal và cấm tuyệt đối `cookie`/`proxy-authorization`.

**Ngoài phạm vi:**
- Mọi khoá Tầng 0 và Tầng 1 — env var giữ nguyên, cố ý. Nói rõ trong doc rằng đây là *quyết định
  bảo mật*, không phải chưa kịp làm.
- Key-value tự do (`config(key text, value jsonb)` không có schema). Nền tảng này bán đúng luận
  điểm "validation sinh từ metadata"; một bảng config không validate được sẽ là thứ mâu thuẫn với
  chính nó, và sẽ thành bãi rác trong 6 tháng.
- Tenant-scope hoá định nghĩa low-code (`docs/features/15-*`) — brief riêng, phải xong trước thì
  tầng low-code mới có ý nghĩa đầy đủ.

## Tiêu chí chấp nhận

- Một khoá khai báo `Operator` mà bị `PUT /admin/config` cố sửa → 403, và có test khẳng định điều
  đó cho **đúng** `CRON_WEBHOOK_ALLOW_PRIVATE_TARGETS` (regression trực tiếp cho audit 04 A#1).
- `GET /admin/config` của tenant A không bao giờ trả về giá trị của tenant B.
- **`GET` không bao giờ trả về giá trị của một khoá secret**, chỉ trả `secretRef` — có test.
- **Tenant A không resolve được `secretRef` của tenant B**, kể cả khi gửi đúng chuỗi ref của B —
  test trực tiếp, vì đây là ranh giới quan trọng nhất của tính năng.
- Endpoint public chỉ trả đúng tập khoá đánh dấu public; một khoá `Tenant` không-public bị yêu cầu
  qua đường đó → không có trong response (không phải 403 — không tiết lộ cả sự tồn tại).
- Webhook gửi được `Authorization` từ `secretRef` tới host đã allowlist, **và** `cron_job_runs`
  không chứa giá trị secret đó ở bất kỳ đâu — test.
- Khoá không được set ở tầng nào → đọc ra đúng mặc định trong code, không phải `null`/lỗi.
- Đổi `SchemaLimits` qua `/platform/config` có hiệu lực **không cần restart** (cùng cơ chế
  hot-swap `ArcSwap` mà registry đã dùng).
- Đọc config trong một request không thêm query DB nào (chứng minh bằng cache hit, không phải
  bằng lời).

## Ranh giới kiến trúc bị đụng tới

- Bề mặt `/admin/*` mở rộng — thuộc HTTP-API Surface, không đụng `CrudService`.
- **`SecretStore` đổi trait** (`metap-control`) — 4 impl, và `../metap-lowcode`'s
  `reconciler-orchestrator` cũng gọi `build_secret_store`. Thay đổi lan sang repo khác, phải báo
  trước.
- **Không** biến config thành `EntityDefinition`. Cùng lý do `policies`/`cron_jobs`/`dashboard_configs`
  không phải entity: đây là bảng nền tảng, không phải dữ liệu nghiệp vụ của tenant.
- Cần **ADR** cho hai điểm: (1) bất biến "tầng là thuộc tính của khoá, cưỡng chế bằng allowlist
  trong Rust"; (2) nới `FORBIDDEN_HEADERS` cho `secretRef` — vì nó sửa đúng một guard bảo mật vừa
  đặt, phải có lý do ghi lại chứ không lặng lẽ đổi.

## Rủi ro / phụ thuộc

- **Rủi ro lớn nhất là chính sự tiện lợi của nó.** Một khi `/admin/config` tồn tại, áp lực "cho
  cái này per-tenant luôn đi" sẽ đến từ mọi phía, và mỗi lần nhượng bộ với một khoá Tầng 1 là một
  lỗ hổng. Allowlist trong code + test 403 ở trên là để chống lại chính áp lực đó, không phải
  chống người dùng.
- Nới `FORBIDDEN_HEADERS` là chỗ dễ sai nhất trong toàn bộ đề xuất: làm ẩu thì gỡ luôn bản vá A#1
  vừa xong. Phải đi kèm test khẳng định literal vẫn bị chặn.
- Phụ thuộc `docs/features/15-tenant-scoped-lowcode-metadata.md` nếu muốn tầng low-code đầy đủ.

## Thứ tự đề xuất

Không còn là "làm hẹp rồi chờ trigger" — trigger đã có (webhook secret + theme). Nhưng vẫn nên chia
3 lát, vì lát 1 không phụ thuộc gì và lát 3 là lát dễ sai nhất:

1. **Khung + Tầng 2.** `ConfigKey` registry, `platform_configs`, `GET/PUT /platform/config`, gỡ 3
   giá trị hardcode (`SchemaLimits` — đóng luôn audit 04 A#7, rate limit, session TTL). Không đụng
   secret, không đụng `SecretStore`, không đụng A#1.
2. **`tenant_configs` + theme.** Thêm tầng tenant vào chuỗi resolve, endpoint public theo hostname,
   allowlist khoá public. Vẫn chưa đụng secret.
3. **Secret.** `SecretStore::get_secret`, `secretRef` trong config, và nới `FORBIDDEN_HEADERS` cho
   webhook — làm sau cùng, khi khung đã ổn định và có chỗ để test tử tế.
