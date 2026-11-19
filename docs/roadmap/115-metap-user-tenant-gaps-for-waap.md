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
| M5 | API key và service account | Key theo tenant/user: tiền tố, băm, phạm vi, hạn, lần dùng cuối, xoay, thu hồi | Cao (chặn S0) | Xong (2026-11-19): service account + API key `mk_`, băm SHA-256, hạn, lần dùng cuối, thu hồi; GraphQL gateway có đủ field; chưa có scope theo key, trần hạn theo tenant |
| M6 | Quyền không gắn entity ("xem dữ liệu request", "duyệt change request") | Policy trên tài nguyên ảo, action tự do | Trung bình | Chưa làm |
| M7 | SSO theo domain email, bắt buộc SSO | `SsoConnection` + định tuyến domain; SAML sau | Thấp | Chưa làm |
| M8 | Owner khác admin, khôi phục owner | Cờ owner, chờ 72 giờ, báo admin | Thấp | Chưa làm |
| M9 | Audit: chuỗi băm, API lọc, xuất file | | Trung bình | Chưa làm |
| M10 | Truy cập hộ của NCC có cấp quyền, có hạn | Grant có hạn, audit đứng tên support | Thấp | Chưa làm |
| M11 | Setting bảo mật của tenant (MFA, hạn API key, TTL phiên, mật khẩu) | Key mới trong `metap-config` tầng Tenant | Thấp | Chưa làm |
| M12 | SCIM | Sau MVP | — | Chưa làm |

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
