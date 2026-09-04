## Phase 68: secret cho webhook — `SecretStore` tổng quát, config write-only, nới `FORBIDDEN_HEADERS` (2026-09-03)

Lát 3/3, lát cuối của `../features/18-config-tiers-db-backed.md`. Cũng là lát **dễ sai nhất** như
brief đã cảnh báo: nó sửa đúng một guard bảo mật vừa đặt cùng ngày (audit 04 A#1).

## Reference được *suy ra*, không phải được *gửi lên*

Brief yêu cầu: "`secretRef` phải được server tự gắn tiền tố `tenant_id`, tenant không được tự đặt
chuỗi ref tuỳ ý". Làm mạnh hơn thế: **suy ra toàn bộ chuỗi**.

`metap_control::tenant_secret_ref(tenant_id, key)` nhận `tenant_id` từ token đã verify và `key` từ
path của route. Không có field nào trong request mang reference cả — nên không có gì để prefix, để
validate, hay để làm sai. Yêu cầu "tenant A không resolve được ref của tenant B, kể cả khi gửi đúng
chuỗi ref của B" trở thành **bất khả thi về mặt cấu trúc** chứ không phải một check phải nhớ viết.
Cùng kỷ luật `S3ObjectStore` đã áp (`{tenant_id}/{key}` dựng bên trong).

Format của reference là **giao** của những gì cả 4 backend chấp nhận (`METAP_TENANT_<32hex>_<KEY>`):
`EnvStore` cần tên hợp lệ kiểu POSIX, GCP Secret Manager chỉ cho `[A-Za-z0-9_-]`, AWS và Vault rộng
hơn cả hai. Một format cho bốn backend nghĩa là credential của một tenant resolve y hệt nhau bất kể
deployment chạy backend nào.

## Config layer không bao giờ giữ credential

Khoá khai báo `secret: true` có hình dạng khác hẳn: `PUT` nhận **plaintext**, plaintext đi thẳng
sang `SecretStore`, và thứ nằm trong `tenant_configs` chỉ là `{"secretRef": ...}`.

Nên "đọc không bao giờ trả về giá trị" là **tính chất cấu trúc**, không phải một filter ai đó phải
nhớ áp: plaintext không có mặt trong dữ liệu của `metap-config` ở bất kỳ thời điểm nào, nên không
đường đọc nào ở đó có thể trả về nó kể cả khi viết sai. `set_tenant` (đường ghi config thường) từ
chối thẳng khoá secret, để tính chất đó là của crate chứ không phải quy ước người dùng crate phải
tuân.

Trong response, entry của khoá secret **bỏ hẳn field `value`** thay vì để rỗng — client không thể
nhầm chuỗi rỗng là credential thật.

## `EnvStore` từ chối ghi, và đó là hành vi đúng

`put_secret`/`delete_secret` trên `EnvStore` trả lỗi. Ghi ở đó nghĩa là `std::env::set_var` trên
process đang chạy: process khác không thấy, restart là mất, không audit được. Tenant admin sẽ set
credential qua API, thấy thành công, rồi phát hiện nó biến mất sau lần deploy kế tiếp — kiểu lỗi
sẽ đốt cháy ai đó.

Từ chối biến việc đó thành một thông báo lỗi nêu đúng tên biến cần export. Và `get_secret` **vẫn
chạy** trên `EnvStore`, nên câu chuyện vẫn mạch lạc: deployment mặc định = operator provision bằng
env var (`dev-tools put-tenant-secret` in ra tên biến), tenant dùng; deployment có Vault/AWS/GCP =
tenant tự phục vụ.

## Nới `FORBIDDEN_HEADERS`: phân biệt literal và secret

`authorization` **vẫn nằm trong** `FORBIDDEN_HEADERS`. Cái được mở là một đường khác hẳn:

| | Literal trong `target_config` | Giá trị từ `SecretStore` |
|---|---|---|
| Nguồn | Text tenant gõ | Credential của chính tenant đó |
| Đích | Bất kỳ | Host đã qua kiểm tra IP + allowlist |
| Đọc lại được? | — | Không (`get_secret` không có bề mặt HTTP) |
| Vào `cron_job_runs`? | — | Không (`webhook::redact`) |

Literal = text kẻ tấn công chọn, gửi tới host kẻ tấn công chọn → giả mạo credential, **vẫn cấm**.
`cookie`/`proxy-authorization` cấm tuyệt đối: không có ca dùng hợp lệ nào cho một job theo lịch
mang session cookie hay proxy credential, nên mở ra chẳng được gì mà mất một đường tấn công.

**Thứ tự là phần chịu lực**, và được comment ngay tại chỗ: target phải qua `WebhookPolicy::check`
**trước** khi credential được đọc. Đảo hai dòng đó là biến tính năng này thành lại đúng primitive
A#1 đã đóng. Có test giữ literal vẫn bị chặn ở mọi cách viết hoa/thường.

`cron-scheduler` **không đọc `tenant_configs`** — nó suy ra reference từ `payload.tenant_id` của
chính job. Nên nó không cần DB access theo tenant, và job không thể nêu tên credential của ai khác
vì nó không nêu tên credential nào cả (`authorizationFromSecret` là một boolean, không phải chuỗi).

## `redact`: tenant admin vừa set được URL vừa đọc được `cron_job_runs`

Credential đi trong *request*, không phải response — nên về nguyên tắc nó không quay lại. Nhưng
tenant admin là người **chọn URL webhook**, nên "trỏ job vào một server echo header" là cách một
bước để đọc ngược credential ra khỏi một bề mặt cố tình write-only. `redact` thay mọi lần xuất hiện
của credential trong response body bằng `[REDACTED]` trước khi nó chạm `cron_job_runs`.

Là lớp phòng thủ phụ chứ không phải chính (credential đáng ra không quay lại), và vì credential đó
là *của chính tenant* nên rò rỉ này nhỏ — nhưng vẫn không phải thứ nên trao tay. Có test cho cả ca
`secret` rỗng (nếu không, `replace("")` sẽ chèn marker vào giữa từng ký tự của body).

## Validator credential là chống header injection

`header_credential` từ chối CR/LF. Không phải chuyện gọn gàng: giá trị này được gửi **verbatim** làm
HTTP header value, nên `\r\n` trong đó là header injection vào chính request mà platform thực hiện
thay mặt tenant — có thể chèn header tuỳ ý, hoặc cả một request thứ hai. `reqwest` chắc cũng từ chối
phần lớn, nhưng nền tảng này không đi mượn validation của một dependency cho một ranh giới nó sở
hữu, và từ chối lúc ghi cho tenant lỗi ngay lúc sửa được thay vì một cron run sau.

## Hạn chế đã biết, ghi ra chứ không giấu

**Một credential cho mỗi khoá khai báo, cho mỗi tenant.** Tenant gọi cả Stripe lẫn Slack với hai
credential khác nhau thì chưa làm được — cần một bề mặt secret có tên (`/admin/secrets/{name}`), là
follow-up chứ không phải lát này. Phần khó (suy ra reference, write-only, resolve đúng thời điểm,
allowlist header) đã xong và tổng quát hoá được.

## Xác minh

`cargo test --workspace` **95/95 suite ok** (từ 94), `clippy --workspace --all-targets -D warnings`
sạch, `fmt --check` sạch.

Unit test mới đáng kể: `a_literal_authorization_header_stays_refused_after_the_secret_path_was_added`
(regression A#1 cho lát này), `a_secret_ref_is_derived_from_the_tenant_and_never_collides`,
`no_key_is_both_secret_and_public`, `a_credential_with_a_newline_is_refused_as_header_injection`,
`env_store_refuses_to_write_a_secret_and_says_what_to_set_instead` (và khẳng định value không lọt
vào chuỗi lỗi), `redact_*` (gồm ca secret rỗng).

**Một lỗi test tự bắt được, đáng ghi lại**: `every_writable_default_passes_its_own_validator` fail
khi thêm khoá secret — vì với khoá secret, `default` (marker `{}` đã lưu) và `validate` (plaintext
caller gửi) **nói về hai giá trị khác nhau**, chạy cái này với cái kia là vô nghĩa. Đã tách thành 2
test đúng ngữ nghĩa và ghi rõ điều đó vào doc comment của `ConfigKeyDef::default`. Test bắt được một
điểm thiết kế mơ hồ chứ không phải một bug — nhưng nếu để nguyên thì người sửa tiếp theo sẽ hiểu sai
hai field đó.

### Chưa verify được trong phiên này

- **6 e2e test mới chưa chạy** (`crates/metap-http/tests/tenant_secret_postgres.rs`) — cần
  `DATABASE_URL` + Postgres thật. Phiên cloud không có Docker.
- **Chưa chạy đường webhook thật đầu-cuối**: chưa có run nào thực sự gửi `Authorization` từ
  `SecretStore` tới một host đã allowlist, và chưa xác nhận sống rằng `cron_job_runs` không chứa
  credential. Logic có test (`redact` unit test, thứ tự check trong `run_webhook` đọc được), nhưng
  **đây là phần đáng verify sống nhất của cả lát** — cần `cron-scheduler` + Postgres + một endpoint
  echo.
- **3 backend Vault/AWS/GCP chưa chạy thật** với `get/put/delete_secret`. `EnvStore` có unit test;
  ba cái kia mới chỉ compile. Vault có service trong CI (`.github/workflows/ci.yml`, thêm từ Phase
  11), nên đường đó verify được khi CI chạy; AWS/GCP thì không.
- Tổng repo hiện có **157 test `--ignored`** chưa chạy trong phiên này (không chỉ của phase này) —
  xem ghi chú sửa số liệu ở `67-tenant-config-tier-public-theme.md`.

Disk quota hết **lần thứ 5 và 6** trong ngày giữa phiên (`cargo clean` giải phóng 31–32GB mỗi lần;
lần thứ 6 là `/tmp` của harness đầy chứ không phải `.shared-target`). Quota thật của phiên ~29GB,
tức là một lượt `cargo build --workspace --tests` gần chạm trần — đáng ghi vào CLAUDE.md nếu còn tái
diễn.

### Cập nhật 2026-09-04: 1 bug test tự tạo, phát hiện lúc verify phiên WAF

Chạy `cargo test -p metap-http -- --ignored` thật lần đầu (Postgres native, phiên `metap-demo-waf`)
phát hiện `platform_config_postgres.rs::an_operator_key_is_refused_even_for_a_platform_admin` và
`tenant_config_postgres.rs::a_tenant_admin_cannot_write_an_operator_or_fleet_key` cùng fail — cả 2
assert chặn cứng "không key nào bắt đầu bằng `cron.`" trên listing của `/platform/config`/
`/admin/config`, viết **trước** khi `CRON_WEBHOOK_AUTHORIZATION` (`cron.webhookAuthorization`,
phase này) tồn tại. Khoá đó là `ConfigLevel::Tenant`, không phải `Operator`, nên nó lên listing của
cả 2 surface hợp lệ (`ConfigStore::platform_writable_view`/`tenant_view` chỉ loại `Operator`). Sửa
2 assert thành kiểm tra đúng tên 2 khoá `Operator` (`cron.webhookAllowPrivateTargets`/
`cron.webhookAllowedHosts`) thay vì so khớp tiền tố — cả 2 test pass thật trên Postgres sống sau
khi sửa.
