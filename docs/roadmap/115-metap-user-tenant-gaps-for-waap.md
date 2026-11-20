# 115. Khoảng trống user, phân quyền, tenant của metap cho WAAP

**Trạng thái:** M2 làm một phần và M1 xong 2026-11-18 (xem 113 P3 và mục "M1: đã làm" bên dưới); còn lại là kế hoạch.

## Vấn đề

Use case của Management plane (WAAP) cần các năng lực về user, phân quyền, tenant. Nguyên tắc của chủ dự án (2026-11-18): mọi thứ liên quan tới ba thứ đó nằm trong `metap`, WAAP chỉ khai báo dữ liệu nghiệp vụ. Kiểm trên `metap` commit `273a833` cho 12 khoảng trống; phân tích đầy đủ, kèm use case bị chặn, ở `waap/docs/delivery/management/03-metap-user-tenant-gaps.md`.

## Danh sách

| # | Thiếu | Đề xuất | Ưu tiên | Trạng thái |
|---|---|---|---|---|
| M1 | Gán role theo phạm vi tài nguyên (org, account, nhóm zone, zone) | Bảng `role_assignments(user, role, scope_type, scope_id)`; metadata khai báo entity thuộc phạm vi nào; `PermissionService` lọc theo phạm vi | Cao (chặn S1 của WAAP) | **Xong** (migration 0045, xem dưới) |
| M2 | Tenant qua API và tự đăng ký | `TenantAdminService` + `POST /auth/signup` + xác nhận email | Cao (chặn S0) | **`TenantAdminService` xong** (113 P3); **tự đăng ký xong** (2026-11-19): `POST /auth/signup` + `/auth/signup/verify`, bật bằng key `signup.enabled`; có CAPTCHA (`CAPTCHA_PROVIDER`/`CAPTCHA_SECRET`) và chặn tên tenant (`signup.blockedTenantNames` + danh sách dành riêng); chưa có chọn gói, gửi lại link |
| M3 | MFA (TOTP, WebAuthn), bắt buộc MFA theo tenant | Yếu tố MFA theo identity, bước hai khi login, key `require_mfa` | Cao | Xong (2026-11-19): TOTP + mã khôi phục, đăng nhập hai bước, key `auth.requireMfa` theo tenant; chưa có WebAuthn, chưa mã hóa secret TOTP khi lưu |
| M4 | Danh sách phiên, đăng xuất một phiên | Bảng phiên + thu hồi từng phiên | Trung bình | Xong (2026-11-19): mỗi lần đăng nhập là một phiên (`metadata.sessions`, token mang `sid`), xem danh sách, kết thúc một phiên hoặc mọi phiên khác, đăng xuất thu hồi phiên ở server; chưa có xem phiên của người khác (admin), giới hạn số phiên, báo đăng nhập mới |
| M5 | API key và service account | Key theo tenant/user: tiền tố, băm, phạm vi, hạn, lần dùng cuối, xoay, thu hồi | Cao (chặn S0) | Xong (2026-11-19): service account + API key `mk_`, băm SHA-256, hạn, lần dùng cuối, thu hồi; GraphQL gateway có đủ field; chưa có scope theo key (trần hạn theo tenant: xong ở M11) |
| M6 | Quyền không gắn entity ("xem dữ liệu request", "duyệt change request") | Policy trên tài nguyên ảo, action tự do | Trung bình | Xong (2026-11-20): `permission.DeclareResource`, `CheckResource`/`CheckResourceIn`, grant `resource:<tên>` qua `SetRolePermissions`, `ListPermissionResources`, gateway `permissionResources` |
| M7 | SSO theo domain email, bắt buộc SSO | `SsoConnection` + định tuyến domain; SAML sau | Thấp | Xong (2026-11-20): luồng OIDC `/auth/sso/*`, `SsoAdminService`, SSO bắt buộc (chỉ owner bật), migration `0053` |
| M8 | Owner khác admin, khôi phục owner | Cờ owner, chờ 72 giờ, báo admin | Thấp | Xong (2026-11-20): owner được bảo vệ, chuyển quyền, khôi phục owner chờ 72 giờ, migration `0052` |
| M9 | Audit: chuỗi băm, API lọc, xuất file | | Trung bình | Xong (2026-11-20): chuỗi băm theo tenant (migration `0050`), `auditEntries` lọc theo người/đối tượng/thời gian, `verifyAuditChain`, `GET /admin/audit/export` (CSV/JSON) |
| M10 | Truy cập hộ của NCC có cấp quyền, có hạn | Grant có hạn, audit đứng tên support | Thấp | Xong (2026-11-20): `SupportAccessService` (grant/list/revoke cho tenant admin, `StartSupportSession` cho platform admin), migration `0051` |
| M11 | Setting bảo mật của tenant (MFA, hạn API key, TTL phiên, mật khẩu) | Key mới trong `metap-config` tầng Tenant | Thấp | Xong (2026-11-20): MFA/TTL phiên/độ dài mật khẩu đã có từ trước; thêm `auth.apiKeyMaxLifetimeDays` (trần hạn API key, áp ở `CreateApiKey`) |
| M12 | SCIM | Sau MVP | — | Xong bản đầu (2026-11-20): `/scim/v2` Users (list có filter `userName eq`, get, create, replace, patch, delete) + ServiceProviderConfig/ResourceTypes/Schemas; chưa có Groups/roles |

## Câu hỏi chờ quyết

1. M1: bảng `role_assignments` chung (đề xuất) hay mở rộng `AUTH_CONTEXT_ENTITY` (hiện đọc một bản ghi mỗi user, không liệt kê và không thu hồi theo phạm vi được).
2. M6: tài nguyên ảo trong policy hay mở rộng danh sách action của entity.

## M1: đã làm (2026-11-18, metap)

Chọn phương án bảng `role_assignments` chung (không mở rộng `AUTH_CONTEXT_ENTITY`) theo đề xuất; câu hỏi chờ quyết số 1 coi như theo đề xuất cho tới khi chủ dự án nói khác.

- Migration `0045`: `metadata.role_assignments(tenant_id, user_id, role, scope_type, scope_id)`, unique theo cả năm cột.
- Metadata: `EntityDefinition.Scopes` (`metadata.Scope("zone","zoneId")`, `ScopeIn` cho field danh sách, `ScopeSelf` khi entity chính là phạm vi); kiểm hợp lệ ở compiler. Entity không khai báo phạm vi nào cho một loại thì role gán theo loại đó không chạm tới nó.
- `RequestContext.ScopedRoles` nạp mỗi request cùng `Roles` (không cache); `Roles` và `IsAdmin()` vẫn chỉ là role toàn tenant, nên admin theo phạm vi không phải admin tenant.
- **Quy tắc an toàn** (`pkg/permission/scope.go`): quyền là OR của các *nhóm role* (nhóm toàn tenant cộng một nhóm cho mỗi bộ role khác nhau mà gán theo phạm vi tạo ra). Một bản ghi truy cập được khi có nhóm cho phép hành động trên entity, bao phủ bản ghi, và qua policy cấp bản ghi **chỉ với role của nhóm đó**, không phải hợp mọi role người dùng giữ ở bất cứ đâu, vì như vậy role cấp ở zone A sẽ rò sang zone B. Các phạm vi cùng bộ role gộp một nhóm (một role trên một nghìn zone là một nhóm).
- crud: list, aggregate (SQL qua `query.WithAccessPlan`), get, get_many, create (phải nằm trong phạm vi), update (bản ghi sau khi sửa vẫn phải trong tầm), delete, transition, capabilities, đọc bản ghi liên quan. Người không có role theo phạm vi đi đúng đường cũ, không đổi hành vi.
- gRPC `UserAdminService`: `AssignRole`/`RevokeRole` nhận `scope_type` + `scope_id`, `ListScopeMembers` ("ai thấy zone này"), `User.scoped_roles`; xoá user kéo theo xoá gán theo phạm vi. GraphQL: `assignUserRole(..., scopeType, scopeId)`, `scopeMembers`, `User.scopedRoles`.
- Test: `pkg/permission/scope_test.go` (thuần logic), `pkg/crud/scoped_roles_postgres_test.go` (Postgres thật: chỉ thấy phạm vi của mình, role cấp ở zone này không rò sang zone khác, tạo và chuyển bản ghi ra ngoài phạm vi bị chặn, ràng buộc theo mảng và theo account, role toàn tenant kết hợp với role theo phạm vi, admin theo phạm vi, aggregate), `pkg/grpcapi/scoped_roles_service_postgres_test.go`. Đã thử phá (bỏ bước kiểm phạm vi) để chắc test thật sự bắt được.

**Giới hạn đã biết:** policy từ chối (deny) được tính theo từng nhóm, nên deny gắn với một role người dùng giữ ở phạm vi A không áp ở phạm vi B. Chưa có UI quản lý gán theo phạm vi; chưa có cách khai báo phạm vi qua `dev-tools`/lowcode ngoài JSON của entity (`scopes`).

## M9, M11, M12: đã làm (2026-11-20, metap, một commit)

**M9 — audit chống sửa và tra cứu.**
- Migration `0050`: mỗi dòng `audit_trail_entries` có `seq` (từ 1, theo tenant), `prev_hash`, `hash`; bảng `audit_chain_heads` giữ đầu chuỗi mỗi tenant. Hàm SQL `metadata.audit_entry_hash` là định nghĩa duy nhất của hash (người ghi và người kiểm dùng chung nên không lệch). Dòng cũ (seq NULL) nằm ngoài chuỗi.
- Ghi: một transaction riêng mỗi entry, khoá dòng head của tenant (`FOR UPDATE`) rồi đọc hash trước, chèn, dời head. Tenant khác nhau không chờ nhau; cùng tenant ghi tuần tự (đánh đổi có chủ đích, đã test 20 writer đồng thời).
- `VerifyChain` đi hết chuỗi, tính lại hash, báo vị trí đầu tiên hỏng: `gap` (mất/chèn dòng), `prev_hash`, `hash` (sửa nội dung), `head` (cắt đuôi). Đây là chống sửa kiểu phát hiện, không phải chống sửa tuyệt đối: người sở hữu DB có thể viết lại cả chuỗi lẫn head; muốn chắc hơn thì lưu `head_hash` ra ngoài (kèm trong báo cáo).
- Tra cứu cả tenant: `audit.Store.Search` (người, loại đối tượng, đối tượng, hành động, khoảng thời gian, keyset), gRPC `AuditAdminService.SearchAudit`/`VerifyAuditChain` (admin), GraphQL `auditEntries`, `verifyAuditChain`.
- Xuất file: `GET /admin/audit/export?format=csv|json&actor=&entity=&record=&actions=&from=&to=` (REST vì là tải file), admin, đi hết các trang rồi stream.

**M11.** `auth.apiKeyMaxLifetimeDays` (tầng Tenant, 0 = không trần). Có trần thì mọi API key phải có hạn và không vượt trần; tenant khác không bị ảnh hưởng.

**M12 — SCIM 2.0, bản đầu.** `/scim/v2` là lớp REST mỏng chạy `UserAdminService` trong cùng tiến trình với bearer token của người gọi (API key của service account admin), nên xác thực, kiểm admin, chặn gỡ admin cuối, mời qua email và audit là đúng luật của admin API. Tạo user = mời qua email (SCIM không mang mật khẩu). `PATCH` nhận `active` dạng bool hoặc chuỗi `"False"` (Entra), `displayName`, `name.formatted`; `userName` không đổi được. Chưa làm: Groups, roles, bulk, filter khác `userName eq`.

## M10: đã làm (2026-11-20, metap)

- Tenant admin cấp quyền: `GrantSupportAccess{reason, role, hours}` (lý do bắt buộc, role phải có trong tenant, 1-168 giờ, mặc định 24). Có thể thu hồi bất cứ lúc nào (`RevokeSupportGrant`).
- Platform admin gọi `StartSupportSession{tenantId}` theo grant còn hiệu lực mới nhất: tạo service account `Support: <email>` trong tenant với **đúng role được cấp** và một API key hết hạn cùng grant (tối đa 8 giờ mỗi phiên). Không thêm đường xác thực mới, nên mọi thao tác của support ghi audit đứng tên tài khoản đó và tenant thấy nó trong danh sách service account.
- Thu hồi grant: thu hồi mọi key, xoá mọi tài khoản phiên ngay. `ListSupportGrants` cũng dọn tài khoản của grant đã hết hạn.
- Audit: cấp, thu hồi, và bắt đầu phiên (đứng tên người support) đều vào trail của tenant.
- Chưa làm: tenant duyệt từng phiên, gửi email báo tenant, lý do riêng cho từng phiên.

## M8: đã làm (2026-11-20, metap)

- `users.is_owner` (migration `0052`, một owner mỗi tenant). Owner gán lười cho admin active lâu đời nhất ở lần đầu có thao tác hỏi tới, nên tenant cũ không cần backfill.
- Admin khác không thể khoá, xoá, hạ quyền, đặt lại mật khẩu hay MFA của owner (403 `owner_protected`; kiểm tra "admin cuối cùng" vẫn chạy trước).
- Chuyển quyền: `TransferOwnership` (chỉ owner, sang admin active), hiệu lực ngay.
- Khôi phục: platform admin `RequestOwnerRecovery{tenantId, newOwnerMemberId, reason}` gửi email mọi admin của tenant, chờ 72 giờ; bất kỳ admin nào `CancelOwnerRecovery` được trong thời gian đó; sau 72 giờ `CompleteOwnerRecovery` cấp admin + owner cho người mới. Mỗi tenant tối đa một yêu cầu đang chờ. Mọi thay đổi owner vào audit.

## M7: đã làm (2026-11-20, metap)

Phát hiện: bản Go có thư viện OIDC (`pkg/tenantauth`) nhưng chưa có route đăng nhập nào, nên M7 gồm cả luồng đăng nhập.

- Luồng: `GET /auth/sso/discover?email=` (chỉ định tuyến khi đúng một tenant nhận domain đó), `GET /auth/sso/start?tenant=` (PKCE + nonce, state lưu DB, dùng một lần, 10 phút), `GET /auth/sso/callback` (xác minh id_token; tìm member theo subject, nếu chưa có thì gắn member cùng email, nếu chưa có nữa thì tạo mới với role mặc định). Không hỏi MFA sau SSO: IdP là yếu tố thứ hai.
- Cấu hình do tenant admin quản (`SsoAdminService`, GraphQL `ssoConfig`/`setSsoConfig`/`deleteSsoConfig`): issuer https và kiểm tra discovery ngay lúc lưu, client secret chỉ ghi (lưu SecretStore, không bao giờ trả về), giới hạn domain email, role mặc định (không bao giờ admin), URI callback hiển thị để đăng ký ở IdP.
- **SSO bắt buộc:** `/auth/login` và đổi tenant trả 403 `sso_required` cho mọi member trừ owner. Chỉ owner được bật (owner giữ đăng nhập mật khẩu làm đường lui, liên kết với M8); xoá cấu hình thì mật khẩu dùng lại.
- Chưa làm: SAML, ánh xạ nhóm/role từ claim của IdP, kiểm `email_verified`, ép SSO ở bước nhận lời mời và đặt lại mật khẩu, xác minh quyền sở hữu domain.
