# 115. Khoảng trống user, phân quyền, tenant của metap cho WAAP

**Trạng thái:** M2 làm một phần 2026-11-18 (xem 113 P3); còn lại là kế hoạch.

## Vấn đề

Use case của Management plane (WAAP) cần các năng lực về user, phân quyền, tenant. Nguyên tắc của chủ dự án (2026-11-18): mọi thứ liên quan tới ba thứ đó nằm trong `metap`, WAAP chỉ khai báo dữ liệu nghiệp vụ. Kiểm trên `metap` commit `273a833` cho 12 khoảng trống; phân tích đầy đủ, kèm use case bị chặn, ở `waap/docs/delivery/management/03-metap-user-tenant-gaps.md`.

## Danh sách

| # | Thiếu | Đề xuất | Ưu tiên | Trạng thái |
|---|---|---|---|---|
| M1 | Gán role theo phạm vi tài nguyên (org, account, nhóm zone, zone) | Bảng `role_assignments(user, role, scope_type, scope_id)`; metadata khai báo entity thuộc phạm vi nào; `PermissionService` lọc theo phạm vi | Cao (chặn S1 của WAAP) | Chưa làm |
| M2 | Tenant qua API và tự đăng ký | `TenantAdminService` + `POST /auth/signup` + xác nhận email | Cao (chặn S0) | **`TenantAdminService` xong** (113 P3); đăng ký tự phục vụ chưa |
| M3 | MFA (TOTP, WebAuthn), bắt buộc MFA theo tenant | Yếu tố MFA theo identity, bước hai khi login, key `require_mfa` | Cao | Chưa làm |
| M4 | Danh sách phiên, đăng xuất một phiên | Bảng phiên + thu hồi từng phiên | Trung bình | Chưa làm |
| M5 | API key và service account | Key theo tenant/user: tiền tố, băm, phạm vi, hạn, lần dùng cuối, xoay, thu hồi | Cao (chặn S0) | Chưa làm |
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
