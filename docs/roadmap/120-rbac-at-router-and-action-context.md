# 120. RBAC ở router, ABAC khi cần, và ngữ cảnh action cho handler của ứng dụng (ADR)

Chủ dự án yêu cầu (2026-11-24), sau roadmap 118 và 119: logic tùy biến của ứng dụng (handler của action) vẫn khó đọc, khó test,
và core CRUD là một hàm dài trộn nhiều mối quan tâm. Tài liệu này là **ADR kèm kế hoạch**: ghi quyết định đã chốt, những gì đã
đo từ mã, những gì chưa chốt, và thứ tự làm. **Đây là thay đổi phá vỡ (breaking change) lớn** ở metap; waap và
`internal/admin` đi theo.

**Trạng thái (2026-11-24): quyết định đã chốt với chủ dự án; P0, P1, P2 và bản đầu của P3 đã làm trên nhánh `feat/120-rbac-router-action-ctx` của metap (chưa merge); phần còn lại của P3 và P4 trở đi chưa làm.** Nhánh làm việc và commit: xem §9.

## 1. Vấn đề (đo từ mã)

| # | Chỗ lệch | Bằng chứng |
|---|---|---|
| 1 | Handler action của app là bảng closure inline, input không có kiểu, dependency truyền lẻ tẻ, chọn `state.Crud` hay `Guarded(...)` là quyết định ngầm | `waap/m-management/zones/actions.go` (10 action) |
| 2 | `CrudService.Create` là hàm tuần tự ~140 dòng: tenant, registry, quyền entity, snapshot, quyền ghi từng field, validate, scope, workflow, transaction, audit, outbox | `pkg/crud/create.go`; `update`, `delete`, `transition` lặp lại một phần khung này |
| 3 | RBAC và ABAC đi chung một đường `PermissionService.CanXEntity` (12 chỗ gọi trong `crud`), mỗi lần đều truy vấn policy | `pkg/permission/permissionservice.go`; `NewPermissionServiceWithCache` **không có nơi nào gọi**, nên production không cache policy |
| 4 | Role của caller được đọc từ DB **mỗi request**, cùng lúc với kiểm tra session bị thu hồi và tài khoản bị vô hiệu | `internal/authctx`, `httpserver/auth.go` (comment: "never cached, so a revocation takes effect on the very next request") |
| 5 | `internal/admin` gọi `adminCaller(ctx)` ở đầu từng phương thức dù route đã có nhóm access `Admin` (một quyết định làm hai lần), và chỉ có **một** chỗ ghi audit cho hàng chục thao tác nhạy cảm | `internal/admin/*_service.go`, `identity_service.go:292` (còn bỏ qua lỗi bằng `_ =`) |
| 6 | `internal/admin/types.go` (2910 dòng) còn kiểu có getter kiểu protobuf, dấu tích của gRPC bị gỡ ở roadmap 116 | `req.GetEntity()` |
| 7 | `CheckResource` (quyền cho resource không có entity) đã có đủ cơ chế nhưng chưa có nơi nào gọi | `pkg/permission/resources.go` |

## 2. Quyết định

### 2.1 RBAC tách khỏi ABAC; RBAC kiểm tra ở router

- **RBAC** là tra bảng role→action trong bộ nhớ qua `RoleMatrix` bất biến cho từng tenant:
  `Allows(roles, entity|resource, action)` là hàm thuần, không I/O. Dữ liệu vẫn lưu trong DB để tenant admin chỉnh được; giá trị
  mặc định khai trong code (`Roles(...)` trên action/entity) và DB ghi đè.
- Kiểm tra ở **router** bằng middleware cổng gắn vào nhóm access có nhãn (`Entity` cho CRUD, `Action` cho action và typed
  operation). Mọi route vẫn phải thuộc đúng một nhóm (test liệt kê route như `AccessGroups` hiện có).
- Khóa là `(entity hoặc resource, action)`. Resource không có entity (`resource:<name>`, `permission/resources.go`) dùng cùng
  ma trận, nên dữ liệu ngoài metap (ClickHouse, Elasticsearch) cũng được phân quyền theo cùng cách.
- **ABAC** (condition, deny, scoped role, quyền theo field, scope theo record) giữ nguyên cơ chế snapshot hiện có nhưng **chỉ
  nạp khi entity có policy cần nó**. Định nghĩa policy ABAC cũng nằm trong bộ nhớ; DB chỉ bị chạm cho dữ liệu của record.
- Ma trận chỉ nói "cổng mở" khi chắc chắn; deny, scoped role và OAuth scope vẫn đi đường ABAC. Khi ma trận chưa nạp được thì
  **từ chối** (fail-closed).

### 2.2 Chấp nhận thu hồi và áp dụng RBAC chậm; session thì tức thì

Chủ dự án chốt (2026-11-24): thay đổi hoặc thu hồi **role và policy** được phép có hiệu lực chậm. Role của caller được cache
trong process với TTL ngắn (đề xuất 30–60 giây, chốt lúc làm) cộng tín hiệu nạp lại theo version khi policy đổi.

**Không nới:** session bị thu hồi, tài khoản bị vô hiệu và khóa MFA vẫn kiểm tra tức thì (một truy vấn rẻ theo khóa chính, hoặc
cache cực ngắn). Đây là điểm đảo một phần quyết định cũ ở `internal/authctx` (role "never cached"): chỉ phần role đổi, không phải
session.

### 2.3 CRUD là một pipeline; câu hỏi quyền cấp entity được tiêm vào, mặc định bật

**Đã điều chỉnh (2026-11-24, chủ dự án chọn):** bản đầu của mục này nói `CrudService` không tự kiểm tra role và chỉ lấy ra qua
`Gated`/`Trusted`. Khi làm P2 thấy hai điều khiến phương án đó không đứng được: (1) `CrudService` vẫn phải hỏi quyền đọc của
**entity khác** (hiển thị quan hệ, `loadRelatedAccess`), mà cổng ở cửa chỉ biết entity chính; (2) nhiều nơi dùng thẳng
`state.Crud` (các `WrapBackend` và action của waap), nên bỏ kiểm tra bên trong sẽ làm các chỗ đó mặc định thành "mở" nếu quên bọc.
Phương án thay thế giữ đúng mục đích (RBAC không nằm rải trong code CRUD, dùng chung một nguồn trong bộ nhớ) mà không đổi mặc định
an toàn:

- `Create/Update/Delete/Transition` thành pipeline `resolve → authorizeEntity → loadCurrent → authorizeRecord → prepare* → persist`
  (`pkg/crud/pipeline.go`). `persist` là chỗ duy nhất mở transaction, ghi, xếp event outbox, commit rồi ghi audit. `prepareCreate`,
  `prepareUpdate`, `prepareTransition` là hàm thuần, test không cần DB.
- Câu hỏi quyền cấp entity cho entity đang làm việc đi qua `EntityAccess` (cùng `PermissionService` trừ khi dịch vụ đến từ
  `Trusted()`), nên có một chỗ duy nhất để đổi, và cổng ở router (§2.1) cùng `CrudService` dùng chung một nguồn trong bộ nhớ.
- `CrudService.Trusted()` trả về bản sao bỏ **chỉ** câu hỏi đó, cho mã (job, bootstrap, sweep) làm việc trên mọi entity của chính dịch
  vụ mình. Quyền theo record, theo field, scope và mask vẫn áp cho người mà `RequestContext` nêu, và đọc quan hệ sang entity khác
  vẫn hỏi quyền.
- `Gated(backend, access)` bọc một `RecordBackend` không tự hỏi câu đó (backend từ xa, wrapper trên dịch vụ trusted) để mỗi thao tác
  hỏi trước. Khác `CrudService`, `Gated` không biết entity có tồn tại không nên entity lạ trả 403 thay vì 404.
- Hậu quả: tên `Trusted` nói thẳng ở chỗ gọi rằng kiểm tra role đang tắt; mọi đường còn lại đi qua `NewCrudService`/`WithAudit` vẫn
  kiểm tra như trước. Chi phí của hai lần kiểm tra (cổng và service) gần như bằng 0 vì cùng đọc từ bộ nhớ.

### 2.4 Handler action có kiểu và có ngữ cảnh

- Handler nhận input/output là struct Go (huma đã dùng cách này ở `typedapi.go`), record đã nạp cho action cấp record, và
  `*action.Ctx` với `Records` (đã bọc `Gated`), `Audit`, `Emit`, `Scope`, `Deps` (một struct dependency cho mỗi domain).
- `a.Audit(...)` và `a.Emit(...)` **chỉ xếp hàng**, được ghi cùng transaction của bước đó khi commit; handler lỗi thì không ghi gì.
- **Audit và event là tùy chọn.** Không gọi thì không có (ngoài audit theo cấu hình `Audit.Enabled` của entity khi đi qua
  `a.Update`). Chỉ thêm một quy tắc có test: mọi thao tác ghi thuộc nhóm `Admin` và bảo mật phải có audit.
- Tương thích ngược: `ActionHandler` cũ chạy song song; port dần. Đã có bản đầu (P3): audit của action ghi là `audit.ActionCustom` ("action") với tên action trong `transition_action`, nằm chung chuỗi băm của tenant.

### 2.5 Hai chế độ truy cập dữ liệu

| | `AsCaller()` (mặc định) | `Within(scope)` |
|---|---|---|
| Dùng cho | Thao tác người dùng, vài record | Nghiệp vụ lớn, nhiều bảng |
| Authz | ABAC từng record | RBAC + một `Scope` khai báo, một lần ở đầu |
| Dữ liệu | CRUD đầy đủ | Repository ép phạm vi bằng `WHERE`/kiểm tra khi ghi, không đánh giá policy từng dòng |

`Scope` tái dùng `ScopeBinding` đã khai trên entity gốc (ví dụ `account → accountId`), không thêm khái niệm mới. Dữ liệu nhạy
cảm theo từng record (mask field theo người đọc) phải dùng `AsCaller()`.

### 2.6 Nghiệp vụ nhiều bước: Operation gồm Step

Handler chạy **ngoài transaction**; mỗi `Step` là một transaction ngắn, idempotent theo tên, retry bỏ qua bước đã xong. Có nhật
ký operation (để retry, không phải audit cho người dùng). `Atomic()` là lựa chọn tường minh cho chỗ cần nguyên tử thật sự. Việc
thật sự lớn chạy dưới dạng job bất đồng bộ (`pkg/jobs`). Lý do: giữ connection khi chờ I/O là nguyên nhân deadlock pool đã sửa ở
metap #88 và #89.

**Cùng transaction (P3):** mỗi `Step` là một unit of work (`tenancy.Router.BeginUnit`). Trong context của nó, `Router.Begin` trả về
một savepoint trên transaction của unit thay vì lấy connection mới, nên ghi record qua CRUD, đọc policy và xếp event vào outbox cùng
chạy trên **một** connection. Điều này giữ đúng quy tắc của #88/#89 (không đường nào giữ một connection trong khi cần cái thứ hai;
một pool chỉ có 2 connection vẫn phục vụ được, và test khẳng định một step giữ đúng một connection). Một thao tác lỗi bên trong unit
chỉ hoàn tác chính nó; unit rollback thì hoàn tác tất cả. Việc không đi qua `Router.Begin` (audit store có chuỗi băm riêng, cache
trong bộ nhớ) đăng ký bằng `tenancy.OnCommit` để chạy sau commit và không chạy khi rollback: audit của CRUD và của action đều theo
cách này, nên một ghi bị hoàn tác không để lại audit entry. Giới hạn: một backend từ xa (không phải `CrudService` trong process) không
tham gia unit; và vì step giữ một connection nên `fn` của step chỉ nên đọc và ghi record của tenant, không gọi dịch vụ khác.

### 2.7 DB ngoài metap

Adapter của store ngoài **bắt buộc nhận `Scope` làm tham số** (`Query(ctx, scope, q)`), tự biên dịch thành `WHERE` hoặc filter.
Kiểu `Scope` nằm trong `pkg/kit` (thuần stdlib) vì `e-edge` và `a-api` chỉ được import `pkg/kit/*` (ADR-0009, depguard).

## 3. Test

| Tầng | Cách test |
|---|---|
| Logic nghiệp vụ | Hàm thuần, table-driven |
| RBAC | `RoleMatrix{...}` literal, không DB |
| Handler | `actiontest.Run(handler, in, ...)` trả kết quả, audit và event đã xếp hàng, các `Update` đã thực hiện; không Postgres, không RabbitMQ |
| Nhiều bước | Chèn lỗi ở bước *n* để kiểm tra resume |
| Adapter DB ngoài | `Scope` cho trước thì `WHERE` sinh ra phải chứa tenant/zone, không có đường thoát |
| Cổng route | Mọi route thuộc đúng một nhóm access |
| Tích hợp | `-tags integration` cho ít luồng thật (reconcile, transaction, outbox) |

## 4. Thay đổi phá vỡ

1. `crud.Create/Update/Delete/Transition` giữ chữ ký; thêm `Trusted()` và `Gated(...)`; `PermissionChecker` tách `EntityAccess` ra.
2. `PermissionService.CanXEntity`: vẫn là nguồn câu trả lời, nhưng đọc từ `MatrixStore` (bộ nhớ) thay vì truy vấn mỗi lần.
3. `internal/authctx`: role cache có TTL (session vẫn tức thì).
4. `internal/admin`: bỏ `adminCaller(ctx)` ở từng phương thức (cổng nhóm `Admin` lo); bỏ kiểu protobuf còn sót.
5. `ActionHandler`: giữ, đánh dấu lỗi thời; handler mới dùng `action.*`.
6. waap: port dần `zones/actions.go` (thử trước `domains.verifyDns` và `zones.applyPreset`), rồi các domain còn lại.

## 5. Phương án đã loại

- **Giữ kiểm tra lần hai trong `CrudService`** (phòng thủ chiều sâu): loại, thay bằng `Gated`/`Trusted` để không kiểm tra hai lần.
- **RBAC chỉ ở code (không DB):** loại, tenant admin cần chỉnh quyền lúc chạy (màn hình policy trong portal đang dùng).
- **Một transaction cho cả nghiệp vụ lớn:** loại, giữ lock lâu và gây lại deadlock pool.
- **Ép lý do khi tắt audit (`NoAudit(lý do)`):** loại cho action của app; chỉ giữ test cho nhóm `Admin`/bảo mật.

## 6. Chưa chốt

- Chữ ký generic cuối cùng của `action.*` và cách sinh metadata (ảnh hưởng form tự sinh của `platform-ui`): chốt sau khi thử trên hai action đầu.
- TTL đã chốt 45 giây cho cả role và policy (chủ dự án, 2026-11-24). Còn chưa chốt: cách báo nạp lại giữa nhiều instance (outbox/RabbitMQ hay kiểm tra version định kỳ); hiện chỉ dựa vào TTL, và policy/role ghi qua process này thì có hiệu lực ngay chỉ với policy ghi qua `PermissionService`.
- Cách tránh để edge nhận rule-set lưng chừng giữa các `Step` (kết quả P0, §7): bước cuối là bước kích hoạt, hoặc một event tổng hợp ở cuối, hoặc `Atomic()` cho thao tác đổi trực tiếp rule-set.
- Nghiệp vụ dở dang: bù trừ, resume hay chấp nhận trạng thái một phần, quyết định theo từng nghiệp vụ.

## 7. Chưa kiểm chứng (không ghi như sự thật)

- ~~`c-config-distributor` có chịu được nhiều event cho một thao tác~~ **Đã kiểm chứng (P0, 2026-11-24, chỉ đọc code):** chịu được. `sync.Zone`
  đọc lại trạng thái hiện tại từ management cho mỗi event, biên dịch và publish idempotent, không gộp event; lần chạy cuối luôn ra
  trạng thái mới nhất, và resync mỗi 60 giây (`RESYNC_INTERVAL_SECONDS`) là chốt hội tụ. Điều cần xử lý không phải số event mà là
  **trạng thái trung gian**: các `Step` commit độc lập nên edge có thể nhận rule-set lưng chừng giữa hai bước (xem §6).
- ~~Có phát event CRUD thông thường ngoài `workflow.EmitTransitioned`~~ **Đã kiểm chứng (2026-11-24):** có. `persist` xếp `workflow.EmitCreated`, `EmitUpdated`, `EmitDeleted` và `EmitTransitioned` trong cùng transaction với ghi.

## 8. Ngoài phạm vi

- Điều kiện quan hệ qua entity khác: `RequiredRelationFields` chỉ thu thập tên field trong condition, **chưa có** cơ chế loader.
  Nếu cần, là tính năng mới, làm riêng.
- Condition record không đẩy xuống SQL được cho list (`CrossRecordConditionInListError`, `UnsupportedContainsConditionInListError`)
  vẫn trả 500 như hiện nay.

## 9. Thứ tự làm và quy ước git

Thứ tự (mỗi bước là một PR, merge **squash** theo quy ước của repo):

| Bước | Nội dung | Repo |
|---|---|---|
| P0 | Kiểm chứng distributor với nhiều event (chỉ đọc). **Xong**, kết quả ở §7 | waap |
| P1 | `RoleMatrix`, role cache có TTL (session tức thì), middleware cổng ở router. **Xong**: cache role 45 giây (`ROLE_CACHE_TTL_SECONDS`), `MatrixStore` giữ policy cấp entity của tenant trong bộ nhớ 45 giây (`POLICY_CACHE_TTL_SECONDS`), cổng `gateEntity`/`gateTransition` trên các route record. Khác bản đầu: ma trận giữ **mọi** policy cấp entity (cả deny, điều kiện theo context, scoped role), không chỉ dòng RBAC thuần, nên quyết định không đổi | metap |
| P2 | Pipeline cho CRUD; `Trusted`/`Gated`. **Xong** (xem §2.3 cho điều chỉnh so với bản đầu) | metap |
| P3 | `action.Ctx`, `Step`/Operation, `actiontest`; `ActionHandler` cũ chạy song song. **Xong** (`pkg/action`, `pkg/action/actiontest`, `AppState.ActionDeps`, `tenancy.Unit`): `action.Entity`/`action.Record` với input và output có kiểu, input giải mã nghiêm ngặt, `Ctx.Records`/`Audit`/`Emit`, `Ctx.Step` là một unit of work (ghi record, event và audit commit cùng nhau, trên một connection), và `EntityAction.Roles` (role được chạy action, policy `action:<tên>` của tenant ghi đè, kiểm tra trước khi đọc input). **Quyết định bỏ**: nhật ký operation để retry bỏ qua bước đã xong (mỗi bước đã nguyên tử và việc cần retry viết bằng upsert; chỉ làm nếu có action không idempotent). **Còn lại**: `Within(scope)` và repository ép scope, làm theo nhu cầu thật khi port ở P4/P5 | metap |
| P4 | `internal/admin` theo pattern mới (bỏ `adminCaller`, thêm audit và transaction qua `persist`) | metap |
| P5 | Port `zones` (hai action đầu rồi phần còn lại). **Hai action đầu xong** (nhánh `feat/120-typed-actions` của waap): `waf.domains.verifyDns` (`netcheck.VerifyDomain`) và `waf.zones.applyPreset` (`zones.ApplyPreset`). Kết quả: `verifyDns` có 9 test đơn vị không cần DB; `applyPreset` giờ là một step nên khi plan từ chối rule thứ hai thì zone không giữ lại policy DDoS, rule thứ nhất hay version nào (test mới fail trên code cũ). Cần thêm ở metap: `Ctx.Current` (record và version của action trên một record), và `actiontest.Records` biết `List` và làm hỏng một lần ghi chọn trước. 8 action còn lại của zones, và scanning/alerting/billing, chưa port | waap |
| P6 | `Scope` trong `pkg/kit` cho adapter DB ngoài | metap, `a-api` |

**Quy ước git cho đợt này** (theo lịch sử các repo):
- Nhánh: `docs/120-rbac-router-action-ctx` (metap-docs), `feat/120-rbac-router-action-ctx` (metap), `feat/120-...` (waap khi tới P5). Mỗi bước dài có thể tách `feat/120-p1-role-matrix`.
- Commit: tiếng Anh, câu mệnh lệnh viết hoa chữ đầu, kết thúc bằng số PR khi merge (`... (#NN)`); tài liệu roadmap dùng `roadmap 120: ...`.
- Merge: **squash**, chỉ khi người dùng nói "merge" và CI xanh; chỉ commit repo nào được nêu tên.
- Con trỏ submodule ở root `metap-org` chỉ bump khi được yêu cầu, sau khi các nhánh đã vào `master`/`main`.
