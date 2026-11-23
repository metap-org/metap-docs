# Audit 07 — SSO, phiên đăng nhập, credential, SSRF (metap Go, 2026-11-23)

Phạm vi: mã Go mới nhất của `metap` sau roadmap 115/116 — đăng nhập một lần (OIDC + SAML,
`pkg/tenantauth`, `pkg/httpserver/routes_sso.go`, `pkg/adminapi/sso_admin_service.go`), xác minh
domain, đổi mật khẩu và khoá tài khoản, xoá workspace, mã hoá credential của tenant
(`pkg/secrets`), SSRF (`pkg/kit/safehttp`, `pkg/cronscheduler/ssrfguard.go`), rate limit.
Phương pháp: đọc mã từng dòng và lần theo luồng; **chưa chạy test hay khai thác thật** — mỗi mục
dưới đây là kết luận từ đọc mã, cần một test tái hiện trước khi sửa.

## Tóm tắt

Phần lõi chắc: xác thực SAML, ràng buộc state, kiểm soát vai trò từ IdP, xác minh domain, mã hoá
credential đều làm đúng. Không có finding HIGH. Có **1 MEDIUM** và 5 LOW/INFO, phần lớn là chỗ
hai cơ chế cùng việc đã lệch nhau.

| # | Mức độ | Vấn đề | Trạng thái |
|---|---|---|---|
| 1 | MEDIUM | `POST /auth/password` kiểm mật khẩu hiện tại bằng `identity.CheckPassword`, không tính vào khoá tài khoản, và route nằm ngoài nhóm `guarded` → người giữ được một session (cookie/token bị trộm) dò mật khẩu không bị khoá | Đã sửa (metap #80): `identity.VerifyCurrentPassword` đếm lần sai cùng bộ đếm khoá với đăng nhập |
| 2 | LOW | Login CSRF ở callback SSO: `state` dùng một lần nhưng không gắn với trình duyệt đã bắt đầu đăng nhập | Đã sửa OIDC (metap #81): cookie `metap_sso_bind` gắn state với trình duyệt; SAML giữ rủi ro đã chấp nhận (ACS là POST chéo site) |
| 3 | LOW | Hai danh sách chặn IP khác nhau cho cùng một việc (SSRF): `cronscheduler.IsBlockedIP` yếu hơn `safehttp.PublicAddress`, và cả hai chưa chặn NAT64/6to4/Teredo | Đã sửa (metap #80): một danh sách `safehttp.PublicAddress`, thêm NAT64/6to4/Teredo |
| 4 | LOW | Route SSO công khai chỉ qua bucket rate limit chung (burst 300), không qua bucket `guarded`; mỗi lượt `start` ghi một dòng DB và (OIDC) gọi ra IdP | Đã sửa (metap #80): `discover`/`start` dùng chung bucket `guarded` |
| 5 | LOW | Xoá workspace chỉ cần gõ tên, không yêu cầu nhập lại mật khẩu/MFA | Đã sửa (metap #81, waap #73): xoá workspace cần mật khẩu của owner, và mã MFA nếu có |
| 6 | INFO | SAML dùng `NameID` làm `external_subject`; với format `emailAddress` thì định danh = email | Đã sửa 2026-11-23: tenant chọn `nameIdFormat` = `persistent`; NameID mờ là subject, địa chỉ chỉ đến từ attribute (bắt buộc khai `emailAttribute`); mặc định vẫn `emailAddress` |

## Chi tiết

### 1. Đổi mật khẩu không bị khoá khi đoán sai (MEDIUM)

`changePassword` (`pkg/httpserver/routes_auth.go`) gọi `identity.CheckPassword`, hàm này chỉ trả
`bool` và **không** gọi `recordFailure`, khác với đường đăng nhập (`identity.go`: 5 lần sai khoá 15
phút). Route cũng được gắn bằng `r.Post`, không phải `guarded.Post`, nên chỉ chịu bucket chung
(5 req/s/IP, burst 300). Kẻ có một session hợp lệ — cookie bị trộm qua XSS ở site khác, máy dùng
chung — có thể thử mật khẩu hiện tại không giới hạn lần. Argon2 làm chậm mỗi lượt nhưng không phải
là cơ chế khoá. Hệ quả là lộ mật khẩu, thường dùng lại ở dịch vụ khác, chứ không chỉ chiếm
workspace này.

Sửa: dùng chung đường verify có đếm lỗi/khoá với đăng nhập (một hàm duy nhất, không hai bản), đưa
route vào `guarded`, và yêu cầu step-up MFA khi người đó đã bật. Kiểm các route cùng kiểu: tắt
MFA / tạo lại recovery code (`/me/mfa*`) cũng nhận mã hiện tại — xác nhận chúng đếm lần sai (chưa
đọc).

### 2. Login CSRF ở callback SSO (LOW)

`ssoCallback` và `samlACS` tiêu thụ `state` một lần và kiểm tenant, nhưng không kiểm rằng trình
duyệt đang gọi callback chính là trình duyệt đã gọi `start`. Kịch bản: kẻ có tài khoản ở IdP của
tenant T bắt đầu luồng, đăng nhập ở IdP bằng tài khoản của mình, giữ lại URL callback (OIDC) hoặc
form POST (SAML) rồi dụ nạn nhân mở nó → nạn nhân bị đăng nhập vào **tài khoản của kẻ tấn công**
trong T và có thể nhập dữ liệu vào đó. Cần có tài khoản IdP trong tenant, nên rủi ro hẹp.

Sửa cho OIDC: ở `start` đặt cookie `__Host-` ngắn hạn chứa mã băm của `state`, kiểm ở callback
(callback là GET top-level, cookie `SameSite=Lax` vẫn gửi). SAML ACS là POST cross-site từ IdP nên
cần `SameSite=None`, đánh đổi lớn hơn lợi ích — chấp nhận rủi ro cho SAML và ghi lại.

### 3. Hai danh sách chặn IP lệch nhau (LOW)

`safehttp.PublicAddress` (dùng cho origin/DNS của khách) chặn loopback, private, link-local,
multicast, `100.64/10`, `0/8`, `192.0.0/24`, `198.18/15`, `240/4`, `fc00::/7`, `fe80::/10`.
`cronscheduler.IsBlockedIP` — dùng cho **webhook** và cho **tải metadata SAML** do admin tenant
nhập URL (`metadataHTTPClient`) — thiếu `0/8` (trừ `0.0.0.0`), `192.0.0/24`, `198.18/15`, `240/4`.
Không đường nào chặn `64:ff9b::/96` (NAT64), `2002::/16` (6to4), Teredo: trên mạng IPv6-only có
DNS64 (nhiều cụm Kubernetes), một tên trả về địa chỉ NAT64 trỏ vào mạng nội bộ IPv4 sẽ lọt qua.

Cả hai đều kiểm tại thời điểm dial (đúng, không bị DNS rebinding) và webhook/metadata không theo
redirect ra địa chỉ chặn, nên đây là lỗ hổng độ rộng của danh sách, không phải kiến trúc. Sửa: một
danh sách duy nhất (`safehttp`), `cronscheduler` và `metadataHTTPClient` gọi vào đó; thêm ba dải
IPv6 trên, và một test bảng chung cho mọi dải.

### 4. Route SSO công khai thiếu bucket nghiêm (LOW)

`mountSSO` gắn route lên router gốc (bucket chung 200 ms/burst 300). Các route đoán credential
khác dùng `guarded` (1 lượt/3 s, burst 20). `/auth/sso/start` và `/auth/saml/{tenant}/start` mỗi lần
ghi một dòng `metadata.sso_states` (dọn dòng hết hạn ngay trong lượt tạo, có index nên rẻ), và
OIDC còn gọi discovery tới IdP của tenant — một client vô danh làm IdP của tenant bị gọi dồn.
`/auth/sso/discover` cho biết domain nào có SSO và `tenantId` tương ứng (chỉ domain đã xác minh,
nên là thông tin ít nhạy cảm).

Sửa: đưa `start`/`discover` vào `guarded`; cache discovery OIDC theo issuer vài phút.

### 5. Xoá workspace không có step-up (LOW)

`RequestWorkspaceDeletion` yêu cầu là owner và gõ đúng tên workspace — chặn thao tác nhầm, không
chặn session bị chiếm. Giảm nhẹ có sẵn: workspace bị tạm ngưng chứ chưa xoá, giữ
`DefaultDeletionRetention`, mọi admin nhận email với link huỷ (token băm, dùng một lần, hết hạn
đúng ngày xoá). Đề xuất: đòi nhập lại mật khẩu hoặc mã MFA, cùng cơ chế step-up với mục 1.

### 6. NameID làm external subject (INFO)

`identityFromAssertion` đặt `ExternalSubject = NameID`, và SP yêu cầu format `emailAddress`. Nếu
IdP cho đổi email, người dùng mất liên kết (rơi về đường link-theo-email, vẫn cần domain đã xác
minh); nếu IdP tái dùng email cho người khác, người đó thừa kế tài khoản. Là rủi ro của chính
IdP mà tenant chọn, và chỉ trong tenant đó. Cân nhắc cho phép cấu hình format `persistent` và
dùng nó khi có.

## Đã kiểm tra, không có vấn đề

- **Xác thực SAML** (`VerifyResponse`): chữ ký so với đúng chứng chỉ trong metadata đã lưu,
  `InResponseTo` khớp ở cả response lẫn assertion, đúng một assertion (chống signature wrapping
  qua nhiều assertion), có audience và đúng audience, kiểm thời gian, `recover` quanh parser,
  validate XML bằng `xml-roundtrip-validator`, không có đường IdP-initiated (thiếu InResponseTo là
  bị từ chối), metadata không phụ thuộc endpoint của IdP lúc đăng nhập. `gosaml2 v0.12.0`,
  `goxmldsig v1.6.1`.
- **State**: dòng `sso_states` xoá-khi-đọc (`DELETE ... RETURNING`), hết hạn 10 phút, ACS kiểm
  `st.Tenant == tenant` trong đường dẫn; `RelayState` sinh từ `crypto/rand`.
- **Vai trò từ IdP**: `defaultRole` và `roleMap` cấm `admin` và `platform_admin` (lúc lưu); đồng
  bộ vai trò chỉ chạm vai trò được quản lý và **không bao giờ gỡ vai trò của owner**; bật
  `enforced` chỉ owner làm được, owner giữ đăng nhập mật khẩu làm đường lui.
- **Tạo/liên kết thành viên**: chỉ khi IdP xác nhận email *và* tenant đã chứng minh sở hữu domain
  (TXT `_metap-verify.<domain>`, giá trị ngẫu nhiên, mỗi domain đã xác minh chỉ thuộc một tenant
  bằng ràng buộc unique), và không bao giờ gắn vào service account. Dò domain (`discover`) chỉ
  trỏ tới tenant có domain **đã xác minh**.
- **Chuyển hướng sau đăng nhập**: `postLoginRedirect` kiểm cùng origin lúc lưu *và* lúc dùng.
- **Tải metadata SAML**: https (http chỉ loopback), giới hạn 1 MB, timeout 15 s, chặn private tại
  thời điểm dial nên redirect sang địa chỉ nội bộ cũng bị chặn.
- **Mã hoá credential** (`pkg/secrets/keyring.go`): AES-256-GCM, nonce ngẫu nhiên 96 bit mỗi lần
  seal, `ref` làm additional data (dòng bị chép sang `ref` khác không mở được), khoá xoay vòng
  bằng cách thêm khoá mới lên đầu, lỗi giải mã không lộ nội dung.
- **`safehttp`**: phân giải rồi dial đúng địa chỉ đã kiểm, không theo redirect, không proxy, thân
  phản hồi bị chặn kích thước ở nơi gọi duy nhất (`netcheck`).
- **Link huỷ xoá workspace**: token ngẫu nhiên, chỉ lưu băm, dùng một lần, hết hạn; một lỗi chung
  cho mọi trường hợp (không dò được token nào tồn tại); route nằm trong `guarded`.

## Chưa kiểm (để lần sau)

`/admin/credentials*` (kiểm quyền từng thao tác và việc "đang dùng" chặn xoá), `pkg/configfile`
(ưu tiên env > file, file không chứa secret), SCIM (`/scim/v2`), phiên hỗ trợ (`support grants`),
đếm lần sai của `/me/mfa*`, và quyền của từng operation huma trong `ops_admin.go` (một bảng
route × guard nên sinh tự động từ OpenAPI để so với mong đợi).
