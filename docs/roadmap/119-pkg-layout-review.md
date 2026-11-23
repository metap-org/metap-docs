# 119. Rà lại bố cục `pkg/` của metap: nên gộp, tách, đổi tên thế nào

Chủ dự án yêu cầu (2026-11-23), sau roadmap 118: rà toàn bộ `pkg/` và đề xuất cách sắp xếp/gộp cho hợp lý. Tài liệu này ghi
hiện trạng đo từ mã (`go list`, đồ thị import, số dòng), những chỗ lệch, và đề xuất theo thứ tự. **Chưa sửa gì**; metap và
waap chưa phát hành ổn định nên được phép đổi đường dẫn import (waap và lowcode sửa theo trong PR đi kèm).

## 1. Hiện trạng

53 package dưới `pkg/` (cộng `internal/admin`, `internal/signin`, 6 binary dưới `cmd/`), khoảng 50 nghìn dòng mã (không tính test). Đồ thị import
**không có vòng** và phần lõi dữ liệu có tầng rõ: `kit` → `metadata`/`permission` → `query`/`workflow`/`audit` → `crud` →
`httpserver`. Vấn đề không nằm ở phụ thuộc mà ở **tên và chỗ đặt**: 30 package phẳng cùng cấp, trong đó một số là ngăn kéo
(`peripherals`, `infra`, `control`), một số chỉ có một người dùng, và cùng một việc có hai nơi.

Ai dùng gì bên ngoài module: waap dùng nhiều nhất `crud` (43 chỗ), `permission` (35), `metadata` (24), `query` (20),
`reconciler` (18), `httpserver` (18). lowcode dùng `httpserver` (17), `metadata` (16), `reconciler` (9), `control` (8),
`infra` (5), `secrets`, `peripherals`. Các package còn lại chỉ metap tự dùng.

## 2. Những chỗ lệch (đo được)

| # | Chỗ lệch | Bằng chứng |
|---|---|---|
| 1 | **`peripherals` là ngăn kéo**: vai trò và vai trò theo phạm vi, người dùng của tenant, preferences, thuộc tính ngữ cảnh, kiểm tra drift metadata, và **ký/giải mã JWT** | 6 file, 965 dòng, 10 nơi import; `jwks` (8 file) cũng ký/xác minh token → **hai đường token**: `peripherals.MintJWT/Decode` (RSA tĩnh) và `jwks` (EdDSA, JWKS); `jwks` tự gọi lại `peripherals` |
| 2 | **`control` ôm bốn việc**: sổ tenant + router tenant + provisioning (đúng tên); **phân giải RequestContext** (`authcontext`); **kho policy Postgres** (`policystore`); **giao diện `SecretStore`** (và bản `userpass`) | 14 file, 9 nơi import; `secrets` (cài đặt kho bí mật) phải import `control` chỉ để có giao diện |
| 3 | **Hai nơi cấu hình**: `infra.AppConfig` (biến môi trường lúc khởi động) và `config` (tầng key trong DB + file YAML) | `infra` 366 dòng; `config` import `kit/env`, `infra` import `config` |
| 4 | **`infra` là ngăn kéo thứ hai**: kết nối pool, health, outbox (ghi), telemetry (cấu hình slog) | lowcode dùng `infra.InitTracing`, `infra.OutboxEvent`, `infra.Connect` |
| 5 | **Cụm nhắn tin rải 5 chỗ**: `infra/eventbus`, `events`, `infra` (outbox), `outboxpublisher`, `notificationworker` | 5 package, tổng 1,1 nghìn dòng, mỗi chỗ 38–460 dòng |
| 6 | **Cụm danh tính rải 7 chỗ**: `identity`, `tenantauth` (cấu hình SSO), `oauthserver`, `jwks`, `captcha`, `accountmail`, và phần ký token trong `peripherals` | không import vòng nhưng không có tên chung; muốn hiểu "đăng nhập" phải mở 7 thư mục |
| 7 | **Một người dùng duy nhất**: `attachments` (156 dòng, chỉ `attachmentroutes`), `dashboards` (111, chỉ `dashboardroutes`), `cache` (178, gần như chỉ `permission`), `storage` + `storage/s3store`, `notificationworker` (38), `jobs` (một file 14 dòng chỉ có doc) | `in=1` trong đồ thị; `pkg/jobs/doc.go` không có mã |
| 8 | **Tên không nói việc**: `runtime` (metrics Prometheus + rate limit), `apigateway`, `app` (bootstrap dịch vụ), `recordclient` | `runtime` chỉ 142 dòng, do `httpserver` và `apigateway` dùng |
| 9 | **Phần lõi dữ liệu đúng nhưng chưa có nhóm**: `metadata`, `permission`, `query`, `workflow`, `audit`, `crud`, `reconciler` | đều đúng tầng, waap dùng trực tiếp; đổi tên sẽ phá nhiều import nhất, ít lợi nhất |

## 3. Đề xuất

Nguyên tắc: **gom theo việc người đọc muốn hiểu, không gom theo kiểu kỹ thuật**; mỗi nhóm một thư mục cha; không đụng phần
lõi dữ liệu và `kit` (đúng và nhiều người dùng); không tạo package mới chỉ để có chỗ chứa.

### 3.1 Thay `peripherals` bằng chỗ đúng của từng phần (làm đầu tiên, giá trị cao nhất)

| Phần hiện ở `peripherals` | Đi đâu |
|---|---|
| `auth.go` (ký/giải mã JWT RSA tĩnh) | gộp vào `jwks` (đổi tên cả package thành `token`): một đường ký, một đường xác minh; RSA tĩnh thành một `KeySource` cạnh EdDSA |
| `roles.go`, `role_assignment.go` (vai trò, vai trò theo phạm vi) | `permission/rolestore` hoặc `identity/roles` (cần chốt: vai trò là dữ liệu truy cập → ưu tiên `permission/rolestore`) |
| người dùng của tenant, `preferences.go`, `context_attributes.go` | `identity` (đã có thành viên, phiên, MFA) |
| `metadata_drift.go` | `metadata` (nó chỉ so định nghĩa entity với bản đã lưu) |

### 3.2 Tách `control` theo việc

- `tenancy` (đổi tên phần còn lại): sổ tenant, router, provisioning, schema tenant, hostname, cache.
- `SecretStore` + `userpass` + bản bộ nhớ → `secrets` (hết việc `secrets` phải import `control`).
- `policystore.go` (kho policy Postgres) → `permission` (cạnh giao diện `PolicyStore`).
- `authcontext.go` (phân giải RequestContext từ token, API key, phiên) → `identity/authctx` hoặc `httpserver/auth` (nó là cầu nối token → quyền; chỉ `httpserver` và vài tool gọi).

### 3.3 Một chỗ cho cấu hình và hạ tầng

- `infra.AppConfig` + `infra.LoadConfig` → `config` (một package cấu hình: khởi động, tầng DB, file). `kit/env` giữ nguyên.
- `infra.Connect` + health → `db` (cùng chỗ `dbmigrate`, `testkit`'s helper): `pkg/db`, `pkg/db/migrate`.
- `infra.InitTracing` → `kit/slogx` hoặc `app`.
- `infra` biến mất.

### 3.4 Gom cụm nhắn tin: `pkg/messaging`

`messaging/eventbus` (từ `infra/eventbus`), `messaging/events` (từ `events`), `messaging/outbox` (ghi từ `infra` + vòng
publisher từ `outboxpublisher`), `messaging/notify` (từ `notificationworker`). `jobs/cron` và `jobs/scheduler` giữ (đã gom
2026-11-23); bỏ `pkg/jobs/doc.go` hoặc đặt tài liệu nhóm vào README.

### 3.5 Gom cụm danh tính: `pkg/auth`

`auth/identity`, `auth/token` (từ `jwks`), `auth/sso` (từ `tenantauth`), `auth/oauthserver`, `auth/captcha`, `auth/mail` (từ
`accountmail`). Giữ `internal/admin` và `internal/signin` như đã chốt (chúng dùng cả nhóm này).

### 3.6 Gom các tính năng tùy chọn có một người dùng

`attachments`, `dashboards` chuyển vào thư mục route tương ứng (`httpserver/attachmentroutes`, `httpserver/dashboardroutes`)
hoặc thành `features/...`: chưa có người dùng thứ hai nên không cần là package công khai riêng. `cache` thành `permission/cache`
nếu vẫn chỉ `permission` dùng (kiểm lại `control`'s RegistryCache trước khi làm). `storage` + `s3store` giữ (tách để không kéo
AWS SDK vào mọi dịch vụ, đã ghi trong `CLAUDE.md`).

### 3.7 Đổi tên cho nói đúng việc

`runtime` → `httpmw` (metrics + rate limit là middleware HTTP); `apigateway` giữ (đã nói đúng); `app` giữ (đã quen, nhiều
tài liệu nhắc); `recordclient` giữ.

### 3.8 Không đổi

`kit/*`, `metadata`, `permission` (trừ phần thêm ở 3.1/3.2), `query`, `workflow`, `audit`, `crud`, `reconciler`, `config`
(trừ phần thêm), `httpserver` và 4 `*routes`, `bootstrap`, `testkit`, `events` payload (chỉ đổi đường dẫn).

## 4. Hình dạng sau khi làm

```
pkg/
  kit/            stdlib-only (không đổi)
  metadata permission query workflow audit crud reconciler     lõi dữ liệu (không đổi tên)
  tenancy/        sổ tenant, router, provisioning
  auth/           identity, token, sso, oauthserver, captcha, mail
  messaging/      eventbus, events, outbox, notify
  jobs/           cron, scheduler
  secrets/ storage/ cache(?)   (secrets nhận SecretStore)
  config/         khởi động + tầng DB + file
  db/             pool, health, migrate
  httpserver/ httpmw/ apigateway/ recordclient/ app/ bootstrap/ testkit/
internal/         admin, signin
```
53 package còn khoảng 40; không còn ngăn kéo (`peripherals`, `infra`) và `control` chỉ còn đúng nghĩa tenancy.

## 5. Chi phí và thứ tự

Mỗi bước là một PR, giữ hành vi, chạy `go test ./...` + tích hợp + `golangci-lint`; waap và lowcode sửa import trong PR
đi kèm (waap có CI, lowcode chưa build được với toolchain hiện tại nên cần kiểm riêng). Ước lượng theo số chỗ import cần sửa
(đo bằng `grep`), không tính thời gian đọc:

| Bước | Việc | Ảnh hưởng ngoài module |
|---|---|---|
| 1 | 3.1 bỏ `peripherals`, một đường token | waap 1 chỗ, lowcode 3 chỗ |
| 2 | 3.2 tách `control` | waap 4, lowcode 8 |
| 3 | 3.3 bỏ `infra`, `config`+`db` | waap 2, lowcode 5 |
| 4 | 3.4 `messaging` | waap 3, lowcode 0 |
| 5 | 3.5 `auth` | 0 (chỉ metap) |
| 6 | 3.6, 3.7 gọn phần một người dùng, đổi tên `runtime` | 0 |

Làm theo thứ tự 1→6: 1–3 sửa chỗ **lệch thật** (hai đường token, ngăn kéo, hai nơi cấu hình); 4–6 là sắp xếp cho dễ đọc, có
thể hoãn. Mỗi bước thêm luật depguard giữ kết quả (ví dụ cấm import `pkg/peripherals` đã xóa, cấm `auth/*` import `httpserver`).

## 6. Điều cần chốt

1. Đồng ý bỏ `peripherals` và `infra` và gom vào chỗ đúng như §3.1–3.3? (đề xuất: có, đây là phần giá trị nhất)
2. Vai trò (roles) nên ở `permission/rolestore` hay `identity/roles`? (đề xuất: `permission/rolestore`)
3. Có làm nhóm `auth/` và `messaging/` (§3.4–3.5) không, hay chỉ sửa các chỗ lệch ở §3.1–3.3? (đề xuất: làm sau, tách PR)
4. `authcontext` ở `identity/authctx` hay `httpserver/auth`? (đề xuất: `httpserver/auth`, vì chỉ transport cần nó)
