# 108. Bootstrap role, tài khoản và quyền khi chạy lần đầu

**Trạng thái:** lát 1 và 2 xong 2026-11-14 (`pkg/bootstrap` trong metap). Lát 3 và 4 chưa làm.

## Vấn đề

Khai báo entity và phân quyền đã ổn, nhưng một app mới boot lên thì phải tạo role, tạo account và
thêm từng quyền bằng tay. Cần khai báo sẵn một lần, chạy lần đầu là có đủ.

## Quyết định (chủ dự án chốt 2026-11-14)

1. **Chạy lại chỉ thêm, không ghi đè.** Policy chỉ được tạo khi chưa có dòng giống hệt; account đã
   tồn tại thì không đụng vào (không đổi mật khẩu, không đổi role). Nhờ vậy chạy mỗi lần boot cũng
   an toàn.
2. **Wildcard (`*`, `waf.*`) được mở ra lúc apply** thành từng dòng policy cụ thể theo registry.
   Engine phân quyền không có wildcard và không thêm. Pattern không khớp entity nào là lỗi, để gõ
   sai tên không âm thầm cấp quyền cho không ai.
3. **Mật khẩu chỉ lấy từ env hoặc secret ref**, không có mật khẩu mặc định. Production thiếu mật
   khẩu cho user mới thì dừng (kiểm tra trước khi ghi bất cứ thứ gì); môi trường khác thì bỏ qua user
   đó và cảnh báo. "Bắt đổi mật khẩu lần đăng nhập đầu" chưa làm.
4. **Khai báo bằng Go cạnh entity** (`bootstrap.Submit` trong `init()`, giống
   `metadata.SubmitEntity`); file JSON chỉ để thêm mục theo từng môi trường (chưa làm).

## Đã làm

- `Bootstrap{Roles, Grants, Users}`, `Grant{Role, Entities, Actions}`, `User{Email, Roles, PasswordRef}`;
  `Validate`, `Merge`, `Submit`/`Collect`.
- `Apply(ctx, Deps, tenantID, Bootstrap, Options{Production})` trả `Result` (số policy tạo/đã có,
  user tạo/đã có/bỏ qua). `Deps` nhận interface nhỏ (`PolicyAdmin`, `Identity`, `PasswordSource`);
  `PgIdentity` và `EnvPasswords` là bản mặc định.
- Policy tạo ra là dạng cơ bản mà tab Basic của admin UI hiển thị (allow, subject context, không
  field, không condition, đúng một role), nên admin sửa được như policy tay.
- Test: unit bằng fake (wildcard, additive, production, validate) và integration với Postgres thật
  cho đường tạo user.

## Giới hạn đã biết

- Chưa có cột `source=bootstrap` trên `policies`. Hệ quả: nếu admin xoá một policy do bootstrap tạo
  thì lần chạy sau tạo lại nó. Thêm cột (migration + `PolicyRow`) khi lát 3 cần phân biệt.
- Role của user chỉ gán lúc tạo user mới; user đã có thì không bổ sung role thiếu.

## Còn lại

- Lát 3: móc vào `provision-tenant` và `app.Main`, thêm `dev-tools apply-bootstrap`; mỗi product có
  một process bootstrap một lần (như `db-migrate`) blank-import các package entity để gom mảnh khai
  báo.
- Lát 4: persona của waap (Platform Admin, Tenant Admin, Security Analyst, Developer, Viewer), cột
  `source`, tài liệu hướng dẫn, ghi chú kiến trúc "product layout".
