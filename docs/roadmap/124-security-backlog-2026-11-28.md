# 124. Vá backlog bảo mật (2026-11-28)

**Trạng thái (2026-11-28): code đã sửa và test trên nhánh `fix/security-backlog` của `metap`, `waap`; chưa merge.**

## 1. Đổi trong `metap`

| Thay đổi | Vì sao | Test |
|---|---|---|
| Cookie phiên/CSRF/SSO có `Secure` theo mặc định; chỉ `COOKIE_SECURE=false` mới tắt (`false`/`0`, giá trị lạ giữ `Secure`) | `pkg/app` gán cứng `CookieSecure = false` | `TestCookiesAreSecureUnlessTurnedOff` |
| Khoá tài khoản: lượt thử được ghi bằng một câu `UPDATE` **trước khi** so mật khẩu; đang khoá thì không so | gửi song song thì bộ đếm không tăng; và khi đang khoá, mật khẩu đúng trả `account_locked`, sai trả `invalid_credentials` | `lockout_postgres_test.go` |
| SAML gắn lượt đăng nhập với trình duyệt bằng cookie `metap_saml_bind` (`SameSite=None; Secure`) | login CSRF | `TestSAMLAnswerOnlySignsInTheBrowserThatStarted` |
| Action phải khai `Requires`, `Roles` hoặc `AnyMember`, compiler từ chối nếu không | action không khai gì thì mở cho mọi thành viên | `compiler_test.go` |
| `ResendInvitation`, `ssoconfig.VerifyDomain`: thay đổi và audit cùng một transaction; gửi mail và tra DNS nằm ngoài | roadmap 123 §3 | test tích hợp sẵn có |

## 2. Phá vỡ tương thích

- Service chạy **http thuần** (trừ `localhost`) phải đặt `COOKIE_SECURE=false`, nếu không trình duyệt bỏ cookie.
- **SAML cần https** (hoặc `localhost`): cookie bind luôn `Secure`.
- Đăng nhập khi đang khoá trả `401 invalid_credentials` thay vì `429 account_locked`. Người đã đăng nhập (đổi mật khẩu,
  step-up) vẫn nhận `account_locked`.
- Vì lượt thử được ghi trước, quá `MaxFailedLogins` lần đăng nhập **đúng** chạy cùng lúc cho một tài khoản có thể bị từ
  chối (lượt sau gặp khoá trước khi lượt đầu kịp xoá).
- Entity có action chưa khai quyền làm service **không boot được**.
- `sso.CheckDomain` giữ nguyên; thêm `sso.DomainRecordPublished`. `signin.SAMLStart` trả `*SSOStart`, `SAMLACS` nhận thêm
  `sameBrowser`.

## 3. Còn mở (từ 123 §3)

`StartRun` chiếm quyền chạy (cần cả thời hạn lẫn cách giao lại message khi đang bị giữ) và job `Direct` chạy trong
handler consumer: cả hai đổi cách bảo đảm giao message, chờ quyết.
