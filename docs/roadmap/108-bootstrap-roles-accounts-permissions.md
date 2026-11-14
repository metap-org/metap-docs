# 108. Bootstrap role, tài khoản và quyền khi chạy lần đầu

**Trạng thái:** xong 4 lát, 2026-11-14 (metap-org/metap#46, metap-org/waap#36).

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

- `Bootstrap{Roles, Grants, Users}`, `Grant{Role, Entities, Actions}`, `User{Email, Roles, SecretRef}`;
  `Validate`, `Merge`, `Submit`/`Collect`.
- `Apply(ctx, Deps, tenantID, Bootstrap, Options{Production})` trả `Result` (số policy tạo/đã có,
  user tạo/đã có/bỏ qua). `Deps` nhận interface nhỏ (`PolicyAdmin`, `Identity`, `PasswordSource`);
  `PgIdentity` và `EnvPasswords` là bản mặc định.
- Policy tạo ra là dạng cơ bản mà tab Basic của admin UI hiển thị (allow, subject context, không
  field, không condition, đúng một role), nên admin sửa được như policy tay.
- Test: unit bằng fake (wildcard, additive, production, validate) và integration với Postgres thật
  cho đường tạo user.

## Lát 3 và 4 (đã làm)

- `app.BootstrapMain`/`RunBootstrap`: process một lần cho từng product, như `db-migrate`. Env:
  `BOOTSTRAP_TENANT_IDS` (danh sách id hoặc `all` = mọi tenant active), `BOOTSTRAP_FILE` (file JSON
  bổ sung theo môi trường, `bootstrap.LoadFile` từ chối field lạ), mật khẩu từ biến env mà
  `secretRef` nêu tên, `NODE_ENV=production` thì thiếu mật khẩu là lỗi.
- `app.Run` tự apply khi `BOOTSTRAP_ON_BOOT=true`. Chỉ dành cho product một service: nhiều service
  cùng apply có thể cùng tạo một policy (bảng `policies` không có unique), và mỗi service chỉ biết
  entity của mình nên wildcard sẽ mở thiếu.
- `bootstrap.ScopedIdentity` đi qua `Router.PoolFor`, nên user của tenant `dedicated_db` được tạo
  đúng database của nó.
- `dev-tools apply-bootstrap <tenantId> <bootstrap.json> [entity.json ...]` cho khai báo thuần JSON;
  `provision-tenant` in nhắc chạy bootstrap sau khi tạo tenant. Chưa gọi tự động từ
  `provision-tenant` vì dev-tools không biết entity của product nào.
- waap: `m-bootstrap/` với persona `security_analyst`, `developer`, `viewer` (Tenant Admin là role
  `admin` có sẵn, Platform Admin vẫn là `dev-tools bootstrap-platform-admin`) và
  `cmd/waf-bootstrap` blank-import entity của cả 3 service.

## Kiến trúc: bootstrap nằm ở đâu khi tách microservice

Mỗi product có một process bootstrap riêng, không nằm trong từng service. Lý do: quyền của một role
trải qua entity của nhiều service, wildcard chỉ mở đúng khi thấy cả registry của product, và việc
này chạy một lần mỗi lần deploy chứ không theo vòng đời của service. Process đó chạy trước hoặc
cùng lúc deploy (job/init container), cùng kiểu với migration.

## Giới hạn đã biết

- Chưa có cột `source=bootstrap` trên `policies`. Hệ quả: nếu admin xoá một policy do bootstrap tạo
  thì lần chạy sau tạo lại nó. Thêm cột (migration + `PolicyRow`) khi lát 3 cần phân biệt.
- Role của user chỉ gán lúc tạo user mới; user đã có thì không bổ sung role thiếu.
- Persona waap mới là entity-level; chưa có field-level hay điều kiện ABAC (ví dụ analyst chỉ thấy
  zone được giao).

## Còn lại (nếu muốn làm tiếp)

- Cột `source` trên `policies` để phân biệt policy do bootstrap tạo.
- Ép đổi mật khẩu ở lần đăng nhập đầu.
- Gọi bootstrap tự động sau `provision-tenant` khi có đường để product đăng ký hook.
