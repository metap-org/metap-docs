# Computed/derived field

- **Trạng thái:** done (2026-09-02) — `EntityField.computed`/`ComputedSpec` (`metap-metadata`),
  validation trong `compiler::validate` (8 test), `computed::{expression_tokens,render_expression}`
  dùng chung giữa validation và runtime (5 test), `CrudService::recompute_fields` wire vào
  `create`/`update`, verify sống qua Postgres thật
  (`computed_field_is_recalculated_on_create_and_update`, xác nhận cả việc bỏ qua giá trị client
  gửi cho field computed). `cargo build/test --workspace` sạch.
- **Người đề xuất:** ghi lại từ thảo luận kiến trúc, `docs/team-charter.md` ("Định hướng đang ghi
  nhận, chưa có trigger" #5)
- **Track sở hữu:** Backend Core
- **Phase roadmap liên quan:** chưa gắn phase

## Vấn đề / động lực

Một field tính từ field khác **cùng một record** (vd `displayName = firstName + " " + lastName`)
— mở rộng tự nhiên từ `searchable`/`sortable` đã có sẵn trên `EntityField`
(`crates/metap-metadata/src/entity.rs`). Hôm nay không có cách khai báo field kiểu này — entity
author phải tự tính ở tầng client (không nhất quán giữa REST/webhook/cron) hoặc lưu trùng dữ liệu
thủ công.

## Phạm vi

**Trong phạm vi:**
- 1 field kind/flag mới trên `EntityField` đánh dấu field là computed — vd
  `computed: Option<ComputedSpec>` với `ComputedSpec { expression: String, depends_on: Vec<String> }`
  (cú pháp `expression` tối giản ở v1 — string template `"{firstName} {lastName}"` hoặc tương
  đương, không phải full expression language).
- Recompute chạy trong đúng pipeline `CrudService::create`/`update` đã có (sau validate, trước
  write) — không phải 1 layer riêng, để REST/webhook/cron dùng chung `CrudService` không bao giờ
  tính ra kết quả khác nhau.
- `depends_on` giới hạn field **cùng record** — field phụ thuộc record khác (vd tên hiển thị lấy
  từ 1 `Reference`) **không** thuộc phạm vi field này (xem Ngoài phạm vi).
- Field computed mặc định KHÔNG `searchable`/`sortable` trừ khi được materialize (ghi giá trị thật
  vào cột/JSONB path, không chỉ tính lúc đọc) — vì filter/sort cần giá trị thật trong DB, không
  tính được ở tầng SQL cho 1 expression tuỳ ý.

**Ngoài phạm vi:**
- `depends_on` phụ thuộc record khác — đó là bài toán materialized view/CQRS khác hẳn, không phải
  "recompute on write" đơn giản. Không làm ở v1.
- Không phải computed field kiểu SQL generated column (Postgres `GENERATED ALWAYS AS`) — vẫn tính
  ở tầng Rust trong `CrudService`, không đẩy xuống DB, để giữ đúng nguyên tắc "mọi write đi qua
  CrudService" và validation logic không phân tán giữa Rust và SQL.

## Tiêu chí chấp nhận

- `EntityField.computed` khai báo được, `MetadataCompiler::validate` bắt lỗi nếu `depends_on`
  tham chiếu field không tồn tại trên cùng entity, hoặc field đó tự tham chiếu chính nó (cycle
  đơn giản nhất).
- `CrudService::create`/`update`: field có `computed` bị client gửi lên trong payload → bị bỏ qua
  (server luôn tự tính, không tin giá trị client gửi — cùng nguyên tắc field server-controlled
  khác, vd `id`/`version`).
- 1 entity test có field computed (vd `displayName` từ `firstName`+`lastName`): tạo record chỉ gửi
  `firstName`/`lastName` → `displayName` tự tính đúng trong response; update `firstName` →
  `displayName` tự tính lại.
- `GET /metadata/entities/{name}` trả đúng `computed` trong field metadata (để FE biết field này
  read-only, không hiện ô nhập).

## Ranh giới kiến trúc bị đụng tới

- `crates/metap-metadata/src/entity.rs` — thêm field mới trên `EntityField` (additive, ~50 struct
  literal chỗ khác trong workspace cần thêm `computed: None` — kinh nghiệm từ lần thêm
  `field_display_hints`/`related_views` trước đó: nên đăng ký qua cơ chế tương tự nếu số lượng
  literal bị vỡ compile lớn, nhưng `computed` là field-level nên nằm trong `EntityField` không
  tránh được như 2 ví dụ kia — kiểm tra số lượng call site thật trước khi code, không giả định).
- `crates/metap-crud/src/crud_service/create.rs`/`update.rs` — thêm bước recompute sau validate.
- `crates/metap-metadata/src/openapi.rs` — field mới cần lộ ra JSON Schema.
- `platform-ui`'s `generated-types.ts`/form field renderer — cần biết field computed để disable
  input, không phải phần của brief này (theo track Frontend Platform riêng khi implement FE).

## Rủi ro / phụ thuộc

- **Không có trigger từ 1 entity thật trong repo** — chủ dự án chọn chủ động implement dựa trên
  2 điểm thiết kế đã nghĩ sẵn trong `team-charter.md`, không chờ nhu cầu thật xuất hiện trước.
  Rủi ro: thiết kế `ComputedSpec`/cú pháp `expression` có thể cần đổi khi có use case thật đầu
  tiên — giữ v1 tối giản (string template, không phải expression engine đầy đủ) để dễ đổi sau.
- Rủi ro số lượng struct-literal call site cần sửa nếu `EntityField` construct trực tiếp (không
  qua builder) ở nhiều nơi trong workspace — cần grep trước khi code, không đoán.
