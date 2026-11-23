# 118. Một router HTTP, một thư viện handler, và phân lớp cho route (BẢN NHÁP, chờ duyệt)

Chủ dự án nhận xét (2026-11-23): tầng HTTP của metap đang lẫn lộn. Cùng một codebase dùng `chi` ở chỗ này và
`http.ServeMux` có sẵn ở chỗ khác; convention của handler không thống nhất; phần admin chưa chia lớp đúng. Cho
phép chọn chỉ dùng mux có sẵn và viết một thư viện handler nhỏ trong metap để mọi nơi dùng chung.

Tài liệu này ghi hiện trạng đo được, quyết định đề xuất, quy ước, và các pha làm. Không đổi hành vi HTTP bên ngoài
(đường dẫn, mã trạng thái, thân phản hồi giữ nguyên); đây là đổi cấu trúc.

## 1. Hiện trạng (đọc từ mã, 2026-11-23)

| Điểm | Thực tế |
|---|---|
| Router | `chi` ở 24 file của metap (66 chỗ dùng API của chi, trong đó 10 chỗ `With/Use/Group/Route/Mount`); `http.NewServeMux` ở `adminapi/scim.go`, và ở waap: `a-api`, `devtools/*`, `spike/*`; `chi` còn lọt vào `m-management/scanning` và `c-config-distributor/ingest` |
| Typed op | `huma` qua `humachi` (`typedapi.go`, `ops_admin.go`, `ops_sessions.go`) cho các thao tác JSON không phải CRUD |
| Kiểu handler | `func(w, r, state *AppState)` bọc bằng closure ở từng `mount*`; mỗi handler tự gọi `requireAuth/requireAdmin/requirePlatformAdmin` (khoảng 30 chỗ trong 11 file) rồi mới làm việc |
| Hai cách làm cùng một việc | `/admin/*`: người dùng, vai trò, SSO, credential… là typed op + service ở `adminapi`; còn policy, config nền tảng/tenant, preferences là handler thường ở `routes_admin.go`/`routes_*config.go`, tự dựng `map[string]any` (`policyToJSON`) |
| Lớp | `routes_admin.go` chứa nghiệp vụ trong tầng transport: `knownActions`/`isKnownAction`, "seed default policies", "sync matrix policies", gọi thẳng `state.Permissions`/`state.Pool`; `routes_auth/signup/account/sso` chứa điều phối đăng nhập |
| Lỗi/phản hồi | Phần lớn đã thống nhất (`httpx.WriteError`, `*crud.ServiceError`) nhưng typed op và handler thường có hai đường ghi lỗi riêng |
| Đặt tên | `routes_x.go`, `ops_x.go`, `x_route.go`, thư mục `*routes/` cho nhóm tuỳ chọn; mỗi nơi một kiểu `Mount`/`mountX` |

## 2. Quyết định đề xuất

### 2.1 Chỉ dùng `http.ServeMux` có sẵn, bỏ `chi`

Go 1.26 (đang dùng) có mẫu `METHOD /path/{param}`, `{name...}`, `{$}` và `r.PathValue`. Những gì `chi` cho mà metap
thực sự dùng chỉ là: tham số đường dẫn, middleware theo nhóm, và mount nhóm con. Cả ba làm được bằng thư viện nhỏ
(§2.2). Bỏ `chi` còn có lợi: một phụ thuộc ít, một cách đọc tham số (`r.PathValue`), và `edge`/`a-api`/`devtools` (chỉ
được dùng `kit`) cùng dùng được chính thư viện đó.

`huma` giữ lại nhưng đổi adapter sang `humago` (cùng module `huma/v2`, chạy trên `ServeMux`).

Rủi ro đã biết của `ServeMux`, xử lý trong thư viện: (a) 404/405 mặc định là text, không phải envelope lỗi chuẩn;
(b) trùng/xung đột mẫu gây panic lúc đăng ký (tốt: lỗi lộ ngay lúc khởi động, có test); (c) không có middleware theo
route — thư viện bọc handler lúc đăng ký; (d) `OPTIONS` không khớp mẫu có method — middleware CORS bọc ngoài mux nên
vẫn trả lời trước khi định tuyến (đúng như hiện nay).

### 2.2 Thư viện: `pkg/kit/httpx/router` (chỉ stdlib)

Nằm trong `kit` để cả `edge` và `a-api` dùng. Bề mặt nhỏ, cố ý:

```go
r := router.New()                       // bọc *http.ServeMux
r.Use(mw...)                            // middleware toàn cục (áp khi phục vụ, bọc ngoài mux)
r.Handle("GET /api/{entity}/{id}", h)   // mẫu stdlib, h là http.Handler
r.Get/Post/Put/Patch/Delete(path, h)    // viết gọn cho "METHOD path"
g := r.Group(mw...)                     // nhóm dùng chung middleware (rate limit, auth)
g.Get("/me", h)
r.Mount("/scim/v2", handler)            // gắn http.Handler con, tự cắt prefix
```

- Lỗi: `r.NotFound`/`MethodNotAllowed` trả envelope chuẩn `{"error":{code,message}}` (kèm `Allow`).
- Handler trả lỗi: `type HandlerFunc func(w http.ResponseWriter, r *http.Request) error` và `router.H(f)` đổi thành
  `http.Handler`; lỗi chảy về một chỗ ghi (`httpx.WriteError`) — nhận mọi lỗi có `HTTPStatus() int` và
  `Code() string` (`*crud.ServiceError` thoả mãn; `kit` không import `crud`, nên dùng interface).
- Bắt buộc có test: ghép nhóm/middleware đúng thứ tự, 405 có `Allow`, xung đột mẫu panic lúc đăng ký, `PathValue`
  với tên có dấu chấm (`waf.zones`), subtree.

Không tự viết lại thứ `ServeMux` đã làm (khớp mẫu, ưu tiên mẫu cụ thể hơn).

### 2.3 Auth là middleware, không phải dòng đầu của mỗi handler

`httpserver.Authenticated(state)`, `.Admin(state)`, `.PlatformAdmin(state)` là middleware: xác thực một lần, đặt
`*permission.RequestContext` vào `context`, handler đọc `rc := httpserver.RC(r)`. Hệ quả: quên kiểm quyền không còn là
lỗi gõ thiếu một dòng — nhóm route khai báo quyền một lần (`g := r.Group(Admin(state))`), và có test duyệt toàn bộ
mẫu đã đăng ký để khẳng định mỗi route nằm trong một nhóm có khai báo (hoặc nằm trong danh sách công khai rõ ràng).
Điều này cũng đóng đúng loại lỗi đã thấy ở audit 08 mục 2 (route không có kiểm quyền).

### 2.4 Ba lớp, quy tắc import một chiều

```
transport   pkg/httpserver (và các nhóm *routes)   giải mã → gọi service → mã hoá. Không SQL, không pgx, không
                                                    nghiệp vụ, không đọc state.Pool.
application pkg/adminapi, pkg/crud, pkg/identity…  Nhận struct yêu cầu, trả struct phản hồi, lỗi *crud.ServiceError.
                                                    Quyền kiểm ở đây (người gọi lấy từ context).
domain/store pkg/permission, identity, cron, …      Quy tắc và truy cập dữ liệu.
```

Kiểm bằng `depguard` (đã có trong CI): `pkg/httpserver/...` không được import `pgx`, `pgxpool`, `database/sql`;
`pkg/adminapi` không import `httpserver`/`net/http` (trừ `scim.go`, chính nó là một skin HTTP và nên chuyển sang
`httpserver`).

### 2.5 Một cách cho một loại việc

| Loại endpoint | Cách làm |
|---|---|
| Vào/ra JSON | **typed op** (`huma`): có OpenAPI, struct yêu cầu/phản hồi, lỗi qua một đường |
| Không phải JSON: chuyển hướng (SSO/SAML/OAuth2), cookie, stream (export audit, tải tệp), health/metrics/jwks | handler thường qua `router.H` |
| Phản hồi | struct có kiểu, không `map[string]any`; phong bì `{"data":…}`/`{"error":…}` |

### 2.6 Đặt tên và đặt chỗ

- Một nhóm route một file `routes_<resource>.go` trong cùng package với `func mount<Resource>(r *router.Router, d Deps)`;
  gộp `ops_*.go` vào cùng resource (typed op và handler thường của một resource nằm cạnh nhau).
- `Deps`: struct hẹp những thứ nhóm đó cần, thay vì truyền cả `*AppState` (làm theo từng pha, không cần một lần).
- Nhóm tuỳ chọn (`oauth2routes`, `cronroutes`, `dashboardroutes`, `attachmentroutes`) giữ là package riêng nhưng cùng
  chữ ký `Mount(r *router.Router, d httpserver.Deps)`; kiểu `httpserver.Mount` đổi theo.

## 3. Phần admin: nên thành thế nào

| Hiện tại | Đề xuất |
|---|---|
| `routes_admin.go`: policy (liệt kê/tạo/xoá, seed mặc định, đồng bộ ma trận, giải thích), vô hiệu cache ngữ cảnh | `adminapi.PolicyAdminService` (cùng khuôn `UserAdminService`) + typed op; `knownActions`, seed, sync chuyển vào service |
| `routes_platform_config.go`, `routes_tenant_config.go`, `routes_preferences.go` | Service `ConfigAdminService`/`PreferencesService` + typed op; handler chỉ giải mã/mã hoá |
| `routes_account.go`, `routes_auth.go`, `routes_signup.go`, `routes_mfa.go`, `routes_sso.go` | Phần điều phối (đăng nhập, ký phiên, mfaGate, tạo tenant khi ký) chuyển vào `identity`/`adminapi.AuthService`; handler chỉ lo cookie, chuyển hướng, mã trạng thái |
| `adminapi/scim.go` | Chuyển phần HTTP sang `httpserver`; `adminapi` còn lại chỉ là service |

## 4. Các pha

Mỗi pha là một PR riêng, giữ nguyên hành vi, bộ test tích hợp hiện có (khoảng 25 file `*_postgres_test.go` trong
`httpserver`) là lưới an toàn; thêm một test chụp OpenAPI (`/metadata/api.json`) trước pha 2 để phát hiện đường dẫn
hay schema đổi ngoài ý muốn.

| Pha | Việc | Ước lượng | Ghi chú |
|---|---|---|---|
| 1 | `kit/httpx/router` + test; quy ước ghi vào `CLAUDE.md`; test chụp OpenAPI và danh sách route hiện có | 2 ngày | Không đụng code hiện có |
| 2 | Đổi `chi` → `router` ở `httpserver` và 4 package `*routes`, `humachi` → `humago`, `scim` dùng `router.Mount`; waap theo (`scanning`, `ingest`) | 3 ngày | **Đổi chữ ký `Mount`: phá build downstream, cần PR waap đi kèm.** Gỡ `chi` khỏi `go.mod` |
| 3 | Auth thành middleware, bỏ các lời gọi `requireX` ở đầu handler; test "mọi route có khai báo quyền" | 3 ngày | Đổi cơ học, rủi ro thấp, giá trị an toàn cao |
| 4 | Admin: policy → `PolicyAdminService` + typed op; config/preferences | 4 ngày | Giữ đường dẫn và thân phản hồi cũ (typed struct sinh đúng JSON cũ; có test so sánh) |
| 5 | Luồng đăng nhập/ký/mfa/sso: đưa điều phối xuống `identity`/service | 5 ngày | Lớn nhất, làm sau cùng; từng luồng một |
| 6 | Quy tắc `depguard` (cấm `chi`, cấm `pgx` trong `httpserver`, cấm `net/http` trong `adminapi`) + dọn đặt tên | 1 ngày | Khoá kết quả |

Tổng khoảng 18 ngày công cho một người. Pha 1–3 đã đủ để có "một router, một kiểu handler, auth khai báo một lần";
pha 4–6 là phân lớp.

## 5. Điều cần chốt

1. Đồng ý bỏ `chi`, dùng `ServeMux` + thư viện nhỏ ở `kit/httpx/router`? (đề xuất: có)
2. Typed op (`huma`) là cách mặc định cho mọi JSON endpoint, kể cả những chỗ đang là handler thường? (đề xuất: có, trừ
   luồng chuyển hướng/stream)
3. Có chấp nhận đổi chữ ký `Mount` (phá downstream một lần, sửa waap ngay trong PR đi kèm) không?
4. Thứ tự: làm pha 1–3 trước rồi mới tới phân lớp admin, hay làm admin trước vì đó là chỗ đau nhất?
