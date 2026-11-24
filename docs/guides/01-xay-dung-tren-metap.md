# Xây dựng nghiệp vụ trên metap: một cách làm duy nhất (BẢN NHÁP)

Bản đồ ranh giới metap/ứng dụng: `waap/docs/architecture/08-cross-cutting/21-metap-vs-app-map.md`.

Cookbook cho service sản phẩm (waap và các sản phẩm sau). Mọi ví dụ trỏ tới code thật trong
`waap/`. Chuẩn gốc và lý do: `waap/docs/architecture/08-cross-cutting/20-code-structure-standard.md`.

Nguyên tắc: `kit`/`runtime` là tiện ích thuần; entity, router CRUD, workflow, phân quyền, event là
của nền tảng metap; service sản phẩm **chỉ khai báo và điền nghiệp vụ vào các điểm mở**. Nghiệp
vụ ngoài CRUD đi cùng đường với CRUD.

## 0. Bài mẫu từ đầu đến cuối: entity có validation, quyền, workflow và một action

Câu hỏi hay gặp: "muốn một router có validation, authn/authz, workflow transition, xử lý lỗi thì
viết thế nào?" Trả lời: **không viết router**. Khai báo entity, metap dựng route và làm các việc
đó; bạn chỉ viết nghiệp vụ riêng. Ví dụ dưới đã được compile thử (không nằm trong repo, chỉ để
minh hoạ); các bài 1-9 phía sau đi sâu từng mảnh.

### Metap làm gì, ứng dụng làm gì

| Việc | Ai làm | Ở đâu |
|---|---|---|
| Route CRUD và transition | metap | `GET/POST /api/{entity}`, `GET/PATCH/DELETE /api/{entity}/{id}`, `POST /api/{entity}/{id}/transitions/{action}` (`internal/httpapi/routes_records.go`) |
| Authn (JWT) | metap | `RequireAuth` trước mọi handler |
| Validation | metap | Đọc từ khai báo field: `Require`, `Min/Max`, `MinLength/MaxLength`, enum, `Unique`, `Reference` |
| Authz | metap | Quyền entity, rồi quyền theo bản ghi (ABAC), rồi `transition:<tên>` riêng từng action (`CanTransitionAction`) |
| Workflow | metap | State machine, guard, audit, phát `<entity>.workflow.transitioned` cùng transaction |
| Lỗi | metap | Một dạng `{"error":{code,message,fieldErrors}}`; `409 version_conflict`, `403`, `404` |
| Khai báo entity, guard, grant | ứng dụng | bước 1, 2 dưới đây |
| Nghiệp vụ ngoài CRUD | ứng dụng | bước 3 |

### Bước 1. Khai báo entity: validation và workflow nằm ở đây

```go
func RequestEntity() metadata.EntityDefinition {
	// Guard của transition: chỉ gửi duyệt khi đã viết lý do.
	submitGuard := permission.AllCondition(
		permission.AttributeCondition("reason", permission.OpNeq, permission.LiteralValue(json.RawMessage(`""`))).
			WithMessage("Write the reason before submitting."),
	)
	title := metadata.NewField("title", "Title", metadata.KindString).Require()
	title.MinLength, title.MaxLength = metadata.Ptr(uint32(3)), metadata.Ptr(uint32(120))
	days := metadata.NewField("durationDays", "Duration (days)", metadata.KindNumber).Require()
	days.Min, days.Max = metadata.Ptr(1.0), metadata.Ptr(90.0)

	return metadata.EntityDefinition{
		Name: "waf.change_requests", Label: "Change request",
		TableName: reconciler.QualifiedTableNameIn("waf.change_requests", "waf"),
		Fields: []metadata.EntityField{
			title, days,
			metadata.NewField("reason", "Reason", metadata.KindString),
			metadata.EnumField("status", "Status", "draft", "pending", "approved", "rejected").Index(),
		},
		Workflow: &metadata.EntityWorkflow{
			StateField: "status", InitialState: "draft", TerminalStates: []string{"approved", "rejected"},
			Transitions: []metadata.WorkflowTransition{
				{Action: "submit", From: "draft", To: "pending", Label: "Submit", Guard: &submitGuard},
				{Action: "approve", From: "pending", To: "approved", Label: "Approve"},
				{Action: "reject", From: "pending", To: "rejected", Label: "Reject"},
			},
		},
		Audit: &metadata.EntityAuditConfig{Enabled: true},
		Actions: []metadata.EntityAction{
			{Name: "bulkClose", Label: "Reject all pending", Scope: metadata.ActionScopeEntity},
		},
	}
}

func init() { metadata.SubmitEntity(RequestEntity) }
```

Chỉ với đoạn này, `POST /api/waf.change_requests` đã từ chối `title` quá ngắn hoặc `durationDays=200`;
`POST .../transitions/submit` bị từ chối khi `reason` rỗng, kèm đúng câu lỗi ở `WithMessage`.
Toán tử guard có: `OpEq`, `OpNeq`, `OpIn`, `OpNotIn`, `OpGt/Gte/Lt/Lte`, `OpContains`, `OpNotContains`.

### Bước 2. Phân quyền: khai báo grant

```go
bootstrap.Submit(func() bootstrap.Bootstrap {
	return bootstrap.Bootstrap{
		Roles: []string{"requester", "approver"},
		Grants: []bootstrap.Grant{
			{Role: "requester", Entities: []string{"waf.change_requests"}, Actions: []string{"read", "create", "update", "transition:submit"}},
			{Role: "approver", Entities: []string{"waf.change_requests"}, Actions: []string{"read", "transition:approve", "transition:reject"}},
		},
	}
})
```

`requester` gọi `approve` nhận `403`. `admin` mặc định toàn quyền trong tenant của nó.

### Bước 3. Thao tác ngoài CRUD: viết action có kiểu

```go
Actions: func(state *web.State, reg *crud.ActionRegistry) error {
	return bulkClose.Register(reg, state.ActionDeps())
},

type bulkCloseIn struct{ Reason string `json:"reason"` }
type bulkCloseOut struct{ Closed int `json:"closed"` }

var bulkClose = action.Entity("waf.change_requests", "bulkClose",
	func(ctx context.Context, a *action.Ctx, in bulkCloseIn) (bulkCloseOut, error) {
		if in.Reason == "" {
			return bulkCloseOut{}, crud.NewServiceErrorFields(http.StatusUnprocessableEntity, errcode.ValidationFailed,
				map[string][]string{"reason": {"is required"}})
		}
		closed := 0
		// Một step là một transaction: ghi bản ghi, event và audit của nó cùng thành công hoặc cùng không.
		err := a.Step(ctx, "reject pending", func(ctx context.Context) error {
			pending, err := a.Records.List(ctx, "waf.change_requests", pendingQuery, a.RC)
			if err != nil {
				return err
			}
			for _, r := range pending.Data {
				if _, err := a.Records.Transition(ctx, "waf.change_requests", r.ID, "reject", r.Version, nil, a.RC, nil); err != nil {
					return err
				}
				closed++
			}
			a.Audit("change_requests.bulk_closed", uuid.Nil, map[string]any{"reason": in.Reason, "closed": closed}) // tuỳ chọn
			return nil
		})
		return bulkCloseOut{Closed: closed}, err
	})
```

Input và output là struct (input được giải mã **nghiêm ngặt**: field lạ hoặc sai kiểu là `400`, không bị bỏ qua).
Handler không tự kiểm authn hay authz: cổng route làm trước khi gọi tới; `a.Records` đọc và ghi với quyền của người gọi
nên không vượt quyền họ, và đổi trạng thái vẫn đi qua `Transition` (guard, quyền, audit, event như khi gọi qua HTTP).
Audit (`a.Audit`) và event (`a.Emit`) là **tuỳ chọn**: không gọi thì không có.

Test không cần DB hay RabbitMQ: `actiontest.Run(bulkClose, actiontest.Call(...), seed)` trả về kết quả, các lần ghi,
event và audit đã xếp hàng (`pkg/action/actiontest`).

### Khi cần route HTTP thật sự riêng

Chỉ cho người gọi là **dịch vụ** (`/internal/*`, webhook), mount qua `Options.Routes`. Một service ngoài metap chỉ import `github.com/metap-org/metap/pkg/web` (mặt công khai của phần HTTP; phần còn lại là `internal/httpapi`):

```go
rc, ok := web.RequireAuth(w, r, state)                       // authn; tự ghi 401
var body in
if !web.DecodeJSONBody(w, r, &body) { return }               // body giới hạn kích thước, tự ghi 400/413
out, err := doWork(r.Context(), state.Crud, rc, body)
if err != nil { web.WriteCrudError(w, err); return }         // lỗi đúng dạng chuẩn
web.WriteJSON(w, 200, map[string]any{"data": out})
```

Với route dành cho người dùng thì dùng action (bước 3), không dùng cách này.

### Ghi nhớ

- Validation nằm ở **khai báo field**, không ở handler.
- Authz nằm ở **grant** (`bootstrap.Submit`) và **guard workflow**, không ở `if role == ...`.
- Đổi trạng thái chỉ qua **`Transition`**.
- Lỗi chỉ qua **`crud.NewServiceError*`**.

## 1. Thêm entity

1. Viết `metadata.EntityDefinition` trong `<service>/entities/`, tên entity/field là hằng ở `model/`.
2. `metadata.SubmitEntity(...)` trong `init()` cạnh định nghĩa. `cmd/` chỉ blank-import package.
3. Bảng do reconciler tạo khi service khởi động; không viết migration tay.

Mẫu: `waap/m-management/billing/entities/plan.go` (`PlanEntity`, `TableName: reconciler.QualifiedTableNameIn(model.Plans, "waf")`).
Cấm: bảng/migration tay cho entity; gõ lại chuỗi `"waf.plans"`.

## 2. Thêm thao tác ngoài CRUD (action)

1. Khai báo trong `EntityDefinition.Actions`: `{Name, Label, Description, Scope}`, scope
   `ActionScopeRecord` (có `{id}`) hoặc `ActionScopeEntity`. Quyền gọi: `Roles` là các role được chạy (policy `action:<tên>` của
   tenant ghi đè), `Requires` là quyền cấp entity người gọi phải có (`read`, `create`, `update`, `delete`, `transition`); cả hai được
   kiểm tra trước khi đọc input. Không khai gì thì mọi người đã đăng nhập đều gọi được.
2. Viết action bằng `pkg/action`: `action.Entity` (thao tác trên cả entity) hoặc `action.Record` (trên một bản ghi: nó được đọc trước,
   với quyền của người gọi, và giải mã vào kiểu bạn chọn; `a.Current` giữ bản ghi và version của nó).
3. Đăng ký: `x.Register(reg, state.ActionDeps())` trong `RegisterActions(cfg)` (trả về
   `func(*web.State, *crud.ActionRegistry) error`). Action cũ viết bằng `crud.ActionHandler` vẫn chạy song song.
4. Handler mỏng: quy tắc nghiệp vụ là **hàm thuần** (test như hàm thường); handler chỉ quyết định đọc gì, ghi gì, audit gì.
   Việc gọi mạng (DNS, webhook, scanner) làm **ngoài** `a.Step`: một step giữ đúng một connection tới khi kết thúc.
5. Phục vụ ở `POST /api/{entity}[/{id}]/actions/{action}`.

Mẫu: `waap/m-management/zones/actions.go` (nhiều action ngắn), `zones/netcheck/checks.go` (`VerifyDomain`: tra DNS ngoài
transaction, rồi ghi domain, lan sang các zone và audit trong một step), `zones/actions.go` (`ApplyPreset`: cả preset hoặc không gì).
Lỗi nghiệp vụ là `*crud.ServiceError` (`crud.NewServiceErrorMsg`/`...Fields`), mã ở `kit/errcode`
hoặc `model`. Cấm: route REST riêng cho một action; tự viết JSON lỗi.

Dữ liệu ngoài metap (ClickHouse, Elasticsearch): adapter đọc qua `scope.Scope` (`pkg/kit/scope`), không nhận tenant trần; xem
`roadmap/121-scope-for-data-outside-metap.md`.

## 3. Thêm ràng buộc khi ghi (guard)

Một guard là một decorator của `crud.RecordBackend`: nhúng `passthrough`, chỉ ghi đè lời gọi cần
chặn. Ghép thành chuỗi ở **một** chỗ, gắn vào `app.Options.WrapBackend`.

Mẫu: `waap/m-management/zones/guard/` (`quota.go`, `validation.go`, `domain.go`, `account.go`,
`reference.go`, `version.go`, mỗi file một guard) và `Backend.chain()` trong `backend.go` (thứ tự ngoài vào trong).
Mỗi guard một việc; guard đứng trong cùng (version) ghi lại điều thực sự xuống DB.
Cấm: guard rải trong handler; một backend gom mọi thứ.

## 4. Workflow

- Khai báo `EntityDefinition.Workflow` (`StateField`, `InitialState`, `TerminalStates`,
  `Transitions` bằng `metadata.Transition(action, from, to, label)`); guard/validator là
  `permission.PolicyCondition`, có thể đọc `$ctx.roles`, `$workflow.actors.<action>`.
- Đổi trạng thái **chỉ** bằng `backend.Transition(ctx, entity, id, action, version, data, rc, reason)`
  hoặc action chuyển trạng thái. Metap kiểm guard và quyền, ghi audit, phát
  `<entity>.workflow.transitioned` cùng transaction.

Mẫu: `waap/m-management/scanning/entities/scanjob.go` (khai báo), `m-management/scanning/run.go` (`doTransition`).
Cấm: `Update` thẳng field trạng thái.

## 5. Phân quyền

- Cấp quyền bằng grant khai báo: `bootstrap.Submit` với `Grant{Role, Entities, Actions}`; entity là
  tên chính xác, `waf.*` hoặc `*`; action là `read|create|update|delete|transition`,
  `transition:<name>` hoặc `*`. `admin` là role dựng sẵn, toàn quyền trong tenant của nó.
- Quyền theo trạng thái: thuộc tính workflow trong policy; theo phạm vi: scoped role (waap dùng cho
  account).

Mẫu: `waap/m-bootstrap/personas.go`.
Cấm: `if role == "..."` trong handler; dùng `platform_admin` cho tính năng sản phẩm.

## 6. Event

- Metap **tự phát** `<entity>.record.created|updated|deleted` và `<entity>.workflow.transitioned`
  cho mọi entity, cùng transaction với ghi dữ liệu. Muốn phản ứng khi một việc xảy ra, đăng ký
  subscribe các topic này (mẫu: `waap/c-config-distributor/subscribe`), luôn kèm resync định kỳ
  làm đảm bảo hội tụ.
- Ví dụ: quyết định gói = `waf.plan_change_requests.record.updated`; rollback = một
  `waf.policy_versions.record.created`.
- Event tuỳ biến: `events.Emit(ctx, tx, events.Event{...})` cần `tx` là transaction của chính lần
  ghi. **Handler hiện không có tx** (CRUD tự mở transaction), nên chưa phát được an toàn từ action;
  đây là chỗ metap cần thêm (một unit-of-work cho action) trước khi waap dùng. Đừng phát ngoài
  transaction và đừng publish thẳng RabbitMQ.

## 7. Gọi service khác và gọi ra ngoài

- Record của service khác: `recordclient.New(url, nil)` (là `crud.RecordBackend` và
  `crud.ActionInvoker`), chuyển tiếp token người gọi. Mẫu: `m-management/zones/guard/reference.go`.
- Việc không phải record giữa hai service: `httpx.NewInternalClient(base, secret, timeout).GetData(...)`
  phía gọi, `httpx.RequireSecret(secret)` phía nhận (503 nếu chưa cấu hình, 401 nếu sai). Mẫu:
  `billing.PlanLookup` và `m-management/billing/actions.go`.
- Địa chỉ do người dùng nhập (origin, hostname): `safehttp.NewClient(...)` (chặn địa chỉ không
  công khai, không theo redirect). Mẫu: `m-management/zones/netcheck/outbound.go`.
- Mọi client khác: `httpx.NewClient(timeout)`. Cấm `http.DefaultClient` và `&http.Client{}` tự dựng.

## 8. Cấu hình và dây nối

Một struct `Config` mỗi service, đọc **một lần** ở `cmd/` bằng `kit/env` (`LoadConfig()`), truyền
xuống. `main.go` chỉ ráp `app.Options`.

**Nguồn giá trị** (thứ tự, cái đầu thắng): biến môi trường, file YAML mà `CONFIG_FILE` trỏ tới, giá
trị mặc định của chỗ đọc. File là nơi đặt cấu hình ổn định của service; env chỉ để ghi đè từng giá
trị theo môi trường và để đưa secret vào (đừng để secret trong file). Mỗi khoá trong file chính là
tên biến nó thay, viết thường và lồng nhau: `smtp.host` là `SMTP_HOST`, `jwks.keys_dir` là
`JWKS_KEYS_DIR`, `cors_origins: [a, b]` là `CORS_ORIGINS=a,b`. Code đọc cấu hình không đổi gì:
`kit/env` tự tìm trong file khi env không có. Dòng đầu của `main`: `configfile.MustLoadFromEnv()`.
Mẫu: `waap/m-deploy/config/management.yaml`.

```go
func main() {
	configfile.MustLoadFromEnv()
	cfg := billingservice.LoadConfig()
	app.Main(app.Options{
		Name:    "billing-service",
		Routes:  billingservice.MountRoutes(cfg),   // chỉ /internal/*, health
		Actions: billingservice.RegisterActions(cfg),
	})
}
```

Mẫu: `waap/m-management/billing/config.go`, `m-management/zones/config.go`. Cấm: `env.*` trong hàm nghiệp vụ.

## 9. Test

- Đơn vị: hàm nghiệp vụ thuần; guard dùng `crudmock.RecordBackend` (xem `m-management/zones/guardedbackend_test.go`,
  `newGuarded`), mock chặt nên sai thứ tự gọi sẽ hỏng test.
- Tích hợp: build tag `integration`, `metap/pkg/testkit` (Postgres thật). Gọi action qua
  `crud.ActionRegistry.Invoke` thay vì dựng HTTP (xem helper `invoke` trong
  `m-management/alerting/alertingservice_postgres_test.go`).
- Cấu hình trong test: dựng `Config{...}` trực tiếp, không `t.Setenv`.

## Còn lệch chuẩn đã biết (cần metap hỗ trợ)

- `m-billing` đọc `control.tenants` bằng SQL qua `state.Router.SharedPool()` (liệt kê khách hàng,
  lấy tên tenant): vi phạm "không SQL trong service sản phẩm". Cần một API thư mục tenant của metap.
- `listCustomerTenants` tự kiểm `rc.IsAdmin()` và tenant vận hành trong handler thay vì bằng grant.
- Event tuỳ biến từ action: xem mục 6.
