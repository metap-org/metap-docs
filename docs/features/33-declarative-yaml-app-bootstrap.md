# Bootstrap app qua YAML (entity + `graphql-gateway`) kèm custom handler

- **Trạng thái:** done — code xong, verify bằng unit test + build/clippy/test/fmt sạch toàn
  workspace `metap` (2026-09-07). Xem "Đã ship / Đã verify" dưới.
- **Người đề xuất:** chủ dự án, 2026-09-07 — "hiện tại đang thiếu app config qua yaml, ví dụ như
  graphql gateway, entity define, hiện tại ăn code nhiều quá, yêu cầu hỗ trợ vừa config vừa ăn được
  code custom handler"
- **Track sở hữu:** Backend Core (`metap-metadata`/`metap-app`) + Backend Ops-Infra
  (`metap-graphql-gateway`) — xem `docs/team-charter.md`
- **Phase roadmap liên quan:** không thuộc phase nào

## Vấn đề / động lực

Hôm nay, mọi downstream binary (`../metap-demo-crm`, `../metap-demo-jira`, `templates/metap-app`)
khai báo **toàn bộ** bằng Rust code trong `main.rs`/`entities/*.rs`:
- Từng `EntityDefinition` (field/list-view/workflow) là 1 module Rust build bằng tay.
- `graphql-gateway`'s danh sách upstream chỉ đọc được qua biến môi trường đánh số
  (`UPSTREAM_<N>_{NAME,GRPC_ADDR,METADATA_URL,LOGIN_URL,SERVICE_EMAIL,SERVICE_PASSWORD}`,
  `crates/metap-graphql-gateway/src/config.rs`) — quản lý cồng kềnh khi có nhiều upstream, không
  review được như 1 file cấu hình.

Chủ dự án muốn giảm lượng Rust phải viết cho phần **thuần khai báo** (định nghĩa entity, danh sách
upstream), chuyển sang YAML — nhưng vẫn giữ được chỗ cắm **code Rust tuỳ biến thật** (custom
handler) cho phần không thể khai báo thuần bằng data.

## Rà soát trước khi thiết kế (đã làm 2026-09-07, không suy đoán)

- **`EntityDefinition` đã là data thuần, YAML-hoá được ngay hôm nay** — `crates/metap-metadata/src/entity.rs`:
  toàn bộ struct (`EntityDefinition`/`EntityField`/`EntityListView`/`EntityWorkflow`/
  `WorkflowTransition`/`ComputedSpec`) là `#[derive(Serialize, Deserialize)]` thuần, không có
  closure/trait object nào. `WorkflowTransition.guard`/`validator` là `PolicyCondition` (khai báo,
  JSON-hoá được — lý do nó được un-skip serde 2026-08-17, chính là để entity DB-authored/low-code
  mang nó qua JSON). `EntityField.computed` (`ComputedSpec`) cũng chỉ là 1 string template
  (`"{firstName} {lastName}"`), không phải code. Nghĩa là: **không có gap kỹ thuật nào ngăn 1 file
  YAML deserialize thẳng thành `Vec<EntityDefinition>`** — cái thiếu chỉ là 1 loader tiện dụng, chưa
  ai viết.
- **Chưa có hook nào gắn code Rust tuỳ biến vào từng entity hôm nay.** Grep `guard`/`validator`/
  `set_fields`: toàn bộ là `PolicyCondition`/`PolicyValue` khai báo, không có closure/callback. Chỗ
  "code tuỳ biến" duy nhất đã tài liệu hoá nằm ở **mức binary/route, không phải mức entity**:
  `metap-app`'s doc comment nói rõ 1 downstream binary tự mount thêm `axum::Router` qua
  `metap_http::build_router`'s `extra_routes`, có `Router::pool_for`/`AuthContext`/`AdminContext`/
  `EventBus::subscribe` sẵn — tức logic tuỳ biến là 1 route riêng viết tay trong `main.rs`, không
  đính kèm vào `EntityDefinition`.
- **Tiền lệ liên quan đã có, và đã CHỦ ĐỘNG loại bỏ hướng "code trong YAML"** —
  `docs/features/06-async-verification-pattern-and-lowcode-custom-logic.md` (2026-08-26): bàn đúng
  câu hỏi "entity low-code gắn logic tuỳ biến kiểu gì" và **explicitly không đề xuất** một sandbox
  script/expression engine để YAML tự mang code chạy được (gọi là "Option C", ghi nhận tồn tại về
  lý thuyết nhưng không làm) — vì đó là 1 hạng mục lớn riêng (sandbox execution, giới hạn tài
  nguyên, bảo mật). Hướng đã dùng ở đó: **named hook + registry Rust do binary tự đăng ký**
  (`metap_infra::HandlerRegistry.on("<entity>.record.created", ...)`) hoặc **webhook trỏ ra service
  ngoài platform**. Thiết kế ở brief này đi theo đúng tiền lệ đó — YAML chỉ mang *tên* hook, Rust
  code thật do binary đăng ký bằng tay trước khi boot, không có "code-as-data" nào lọt vào YAML.
- **`graphql-gateway` config hôm nay chỉ đọc env var, không có YAML/TOML loader nào** — comment
  trong `config.rs` nói rõ "no config-parsing dependency added". `waf-graphql-gateway`
  (`../metap-demo-waf/data-plane/graphql-gateway`) đã có tiền lệ **bọc** gateway generic bằng
  `build_with_extensions` để thêm 7 custom mutation viết tay bằng Rust — đúng mẫu "config (upstream
  list) tách khỏi code (mutation logic)" brief này muốn nhân rộng, không phải phát minh mới.

## Phạm vi (đề xuất — cần chốt trước khi code, xem câu hỏi mở dưới)

**Trong phạm vi (đề xuất):**
- Loader YAML cho `EntityDefinition`: đọc 1 thư mục `entities/*.yaml` (hoặc 1 file gộp, xem câu hỏi
  mở #3), deserialize thẳng bằng serde đã có, đăng ký vào `MetadataRegistry` — thay cho việc build
  từng `EntityDefinition` bằng tay trong `main.rs`. Không đổi `EntityDefinition` type, không đổi
  `MetadataRegistry` — chỉ thêm 1 hàm nạp (`metap-app`, cạnh `bootstrap_platform`).
- Loader YAML cho `graphql-gateway`'s danh sách upstream: 1 file `gateway.yaml` liệt kê upstream
  (name/grpc_addr/metadata_url/login_url/service_email/service_password) thay cho biến môi trường
  đánh số — **`build_with_extensions`'s custom mutation code (Rust) không đổi gì**, hai việc tách
  bạch.
- Cơ chế "custom handler" theo tên: YAML cho phép 1 field khai báo tên hook (vd
  `onBeforeCreate: "verifyDomainOwnership"`), binary tự đăng ký hàm Rust thật cho tên đó vào 1
  registry trước khi boot (mẫu `HandlerRegistry` đã có ở Phase 40) — thiếu tên → lỗi ngay lúc boot
  (fail-fast), không phải runtime 404 âm thầm.

**Ngoài phạm vi (rõ ràng không làm):**
- Không làm sandbox script/expression engine để YAML tự mang logic chạy được (đã bác ở
  `docs/features/06-...md`, giữ nguyên quyết định đó).
- Không đổi cách `metap-lowcode` định nghĩa entity qua API (khác tầng — DB-authored/runtime, không
  phải file YAML đọc lúc boot).
- Không bắt buộc migrate `../metap-demo-crm`/`../metap-demo-jira` sang YAML — vẫn giữ được cách khai
  báo Rust cũ song song (YAML là lựa chọn thêm, không phải thay thế bắt buộc).

## Câu hỏi mở — đã chốt (2026-09-07)

1. **Thứ tự làm**: chốt làm đồng thời cả hai (2 việc không đụng chung code, độc lập nhau).
2. **Custom handler áp cho những điểm nào**: chốt chỉ entity-event hook — dùng nguyên
   `metap_infra::HandlerRegistry.on("<entityName>.record.created", ...)` đã có, không thêm field
   "tên hook" nào vào schema YAML/`EntityDefinition`. Rà soát lúc code phát hiện: cơ chế này đã
   **miễn phí** với entity nạp từ YAML — tên entity (đã có sẵn trong file YAML) là đủ để binary tự
   đăng ký handler theo đúng pattern đó trong `main.rs`, không cần cơ chế nối dây mới nào giữa
   loader YAML và `HandlerRegistry`. Không mở rộng sang custom GraphQL resolver/mutation (giữ
   nguyên `waf-graphql-gateway`'s `build_with_extensions` làm ví dụ riêng, không đụng gateway
   generic).
3. **YAML nạp từ đâu**: chốt 1 thư mục quét lúc boot cho entity (`load_entity_definitions_from_dir`,
   không đệ quy, mỗi file 1 entity, thứ tự tên file để đăng ký deterministic), 1 file gộp cho
   gateway (`UPSTREAM_CONFIG_FILE`, 1 key `upstreams:` chứa danh sách). Không hot-reload — đọc 1
   lần lúc boot, đúng bất biến "`MetadataRegistry` read-only sau boot" đã có.

## Đã ship / Đã verify (2026-09-07, repo `metap`)

**Đã ship:**
- `metap-metadata::EntityField`/`EntityDefinition`: thêm `#[serde(default)]` cho 9 field
  `Option`/`Vec` trước đó chỉ có `skip_serializing_if` (thiếu `default` — khiến 1 field vắng mặt
  hoàn toàn, không chỉ `null`, là lỗi deserialize). Thay đổi thuần cộng thêm (permissive hơn), tìm
  ra khi viết YAML tối giản đầu tiên và phát hiện nó fail deserialize dù đúng shape.
- `metap-app::entities_yaml` (module mới) — `parse_entity_yaml`/`load_entity_definitions_from_dir`:
  nạp `Vec<EntityDefinition>` từ 1 thư mục `*.yaml`/`*.yml`, dùng đúng key `camelCase` của
  `EntityDefinition` (không phát minh convention mới). Không tự đăng ký vào `MetadataRegistry` —
  caller vẫn tự gọi `.register(entity)?` từng cái, giống hệt entity viết tay Rust hôm nay. Re-export
  qua `metap::app`/`metap::prelude::load_entity_definitions_from_dir`.
- `metap-graphql-gateway::config`: thêm `UPSTREAM_CONFIG_FILE` (YAML, key `upstreams:`) làm nguồn
  thay thế cho vòng lặp `UPSTREAM_<N>_*` — 2 nguồn loại trừ nhau, mặc định (không set biến này) giữ
  nguyên hành vi cũ 100% (`../metap-demo-waf` không cần đổi gì). `parse_upstreams_yaml` tách riêng,
  test được mà không đụng filesystem/env var.
- Dependency mới: `serde_norway` (fork còn bảo trì của `serde_yaml` đã archive, API tương thích) —
  thêm vào `[workspace.dependencies]` (dùng ở 2 crate).
- `.env.example`/`README.md` của `metap-graphql-gateway` cập nhật mô tả cả 2 nguồn cấu hình.

**Đã verify:** `cargo build/clippy --all-targets -D warnings/test --workspace` sạch toàn bộ
workspace `metap` (96 test suite, không suite nào fail), `cargo fmt --all --check` sạch. Unit test
mới: 3 test cho `entities_yaml` (parse camelCase tối giản, lỗi YAML sai định dạng có thông báo rõ,
nạp nhiều file theo thứ tự tên file), 3 test cho `parse_upstreams_yaml` (parse 2 upstream, danh
sách rỗng bị từ chối, YAML sai định dạng có thông báo rõ) — không cần Postgres/service thật, cả 6
test đều pure-logic.

**Còn nợ (ngoài phạm vi session này):** chưa có ví dụ thật dùng YAML loader trong
`../metap-demo-crm`/`../metap-demo-jira`/`../metap-demo-waf` (out of scope repo) — brief này chỉ
ship cơ chế ở `metap` core, chưa migrate binary nào sang dùng nó (đúng "Ngoài phạm vi" đã ghi —
không bắt buộc migrate).

## Ranh giới kiến trúc bị đụng tới

- `metap-app` (thêm loader, không đổi `bootstrap_platform` hiện có).
- `metap-graphql-gateway::config` (thêm nhánh đọc file, giữ nguyên nhánh env var để không phá
  `../metap-demo-waf` đang dùng).
- Không đụng `metap-metadata`/`MetadataRegistry` type, không đụng `metap-lowcode` (khác tầng).
- Cần ADR? Chưa rõ tới khi câu hỏi mở #2 (phạm vi custom handler) được chốt — nếu chỉ dừng ở
  entity-event hook (tiền lệ đã có từ Phase 40) thì không cần ADR mới; nếu mở rộng sang custom
  GraphQL resolver mức field thì nên có 1 ADR ngắn ghi lại ranh giới (giống cách
  `docs/features/15-...md` cần ADR cho `MetadataResolver`).

## Rủi ro / phụ thuộc

- Không phụ thuộc phase nào. Không phụ thuộc feature #12/#15 (đã merge, độc lập).
- Rủi ro chính: nếu câu hỏi mở #2 mở quá rộng (custom handler ở nhiều điểm khác nhau), dễ trôi dần
  thành đúng thứ Phase 06 đã cố tình tránh (1 engine tuỳ biến vô hạn). Giữ kỷ luật "YAML chỉ mang
  tên, Rust registry mới là code thật" xuyên suốt để không lặp lại rủi ro đó.
