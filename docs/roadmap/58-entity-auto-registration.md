## Phase 58: `submit_entity!`/`register_all_submitted()` — entity tự đăng ký qua `inventory`, xoá bước liệt kê 2 lần (2026-09-01)

Trigger: rà soát định kỳ phát hiện gap tài liệu — `submit_entity!`/`register_all_submitted()` đã
code xong và commit ở cả 4 repo (`metap` + 3 downstream: `metap-demo-crm`/`metap-demo-jira`/
`metap-demo-waf`) nhưng chưa có entry roadmap tương ứng. Entry này ghi lại lịch sử của công việc
đã có sẵn — không đổi code trong phiên viết doc.

Vấn đề gốc: một binary downstream trước đây phải liệt kê mỗi entity 2 lần — 1 lần khai báo `mod`
và 1 lần gọi tay `registry.register(x_entity())?` trong `main.rs` — dễ quên khi thêm entity mới
(quên bước 2 im lặng bỏ sót entity, không có lỗi biên dịch nào báo).

## Thiết kế (`metap`, `crates/metap-metadata/src/registry.rs`)

- `EntityFactory(pub fn() -> EntityDefinition)` + `inventory::collect!(EntityFactory)` — 1 factory
  entity nộp qua `submit_entity!` được thu thập lúc link-time.
- `submit_entity!($factory:path)` (macro_rules, `#[macro_export]`) — gọi 1 lần ngay cạnh hàm định
  nghĩa entity (ví dụ `fn zone_entity() -> EntityDefinition`), nộp vào `inventory` qua constructor
  chạy trước `main()` — không cần init call tường minh, chỉ cần `mod` khai báo module đó ở đâu đó
  binary có link tới.
- `MetadataRegistry::register_all_submitted(&mut self)` — duyệt `inventory::iter::<EntityFactory>`,
  gọi `register()` cho từng cái. Thứ tự duyệt là link order (không phải declaration order); 2
  entity trùng tên vẫn fail giống hệt cách `register()` tự gọi tay đã fail, chỉ khác chỗ lộ ra lỗi.

Wired vào `templates/metap-app` làm pattern tham chiếu cho project downstream mới.

## Áp dụng ở 3 repo downstream (cùng ngày 2026-09-01)

- **`metap-demo-crm`** (`1d59f37`): 4 entity (`customer`/`sales_order`/`inventory_movement`/
  `journal_entry`) tự đăng ký qua `submit_entity!`; `main.rs`'s 4 lời gọi
  `registry.register(...)` gộp thành 1 `register_all_submitted()`.
- **`metap-demo-jira`** (`b544ff9`): 8 entity, cùng pattern. **Giữ nguyên có chủ đích**: thứ tự
  `reconcile()` ngay dưới đó (project → sprint → epic → ... theo FK-dependency) không đổi —
  `registry.register()` tự nó không quan tâm thứ tự (`HashMap` insert), nhưng `reconcile()` thì
  có, và thứ tự duyệt `inventory` (link order) không phải thứ đáng tin cậy để giữ đúng FK-order —
  nên 2 việc tách bạch, không gộp chung dù trông giống nhau.
- **`metap-demo-waf`/`data-plane`** (`a1c30e9`): 9 entity, cùng pattern, cộng 1 regression test
  assert đủ 9 entity được pickup. **Bundle kèm 1 fix không liên quan phát hiện cùng lúc**:
  `data-plane` chưa từng gọi `tracing_subscriber` init — mọi `tracing::info!` (kể cả access-log
  span của `build_router`) bị âm thầm drop, chỉ dòng `eprintln!` boot-time hiện ra console. Sửa
  bằng gọi `metap::infra::init_tracing()` đầu `main()`, đổi hết `eprintln!` còn lại sang
  `tracing::info!`.

## `metap-lowcode` — cố ý không áp dụng, không phải gap

Rà soát khi viết entry này: `metap-lowcode` không gọi `register_all_submitted()` ở đâu cả. Không
phải thiếu sót — `metap-lowcode` không sở hữu entity code-authored nào theo kiểu `entities/*.rs`
như 3 repo trên. `crates/metap-lowcode/src/store.rs`'s `merge_with` **nhận** một
`base_registry: &MetadataRegistry` đã dựng sẵn từ binary host (binary đó đã tự gọi
`register_all_submitted()` cho entity code-authored của chính nó) rồi merge thêm entity
DB-authored (từ `publish`) vào — `metap-lowcode` là bên tiêu thụ 1 registry có sẵn, không phải
bên khai báo hàm entity nào để nộp qua `submit_entity!`. Không cần đổi gì ở đây.

## Xác minh

Không đổi code trong phiên viết doc này — chỉ ghi lại lịch sử 4 commit đã có sẵn trước khi viết
entry này (`metap` `2c8abac`, `metap-demo-crm` `1d59f37`, `metap-demo-jira` `b544ff9`,
`metap-demo-waf` `a1c30e9`, tất cả 2026-09-01). Build/test cho các thay đổi code đó đã chạy ở
phiên làm việc trước lúc code — không re-run trong phiên viết doc này vì không có code nào đổi.
