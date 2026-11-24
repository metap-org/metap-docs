# 121. Scope cho dữ liệu ngoài metap: ClickHouse, Elasticsearch (ADR, P6 của roadmap 120)

Roadmap 120 §2.7 chốt một câu: adapter của store ngoài metap phải nhận `Scope` làm tham số bắt buộc. Tài liệu này là ADR cho phần đó:
hình dạng của `Scope`, ai dựng nó, adapter dùng nó thế nào, và những gì cố ý chưa làm. **Trạng thái (2026-11-24): đã làm (§7).**

## 1. Vấn đề (đo từ mã)

- Dữ liệu ngoài metap (security event ở ClickHouse trong `waap/a-api`) không có entity, không có policy engine, không có ABAC của
  metap. Quyền gọi thì đã có: RBAC theo `(resource, action)` và `permission.CheckResource` / `CheckResourceIn` (roadmap 115 M6, có
  khung nhưng chưa nơi nào gọi). Còn **phạm vi dữ liệu** thì chưa có gì dùng chung.
- `a-api` đã nhận `tenantID uuid.UUID` ở mọi hàm đọc của store (`List`, `Get`, `Aggregate`) và dựng mọi câu `WHERE` qua `newWhere(tenantID)`,
  nên không có đường nào quên tenant. Nhưng nó chỉ biết tenant. Bảng `CLAUDE.md` của waap ghi thẳng: "Not yet: zone/role-level read
  filtering". Một người chỉ có vai trò trên một zone (`ScopedRole`, roadmap 115 M1) hiện thấy event của **mọi** zone của tenant nếu
  có quyền đọc event.
- Mỗi store ngoài sẽ phải tự nghĩ lại điều này (ClickHouse dùng `WHERE`, Elasticsearch dùng filter). Không có kiểu chung nghĩa là mỗi
  adapter tự quyết "quên giới hạn thì sao".

## 2. Quyết định

1. **Một kiểu `Scope` trong `pkg/kit/scope`** (chỉ stdlib), vì `e-edge` và `a-api` chỉ được import `pkg/kit/*` (ADR-0009, depguard).
   Nó nói: dữ liệu của tenant nào, và với mỗi **chiều** (dimension) có giới hạn, những giá trị nào được thấy (ví dụ chiều `zone`
   chỉ gồm hai zone). Chiều không nhắc đến thì không bị giới hạn.
2. **Giá trị zero không dùng được.** `Scope{}` không có tenant nên `Valid()` lỗi và mọi adapter từ chối nó. Không có "scope rỗng nghĩa
   là mọi thứ".
3. **Giới hạn chỉ thu hẹp.** Gọi `Limit` hai lần trên một chiều là giao của hai tập, không phải hợp. Một chiều có giới hạn mà tập giá
   trị rỗng nghĩa là **không thấy gì** (đóng, không mở): adapter trả kết quả rỗng mà không truy vấn.
4. **Adapter biên dịch bằng allowlist.** `Scope.Plan(tenantColumn, columns)` nhận ánh xạ chiều → cột do adapter khai báo; một chiều
   không có trong ánh xạ là **lỗi**, không bao giờ bị bỏ qua. Kết quả là danh sách mệnh đề (cột, các giá trị) mà adapter tự dựng thành
   `WHERE` có tham số (kiểu cột là việc của adapter: ClickHouse cần `toUUID(?)`). Không nối chuỗi, không SQL thô từ Scope.
5. **Hàm đọc của adapter nhận `scope.Scope`, không nhận tenant trần.** Hàm ghi (ingest) vẫn nhận tenant: ghi là việc của control-plane
   với bí mật ingest, không phải của người đọc.
6. **Ai dựng Scope:**
   - Service **dùng metap**: `permission.ScopeFor(ctx, rc, resource, action)` trả `Scope` cho đúng người gọi: không giới hạn nếu một
     role toàn tenant của họ được quyền trên resource đó; nếu không thì chỉ các `ScopedRole` mà role đó được quyền (`CheckResourceIn`)
     trên zone/account tương ứng; nếu không có gì thì từ chối. Dùng cùng ma trận policy trong bộ nhớ của roadmap 120 P1.
   - Service **không dùng metap** (`a-api`): dựng từ token đã xác minh. Hôm nay token chỉ mang tenant, nên Scope là `ForTenant(tenant)`
     (giới hạn chiều zone chưa có nguồn, xem §4).
7. **Resource, không phải entity.** Quyền gọi vẫn theo `resource:<tên>` (đã có), nên một store ngoài không cần giả làm entity để được
   phân quyền.

## 3. Hình dạng API (đủ để viết mã)

```go
package scope // pkg/kit/scope

type Scope struct{ /* tenant, limits */ }

func ForTenant(tenant string) Scope
func (s Scope) Limit(dimension string, values ...string) Scope // thu hẹp, trả bản sao
func (s Scope) Tenant() string
func (s Scope) Valid() error
func (s Scope) Allows(dimension, value string) bool
func (s Scope) Plan(tenantColumn string, columns map[string]string) (Plan, error)

type Plan struct {
	Clauses []Clause // tenant trước, rồi mỗi chiều có giới hạn theo thứ tự tên
	None    bool     // một chiều có giới hạn nhưng không còn giá trị nào: không truy vấn
}
type Clause struct{ Column string; Values []string }
func (c Clause) SQL(placeholder string) (string, []any) // "col = ?" hoặc "col IN (?, ?)"; placeholder như "toUUID(?)"

// metap
func (s *PermissionService) ScopeFor(ctx context.Context, rc *RequestContext, resource, action string) (scope.Scope, error)
```

## 4. Chưa làm (có chủ đích)

- **Đưa danh sách zone vào token** hoặc để `a-api` hỏi management. Hôm nay `a-api` dựng `ForTenant`, nên giới hạn theo zone **chưa có
  hiệu lực end to end**: kiểu, biên dịch, và `ScopeFor` có, nhưng nguồn cho `a-api` thì chưa. Hai hướng, chọn sau khi có nhu cầu thật:
  (a) claim `zones` trong token (đơn giản, nhưng token dài với người có nhiều zone và quyền đổi chỉ có hiệu lực khi token mới);
  (b) `a-api` hỏi management (chính xác hơn, thêm một phụ thuộc và một lần gọi mỗi request, hoặc cache ngắn).
- **Adapter Elasticsearch.** Repo chưa có; `Plan` đủ để viết nó (mệnh đề thành `term`/`terms` filter), không làm trước.
- **Đẩy `ScopeFor` vào `pkg/action`** (`Ctx.Scope`): để khi có action đọc store ngoài. waap hiện đọc event qua `m-analytics.Backend` như
  một entity ngoài, với token của caller, nên `a-api` là nơi áp Scope.
- **Ghi (ingest) theo scope.**

## 5. Phương án đã loại

- **`Scope` là một câu `WHERE` dựng sẵn:** loại. Nối chuỗi từ nơi khác vào SQL là chính chỗ lỗi, và mỗi store ngoài có cú pháp khác.
- **Scope rỗng nghĩa là không giới hạn:** loại. Quên dựng Scope sẽ mặc định thành mở. `Scope{}` bị từ chối.
- **Bỏ qua chiều adapter không biết:** loại. Một giới hạn bị lặng lẽ bỏ qua là một lỗ phân quyền; nó phải là lỗi.
- **Đặt kiểu trong `pkg/permission`:** loại, vì `a-api` không được import nó.

## 6. Kiểm chứng

- `pkg/kit/scope`: giá trị zero bị từ chối, `Limit` chỉ thu hẹp (giao), chiều rỗng cho `None`, chiều lạ là lỗi, thứ tự mệnh đề ổn
  định, `Clause.SQL` đúng cho một và nhiều giá trị và không bao giờ nhúng giá trị vào chuỗi SQL.
- `ScopeFor`: admin và role toàn tenant được không giới hạn; chỉ có `ScopedRole` thì được đúng các zone đó; role scoped ở zone khác
  thì không; không có gì thì bị từ chối.
- `a-api`: mọi câu truy vấn đọc chứa `tenant_id = ?`; có giới hạn zone thì chứa `zone_id IN`; chiều rỗng không truy vấn; test trên các
  `Plan*` hiện có vẫn qua.

## 7. Trạng thái (2026-11-24)

**Đã làm**, trên các nhánh `feat/120-*` (metap `5932cb4`) và `feat/120-typed-actions` (waap `7f8c7c3`):

- `metap/pkg/kit/scope`: `Scope`, `ForTenant`, `Limit` (chỉ thu hẹp), `Valid`, `Allows`, `Limits`, `Plan`, `Clause.SQL`. 10 test: giá trị zero bị từ
  chối, `Limit` hai lần là giao, chiều rỗng cho `None`, chiều lạ là lỗi, thứ tự ổn định, giá trị không bao giờ vào câu SQL.
- `metap/pkg/permission`: `PermissionService.ScopeFor(ctx, rc, resource, action) (scope.Scope, PermissionDecision, error)`; 6 test (admin và role
  toàn tenant: cả tenant; chỉ `ScopedRole`: đúng các zone được cấp; không có gì: bị từ chối kèm Scope zero; quyền một action không kéo theo action
  khác; nhiều loại scope thu hẹp cùng nhau; resource chưa khai báo là lỗi).
- `waap/a-api`: `List`, `Get`, `Aggregate` nhận `scope.Scope`; tenant luôn đứng đầu mệnh đề `WHERE`; giới hạn `zone` thêm `zone_id IN (...)` cho cả ba kiểu
  đọc; giới hạn rỗng trả rỗng và không truy vấn; chiều store không lọc được là lỗi; Scope zero bị từ chối. Có test kế hoạch SQL và một test trên
  ClickHouse thật (list, get, aggregate với scope giới hạn theo zone).

**Chưa nối (đúng như §4):** `Identity.Scope()` của `a-api` là cả tenant vì token chưa mang quyền theo zone, nên giới hạn theo zone có tác dụng ở
mọi nơi *trừ* việc có ai đó đưa nó vào. Bước nối còn lại là chọn (a) claim `zones` trong token hoặc (b) `a-api` hỏi management, rồi để
management dựng Scope bằng `ScopeFor`. Kiểu, biên dịch, `ScopeFor` và `a-api` đã sẵn sàng cho bước đó.
