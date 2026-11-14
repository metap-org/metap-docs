# 106. Mỗi field một cột thật, bỏ `data jsonb`

Trạng thái: **đang làm** (2026-11-14); đã diễn tập WAF trên DB sao chép (mục 8). S1 đến S5 đã xong ở `metap` (commit `1604ef2`), S6 và S7 chưa. Trigger: chủ dự án không muốn giá trị field của entity nằm trong cột `data jsonb`.
Quyết định (chủ dự án, 2026-11-14): mỗi field một cột có kiểu; có migration chuyển dữ liệu hiện có từ `data` sang cột.

## 1. Hiện trạng

Mỗi entity có bảng riêng (Phase 86 đã bỏ bảng `records` chung), nhưng mọi field nằm trong `data jsonb`. Chỉ hai trường hợp có cột thật:
field `Reference` có `RefEntity` và field khai `Storage: column`. Cột thật đó là **bản sao** đồng bộ bằng trigger (`OriginGenerated`),
nguồn sự thật vẫn là `data`. Index ở các field còn lại là index biểu thức `((data ->> 'f')::type)`.

Hệ quả đã gặp: sự cố số 8 và 9 trong `metap-demo-waf/CLAUDE.md` (trigger đồng bộ biến mất, backfill không chạm dòng nào) đều sinh ra từ
việc phải giữ hai bản dữ liệu. Index biểu thức khó đọc, không có ràng buộc kiểu, không có `NOT NULL`, không có FK cho field thường.

## 2. Thiết kế đích

**Bảng entity** = cột hệ thống + **một cột cho mỗi field**, đặt tên đúng tên field, kiểu theo `FieldKindSQLType`
(`text`, `double precision`, `numeric(18,4)`, `boolean`, `date`, `timestamptz`, `uuid`, `jsonb` cho kind `json`).
Cột `data` bị bỏ. Cột hệ thống giữ nguyên: `id, tenant_id, code, status, version, deleted, created_at, updated_at, created_by, updated_by`.
Tên field trùng cột hệ thống (trừ `code` và `status` đã được ánh xạ sẵn) vẫn bị `Compile` từ chối.

**Ranh giới dịch vụ không đổi.** `RecordDto.Data` vẫn là một map JSON; REST/GraphQL/gRPC, permission, validation, workflow, audit không phải biết
cách lưu. Việc dịch giữa JSON và cột nằm gọn trong `pkg/crud` và `pkg/query`:

- **Đọc:** danh sách cột chọn ra gồm `jsonb_build_object` chỉ các field không NULL, gán bí danh `data`. Mã quét dòng (`scanRecordRow`) giữ nguyên.
- **Ghi:** `INSERT ... SELECT ... FROM jsonb_to_record($n::jsonb) AS r("f" <kiểu>, ...)` và `UPDATE ... SET "f" = r."f" ... FROM jsonb_to_record(...)`.
  Payload vẫn là JSON đã qua `ValidatePayload`; Postgres ép kiểu theo cột. Field `system` không nằm trong `SET` của `Update` (giữ hành vi hiện tại).
- **Truy vấn:** `QueryPlanner`, JQL, aggregate, bộ lọc record-level và `find_referencing_records` dùng `"f"` thay cho `data ->> 'f'`.
  `FieldHasRealColumn` luôn đúng nên các nhánh sẵn có tự chuyển.

**Index và ràng buộc** tính trực tiếp trên cột: `unique` là partial unique index (`WHERE deleted = false`), tenant-scoped như Phase 103;
`searchable` là GIN (trigram hoặc tsvector) trên cột; `Reference` có FK như cũ. Không còn trigger đồng bộ, không còn `OriginGenerated`.

**Sai khác dữ liệu chấp nhận được** (ghi lại để không ai ngạc nhiên): `date`/`timestamptz` đọc ra theo định dạng ISO của Postgres
(`timestamptz` có offset `+00:00`); `money` đọc ra có 4 số thập phân; field không có giá trị không xuất hiện trong `data` (giống trước).
Giá trị JSON `null` của field thường được coi là "không có giá trị".

## 3. Migration dữ liệu hiện có

Reconciler đã có sẵn thành phần: `AddColumn`, `BackfillColumn` (batch, checkpoint, resume), preflight bằng `pg_input_is_valid`, quarantine,
`dev-tools reconcile-plan` (xem SQL trước khi chạy). Thêm:

1. **`AddColumn`** cho mọi field chưa có cột (nullable), kèm tạo index mới trên cột.
2. **`BackfillColumn`** `col = (data ->> 'f')::type` theo lô, có checkpoint. Dùng `BackfillScope::AllTenants` cho bảng dùng chung nhiều tenant.
   Dòng không ép được kiểu đi vào quarantine như thao tác `widen_type` hiện nay, không bị bỏ qua im lặng.
3. **Cổng kiểm** trước khi bỏ `data`: mọi field có giá trị trong `data` đều có giá trị khớp trong cột (đếm theo từng field), không còn dòng quarantine chưa xử lý.
4. **`DropDataColumn`**: chỉ chạy khi qua cổng kiểm; kéo theo bỏ index biểu thức cũ. Là bước **riêng, chủ động**, không tự chạy trong `reconcile()` thường.
   Trước đó entity ở chế độ chuyển tiếp: ghi vào cả `data` lẫn cột, đọc từ cột.

Bài học đã ghi ở Phase 79 (mất dữ liệu vì giả định bảng rỗng): trước mỗi migration phải **truy vấn số dòng thật** theo (tenant, entity),
chụp snapshot DB, và so sánh số dòng/giá trị sau khi chuyển. Không dựa vào "không có seed script".

## 4. Các lát

| Lát | Nội dung | Kiểm chứng |
|---|---|---|
| S1 (xong) | `metadata`/`reconciler`: `Compile` mọi field thành cột; index/unique/searchable trên cột; op `DropDataColumn`; chế độ chuyển tiếp | unit không DB; e2e reconcile trên PG16: bảng mới không có `data`, bảng cũ hội tụ 1 lượt |
| S2 (xong) | `crud`: đọc/ghi/transition/delete/get_many theo cột; `unique_violation` theo tên index mới | e2e `crud_service_postgres`: mọi kind, partial update, system field, version conflict |
| S3 (xong) | `query`: planner, JQL, aggregate, record-level policy, keyset cursor; `find_referencing_records` | e2e planner/aggregate; so kết quả với bản `data` cũ trên cùng dữ liệu |
| S4 (xong) | Nơi đọc `data` còn lại: `AUTH_CONTEXT_ENTITY`, `workflow`, audit đọc bản ghi cũ, `grpcapi` | grep không còn `data ->>` ngoài bảng nền tảng |
| S5 (xong) | Công cụ migration: `dev-tools migrate-to-native-columns <tenant> <entity>` (plan, chạy, kiểm số dòng), cổng kiểm, `DropDataColumn` | e2e: dữ liệu giả lập đủ kind, kể cả dòng không ép được kiểu |
| S6 | `metap-lowcode`: publish entity tạo cột; đổi field = add/rename/widen/drop có sẵn; entity cũ migrate | build + test repo đó |
| S7 | Downstream: `metap-demo-waf` (9 entity, dữ liệu thật), `metap-demo-jira`; cập nhật CLAUDE.md và tài liệu | chạy migration trên bản sao DB, so số dòng; portal đọc ghi được |

Mỗi lát phải qua `go build ./... && go vet ./... && go test ./...` và `go test -tags integration` trên các package nó chạm trước khi sang lát sau.

## 5. Rủi ro và điểm mở

- **Đổi schema khi đổi field** (đặc biệt entity low-code): `ADD COLUMN` nullable là tức thời, đổi kiểu/bỏ cột đã có thao tác migration an toàn từ Phase 79;
  khác trước đây là *mọi* đổi field đều là DDL. Chấp nhận, đã thống nhất.
- **Giới hạn số cột** của Postgres (1600) là trần cứng cho số field một entity; chưa entity nào gần mức đó.
- **Khác biệt định dạng ngày/tiền** ở mục 2 có thể làm lệch test snapshot ở downstream; sẽ sửa test, không đổi cách lưu.
- **Chưa quyết:** có cần `NOT NULL` cho field `required` không. Đề xuất chưa làm trong đợt này (thêm `NOT NULL` vào bảng đang có dòng cần backfill giá trị mặc định); validation ở `CrudService` vẫn là chốt chặn.
- **Chưa quyết:** có giữ chế độ ghi kép (`data` và cột) sau lát S5 không. Đề xuất bỏ ngay sau khi qua cổng kiểm để không có hai nguồn sự thật.

## 6. Kết quả S1 đến S5 (2026-11-14)

Đã kiểm trên PostgreSQL 16 thật: unit và integration của `pkg/reconciler`, `pkg/crud`, `pkg/query`, `pkg/grpcapi`, `pkg/httpserver`, `pkg/peripherals` đều xanh
(riêng `pkg/cache` và `pkg/secrets` cần Redis và Vault, không có trong môi trường này, không liên quan). Chi tiết đã làm khác với dự kiến:

- Bảng còn cột `data` bị **từ chối** (`ErrLegacyDataColumn`) thay vì chuyển tự động; phải chạy `dev-tools migrate-to-native-columns` trước khi triển khai mã mới. Lý do: dịch vụ mới chỉ đọc cột, bảng chưa backfill sẽ trả NULL.
- Trigger đồng bộ cũ phải bị bỏ khi chuyển, nếu không nó ghi đè giá trị cột bằng NULL (đã có test).
- Thao tác migration khai báo là DDL cột: mở rộng kiểu là `ALTER COLUMN TYPE ... USING` (viết lại bảng, khoá ACCESS EXCLUSIVE); đổi tên field đã là `RenameColumn` của Diff.
- Field `searchable` chỉ cho kind `string`/`enum` (trigram và tsvector chạy trên text); kind khác bị `Compile` từ chối.
- Field tên `data` bị từ chối (trùng cột cũ trong lúc chuyển).
- `ParamBuilder.SetEntity` mang danh sách field cho điều kiện policy record-level; field không khai báo trong điều kiện luôn không khớp (`NULL::text`), đúng như trước.
- `CopyGenericRecords` (bảng `records` chung đã bị bỏ ở migration 0033) viết lại để rải `data` lên các cột của bảng đích; không còn nguồn nào dùng nó trong thực tế.

Chưa làm: `NOT NULL` cho field `required` (đề xuất ở mục 5, giữ nguyên), S6 (`metap-lowcode`), S7 (`metap-demo-waf`, dữ liệu thật).

## 7. Đổi entity sau này, và rollback (2026-11-14)

Trả lời câu "đổi entity thì làm gì, migrate xong mà ứng dụng lỗi thì lùi thế nào":

- **Không có file migration sinh ra.** Đổi `EntityDefinition` trong code rồi khởi động lại; `reconcile()` tự áp phần chỉ-thêm (cột nullable, index `CONCURRENTLY`, FK).
  Xem trước bằng `dev-tools reconcile-plan`, cổng CI bằng `--fail-on-destructive` (thoát khác 0 nếu kế hoạch bỏ FK/unique/trigger hoặc có đổi kiểu cột).
- **Đổi kiểu field không còn bị bỏ qua im lặng**: `ErrColumnTypeChanged`, phải chạy `widen_type` (viết lại bảng) rồi mới reconcile.
- **Rollback thiết kế chỉ-thêm**: thêm field/index thì bản cũ bỏ qua cột mới, lùi chỉ là deploy bản cũ. Đổi tên/xoá field/đổi kiểu là thao tác một chiều, làm theo hai bản phát hành (thêm trước, dọn sau khi hết thời gian có thể lùi).
- **Rollback lần chuyển sang cột thật**: `dev-tools restore-legacy-data` dựng lại `data` từ các cột (cả sau `--drop-data`), tạo lại trigger đồng bộ cũ, xoá dấu "đã backfill" để lần chuyển tiếp theo chép lại từ đầu. Đã test trên PostgreSQL 16: chuyển, drop, ghi một dòng bằng cột, restore, bản cũ ghi một dòng vào `data`, chuyển lại, dòng đó sang cột.
- **Còn thiếu**: lùi được từng thao tác DDL riêng lẻ (mỗi thao tác là một transaction riêng, chạy dở thì chạy lại để hội tụ); lùi `widen_type`; tự động hoá snapshot trước migration.

## 8. Diễn tập migration cho WAF (2026-11-14)

Chạy trên một DB sao chép, không phải DB dev thật (phiên làm việc không có bản sao đó). Dựng DB bằng metap trước PR #42 để
10 bảng `waf.waf_*` có đúng dạng cũ (cột `data jsonb`), seed khoảng 490 dòng cho 2 tenant (number, boolean, enum, json, reference).

- `migrate-to-native-columns --all-tenants` rồi `--drop-data` cho cả 10 bảng: sạch. So lại bằng SQL độc lập 56 cột trên 9 bảng, khớp hết với `data`
  (bảng thứ 10, `ip_access_lists`, migrate sạch nhưng không có cột nào để so).
- 3 service mới boot 2 lần trên DB đã chuyển: cả 2 lần `ops_applied=0` cho 10 entity.
- `zones-service` đọc dữ liệu đã chuyển qua gRPC, sắp xếp theo số đúng thứ tự; `datetime` ra dạng `+00:00` như đã nêu ở mục 2.
- `restore-legacy-data` khôi phục 49/49 dòng khớp, sau đó migrate lại được.

Hai điều phát hiện:

1. **`loadEntityFromFile` nhận file sai dạng thành entity rỗng**: lệnh migrate không làm gì mà vẫn báo "Verification is clean". Đã sửa (entity không tên là lỗi, nhận cả dạng trần lẫn dạng bọc `{"data": ...}`), có test.
2. **Một dòng có giá trị sai kiểu trong `data` (ví dụ `"n/a"` ở field số) làm migration dừng**, thông báo là lỗi Postgres không nêu dòng hay field. Hành vi an toàn (không bỏ sót dữ liệu) nhưng khó dò: trước khi chạy trên DB thật nên quét các dòng như vậy bằng SQL.

Chưa làm: chạy trên DB dev thật của WAF (cần snapshot trước, chạy theo thứ tự domains, zones, ddos_policies, firewall_rules, ip_access_lists,
scan_jobs, scan_findings, alert_policies, alert_notifications, incidents), `metap-lowcode` (S6, hoãn theo ưu tiên của chủ dự án), `metap-demo-jira`.

