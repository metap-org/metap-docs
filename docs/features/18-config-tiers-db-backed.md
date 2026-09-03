# Config trong database, phân tầng operator / platform-global / per-tenant

- **Trạng thái:** proposed — chưa code, cần chốt phạm vi trước
- **Người đề xuất:** chủ dự án, 2026-09-03 ("những config như này có thể set trong database, đầu
  API admin/config hay gì đó - per tenant, ngoài ra thì với lowcode thì có thêm mức config-global")
- **Track sở hữu:** HTTP-API Surface (bề mặt `/admin/*`) + Backend Core (tầng đọc/cache)
- **Phase roadmap liên quan:** không thuộc phase nào

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
  chính là audit 04 **B#7**, và doc comment của `Default` tự nói "starting guardrails, not a
  permanent tuning".
- Rate limit (200ms / burst 300) — hardcode 2 chỗ: `metap-http/src/lib.rs:73`,
  `graphql-gateway/src/server.rs:131`.
- `TOKEN_TTL_SECONDS = 3600` — hardcode ở `metap-http/src/routes/auth.rs:21`.
- Từ env: `POLICY_CACHE_TTL_SECONDS`, `AUTH_CONTEXT_CACHE_TTL_SECONDS`, `CRON_TICK_MS`,
  `CRON_BATCH_SIZE`, `OUTBOX_POLL_MS`, `OUTBOX_BATCH_SIZE`.

**Tầng 3 — thật sự per-tenant.** Danh sách này **gần như rỗng nếu chỉ nhìn env var** — bằng chứng
cho nhận định ở trên. Ứng viên thật đến từ các hằng số tầng 2 khi cần khác nhau giữa các tenant:
TTL session (tenant enterprise muốn ngắn hơn), rate limit riêng theo gói dịch vụ, `max_limit` của
list view, locale mặc định của tenant. Cộng với những gì **đã** per-tenant và đã nằm trong DB rồi
(`tenant_auth_configs`, `dashboard_configs`).

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

**Trong phạm vi (nếu chốt làm):**
- Một `ConfigKey` registry typed trong Rust: mỗi khoá khai báo tầng cho phép (`Operator` /
  `PlatformGlobal` / `Tenant`), kiểu dữ liệu, giá trị mặc định, và validation.
- Bảng `platform_configs` (global) + `tenant_configs` (per-tenant), cùng shape `jsonb` như
  `tenant_auth_configs` đã dùng.
- `GET/PUT /admin/config` (tenant admin, chỉ thấy/sửa được khoá `Tenant`) và
  `GET/PUT /platform/config` (`PlatformAdminContext`, khoá `PlatformGlobal`).
- Cache đọc: cùng cơ chế đã có (`metap-cache` cho `PermissionService`, `RegistryCache`,
  `ArcSwap` cho `MetadataRegistry`) — **không** query DB mỗi request.
- Bắt đầu bằng đúng 3 khoá tầng 2 có giá trị rõ ràng và đang hardcode: `SchemaLimits`, rate limit,
  session TTL.

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
- Khoá không được set ở tầng nào → đọc ra đúng mặc định trong code, không phải `null`/lỗi.
- Đổi `SchemaLimits` qua `/platform/config` có hiệu lực **không cần restart** (cùng cơ chế
  hot-swap `ArcSwap` mà registry đã dùng).
- Đọc config trong một request không thêm query DB nào (chứng minh bằng cache hit, không phải
  bằng lời).

## Ranh giới kiến trúc bị đụng tới

- Bề mặt `/admin/*` mở rộng — thuộc HTTP-API Surface, không đụng `CrudService`.
- **Không** biến config thành `EntityDefinition`. Cùng lý do `policies`/`cron_jobs`/`dashboard_configs`
  không phải entity: đây là bảng nền tảng, không phải dữ liệu nghiệp vụ của tenant.
- Cần **ADR** cho đúng một điểm: bất biến "tầng là thuộc tính của khoá, cưỡng chế bằng allowlist
  trong Rust" — vì nó là thứ duy nhất ngăn bề mặt tiện lợi này trở thành đường vòng qua mọi guard
  operator đã đặt.

## Rủi ro / phụ thuộc

- **Rủi ro lớn nhất là chính sự tiện lợi của nó.** Một khi `/admin/config` tồn tại, áp lực "cho
  cái này per-tenant luôn đi" sẽ đến từ mọi phía, và mỗi lần nhượng bộ với một khoá Tầng 1 là một
  lỗ hổng. Allowlist trong code + test 403 ở trên là để chống lại chính áp lực đó, không phải
  chống người dùng.
- Phụ thuộc `docs/features/15-tenant-scoped-lowcode-metadata.md` nếu muốn tầng low-code đầy đủ.
- Nếu chỉ làm 3 khoá tầng 2 (khuyến nghị), không phụ thuộc gì và không đụng ai.

## Khuyến nghị

Làm **bản hẹp**: 3 khoá Tầng 2 đang hardcode (`SchemaLimits` — đóng luôn audit 04 B#7, rate limit,
session TTL), một bảng `platform_configs`, một `GET/PUT /platform/config`. Chưa làm
`tenant_configs` cho tới khi có một nhu cầu per-tenant *thật* (hôm nay khảo sát ra gần như không
có — thứ per-tenant thật thì đã nằm trong DB sẵn rồi). Như vậy vừa gỡ được cái đang thực sự vướng
(giá trị hardcode ở bề mặt public nhất), vừa không dựng sẵn một bề mặt per-tenant chưa ai cần —
đúng quy ước "không xây trước khi có trigger" của `docs/team-charter.md`.
