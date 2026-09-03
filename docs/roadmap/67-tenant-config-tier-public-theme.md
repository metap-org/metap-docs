## Phase 67: tầng config per-tenant + bề mặt branding trước khi login (2026-09-03)

Lát 2/3 của `../features/18-config-tiers-db-backed.md`, làm ngay sau Phase 66. Gồm bảng
`tenant_configs`, `GET/PUT/DELETE /admin/config`, và `GET /public/config` — endpoint **không auth**
resolve tenant theo `Host` để màn hình login hiện đúng branding của tenant.

## Mắt xích thứ ba, không phải bảng thứ hai

Một khoá vẫn khai báo **một** tầng ghi được, nhưng khoá tầng `Tenant` đồng thời nhận **fleet
default** từ tầng platform. Nên một lần đọc đi qua:

```
default khai báo trong Rust  ←  platform_configs  ←  tenant_configs
```

mỗi tầng chỉ ghi đè khoá nó thực sự có row. Đây đúng là mức "config-global" chủ dự án nêu, và là lý
do `PUT /platform/config` giờ nhận cả khoá tenant — **như default, không phải như quyết định**.
Response trả kèm `level`/`tenantOverridable` để platform admin phân biệt được hai thứ đó.

## `auth.sessionTtlSeconds` chuyển xuống tầng tenant — và có test chứng minh nó *được đấu dây*

Chọn khoá này làm bằng chứng vì nó đã được wire sẵn từ lát 1: tenant tự quyết user của **chính nó**
đăng nhập bao lâu thì chỉ ảnh hưởng user của chính nó, còn bound (60s–30 ngày) vẫn nằm trong Rust do
operator kiểm soát — nên cái tệ nhất một tenant admin làm được là chọn một giá trị hợp lệ.

Test e2e `a_tenants_session_ttl_override_changes_the_tokens_it_mints` lấy token thật qua
`GET /auth/token` rồi decode `exp`. Lý do phải có: **một bề mặt config báo cáo giá trị mà không gì
tuân theo còn tệ hơn là không có bề mặt nào** — `/admin/config` trả 600 mà token vẫn 3600 thì lỗi đó
sẽ không ai phát hiện cho tới lúc cần.

## Endpoint public: thu hẹp chứ không tin thêm

Đây là thứ rủi ro nhất lát này — đọc config per-tenant mà không có credential nào. Bốn quyết định:

1. **Khoá nào được public là flag `ConfigKeyDef::public` trên chính khoá, trong Rust.** Khoá thêm về
   sau mặc định là private. Không dùng cột `is_public` trong DB — cột đó lại chính là thứ cần được
   bảo vệ, quay vòng vô hạn (cùng lập luận với tầng ở Phase 66).
2. **Hostname lạ trả về giá trị fleet-wide, không phải 404.** Trả lời khác nhau giữa hostname đã
   đăng ký và chưa đăng ký là biến endpoint này thành **oracle dò sự tồn tại của tenant** cho bất kỳ
   ai gửi được `Host`. Test khẳng định hai response cùng shape.
3. **`X-Forwarded-Host` không bao giờ được đọc.** Header đó ai cũng set được; tôn trọng nó là biến
   thứ duy nhất endpoint này khoá theo thành một tham số tự do.
4. **Mapping hostname → tenant là operator-written** (`control.tenant_hostnames`,
   `dev-tools set-tenant-hostname`), không có bản HTTP. Tenant tự claim hostname tuỳ ý thì claim
   được của tenant khác và phục vụ branding của mình trên màn hình login của họ.

## Validator theme là phần thực chất, không phải phần thủ tục

Ba giá trị này được render vào một trang phục vụ cho bất kỳ ai, nên:

- `theme.primaryColor` chỉ nhận hex literal (`#rgb`/`#rrggbb`). Từ chối cả tên màu và `rgb()` —
  không phải vì chúng nguy hiểm, mà vì cho phép chúng nghĩa là phải parse CSS, còn kiểm tra hex là
  thứ viết đúng được. `#0af; background: url(...)` bị chặn.
- `theme.logoUrl` chỉ nhận `https://` hoặc path site-relative. Chặn `javascript:` (chạy script),
  `data:` (chở được `image/svg+xml`, mà SVG chạy script), `http:` (mixed content + sửa được trên
  đường truyền), và `//host` protocol-relative (browser resolve sang origin của người khác).
- `theme.displayName` giới hạn 64 ký tự, từ chối control character.

## Đọc không chạm DB, kể cả `Router::begin`

Tầng platform vẫn là `ArcSwap`. Tầng tenant là `moka` cache keyed theo tenant, TTL 30s bằng
`RegistryCache` — số tenant không có trần nên snapshot vĩnh viễn là leak chứ không phải cache.
`AppState::effective_config(tenant)` kiểm cache **trước khi** quyết định có mở transaction hay
không, vì `Router::begin` tự nó đã là một round trip.

Hạn chế đã biết, ghi ra chứ không giấu: deployment nhiều instance thì write chỉ thấy ở instance khác
sau TTL (tầng tenant) hoặc phải restart (tầng platform). Cùng đặc tính `RegistryCache` đã có; kênh
invalidation cross-process thì nền tảng này chưa có ở đâu cả.

## Xác minh

`cargo test --workspace` 94/94 suite ok (từ 93), clippy `--workspace --all-targets -D warnings` sạch,
fmt sạch. **7 e2e test mới chưa chạy** — phiên cloud không có Docker/Postgres, cần
`cargo test --workspace -- --ignored`.

> **Sửa số liệu (2026-09-03, phát hiện khi làm Phase 68):** bản đầu của mục này ghi "tổng cộng 37
> test `--ignored` trong repo". **Sai** — đếm thật bằng `cargo test --workspace` rồi cộng các dòng
> `N ignored` cho ra **157**. Con số 37 là ước lượng chứ không phải kết quả đo, và nó làm mức phủ
> e2e chưa-chạy của repo trông nhỏ hơn thực tế đáng kể.

Disk quota hết lần thứ 4 trong ngày giữa phiên, `cargo clean` giải phóng 30.5GB.

## Còn lại

Lát 3 (`SecretStore::get_secret` + `secretRef` + nới `FORBIDDEN_HEADERS` cho webhook secret) là phần
duy nhất còn lại của brief 18 — và là lát dễ sai nhất, vì nó sửa đúng một guard bảo mật vừa đặt.

**FE chưa đụng.** `@metap/platform-ui` chưa đọc `GET /public/config`; màn hình login vẫn dùng theme
mặc định của design system. Backend đã sẵn sàng, phần FE là việc riêng ở repo `../platform-ui`.
