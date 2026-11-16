# 114. Quản lý role và phân quyền theo role

**Trạng thái:** P1–P3 đã code 2026-11-16 (metap, platform-ui, waap; nhánh `identity-membership` ở metap và platform-ui, chưa commit). Frontend P2–P3 chưa xem trên trình duyệt.

## Vấn đề

Kiểm tra trong code ngày 2026-11-16:

**Role không có bảng riêng**

- Danh mục role là suy ra: `identityDeps.knownRoles` (`metap/pkg/grpcapi/identity_service.go`) gộp các role mà policy của tenant nhắc tới (`peripherals.PolicyRoles`) với `admin` dựng sẵn (thêm `platform_admin` ở platform tenant).
- Vì thế không tạo được role trống, không có tên hiển thị hay mô tả, và không xoá được. Role chỉ "xuất hiện" khi đã có policy nhắc tới, và chỉ "mất đi" khi không còn policy nào nhắc tới.
- Gán role cho user thì đã có (`AssignRole`/`RevokeRole`, panel user ở P1 của 113). Nhưng phía role thì `RoleUsersPanel` trong `PermissionMatrix.tsx` bắt gõ **user ID** để gán.

**Màn phân quyền đi theo entity**

- `PoliciesAdminPage` bắt chọn một entity trong dropdown trước, rồi mới hiện bảng role × action cho riêng entity đó. Hai tab "Cơ bản" (matrix) và "Nâng cao" (ABAC) cũng theo từng entity.
- Câu hỏi admin hay đặt là "role này được làm gì?". Để trả lời hoặc cấu hình một role cho N entity, admin phải chọn rồi lưu N lần.
- Lưu: `syncPolicyMatrix(entity, grants)` → `SyncBasicPolicies` xoá mọi dòng "cơ bản" của **một entity** rồi ghi lại, mỗi dòng một role (`roles = [role]`, hoặc `NULL` là "mọi người"). Dòng cơ bản là dòng `condition IS NULL AND field IS NULL AND subject = 'context' AND effect = 'allow'`, khớp với `isBasicShapedRow` ở frontend.
- Action của policy: `read`, `create`, `update`, `delete`, `transition`. Engine đã hiểu thêm `transition:<action>` để cấp từng action workflow (`permission.TransitionActionPolicy`), nhưng `isKnownAction` của API matrix chỉ nhận 5 action gốc, nên giao diện không cấp được theo từng action workflow.

**Màn admin trông khác màn entity**

- `GeneratedList` gắn với metadata: nó đọc `useEntity(entityName)`, field và cột lấy từ `EntityDefinition`, dữ liệu lấy qua query GraphQL CRUD chung.
- User và role không phải entity metadata. Chúng đi qua gRPC riêng, vì có những luật mà CRUD chung không có: admin cuối cùng, identity dùng chung, thu hồi phiên. Vì thế `UsersAdminPage`, `CronJobsAdminPage` và `TenantsAdminPage` mỗi màn tự dựng `Table` riêng, nên toolbar, tìm kiếm, phân trang và panel chi tiết đều lệch so với các màn entity.

## Quyết định

1. **Phân quyền theo role.** Trang chi tiết role có một bảng gồm mọi entity × action. Màn theo entity hiện tại chuyển thành chế độ **chỉ xem** ("ai làm được gì trên entity này") để đối chiếu.
2. **Role là dữ liệu thật:** có bảng `metadata.roles` theo tenant. `name` là khoá và không đổi được. `label`/`description` sửa được.
3. **Xoá role đang dùng thì chặn.** Role còn thành viên đang hoạt động, hoặc còn bất kỳ policy nào nhắc tới (kể cả dòng nâng cao), thì không xoá được. Lỗi `role_in_use` liệt kê những gì đang chặn, giống `record_referenced` ở `ReferencedByErrorMessage`. Role dựng sẵn thì không bao giờ xoá được.
4. **Không biến user/role thành entity metadata.** Thay vào đó tách phần trình bày của `GeneratedList` ra một list dùng chung, không phụ thuộc nguồn dữ liệu, để mọi màn admin trông giống màn entity.
5. API mới đi qua gRPC + GraphQL gateway như 113. Không thêm REST.

## Thiết kế

### Dữ liệu

`metadata.roles` (migration mới, sau `0040`):

| Cột | Ghi chú |
|---|---|
| `tenant_id` | |
| `name` | `varchar(80)`, khớp `user_roles.role`; unique `(tenant_id, name)` |
| `label`, `description` | hiển thị; trống thì app dịch theo `name` (i18n) như hiện nay |
| `builtin` | `admin`, `platform_admin`: không sửa quyền, không xoá |
| `created_at`, `created_by`, `updated_at` | |

- **Backfill:** mỗi tenant một dòng cho mỗi role đang có trong `policies.roles` hoặc `user_roles.role`, cộng các role dựng sẵn. Đếm trước và sau theo tenant (bài học mất dữ liệu ở `waap/CLAUDE.md`).
- **`knownRoles` và `checkRoles` đọc từ bảng này.** Gán role chưa có trong bảng là lỗi validation như P1 của 113. Ghi policy nhắc tới role không có trong bảng cũng là lỗi.
- **`pkg/bootstrap`** (108) thêm role vào bảng khi apply. Giữ nguyên nguyên tắc chỉ thêm, không ghi đè `label` admin đã sửa.
- **Pseudo-role "Mọi người"** (dòng policy `roles = NULL`) vẫn không phải dòng trong bảng. Giao diện ghim nó lên đầu danh sách role như matrix hiện tại.

### API: `metap.identity.v1.RoleAdminService`

Bật cùng chỗ với `UserAdminService` (service phát token), chỉ dành cho admin của tenant. Mọi thay đổi đều ghi audit với `entity = metadata.roles`.

- `ListRoles` (đã có ở `UserAdminService`, chuyển sang đây): tìm kiếm, số thành viên, số quyền.
- `GetRole`, `CreateRole {name, label, description, copyFrom?}`. `copyFrom` chép quyền cơ bản của một role khác.
- `UpdateRole {label, description}`, `DeleteRole`: chặn theo quyết định 3.
- `ListRoleMembers` (phân trang). `AssignRole`/`RevokeRole` giữ nguyên ở `UserAdminService`, gọi được từ cả hai phía.
- `GetRolePermissions(role)`: mọi dòng cơ bản của role trên mọi entity, kèm danh sách dòng nâng cao có nhắc tới role (để hiện dấu trên ô).
- `SetRolePermissions(role, grants[{entity, action}])`: chạy trong một transaction. Xoá mọi dòng cơ bản có `roles = [role]` của tenant rồi ghi lại theo `grants`. Sau đó xoá cache policy của mọi entity bị ảnh hưởng (`invalidatePoliciesCache`). `action` nhận 5 action gốc cộng `transition:<action>` khi entity có workflow chứa action đó.
- Gateway: `role`, `roles`, `rolePermissions`, `createRole`, `updateRole`, `deleteRole`, `setRolePermissions`. `syncPolicyMatrix` ở lại cho chế độ theo entity cho tới khi bỏ hẳn.

Dòng cơ bản có nhiều role trong một dòng (`roles = [a, b]`, chỉ có thể do ghi tay hoặc dữ liệu cũ, vì `SyncBasicPolicies` và bootstrap đều ghi một role một dòng) sẽ được migration tách thành từng dòng một role. Nhờ đó `SetRolePermissions` chỉ phải làm việc với dạng `[role]`.

### Giao diện (platform-ui)

**List dùng chung.** Tách phần trình bày của `GeneratedList` gồm toolbar, ô tìm kiếm, bộ lọc, bảng mật độ 112, phân trang `1 2 3 … last` (111), menu hành động trên dòng, panel/drawer chi tiết, thành một component nhận `rows`, `columns`, `total`, trạng thái paging/search/filter và callback. Component này chỉ ghép các atom `@metap/ui`, không tạo primitive mới.
- `GeneratedList` thành lớp nối metadata vào component đó. Hành vi và props (`GeneratedListProps`) giữ nguyên.
- `UsersAdminPage` và `RolesAdminPage` (mới) dùng component này qua gRPC/GraphQL. `CronJobsAdminPage` và `TenantsAdminPage` chuyển sau, khi đụng tới (`TenantsAdminPage` ở P3 của 113).

**Màn Roles.**
- List: tên, mô tả, số thành viên, số quyền, nhãn "dựng sẵn". Nút "Tạo role" có tuỳ chọn "Sao chép quyền từ…".
- Trang chi tiết role có 3 tab:
  - **Quyền:** bảng mô tả ở dưới.
  - **Thành viên:** thêm người bằng ô tìm user theo email/tên (dùng `users` của 113, không gõ ID); gỡ từng người.
  - **Lịch sử:** `AuditLog` với `metadata.roles`.

**Bảng quyền (tab Quyền).**

```
Role: security_analyst                          [Sao chép từ role…] [Huỷ] [Lưu (7 thay đổi)]
Tìm entity…   [Chỉ hiện entity có quyền]
                                  Xem   Tạo   Sửa   Xoá   Workflow
▾ Zones & bảo vệ     [preset ▾]   ■     ◧     ◧     □      ◧
    Zone                          ✓     □     ✓     □     2/4 ▾
    DDoS policy                   ✓     ✓     ✓     □      –
    Firewall rule                 ✓     ✓     ✓    ✓●      –        ● = có điều kiện (ABAC)
▸ Scanning           [preset ▾]   ■     □     □     □      □
▸ Alerting & Incident [preset ▾]  …
```

- **Mọi entity trên một màn, nhóm theo module.** Nhóm lấy từ app qua prop `groupFor(entity)` (waap dùng cùng nhóm với sidebar của 112). Mặc định nhóm theo prefix của tên entity.
- **Tích nhanh:** có thể tích cả dòng, cả cột trong một nhóm, hoặc cả nhóm. Mỗi dòng/nhóm có preset: *Không*, *Chỉ xem* (`read`), *Xem + sửa* (`read`, `create`, `update`, `transition`), *Toàn quyền* (thêm `delete`).
- **Workflow:** ô tích là `transition` (mọi action workflow). Mũi tên mở popover chọn từng action (`transition:<action>`) và hiện "2/4". Entity không có workflow thì hiện "–".
- **Dấu ●** khi có dòng nâng cao (có điều kiện, theo field, hoặc `deny`) nhắc tới role trên ô đó. Bấm vào thì mở popover liệt kê các dòng đó và cho sửa ngay bằng `ConditionBuilder` hiện có. Dòng nhắc tới nhiều role thì có cảnh báo "cũng áp dụng cho role X, Y".
- **Lưu một lần:** mọi thao tác chỉ sửa trạng thái cục bộ. Nút Lưu hiện số thay đổi và gọi `setRolePermissions` một lần. Rời trang khi còn thay đổi chưa lưu thì có hỏi lại.
- **Role dựng sẵn** `admin` hiện bảng toàn quyền ở dạng chỉ xem kèm dòng "admin bỏ qua mọi policy".

**Màn theo entity (chỉ xem).** `PoliciesAdminPage` giữ ô tìm `PermissionSearch` và hiện bảng role × action của một entity, mỗi role là link sang tab Quyền của role đó. Tab "Nâng cao" theo entity ở lại tới khi popover trong bảng quyền thay được hết.

## Giai đoạn

### P1. List dùng chung
- Tách component list từ `GeneratedList`. Các màn entity của waap (`ZonesPage`, …) không đổi hành vi.
- `UsersAdminPage` chuyển sang dùng component này.
- Kiểm tra: platform-ui typecheck/lint/format; waap web `tsc`/`oxlint`/`vite build`; chủ dự án xem trên trình duyệt.

### P2. Role là dữ liệu + màn Roles
- Migration `metadata.roles` + backfill + tách dòng nhiều role.
- `RoleAdminService` (trừ phần quyền), `knownRoles`/`checkRoles` đọc bảng, bootstrap ghi role.
- Gateway + `RolesAdminPage` (list, tạo, sửa, xoá bị chặn, tab Thành viên, tab Lịch sử).
- Test: chặn xoá (còn thành viên, còn policy, dựng sẵn), backfill đúng số role theo tenant, tạo role trùng tên.

### P3. Bảng quyền theo role
- `GetRolePermissions`/`SetRolePermissions`, nhận `transition:<action>`. Cache được xoá cho mọi entity bị ảnh hưởng.
- Tab Quyền (nhóm, preset, workflow, dấu ●, sao chép từ role), `copyFrom` khi tạo role.
- `PoliciesAdminPage` chuyển sang chỉ xem.
- Test: lưu không đụng dòng nâng cao, quyền có hiệu lực ngay sau khi lưu ở cả 3 service của waap, `transition:<action>` chặn đúng action.

## Ngoài phạm vi / chưa chốt

- **Quyền wildcard** (ví dụ "mọi entity của module X"): engine hiện không có. Bootstrap (108) mở wildcard ra thành từng dòng lúc apply. Preset theo nhóm đã đủ cho P3, nên không thêm wildcard vào engine.
- **Action tuỳ biến của entity** (`verifyDns`, `run`, … qua `ActionService.Invoke`, 109) chưa có action policy riêng để hiện thành cột trong bảng. Nếu cần phân quyền riêng cho từng action thì làm thành một roadmap riêng.
- **Role cấp platform** (dùng chung cho mọi tenant) và kế thừa role: không làm.
- **Phân quyền theo field** (`field`) vẫn chỉ chỉnh được ở popover/tab nâng cao, không có cột riêng.

## P1: đã làm (2026-11-16)

- `platform-ui/src/list/DataList.tsx` (mới, export ở `index.ts`): tiêu đề, ô tìm kiếm, nút bật dòng lọc (mỗi cột tự đưa control lọc qua `column.filter`), làm mới, chọn cột hiển thị (lưu `localStorage` theo `columnVisibilityKey`; cột `required` luôn hiện), sắp xếp khi bấm header, bảng ảo hoá dạng grid, chọn nhiều dòng (`selection.isSelectable`, thanh hành động hàng loạt), cột hành động trên dòng, `onRowClick`, `wrapPrimaryCell`, phân trang số và chọn cỡ trang.
- `GeneratedList` viết lại trên `DataList`, props giữ nguyên. Vẫn tự lo metadata, đồng bộ URL, export, xoá từng dòng/hàng loạt. Key lưu cột ẩn giữ như cũ (`platform-ui.list.hiddenColumns.<entity>.<view>`). Khác một chút: ô "chọn tất cả" chỉ chọn dòng xoá được (trước chọn cả dòng không xoá được).
- `UsersAdminPage` dùng `DataList`: lọc trạng thái và role chuyển xuống dòng lọc dưới cột, giống màn entity; thêm làm mới, chọn cột, chọn cỡ trang (tới 200, `identity.MaxListLimit`); bấm dòng mở panel chi tiết như trước.
- Kiểm tra: platform-ui `tsc` sạch, `oxlint` 0 lỗi (warning `useVirtualizer` chuyển từ `GeneratedList` sang `DataList`, không thêm mới); waap web `tsc -b`, `oxlint`, `vite build` đạt. Chạy trong container `web` của stack dev, vì `node_modules` của platform-ui do container cài.

## P2: đã làm (2026-11-16)

**Khác với kế hoạch ở trên**, chốt trong lúc làm:

- **`metadata.roles` nằm cạnh `user_roles`, không nhân bản theo schema của tenant như `policies`.** Một bảng dùng chung có cột `tenant_id`; câu query không ghi schema vẫn tìm ra qua `search_path` của `Router`. Tenant `dedicated_db` có bản riêng trong DB của mình, giống `user_roles`. Không thêm vào `tenantScopedTables`.
- **Ghi policy hay gán role với tên chưa có không bị từ chối, mà tự thêm role vào danh mục** (`peripherals.EnsureRoles`, gọi trong `AssignRole`, `PostgresPolicyStore.CreatePolicy` và `SyncBasicPolicies`). Lý do: bootstrap (108), `dev-tools`, `seedDefaultPolicies` và test đều ghi role trực tiếp; từ chối thì vỡ hết. Nhờ vậy luôn giữ được điều kiện "role nào đang dùng đều có dòng trong bảng". Gõ sai tên vẫn bị chặn ở API admin (`checkRoles` đọc danh mục), là nơi người dùng thật gõ tên.
- **Role dựng sẵn** (`admin`, thêm `platform_admin` ở platform tenant) được tạo dòng khi dùng lần đầu (`EnsureBuiltinRoles`), nên tenant tạo sau migration không cần bước riêng.
- **Không thêm `ListRoleMembers`**: tab Thành viên dùng `users(role: …)` sẵn có.
- **`copyFrom` để sang P3**: chép quyền là ghi policy, nên phải xoá cache policy, mà việc đó thuộc phần `SetRolePermissions`.
- **Tên role mới**: `^[a-z][a-z0-9_]{0,79}$`, cấm `__everyone__`. Role có từ trước giữ tên cũ.
- `UserAdminService.ListRoles` chuyển sang `RoleAdminService.ListRoles`. Field GraphQL `roles` giữ tên, thêm các trường mới.

**metap**

- Migration `0041_roles.sql`: bảng `metadata.roles` (`id`, `tenant_id`, `name`, `label`, `description`, `builtin`, thời gian, `created_by`; unique `(tenant_id, name)`). Backfill từ `user_roles`, `control.tenants` (admin), platform tenant, và `policies` của mọi schema (vòng lặp trên `information_schema`). Cùng lượt tách dòng policy cơ bản nhiều role thành từng dòng một role.
- `pkg/peripherals/roles.go`: `EnsureRoles`, `EnsureBuiltinRoles`, `ListRoleRows`, `RoleNames`, `GetRoleByName`, `CreateRole`, `UpdateRole`, `DeleteRole`, `PoliciesNamingRole`, `PolicyCountsByRole`.
- gRPC `metap.identity.v1.RoleAdminService` (`pkg/grpcapi/role_admin_service.go`): `ListRoles` (tìm kiếm, số thành viên đang hoạt động, số policy), `GetRole`, `CreateRole`, `UpdateRole`, `DeleteRole`, `ListRoleAudit`. Xoá bị chặn trả `409 role_in_use` kèm `fieldErrors.members` (email) và `fieldErrors.policies` (`entity action`), mỗi loại tối đa 5 rồi `+N`. Audit ghi dưới `metadata.roles` theo `id` của role. Mã lỗi mới: `errcode.RoleInUse`, `errcode.RoleExists`.
- Gateway: `roles(search)`, `role(name)`, `roleAudit(name, cursor, limit)`, `createRole`, `updateRole`, `deleteRole`.
- Test: `TestRoleAdminLifecycle` (tên sai, trùng tên, role trống gán được ngay, đếm, tìm theo mô tả, sửa, chặn xoá theo thành viên rồi theo policy, xoá, role dựng sẵn, audit khi tạo lại cùng tên, role vào danh mục qua `AssignRole`, không phải admin thì bị từ chối); test gateway cho `roles` và lỗi `role_in_use`.

**waap**

- DB dev: đã chạy `db-migrate` (0041) sau khi chạy thử trong transaction và snapshot. Kết quả 6 dòng: platform tenant `admin`, `platform_admin`; tenant `a148…` `admin`, `developer`, `security_analyst`, `viewer`. 31 policy không đổi (không có dòng nhiều role).
- Menu "Vai trò" (`/admin/roles`) trong nhóm quản trị.

**platform-ui**

- `RolesAdminPage` (mới): list trên `DataList` (tên hiển thị + mã, mô tả, số thành viên, số quyền, nhãn dựng sẵn, tìm kiếm); dialog tạo; panel chi tiết 3 tab: Thông tin (sửa tên hiển thị/mô tả, xoá, hiện ai/quyền nào đang chặn xoá), Thành viên (thêm bằng ô tìm user theo email/tên, gỡ), Lịch sử.
- `roleAdminApi.ts` (mới), `AdminHistory.tsx` (lịch sử dùng chung cho user và role, tách từ `UsersAdminPage`). `AdminRole` có thêm trường; `roleDisplayName`. Trang Users hiện tên hiển thị của role trong ô chọn.

**Chưa kiểm tra live qua gateway** với tài khoản thật (chưa có token admin trong phiên này); phần backend dựa vào test tích hợp chạy gRPC + Postgres thật. Màn Roles cần chủ dự án xem trên trình duyệt.

## P3: đã làm (2026-11-16)

**Khác với kế hoạch ở trên**, chốt trong lúc làm:

- **Dòng nâng cao không sửa ngay trong popover.** Chấm trên ô (số dòng nâng cao của entity) mở popover liệt kê từng dòng (cho phép/từ chối, action, field, có điều kiện, dùng chung với role nào) và link "Mở trong trình sửa quyền" tới `/admin/policies?entity=…&tab=advanced`. Tab Nâng cao theo entity ở lại làm nơi sửa. Nhúng `ConditionBuilder` vào popover hẹp sẽ chật và trùng chức năng.
- **Không cần xoá cache policy.** Mọi `PermissionService` metap dựng đều không có cache (`NewPermissionService`), nên quyền vừa lưu có hiệu lực ở request kế tiếp ở mọi service. Ghi chú này nằm trong `role_permissions.go`; nếu sau này bật cache thì phải thêm bước xoá.
- **Migration `0042`**: `policies.action` từ `varchar(20)` lên `varchar(120)` ở mọi bảng `policies` (bảng chung và bản của từng schema tenant). `transition:markFalsePositive` dài 28 ký tự, không lưu được trước đó.
- **Ma trận sửa theo entity bị gỡ** (`PermissionMatrix.tsx`, kèm ô gán user bằng id). `syncPolicyMatrix` (GraphQL/REST) vẫn còn cho client khác, giao diện không dùng nữa.
- Bảng quyền nằm trong tab "Quyền" của panel role (panel rộng `sm:max-w-4xl`, tab giữ nguyên khi chuyển nên thay đổi chưa lưu không mất). `copyFrom` có ở cả dialog tạo role (backend chép quyền cơ bản) và trong bảng quyền ("Sao chép quyền từ…" chỉ thay bản nháp, phải bấm Lưu).
- Role `admin` không có bảng quyền (bỏ qua mọi policy), `SetRolePermissions` từ chối nó.

**metap**

- `peripherals`: `RoleBasicGrants`, `ReplaceRoleBasicGrants` (bỏ role khỏi dòng cơ bản dùng chung, xoá dòng `[role]`, ghi lại, trả về entity bị ảnh hưởng), `AdvancedPoliciesNamingRole`.
- `RoleAdminService.GetRolePermissions` / `SetRolePermissions` (`pkg/grpcapi/role_permissions.go`): action hợp lệ là 5 action gốc hoặc `transition:<action>`; tối đa 5000 quyền mỗi lần; audit `granted`/`revoked` dưới `metadata.roles`. `CreateRole.copy_from`.
- Gateway: `rolePermissions(name)`, `setRolePermissions(role, grants)`, `createRole(copyFrom)`.
- Test `TestRolePermissionsReplaceBasicGrantsOnly`: dòng nâng cao và dòng dùng chung được giữ đúng, quyền có hiệu lực ngay qua `PermissionService` + policy store thật (kể cả chỉ cho `pause` mà không cho `resume`), action sai, role admin, role không tồn tại, `copyFrom`, audit.

**platform-ui**

- `RolePermissionsEditor` (`admin/policies/`): mọi entity trên một màn, nhóm thu gọn được (`groupFor`, mặc định theo namespace), mẫu cho từng dòng và từng nhóm (Không có quyền / Chỉ xem / Xem và sửa / Toàn quyền, hiện "Tuỳ chỉnh" khi không khớp mẫu), ô tích cả cột trong nhóm (ba trạng thái), cột Workflow (ô "mọi action" + popover từng action, bỏ một action thì các action còn lại vẫn giữ), tìm entity, "chỉ hiện entity có quyền", sao chép từ role, Bỏ thay đổi, Lưu (n thay đổi), cảnh báo khi đóng tab còn thay đổi. Logic thuần ở `rolePermissionsModel.ts`.
- `EntityAccessView`: tab Cơ bản của `PoliciesAdminPage` thành bảng chỉ xem role × action, tên role link sang `/admin/roles?role=…`. `PoliciesAdminPage` đọc/ghi `?entity=` và `?tab=`.
- `RolesAdminPage`: tab Quyền, `?role=` mở thẳng một role, props `permissionGroups` / `advancedHref`.

**waap**

- `/admin/roles` nhóm entity theo hai nhóm của sidebar: Bảo vệ (zone, domain, DDoS, firewall, IP list) và Vận hành (còn lại).
- DB dev: đã chạy `0042` sau khi snapshot; 31 policy giữ nguyên.

## Sau P3: màn Policies theo vai trò (2026-11-16)

Chủ dự án xem P3 và thấy màn Policies "chưa update": bảng quyền theo role chỉ nằm trong tab Quyền của panel role, còn menu Policies chỉ còn màn xem theo entity. Đổi lại:

- `PoliciesAdminPage` có hai tab. **Theo vai trò** (mặc định): cột trái là danh sách role (tìm kiếm, số quyền, số thành viên), cột phải là `RolePermissionsEditor` của role đang chọn (cùng component với tab Quyền của panel role, nên hai chỗ luôn giống nhau). **Theo entity**: ô tìm quyền, chọn entity, tab Cơ bản (chỉ xem) và Nâng cao (sửa quyền có điều kiện).
- URL: `?view=roles&role=…`, `?view=entity&entity=…&tab=basic|advanced`; `?entity=` không có `view` vẫn mở màn theo entity như link cũ. `admin/policies/policyLinks.ts` dựng các link này; chấm "quyền nâng cao" trong bảng quyền và tên role ở màn theo entity đều link vào đây.
- Tab Theo vai trò giữ nguyên khi chuyển sang Theo entity, nên thay đổi chưa lưu không mất.
- waap truyền cùng cách nhóm entity (Bảo vệ / Vận hành) cho cả `/admin/roles` và `/admin/policies`.
