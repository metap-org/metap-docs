# 113. Quản trị user và tenant

**Trạng thái:** P1 và P2 làm xong 2026-11-16 (metap, platform-ui, waap; nhánh `identity-membership` ở metap và platform-ui, chưa commit). P3–P4 còn là kế hoạch. Frontend chưa kiểm tra trên trình duyệt (theo quy ước FE: người dùng tự xem). Xem "P1: đã làm" và "P2: đã làm" ở cuối để biết chỗ nào khác kế hoạch ban đầu.

## Vấn đề

Phần quản lý user/tenant của metap mới ở mức demo. Hiện trạng (kiểm tra trong code, 2026-11-16):

**User trong một tenant**

- `metadata.users` chỉ có `id, tenant_id, email, password_hash` (+ cột OIDC `auth_provider`, `external_subject` từ `0020`). Không có tên hiển thị, trạng thái, lần đăng nhập cuối, không xoá được user.
- `GET /admin/users` đọc từ `user_roles`, trả `userId + roles`: **không có email**, user chưa có role thì không hiện, không phân trang/tìm kiếm.
- Role là chuỗi tự do: gõ sai là tạo role "ma"; không chặn tự gỡ quyền admin của mình hay gỡ admin cuối cùng.
- Không đổi/quên/đặt lại mật khẩu, không có chính sách mật khẩu, không mời user qua email (admin tự đặt mật khẩu hộ).
- JWT không thu hồi được: khoá user hay gỡ role thì token cũ vẫn chạy tới khi hết hạn (role thì tra lại mỗi request, nhưng "user đã bị khoá" thì không có gì để tra).
- Login không giới hạn số lần thử, không kiểm tra trạng thái tenant (tenant bị suspend vẫn login được, chỉ bị 403 ở request sau).
- Thao tác quản trị (tạo user, gán role...) không có audit.

**Tenant**

- Go metap có mô hình (`control.tenants`, `TenantStatus`, `Router`) và hàm provisioning (`control.ProvisionSchemaTenant/ProvisionDedicatedDbTenant`), nhưng **không có API nào**: tạo tenant chỉ qua `dev-tools provision-tenant`.
- `control.tenants` không có cột tên.
- `TenantsAdminPage`, tenant switcher và impersonation trong platform-ui gọi vào `metap-lowcode` (`control-api`, `control-plane-graphql`), vẫn là Rust, không chạy cùng stack Go của waap.

**Định danh**

- `users_email_unique` là unique toàn hệ thống: một email chỉ thuộc một tenant. Login không có tenant tìm user theo email ở DB dùng chung.
- Tenant `dedicated_db` tạo admin trong DB riêng của tenant (`ProvisionDedicatedDbTenant`), nên login không kèm `tenantId` không bao giờ thấy user đó.

## Quyết định (chủ dự án, 2026-11-16)

1. **Một người thuộc được nhiều tenant.** Tách *identity* (một người: email, mật khẩu) khỏi *membership* (người đó trong một tenant: role, trạng thái), làm ngay trong P1 vì P1 đằng nào cũng đổi bảng `users`.
2. **Quản trị tenant chuyển hẳn về Go metap.** Phần tenant/impersonation của `metap-lowcode` (`control-api`, `control-plane-graphql`) bị thay thế; platform-ui thôi gọi tới đó.
3. **Transport: gRPC + GraphQL, REST chỉ khi bắt buộc.** API quản trị mới là service gRPC trong metap, portal gọi qua GraphQL gateway (gateway gọi gRPC, không forward REST như `platform.go` hiện nay). REST chỉ còn cho các endpoint phải set cookie HttpOnly trên origin của issuer: login, chọn/chuyển tenant, logout, refresh, đổi mật khẩu (xoay phiên), và (P2) nhận lời mời, đặt lại mật khẩu.

## Mô hình dữ liệu

Mọi bảng định danh nằm ở **DB dùng chung**, kể cả với tenant `dedicated_db` (sửa luôn lỗi user của tenant DB riêng không login được). `user_roles` giữ nguyên chỗ cũ (đi qua `Router`).

- `metadata.identities` (mới): `id`, `email` (unique theo `lower(email)`), `password_hash` (null với user chỉ dùng SSO), `status` (`active`/`disabled`, khoá toàn hệ thống, chỉ platform admin), `session_epoch`, `failed_logins`, `locked_until`, `last_login_at`, `last_tenant_id`, `created_at`, `updated_at`.
- `metadata.users` thành **membership**, giữ nguyên `id`. `id` vẫn là `sub` của JWT, `user_id` trong `user_roles`, actor của audit, `created_by` khắp nơi, nên không phải đổi tham chiếu nào. Thêm `identity_id` (FK), `display_name`, `status` (`invited`/`active`/`disabled`), `session_epoch`, `last_login_at`, `deleted_at`, `invited_by`. Unique `(tenant_id, identity_id) WHERE deleted_at IS NULL`. Bỏ `users_email_unique`. Cột OIDC ở lại membership vì IdP cấu hình theo tenant.
- `email`, `password_hash` trên `users` được backfill sang identity rồi bỏ ở một migration sau, khi mọi đường đọc đã chuyển (2 bước, để rollback được).
- Danh mục role P1 là suy ra, không thêm bảng: `admin` dựng sẵn + role xuất hiện trong policy/bootstrap của tenant, kèm số user. Mô tả role do app dịch (i18n).

## Phiên và token

- Thêm claim `idn` (identity id) và `sep` (epoch của identity + membership lúc cấp).
- `ResolveRequestContext` (đã tra role mỗi request, không cache) tra thêm trạng thái + epoch của membership và identity trong cùng lượt. Token có `sep` cũ hơn, hoặc user/identity bị khoá/xoá, bị từ chối ngay.
- Tăng epoch khi: khoá user, xoá khỏi tenant, đổi/đặt lại mật khẩu, gỡ role admin (thận trọng).
- Token cấp trước khi deploy (không có `sep`) được chấp nhận tới khi hết hạn: TTL phiên vốn có giới hạn.

## Luồng đăng nhập

- `POST /auth/login {email, password, tenantId?}` kiểm tra identity (lockout theo identity + rate limit theo IP), rồi lấy membership `active` thuộc tenant `active`:
  - 0 membership: 401, cùng thông báo với sai mật khẩu.
  - 1 membership: cấp phiên như hiện nay.
  - nhiều membership và không có `tenantId`: trả `{selectTenant: true, tenants: [{id, name}], ticket}`. `ticket` là token ký ngắn hạn (2 phút, chỉ có `idn`, audience riêng).
- `POST /auth/select-tenant {ticket, tenantId}` cấp phiên. Mặc định chọn sẵn `last_tenant_id`.
- `POST /auth/switch-tenant {tenantId}` khi đang đăng nhập: cấp phiên mới cho membership khác của cùng identity, không nhập lại mật khẩu.
- Platform admin = identity có membership ở platform tenant với role `platform_admin`; vào khu platform bằng `switch-tenant` như mọi tenant khác.

## gRPC + GraphQL

Service gRPC mới trong `pkg/grpcapi` (proto riêng, sinh code bằng `protoc` chạy trong docker vì máy dev chưa cài), chạy trên **service phát token** (waap: `zones-service`, chủ của identity):

- `metap.identity.v1.UserAdminService` (admin của tenant): `ListUsers` (tìm kiếm, lọc role/trạng thái, phân trang offset + total), `GetUser`, `CreateUser`, `UpdateUser`, `DisableUser`, `EnableUser`, `RemoveUser`, `ResetPassword`, `AssignRole`, `RevokeRole`, `ListRoles`. Chặn gỡ admin cuối cùng và tự hạ quyền/khoá chính mình. Mỗi thao tác ghi audit (`entity = metadata.users`), nên trang chi tiết user dùng lại được `AuditLog`.
- `metap.identity.v1.AccountService` (chính mình): `GetMe`, `UpdateProfile`, `ListMyTenants`.
- (P3) `metap.platform.v1.TenantAdminService`.

`pkg/graphqlgateway` thêm field GraphQL gọi các service này qua gRPC: `users`, `user`, `roles`, `me`, `myTenants`, và các mutation tương ứng. Thay `adminUsers`, `tenantUsers`, `createAdminUser`, `assignUserRole`, `revokeUserRole` đang forward REST. REST `/admin/users*` và `/users` bị gỡ khi platform-ui đã chuyển xong, trong cùng phase.

## Các phase

### P1. Quản lý user trong tenant + tách identity

1. Migration `0040`: `identities`, cột membership trên `users`, backfill mỗi user cũ thành 1 identity cùng email, bỏ unique email toàn cục. Chuyển user của tenant `dedicated_db` về DB chung: `dev-tools`, chạy một lần, có báo cáo.
2. Identity/membership trong `peripherals`: kiểm tra mật khẩu qua identity, lockout, chặn tenant không active; `CreateUser`/provisioning/OIDC JIT/`m-bootstrap`/`dev-tools` tạo identity + membership (identity đã có thì gắn membership mới).
3. Luồng login/select/switch ở mục trên; claim `idn`/`sep` + kiểm tra trong `ResolveRequestContext`.
4. Proto + `UserAdminService`, `AccountService`, audit, các chặn an toàn.
5. GraphQL gateway gọi gRPC; gỡ REST admin users.
6. platform-ui:
   - `UsersAdminPage` viết lại: bảng gọn, tìm/lọc/phân trang, dialog tạo, panel chi tiết gồm role (chọn từ danh mục), trạng thái, audit, khoá/mở/xoá/đặt lại mật khẩu.
   - Menu tài khoản: đổi mật khẩu.
   - Tenant switcher dùng `myTenants` + `switch-tenant`.
   - Màn chọn tenant sau login.
7. waap: `zones-service` chạy service identity; gateway cấu hình địa chỉ gRPC của issuer; route `/auth/*` mới trong vite proxy và nginx.

### P2. Mời user, tự phục vụ

- Lời mời: `invitations` (token băm, hết hạn, dùng một lần). Admin mời bằng email; người mới đặt mật khẩu, người đã có identity chỉ cần chấp nhận. Gửi qua `notification-worker`; dev dùng một mock mail trong `devtools/`, giống `dns-mock`.
- Quên mật khẩu, chính sách mật khẩu theo tenant (`tenant_config`).

### P3. Quản trị tenant trên platform

- `control.tenants` thêm `name` (+ `slug` nếu cần).
- `TenantAdminService`: `ListTenants` (tìm, lọc trạng thái/tier), `GetTenant`, `CreateTenant` (dùng lại provisioning, admin đầu tiên qua lời mời P2), `UpdateTenant` (tên, tier, trial), `SuspendTenant`, `ResumeTenant`, `DeleteTenant`, `ListTenantMembers`, `Impersonate` (có hạn, có audit, dùng lại `impersonation_sessions` của `0027`, claim `imp`, banner trên UI).
- GraphQL tương ứng; `TenantsAdminPage` viết lại trên API này; bỏ `tenantAdmin.ts`/`impersonation.ts` (phần gọi metap-lowcode). Ghi trong docs của metap-lowcode rằng `control-api`/`control-plane-graphql` (phần tenant) đã bị thay.

### P4. Để sau

MFA (TOTP), màn cấu hình SSO cho tenant (OIDC đã có ở backend), SCIM, API token theo user (mở rộng từ OAuth client).

## Rủi ro và lưu ý

- **Migration dữ liệu thật:** snapshot DB trước, đếm user theo tenant trước/sau (bài học mất dữ liệu ở `waap/CLAUDE.md`).
- `metap-demo-jira` dùng `dedicated_db`: phải build và login lại được sau P1.
- `metap-lowcode` (Rust) còn đọc `metadata.users` trực tiếp ở chỗ nào thì sẽ vỡ khi bỏ cột ở bước 2; kiểm tra trước khi bỏ.
- Các field platform khác của gateway (policies, cron, config, OAuth client, dashboard) vẫn forward REST. Ngoài phạm vi 113, nhưng cùng hướng "gRPC trước".

## P1: đã làm (2026-11-16)

**Khác với kế hoạch ở trên**, chốt trong lúc làm:

- **Thu hồi phiên dùng `iat`, không thêm claim `sep`.** Mọi token mint ra giờ có `iat`. `identities.sessions_revoked_at` và `users.sessions_revoked_at` là mốc thời gian: token có `iat` sớm hơn mốc này (làm tròn xuống giây) thì bị từ chối. Token cũ không có `iat` thì bị từ chối ngay khi có một lần thu hồi. Không cần claim `idn`: `switch-tenant` tìm identity từ membership của chính token.
- **Chọn tenant khi login không có vé (ticket).** Khi có nhiều tenant mà chưa chọn, `POST /auth/login` trả `{selectTenant, tenants}` và không tạo phiên; form gửi lại cùng email/mật khẩu kèm `tenantId`. Ít thành phần hơn, lockout vẫn áp dụng như thường. Có `last_tenant_id` thì đăng nhập thẳng vào tenant đó.
- **`users.password_hash` bỏ luôn ở `0040`** (chép sang identity trước): giữ hai bản mật khẩu chỉ để chúng lệch nhau. `users.email` giữ lại làm bản sao của email identity.
- **Tenant chưa có dòng `control.tenants` vẫn được tính là active**, giống fallback của `Router`. Kiểm tra trạng thái chỉ áp dụng cho tenant đã đăng ký.
- **Mật khẩu tối thiểu 8 ký tự** chỉ kiểm ở chỗ người dùng tự chọn mật khẩu (tạo user qua API, đặt lại, đổi). Seed và test vẫn dùng mật khẩu ngắn được.
- **Admin không đặt lại được mật khẩu của người còn thuộc tenant khác** (`User.sharedIdentity`). Nếu cho phép, admin của tenant A có thể chiếm tài khoản của người đó ở tenant B. Người đó tự đổi mật khẩu, hoặc dùng "quên mật khẩu" ở P2.
- **Tạo user mới mà không có mật khẩu thì bị từ chối** cho tới khi có lời mời (P2). Email đã có tài khoản thì được thêm vào tenant mà không cần mật khẩu, và mật khẩu cũ giữ nguyên.
- **Role phải có trong danh mục**: `admin` (thêm `platform_admin` ở platform tenant) cộng mọi role mà một policy của tenant nhắc tới. Gõ sai tên role giờ là lỗi validation, không còn tạo ra role "ma".
- **Tenant switcher cũ (impersonation qua metap-lowcode) bị gỡ khỏi shell.** Nó gọi `/platform/*` của Rust `control-api`, vốn không chạy cùng stack Go. Switcher mới chỉ hiện khi người dùng thuộc từ 2 tenant trở lên. Impersonation quay lại ở P3, trên API Go.

**metap**

- Migration `0040_identities_memberships.sql`: bảng `metadata.identities`; `users` thêm `identity_id` (NOT NULL, backfill mỗi email một identity), `display_name`, `status`, `sessions_revoked_at`, `last_login_at`, `invited_by`, `deleted_at`; unique `(tenant_id, identity_id) WHERE deleted_at IS NULL` thay cho unique email toàn cục; `control.tenants.name`.
- `pkg/identity` (mới): `Authenticate` (lockout 5 lần / 15 phút; lỗi "đang khoá" chỉ trả ra khi mật khẩu đúng), `Memberships`, `PickMembership`, `AddMember`, `ListMembers` (tìm kiếm, lọc, phân trang, ẩn service principal của OAuth client), `SetMemberStatus`, `RemoveMember`, `SetPassword`, `CheckSession`. Luôn chạy trên DB dùng chung. `peripherals.CreateUser/VerifyCredentials/FindUserByID/ListTenantUsers` và JIT của `tenantauth` thành lớp bọc mỏng gọi sang đây. Provisioning `dedicated_db`, `dev-tools create-user` và `bootstrap.ScopedIdentity` ghi member vào DB dùng chung.
- `control.ResolveRequestContext` nhận thêm `iat` và gọi `identity.CheckSession`. HTTP trả `401 session_revoked` / `account_disabled`; gRPC trả cùng mã đó qua trailer.
- REST (chỉ những gì cần cookie): `POST /auth/login` (luồng mới), `POST /auth/switch-tenant`, `POST /auth/password`.
- gRPC `metap.identity.v1` (`pkg/grpcapi/proto/metap_identity.proto`): `UserAdminService` (List/Get/Create/Update/Disable/Enable/Remove/ResetPassword/AssignRole/RevokeRole/ListRoles/ListUserAudit) và `AccountService` (GetMe/UpdateProfile/ListMyTenants/ListTenantUsers). Mọi thay đổi ghi audit dưới `metadata.users`. Chỉ service phát token mới bật (`app` tự bật khi `CanIssueTokens`). Sinh code: `pkg/grpcapi/proto/gen.sh` (protoc chạy trong docker).
- `pkg/graphqlgateway` `WithIdentity(client)`: `users`, `user`, `roles`, `userAudit`, `me`, `myTenants`, `tenantUsers`, `createUser`, `updateUser`, `disableUser`, `enableUser`, `removeUser`, `resetUserPassword`, `assignUserRole`, `revokeUserRole` (giờ trả `User`), `updateMyProfile`, tất cả qua gRPC. Đã gỡ `adminUsers`, `createAdminUser` (forward REST) cùng các route REST `/admin/users`, `/admin/users/{id}/roles*`, `/users`. Chỉ giữ `/admin/users/{id}/context/invalidate`.
- Test: `pkg/identity` (nhiều tenant, lockout, thu hồi phiên, list), `grpcapi` (`TestUserAdminLifecycle`: các chặn an toàn, identity dùng chung, audit phân trang; `TestUserAdminNeedsTheAdminRole`), `httpserver` (`TestLoginTenantChoiceSwitchAndPasswordChange`), `graphqlgateway` (`TestIdentityFieldsCallGRPCAsTheCaller`).

**waap**

- `waf-graphql-gateway` gọi `ConnectIdentity` tới gRPC của zones-service và truyền `WithIdentity`.
- DB dev: đã chạy `db-migrate` (0040), 3 user thành 3 identity, đếm lại khớp. Snapshot trước khi chạy nằm trong scratchpad của phiên làm việc.
- Đã kiểm tra live qua gateway: tạo, login, khoá (phiên đang mở bị từ chối ngay), xoá, xem audit. Các route `/auth/*` mới cũng đã chạy được qua proxy của vite.

**platform-ui**

- `UsersAdminPage` viết lại: tìm kiếm, lọc theo trạng thái/role, phân trang; dialog thêm user; panel chi tiết (tên hiển thị, role, khoá/mở, xoá, đặt lại mật khẩu, lịch sử).
- `LoginForm` có bước chọn tenant và thông báo riêng cho các lỗi khoá/vô hiệu/không có tenant.
- `AppShellLayout`: switcher theo membership, "Đổi mật khẩu" trong menu tài khoản.
- `userAdminApi.ts` / `accountApi.ts` mới. `PermissionMatrix` hiện email thay cho id. `relativeTime` chuyển sang `lib/dates.ts`.

**Còn lại của P1**

- Tenant `dedicated_db` đã có user trong DB riêng: chưa có công cụ chuyển sang DB chung. DB dev không có tenant nào loại này.
- Chưa có rate limit theo IP cho `/auth/login`; lockout hiện tính theo từng identity.
- `TenantsAdminPage`, `tenantAdmin.ts`, `impersonation.ts` vẫn trỏ sang metap-lowcode, sẽ thay ở P3.

## P2: đã làm (2026-11-16)

**Khác với kế hoạch ở trên**, chốt trong lúc làm:

- **Thư gửi thẳng từ service cấp token qua SMTP, không qua `notification-worker`.** Worker đó hiện chỉ ghi log, và stack dev của waap không chạy cả nó lẫn `outbox-publisher`; đi đường outbox → RabbitMQ → worker thì phải dựng thêm hai service chỉ để gửi hai loại thư. Thay vào đó `pkg/accountmail` gửi đồng bộ với timeout 10s. Relay từ chối hay chưa cấu hình SMTP thì lời mời vẫn được tạo và link trả về cho admin để tự gửi. Thư quên mật khẩu gửi ở goroutine nền (xem dưới).
- **Mail mock tự viết** (`waap/devtools/mail-mock`, chỉ thư viện chuẩn) thay vì Mailpit: môi trường dev không kéo được image từ Docker Hub. SMTP :1025, UI + API :8025.
- **Chính sách mật khẩu chỉ có độ dài tối thiểu** (`auth.passwordMinLength`, cấp tenant, 8–128, mặc định 8). Vì mật khẩu thuộc identity và dùng chung cho mọi tenant, mật khẩu phải thoả **tenant khắt khe nhất** mà người đó thuộc (kể cả tenant đang được mời vào). Áp ở mọi chỗ người dùng chọn mật khẩu: tạo user có mật khẩu, admin đặt lại, đổi mật khẩu, nhận lời mời, đặt lại qua link. Chưa có yêu cầu loại ký tự.
- **Nhận lời mời chỉ đặt được mật khẩu khi tài khoản chưa có mật khẩu và không thuộc tenant nào khác** (`identity.CanSetPasswordOnAccept`, kiểm cả lúc tạo lẫn lúc nhận). Tài khoản đã có mật khẩu phải nhập mật khẩu hiện tại (đi qua `Authenticate`, nên vẫn tính lockout). Lý do: admin mời thấy được link; nếu link đủ để đặt mật khẩu, admin tenant A chiếm được tài khoản của người đó ở tenant B. Tài khoản chỉ dùng SSO (không mật khẩu, có tenant khác) thì chưa nhận được bằng link (`invitation_needs_sign_in`).
- **Rút lời mời = xoá member đang ở trạng thái `invited`** (`RemoveUser`, giờ cũng thu hồi lời mời đang mở). Không thêm RPC riêng.
- **Token nằm trong body POST**, không trên URL của API: `POST /auth/invitations/lookup` thay cho `GET /auth/invitations/{token}`. Trang web đọc `?token=` của link trong thư một lần rồi xoá khỏi thanh địa chỉ.
- **Quên mật khẩu luôn trả 202 giống nhau** và gửi thư ở goroutine nền, để cả câu trả lời lẫn thời gian phản hồi đều không lộ email nào có tài khoản. Tối đa 3 link mỗi identity mỗi giờ; đếm và ghi dưới advisory lock theo identity, vì test bắt được race khi nhiều request đến cùng lúc. Đặt lại xong không tự đăng nhập; mọi phiên bị thu hồi.
- Chưa có rate limit theo IP cho `/auth/password/forgot` và `/auth/invitations/*` (giống `/auth/login`, vẫn nằm trong "Còn lại của P1").

**metap**

- Migration `0043_invitations_password_resets.sql`: `metadata.invitations` (tenant, member, identity, `new_identity`, `token_hash` SHA-256, người mời, hết hạn 7 ngày, `accepted_at`, `revoked_at`) và `metadata.password_resets` (identity, `token_hash`, hết hạn 1 giờ, `used_at`), ở DB dùng chung.
- `pkg/identity/invitations.go`: `CreateInvitation` (thu hồi link cũ), `FindInvitation`, `PendingInvitation`, `RevokeInvitations`, `AcceptInvitation` (đúng một lần), `CanSetPasswordOnAccept`, `FindIdentityByEmail`, `CreatePasswordReset`, `ResetPasswordWithToken`, `IdentityTenants`, `RequiredPasswordLength`, `CheckPasswordPolicy`. `AddMember` nhận `Status: invited`. `WeakPasswordError` mang độ dài tối thiểu.
- `pkg/accountmail`: soạn thư (song ngữ Việt/Anh) và gửi qua SMTP (STARTTLS khi relay có, AUTH PLAIN khi có user, `SMTP_FROM` dạng `Tên <địa chỉ>`). Env: `SMTP_HOST/PORT/USER/PASSWORD/FROM` (đã có từ cron), `PUBLIC_APP_URL` (mới).
- REST công khai (`pkg/httpserver/routes_account.go`): `POST /auth/invitations/lookup` (email, tenant, `passwordMode` `set`/`confirm`/`provider`, độ dài tối thiểu), `POST /auth/invitations/accept` (set cookie phiên), `POST /auth/password/forgot` (202), `POST /auth/password/reset` (204). Mã lỗi mới: `invitation_invalid`, `invitation_needs_sign_in`, `reset_token_invalid`.
- gRPC `UserAdminService.InviteUser`, `ResendInvitation`; `User.invitation_expires_at`. Gateway: `inviteUser(input)`, `resendInvitation(id)`, `User.invitationExpiresAt`.
- Test: `TestInvitationAcceptSetsOrConfirmsThePassword`, `TestForgotPasswordResetsThroughTheEmailedLink` (REST, kể cả chính sách 12 ký tự theo tenant và giới hạn số link mỗi giờ), `TestInviteUserEmailsALinkAndResendReplacesIt`, `TestInviteWithoutEmailReturnsTheLinkAndPolicyAppliesToAdminPasswords` (gRPC), `pkg/accountmail` unit.

**waap**

- `devtools/mail-mock` (module riêng, có test với `net/smtp`, README). Cả hai file compose chạy nó; zones-service có `SMTP_HOST=mail-mock`, `PUBLIC_APP_URL` (dev `http://localhost:5173`, compose production `http://localhost:8081`, ghi đè qua `.env`).
- Route công khai `/accept-invite`, `/forgot-password`, `/reset-password`.
- DB dev: đã chạy `0043` sau khi snapshot. Đã chạy live luồng quên mật khẩu qua SMTP thật tới mail-mock bằng một tài khoản tạm trong tenant tạm (thư tới, đặt lại 204, dùng lại link 400, mật khẩu cũ 401, mới 200), rồi xoá tài khoản tạm.

**platform-ui**

- Trang Users: dialog thêm user mặc định là **gửi lời mời**, ô "Đặt mật khẩu ngay thay vì gửi lời mời" giữ cách cũ. Thư không gửi được thì dialog hiện link kèm nút sao chép. Panel của người đang được mời có mục Lời mời (hạn, "Gửi link mới"), nút xoá thành "Huỷ lời mời", ẩn khoá/mở.
- `AcceptInvitePage`, `ForgotPasswordPage`, `ResetPasswordPage` (`auth/AccountSelfServicePages.tsx`); link "Quên mật khẩu?" trên form đăng nhập; các hàm trong `accountApi.ts`.

**Còn lại của P2**

- Chưa có màn cấu hình `auth.passwordMinLength` cho tenant admin trong portal; đặt được qua API cấu hình tenant sẵn có.
- Chưa xem giao diện trên trình duyệt; luồng nhận lời mời mới kiểm bằng test tích hợp, chưa chạy live (cần tài khoản admin để mời).
