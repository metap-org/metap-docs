# 05.6 Tham chiếu metadata của entity

Metadata là phần khai báo một lần cho mỗi entity; từ đó platform tự sinh bảng, ràng buộc, CRUD, validation, danh sách, workflow, sự kiện, audit và API. Tài liệu này giải thích từng thành phần. Nguồn sự thật là `pkg/metadata/entity.go` (repo `metap`); nếu tài liệu và code khác nhau thì code đúng.

Quy ước khai báo: dùng DSL ở `pkg/metadata/dsl.go` (`metadata.NewField`, `EnumField`, `RefField`, các hàm chuỗi `.Require().Index().Sort().Search().Uniq()`), và đăng ký entity một lần bằng `metadata.SubmitEntity` trong `init()` cạnh định nghĩa. Xem [03-core-services.md](03-core-services.md) cho `MetadataRegistry`.

## 1. EntityDefinition

| Thành phần | Vai trò |
|---|---|
| `Name` | Định danh, dạng `waf.zones`. Là khoá cho policy, topic sự kiện và tên type GraphQL. |
| `Label` | Tên hiển thị. |
| `TableName` | Bảng riêng của entity, có schema (ví dụ `waf.waf_zones`). Bảng chung `records` đã bị cấm. |
| `Fields` | Danh sách field; mỗi field là một cột có kiểu. |
| `ListViews` | Các cách xem danh sách. |
| `Workflow` | Máy trạng thái, tuỳ chọn. |
| `UniqueConstraints` | Ràng buộc duy nhất trên nhiều field cùng lúc. |
| `Audit` | Bật audit trail, tuỳ chọn. |
| `Actions` | Thao tác ngoài CRUD (roadmap 109). |

Hai thành phần không nằm trong `EntityDefinition` mà đăng ký riêng: `RelatedView` và `FieldDisplayHint` (mục 6).

## 2. EntityField

### Kiểu (`Kind`)

`Kind` quyết định kiểu cột và cách validate:

| `Kind` | Cột SQL |
|---|---|
| `string`, `enum` | `text` |
| `number` | `double precision` |
| `money` | `numeric(18,4)` |
| `boolean` | `boolean` |
| `date` | `date` |
| `datetime` | `timestamptz` |
| `id`, `reference` | `uuid` |
| `json` | `jsonb` |

Mọi field đều là cột có kiểu và nullable (roadmap 106); không còn cột `data jsonb`. `Storage` vì vậy không còn tác dụng. Khi đọc lại: `datetime` có dạng `+00:00` (không phải `Z`), `money` có 4 chữ số thập phân, cột NULL là key vắng mặt.

### Validate

- `Required`: bắt buộc.
- `Min`, `Max`: chỉ cho `number` và `money`, và `Min` không lớn hơn `Max`.
- `MinLength`, `MaxLength`: chỉ cho `string`.
- `EnumValues`: tập giá trị của `enum`, bắt buộc khai báo khi `Kind` là `enum`.

### Tham chiếu

`RefEntity` và `RefDisplayField`: xem mục 4.

### Truy vấn

- `Indexed`: tạo index.
- `Sortable`: cho phép sắp xếp theo field này.
- `Searchable` và `SearchMode`: tìm kiếm bằng index trigram (cần extension `pg_trgm`).

### Duy nhất

- `Unique`: partial index chỉ tính bản ghi chưa xoá mềm (`WHERE deleted = false`), nên bản ghi đã xoá không giữ giá trị.
- `UniqueScope`: `tenant` (mặc định, duy nhất trong từng tenant) hoặc `global` (duy nhất trên mọi tenant dùng chung bảng, ví dụ `hostname` do edge phục vụ). `global` chỉ có phạm vi trong một bảng; tenant `dedicated_db` có bản sao bảng riêng nên không bị tính.

### Khác

- `Computed`: field tính từ field khác bằng template chuỗi (`Expression`, `DependsOn`); không phải ngôn ngữ biểu thức đầy đủ.
- `System`: field do platform ghi, không phải người dùng (ví dụ `lastUsedAt`). Create và update từ chối giá trị này trong payload; nó chỉ ghi qua `CrudService.SetSystemFields`, không tăng `version`, không phát sự kiện outbox, không ghi audit.

## 3. ListView, Workflow, UniqueConstraints, Audit, Actions

### ListView

Một cách xem danh sách: `Fields` (cột hiển thị), `Filters` (field được phép lọc), `DefaultSort`, `MaxLimit` (số bản ghi tối đa mỗi trang), `RequiredFields` (field luôn có trong kết quả). Client chỉ lọc và sắp xếp được trên field mà metadata cho phép, nên đầu vào của client không bao giờ thành toán tử SQL tuỳ ý; mọi truy vấn đi qua `QueryPlanner`.

### Workflow

Máy trạng thái: `StateField` (field giữ trạng thái), `InitialState`, `TerminalStates`, `Transitions`.

Mỗi transition có `Action`, `From`, `To`, `Label` và tuỳ chọn:

- `Guard`: điều kiện được phép chuyển.
- `Validator`: điều kiện dữ liệu hợp lệ.
- `GuardMessage`, `ValidatorMessage`: thông báo thân thiện thay cho lý do kỹ thuật của evaluator.
- `SetFields`: các field tự đặt khi chuyển.

`Guard` và `Validator` là `PolicyCondition`, cùng kiểu với policy, và có thể đọc người gọi (`$ctx.roles`, `$ctx.userId`) cùng người đã làm từng action trước đó (`$workflow.actors.<action>`). Nhờ vậy viết được ràng buộc kiểu "người duyệt khác người gửi".

### UniqueConstraints

Tổ hợp từ 2 field trở lên phải duy nhất; một field đơn thì dùng `Unique` của field đó. Có `Scope` riêng (cùng ngữ nghĩa `UniqueScope`) và cũng là partial index.

### Audit

`Enabled` bật audit trail; `RedactedFields` che field nhạy cảm. Mỗi create, update, delete và transition ghi một dòng gồm diff theo field, người thực hiện, lý do và `version_after`. Dòng audit được ghi sau khi commit nghiệp vụ, nên một lần crash hiếm có thể mất đúng một dòng audit chứ không mất dữ liệu nghiệp vụ.

### Actions

Thao tác ngoài CRUD và transition: `Name` (camelCase, ví dụ `verifyDns`), `Label`, `Description`, `Scope` (`record` cần id, `entity` thì không). Quyền không khai báo ở đây: handler luôn đọc và ghi qua CRUD bằng quyền của người gọi. Chi tiết ở [roadmap 109](../../roadmap/109-grpc-first-actions.md).

## 4. `RefEntity`: phía "nhiều" trỏ tới phía "một"

`RefEntity` khai báo trên field của entity con, trỏ tới entity cha. Ví dụ `waf.firewall_rules` có field `zoneId`:

```go
metadata.RefField("zoneId", "Zone", "waf.zones", "hostname").Index()
//                 field    nhãn    RefEntity    RefDisplayField
```

- **Kiểu và cột:** `Kind` là `reference` nên cột là `uuid`; reconciler tạo khoá ngoại thật từ `waf_firewall_rules.zoneId` sang `waf_zones`.
- **Validate:** tạo hoặc sửa với `zoneId` trỏ vào bản ghi không tồn tại thì bị từ chối.
- **Chặn xoá:** xoá một Zone khi còn FirewallRule trỏ vào bị chặn bằng `record_referenced`; lỗi liệt kê mọi bản ghi đang chặn (khoá `<entity>.<field>`, giá trị là các id).
- **`RefDisplayField`:** field của entity cha dùng để hiển thị thay cho uuid (ở đây là `hostname`).
- **Một chiều:** `waf.zones` không có field nào liệt kê các rule của nó; chiều ngược lại cần `RelatedView`.
- **Ràng buộc:** entity cha phải nằm cùng schema với entity con (đúng với mọi quan hệ hiện có); `compile()` chỉ thấy một entity mỗi lần nên không tra được tham chiếu khác schema.
- Với `reference`, `RefEntity` bắt buộc phải có (validate từ chối nếu thiếu).

## 5. `RelatedView`: phía "một" hiển thị các bản ghi "nhiều"

`RelatedView` là khai báo thuần hiển thị: không tạo cột, không tạo ràng buộc. Nó nói rằng khi xem một bản ghi, hãy hiện thêm các bản ghi của entity khác trỏ tới nó. Ví dụ của `waf.zones`:

```go
{Name: "firewallRules", Label: "Firewall Rules",
 Entity: "waf.firewall_rules", FilterField: "zoneId",
 Fields: []string{"name", "ruleType", "action"}, Limit: &limit10}
```

Nghĩa là: lấy các `waf.firewall_rules` có `zoneId` bằng id của Zone đang xem, chỉ hiện ba cột `name`, `ruleType`, `action`, tối đa 10 dòng.

| Thành phần | Vai trò |
|---|---|
| `Name`, `Label` | Định danh và nhãn của mục. |
| `Entity` | Entity con cần liệt kê. |
| `FilterField` | Field `reference` của entity con trỏ về entity đang xem; thường chính là field `RefEntity` ở mục 4. |
| `Fields` | Các cột hiển thị. |
| `Limit` | Số dòng tối đa (tuỳ chọn). |

Đăng ký riêng bằng `metadata.SubmitRelatedViews(entity, factory)`, không nằm trong `EntityDefinition`, để entity cha không phải biết trước các entity con.

### Cặp đôi

| | `RefEntity` | `RelatedView` |
|---|---|---|
| Khai báo ở | entity con, trên field | entity cha, đăng ký riêng |
| Hướng | con → cha | cha → danh sách con |
| Tạo cột, khoá ngoại, kiểm tra toàn vẹn | có | không |
| Dùng để | ràng buộc dữ liệu, hiển thị tên cha | trang chi tiết của cha hiện các con |

Một `RelatedView` chỉ có ý nghĩa khi `FilterField` là một field `reference` thật; nếu entity con không có `RefEntity` thì không có khoá để lọc.

## 6. FieldDisplayHint

Gợi ý để UI chung hiển thị một field: `Field`, `ResolveVia` (giải một giá trị thô thành nhãn qua entity khác) và `EnumTones` (màu cho từng giá trị enum, ví dụ `active` là `default`, `suspended` là `destructive`). Đăng ký riêng bằng `metadata.SubmitFieldDisplayHints(entity, factory)`.

## 7. Metadata sinh ra những gì

- **Bảng và ràng buộc:** reconciler tạo cột, index, khoá ngoại và unique, theo hướng chỉ thêm, không bao giờ xoá cột hay dữ liệu.
- **API:** CRUD, list, aggregate, transition và action trên gRPC (`RecordService`, `ActionService`) và trên GraphQL gateway (`<verb><TypeName>`).
- **Còn lại:** validation theo field, phân quyền theo entity, field và bản ghi, sự kiện outbox, audit và OpenAPI.

Metadata không chứa quyền. Quyền nằm ở policy (RBAC/ABAC; mức entity mặc định cấm, mức field và bản ghi mặc định cho phép khi chưa có policy), và bootstrap ([roadmap 108](../../roadmap/108-bootstrap-roles-accounts-permissions.md)) là cách khai báo sẵn policy.

## 8. Khi sửa metadata

- Thêm field hoặc entity: reconciler tự thêm cột và index ở lần boot sau.
- Đổi kiểu một field: bị từ chối (`ErrColumnTypeChanged`) cho tới khi dùng migration `widen_type`.
- `dev-tools reconcile-plan ... --fail-on-destructive` cho biết trước các thao tác phá huỷ để đưa vào bước review hoặc CI.
- `GET /metadata/entities/{entity}` trả về metadata dạng `EntitySummary` (gồm `relatedViews`, `fieldDisplayHints`, `actions`, `version` là hash của định nghĩa).
