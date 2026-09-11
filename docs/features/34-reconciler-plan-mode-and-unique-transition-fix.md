# `metap-reconciler`: root-cause "transition-state" gap + `plan()` generate-SQL mode

- **Trạng thái:** done (2026-09-11) — `metap` commit chứa `crates/metap-reconciler/src/{diff,plan,executor,lib}.rs`,
  `crates/metap-reconciler/{src/diff/tests.rs,tests/reconcile_postgres.rs}`,
  `crates/metap-dev-tools/src/main.rs` (`reconcile-plan` subcommand).
- **Người đề xuất:** chủ dự án, tiếp nối 2 nợ kỹ thuật còn treo từ Phase 82
  (`../roadmap/82-record-referenced-ux-and-metadata-control-schema-split.md`'s "Còn lại") — nợ
  thứ 3 ("entity low-code CRM vẫn ở `entities` chung") không còn cấp thiết sau khi `metap-demo-crm`
  bị đánh dấu deprecated cùng ngày (xem `../CLAUDE.md` root, không phải file này).
- **Track sở hữu:** Backend Core
- **Phase roadmap liên quan:** không thuộc phase nào (nợ kỹ thuật độc lập, không gắn 1 phase sản phẩm)

## Vấn đề / động lực

Hai việc riêng nhưng dùng chung 1 primitive, nên gộp 1 brief:

1. **Bug thật, tái hiện 2 lần độc lập** (WAF's `waf.ddos_policies.zoneId` — Phase 80; Jira's
   `jira.projects.key` — Phase 82): 1 field `unique: true` đang ở dạng `UNIQUE` constraint chặn
   (blanket, kiểu cũ trước Phase 79) không bao giờ tự chuyển sang partial unique index (kiểu mới,
   `WHERE deleted = false`) qua 1 lần `reconcile()` bình thường — `ops_applied` không hội tụ về 0,
   cả 2 lần phải workaround thủ công (`DROP INDEX` tay ngoài reconciler). Root cause chưa từng
   tìm ra, chỉ ghi lại là gap mở ở cả 2 phase.
2. **Yêu cầu sản phẩm riêng**: `metap-reconciler`'s bước áp DDL hiện chỉ có 1 chế độ — tính SQL rồi
   chạy thẳng luôn. Chủ dự án muốn thêm chế độ thứ 2: tính SQL, in ra cho dev tự xem rồi tự chạy —
   giống cách các migration tool hiện đại tách `generate`/`apply` (Prisma `migrate dev` vs
   `migrate diff --script`, Drizzle `generate`/`migrate`).

Đọc code tìm ra root cause thật của (1): `diff()`'s vòng lặp "Step 3: indexes" (rebuild khi tên đã
tồn tại nhưng định nghĩa khác) luôn dùng `DropIndexConcurrently`, kể cả khi tên đó đang backing 1
`UNIQUE` constraint sống — vòng lặp orphan-cleanup ngay bên dưới (`diff.rs:162-174`) *đã* có guard
đúng cho đúng tình huống này (comment giải thích rõ: Postgres từ chối `DROP INDEX` trực tiếp lên
index đang backing constraint), nhưng guard đó chỉ áp cho nhánh "không còn desired nữa", không áp
cho nhánh "rebuild vì định nghĩa khác" — đúng nhánh case bug này rơi vào. `topo_sort`'s thứ tự cũ
(`DropIndexConcurrently`/`CreateIndexConcurrently` cùng rank 5, `DropUnique` rank 6) khiến
`DROP INDEX` chạy trước khi constraint bị gỡ — thất bại hoặc no-op, rồi `CREATE INDEX ... IF NOT
EXISTS` sau đó thấy tên vẫn bị chiếm nên im lặng bỏ qua — khớp chính xác triệu chứng đã ghi.

## Phạm vi

**Trong phạm vi:**
- Fix root cause: đổi `rank()` trong `crates/metap-reconciler/src/diff.rs`'s `topo_sort` — mọi
  op *drop* trong nhóm FK/unique/index chạy trước mọi op *create/add* trong nhóm đó (không còn
  ghép cùng rank theo cặp create/drop như cũ). `DropUnique` (rank 6) giờ luôn chạy trước
  `DropIndexConcurrently`/`CreateIndexConcurrently` (rank 7/8) — tự động đúng thứ tự cho mọi
  transition cùng tên, không chỉ ca này.
- `metap-reconciler::plan(desired, actual, renames) -> Vec<PlannedOp>` (file mới `src/plan.rs`) —
  thuần Rust, không mở connection nào, tái dùng `diff()` + `executor::build_sql` (đổi visibility
  `pub(crate)`, không đổi logic). `reconcile()`/`executor::execute()` giữ nguyên hoàn toàn — `plan()`
  là hàm cộng thêm, gọi `diff()` độc lập một lần nữa, không chèn vào đường auto-apply hiện có.
- `dev-tools reconcile-plan <tenantId> <entityJsonPath> <tableName>` — subcommand mới, cùng kiểu
  input `<entityJsonPath>` với `migrate-to-dedicated-table` đã có (file JSON = nguyên văn
  `GET /metadata/entities/<entity>` trả về). In SQL ra stdout, không thực thi gì. Auto-reconcile
  lúc boot của mọi service downstream (`crm-server`/`jira-server`/WAF's 3 service/`metap-lowcode`'s
  orchestrator) không đổi gì — đây là bề mặt thêm vào, opt-in.
- Test hồi quy mới, chạy được cả 2 dạng:
  - Unit, không cần Postgres: `diff/tests.rs`'s
    `unique_constraint_to_partial_index_same_name_drops_constraint_before_rebuilding_index` —
    dựng tay `actual`/`desired` đúng hình dạng lịch sử, assert thứ tự op.
  - E2e, Postgres thật: `tests/reconcile_postgres.rs`'s
    `unique_field_converges_from_a_legacy_blanket_constraint_to_a_partial_index_in_one_pass` —
    reconcile thật, tự tay revert về hình dạng constraint cũ (cùng tên tự động lấy từ
    `pg_indexes`, không hardcode naming scheme), reconcile lại, assert hội tụ trong đúng 1 pass
    sửa lỗi + xác nhận index thật sự là partial (`pg_index.indpred IS NOT NULL`), không chỉ
    `ops_applied == 0`.

**Ngoài phạm vi:**
- Nợ kỹ thuật #2 (per-tenant schema isolation thật) — vẫn chưa có trigger, cần quyết định sản
  phẩm riêng, không đụng trong brief này.
- Đổi `reconcile()`/boot-time auto-apply thành mặc định generate-only — không phải yêu cầu, mọi
  service vẫn tự apply như cũ; `plan()`/`reconcile-plan` chỉ là lựa chọn thêm.
- Sửa README/CLAUDE.md của `metap-demo-crm`/`metap-demo-jira` (không nằm trong scope truy cập của
  phiên thực hiện việc này).

## Tiêu chí chấp nhận

- `cargo test -p metap-reconciler` (unit, không cần DB): tất cả pass, kể cả 3 test đã có từ trước
  liên quan tới thứ tự op (`topo_order_create_table_before_columns_before_indexes_before_fks`,
  `invalid_index_is_dropped_then_recreated_drop_strictly_before_create`,
  `orphan_index_and_fk_and_unique_are_dropped`) — không sửa 1 dòng nào trong 3 test đó, xác nhận
  rank mới không phá behavior cũ.
- `cargo test -p metap-reconciler -- --ignored` (e2e, Postgres thật): tất cả pass, gồm cả test mới
  ở trên. **Test mới xác nhận fail trên code cũ** (verify tay: revert tạm rank về bản gốc, chạy
  lại — fail đúng ở assert `ops_applied == 0`, `left: 1, right: 0` — rồi khôi phục fix, pass lại) —
  không chỉ "pass sau khi sửa", mà chứng minh được test thật sự bắt được bug.
- `cargo build --workspace` + `cargo clippy --workspace --all-targets -- -D warnings` sạch toàn
  workspace `metap` (không chỉ `metap-reconciler`/`metap-dev-tools`).
- `dev-tools reconcile-plan` chạy tay thật: in đúng SQL, xác nhận **không** có side effect nào lên
  DB (`\dt`/`\d` trước/sau giống hệt nhau) — verify sống, không chỉ đọc code.

## Ranh giới kiến trúc bị đụng tới

`crates/metap-reconciler` (thay đổi thứ tự thực thi trong `diff.rs`, thêm module `plan.rs`, đổi 1
hàm private trong `executor.rs` thành `pub(crate)` — không đổi public API của `reconcile()`/
`execute()`) và `crates/metap-dev-tools` (subcommand mới, không đụng subcommand cũ nào). Không cần
ADR — không đổi mô hình dữ liệu, không đổi hợp đồng public của `reconcile()`, chỉ sửa đúng 1 bug
thứ tự thực thi nội bộ và thêm 1 capability hoàn toàn cộng thêm.

## Rủi ro / phụ thuộc

- Rủi ro chính đã kiểm chứng bằng test, không chỉ suy luận: thứ tự rank mới **strictly** general
  hơn thứ tự cũ (mọi drop luôn trước mọi create trong cùng nhóm), nên không có tổ hợp op nào mà
  thứ tự cũ đúng còn thứ tự mới sai — không tìm thấy phản ví dụ khi rà lại toàn bộ test cũ.
- Không phụ thuộc feature khác. Không cần chủ dự án tự chạy lại gì thêm — mọi service downstream
  tự động được fix khi bump lên commit `metap` này (đường auto-apply không đổi hành vi, chỉ đúng
  hơn cho đúng 1 tình huống hiếm — chuyển constraint→index cùng tên).
