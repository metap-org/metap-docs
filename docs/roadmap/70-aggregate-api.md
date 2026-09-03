## Phase 70: Aggregation API (`plan_aggregate`) — `GROUP BY`/`COUNT` cho mọi entity (2026-09-03)

Trigger: rà soát "còn phần nào rất to chưa làm" cho `metap-demo-waf` phát hiện **`metap` core chưa
có bất kỳ khả năng aggregate nào** — grep `GROUP BY`/`COUNT`/`aggregate` trong `metap-query` +
`metap-crud` + `metap-dashboards` ra 0 kết quả; `metap-dashboards` chỉ có đúng 1 struct
`DashboardConfig` lưu layout JSON, không tính toán gì; không có route analytics nào trong 15 route
group của `metap-http`. Nghĩa là mọi màn hình dashboard/analytics của mọi app dựng trên platform
này đều phải kéo record thô về đếm ở browser — sai 2 lần: chặn ở `max_limit` của list view (nên
mọi con số âm thầm thành "tối đa 50"), và chuyển cả bảng dữ liệu qua mạng để hiện 1 số nguyên.

Đây là gap ở **tầng platform**, không phải của riêng app WAF — nên sửa ở core.

## Đã làm

- **`metap-query::aggregate`** (module mới) — `plan_aggregate()` cùng shape chữ ký với
  `plan_list()`: nhận `MetadataRegistry` + `PermissionService` + `RequestContext` +
  `record_read_policies`, trả `PlannedAggregateQuery { sql, params }`.
  - 5 hàm aggregate (`count`/`sum`/`avg`/`min`/`max`) là **enum đóng**, không phải chuỗi client gửi
    lên — "hàm SQL nào được chạy" không phải input của client.
  - `group_by` chỉ nhận field đã `indexed`, có `enum_values`, nằm trong `filters` của list view,
    hoặc tên `status`. Field khác trả 400 chứ không nội suy vào SQL.
  - `sum`/`avg`/`min`/`max` bắt buộc field `Number`/`Money`; `count:<field>` ("số bản ghi có set
    field này") là ngoại lệ duy nhất được dùng trên field không phải số.
  - `bucket` (`hour`/`day`/`week`/`month`) → `date_trunc`, đọc `createdAt`/`updatedAt` (cột thật)
    hoặc 1 field `Date`/`Datetime` của entity; `since`/`until` bind text rồi cast, không nội suy.
  - `MAX_GROUPS = 500` — chặn cứng số nhóm trả về. Không có cái này thì `GROUP BY sourceIp` là
    đường dump cả bảng qua 1 endpoint analytics.
- **Bảo mật đi kèm `plan_list` chứ không phải bản yếu hơn**: luôn scope `tenant_id`, luôn
  `deleted = false`, và **record-level (ABAC) read policy được gấp vào `WHERE` trước khi
  aggregate** — người không đọc được 1 row thì cũng không được biết row đó tồn tại qua phép đếm.
  Ghi rõ trong doc comment: **field-level masking chưa có tương đương** và cố ý không giả vờ có —
  cần luồng readable-field set vào planner, để lại làm sau (hiện trạng trước phase này là *không có
  gì cả*, nên đây không phải bước lùi).
- **`CrudService::aggregate`** — cùng cổng permission `list` dùng (`can_read_entity` + snapshot +
  record policies), thực thi qua `Router::begin` (tenant-routed), map `InvalidAggregateError` →
  400 `invalid_aggregate`.
- **`POST /api/{entity}/aggregate`** (`metap-http`) — `POST` vì tập metric/filter là object có cấu
  trúc chứ không phải map chuỗi phẳng; về ngữ nghĩa vẫn là read, gác bằng `AuthContext` y hệt
  `list_records`. axum khớp segment tĩnh `aggregate` trước `{id}` nên không đụng route cũ.
- Kết quả trả về: `to_jsonb(agg)` — 1 JSON object/row **dựng trong database**. Lý do: projection do
  caller định hình nên không biết kiểu cột lúc compile-time, không thể mô tả cho sqlx như tập cột
  có kiểu. Mọi metric ép `::float8` để client charting không gặp series này là number, series kia
  là string.

## Xác minh

**Chưa build, chưa test, chưa chạy** — phiên này chủ dự án yêu cầu rõ "không verify code, không
cần build, không cần test, chỉ code". Toàn bộ phase này (và Phase 71) cần 1 vòng
`cargo build`/`clippy`/e2e trước khi tin được. Rủi ro đã biết: `to_jsonb` + `apply_params` mới chỉ
đọc code chứ chưa chạy thật lần nào.

## Consumer đầu tiên

`metap-demo-waf`'s portal (Phase 71) — Dashboard/Analytics/Findings đều dựng trên endpoint này.
