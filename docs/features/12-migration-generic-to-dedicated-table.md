# Migration path: generic `records` table → bảng riêng cho 1 entity

- **Trạng thái:** in-progress — cơ chế cốt lõi + CLI đã code và verify sống trong `metap` 2026-09-06,
  còn nợ 1 lần chạy thật trên `../metap-demo-crm` (xem "Đã ship / Đã verify" ở cuối file)
- **Người đề xuất:** ghi lại từ thảo luận kiến trúc, `docs/team-charter.md` ("Định hướng đang ghi
  nhận, chưa có trigger" #4); **kích hoạt 2026-09-06** — chủ dự án chốt chủ động đóng khoảng hở
  hiệu năng này ngay, không chờ 1 entity thật đo được nghẽn ("không được để lỗ hổng hiệu năng ở
  đây") — xem ADR cập nhật, `docs/architectures/09-adr/00-index.md`.
- **Track sở hữu:** Backend Core
- **Phase roadmap liên quan:** liên quan `docs/features/04-table-per-entity.md` (đã `done`, wire
  vào Phase 21) nhưng đây là ý khác — di chuyển 1 entity ĐANG SỐNG trên `records` sang bảng riêng
  mà không mất dữ liệu, không phải tạo entity mới trực tiếp trên bảng riêng.

## Vấn đề / động lực

`docs/features/04-table-per-entity.md` đã chứng minh table-per-entity chạy được thật
(`metap-demo-jira`) cho entity **mới tạo trực tiếp** trên bảng riêng. Câu hỏi khác: 1 entity đã có
dữ liệu thật trên bảng `records` chung (JSONB), muốn chuyển sang bảng riêng — quy trình migrate dữ
liệu đang sống là gì?

**Quyết định cơ chế (2026-09-06, chủ dự án chốt trực tiếp):** **downtime chấp nhận được** — tắt
hẳn service trong lúc migrate, không cần zero-downtime dual-write/shadow-read. Lý do: metap còn ở
giai đoạn dev/thiết kế, chưa có dữ liệu sản xuất thật ("metap mới phase dev") — độ phức tạp thiết
kế/rủi ro bug của dual-write (2 nguồn ghi phải luôn nhất quán, rollback giữa chừng khó) không đáng
so với cái giá phải trả chỉ để né vài giây/vài phút downtime không ai đang phụ thuộc vào lúc này.
**Quyết định này đảo lại khi metap có khách hàng/dữ liệu sản xuất thật** — lúc đó phải quay lại
thiết kế zero-downtime.

## Phạm vi

**Trong phạm vi:**
1. Tool/lệnh migrate 1 entity (khả năng sẽ là `dev-tools migrate-to-dedicated-table <entity>` hay
   tương tự, chưa chốt tên) — cho entity bất kỳ đang sống trên `records`, không giới hạn
   accounting/inventory như định hướng cũ trong `docs/architectures/05-building-blocks/04-data-model.md`.
2. Quy trình 1 lần (không phải reconciler chạy liên tục):
   a. `metap-reconciler::reconcile()` tạo bảng đích (mechanism đã có, không đổi — `compile()` +
      `introspect()` + `diff()` + `executor::execute`).
   b. Batch-copy: đọc toàn bộ record JSONB hiện có của entity đó trên `records`, ghi sang bảng
      mới — cần viết mới (không tái dùng thẳng `backfill::run_batched_update`, hàm đó phục vụ
      "sửa cột trong cùng 1 bảng", không phải "copy bảng nguồn khác sang bảng đích khác" — nhưng
      giữ cùng nguyên tắc checkpointed/tự lành sau crash).
   c. Đổi `EntityDefinition.table_name` từ `"records"` sang bảng đích (`qualified_table_name_for`).
   d. Khởi động lại service.
3. Không cần feature flag/dual-write — service tắt trong bước b/c nên không có request nào đọc/ghi
   nhầm bảng giữa lúc migrate.

**Ngoài phạm vi:**
- Không phải bản thân table-per-entity mechanism (`metap-reconciler`, đã xong) — đây là *migration
  path* cho dữ liệu đã tồn tại, dùng lại mechanism đó làm đích đến.
- Zero-downtime migration — ngoài phạm vi cho tới khi có dữ liệu sản xuất thật (xem "Quyết định cơ
  chế" trên).

## Tiêu chí chấp nhận

- Migrate 1 entity thật (test trên `metap-demo-crm`, ví dụ `crm.customers`) từ `records` sang bảng
  riêng, không mất record nào, `version` (optimistic locking) giữ nguyên giá trị sau migrate.
- Service dừng trong lúc migrate, khởi động lại phục vụ đúng từ bảng mới — verify bằng request
  thật trước/sau (list/get/create/update/delete/transition đều hoạt động đúng).
- `workflow_events`/outbox không bị gián đoạn theo entity đó — record tạo/sửa *trước* lúc dừng
  service verify vẫn còn nguyên vẹn *sau* migrate.

## Ranh giới kiến trúc bị đụng tới

- `metap-reconciler` (đích đến, đã có sẵn — `compile`/`introspect`/`diff`/`executor`).
- Batch-copy loop mới (chưa có sẵn — checkpointed, tự lành nếu crash giữa chừng, không cần
  rollback vì chạy lúc service đã tắt, không có ghi đồng thời).
- `EntityDefinition.table_name`: đổi trong code (recompile+redeploy), không phải data-driven —
  giống cách `metap-demo-jira`'s entity code-authored set `table_name` hôm nay.
- Có thể cần thêm 1 binary/subcommand mới trong `dev-tools` hoặc 1 crate ops riêng (giống
  `db-migrate`) — chưa chốt.

## Rủi ro / phụ thuộc

- Rủi ro mất dữ liệu nếu batch-copy sai (bỏ sót record, sai kiểu dữ liệu lúc convert JSONB path
  sang cột typed) — cần test kỹ trên dữ liệu thật trước khi chạy trên môi trường có ý nghĩa.
- Phụ thuộc `docs/features/04-table-per-entity.md` (đích đến migration) — đã sẵn sàng.
- Quyết định downtime-acceptable là tạm thời theo giai đoạn dự án — phải revisit khi metap tiến
  gần production thật (xem ADR).

## Đã ship / Đã verify (2026-09-06, `metap` repo)

**Code:**
- `metap-reconciler::migrate` (module mới, `crates/metap-reconciler/src/migrate.rs`) —
  `copy_generic_records` (bước b: batch-copy có checkpoint, keyset `id > cursor`, tái dùng đúng 3
  hàm checkpoint của `backfill.rs` — `load_cursor`/`save_progress`/`mark_completed` trên
  `reconciler_backfill_progress`, dưới `op_id` riêng `MIGRATE_OP_ID` để không đụng `op_id` của một
  `BackfillColumn` thật) và `migrate_generic_to_dedicated` (gộp bước a + b: gọi `reconcile()`
  nguyên bản để tạo/cập nhật bảng đích, rồi copy dữ liệu sang). Không tái dùng thẳng
  `backfill::run_batched_update` đúng như brief yêu cầu (nó `UPDATE` tại chỗ trên 1 bảng, không
  `INSERT` chéo bảng) — nhưng tái dùng nguyên tắc checkpoint/tự lành và cả code checkpoint thật.
  Mỗi batch là **1 câu SQL** (`WITH batch AS (SELECT ...), ins AS (INSERT ... RETURNING id) SELECT
  b.id FROM batch b LEFT JOIN ins i ON i.id = b.id`) — cursor luôn tiến theo `batch` (những gì đã
  *đọc* được), không phụ thuộc `ins` ghi được bao nhiêu dòng, nên một lần resume re-fetch đúng
  batch đã áp dụng một phần vẫn tiến cursor thay vì lặp vô hạn.
- `dev-tools migrate-to-dedicated-table <tenantId> <entityJsonPath> [sourceTable]`
  (`crates/metap-dev-tools/src/main.rs`) — wrapper CLI mỏng. `entityJsonPath` là file JSON đúng
  hình dạng `GET /metadata/entities/{entity}` trả về (cùng cách `metap-graphql-gateway`'s
  `schema_builder.rs` đã parse từ lâu) — lấy **trước** khi dừng service, vì lệnh này tự nó không
  gọi service đang chạy, chỉ chạm DB. Chưa chốt tên lệnh trong brief gốc ("chưa chốt") — tên này là
  quyết định thật của lần code này.
- Không đổi `EntityDefinition.table_name`/restart tự động — đúng phạm vi brief (bước c/d là việc
  của downstream binary, không phải của `metap-reconciler`/`dev-tools`).

**Verify (real, không phải giả định):**
- `cargo build --workspace` + `cargo clippy --workspace --all-targets -- -D warnings` +
  `cargo test --workspace` (unit) sạch trên toàn bộ 30 crate, không riêng phần đụng tới.
- 2 test e2e mới (`crates/metap-reconciler/tests/migrate_postgres.rs`, `--ignored`, chạy thật
  trên Postgres local, không mock): migrate 3 dòng (kèm 1 dòng `deleted=true`, `version` khác 1)
  từ `records` sang bảng riêng — không mất dòng nào, `version` giữ nguyên, cột `storage: column`
  được đồng bộ đúng qua trigger của `reconcile()` (không phải do copy ghi trực tiếp); resume sau
  "crash" giả lập (seed sẵn checkpoint) chỉ quét đúng phần còn lại, không quét lại/không trùng.
  Toàn bộ 20 test e2e khác đã có sẵn của `metap-reconciler` (`migration`/`orchestrator`/`reconcile`)
  vẫn xanh sau khi đổi 3 hàm `backfill.rs` từ private sang `pub(crate)`.
- Chạy CLI **thật** (không phải chỉ test): `dev-tools migrate-to-dedicated-table` với entity JSON
  giả (`test.cli_customers`, 1 field `storage: column`) và 2 dòng seed trực tiếp vào `records` trên
  Postgres local — output đúng, `psql` xác nhận `id`/`code`/`version` giữ nguyên và cột promote
  (`balance`) đúng giá trị. Phát hiện và sửa 1 bug thật trong lúc verify: `router_for`'s
  `shared_pool` dùng `max_connections(1)` (đúng cho mọi subcommand khác) nhưng
  `metap_reconciler::reconcile()` giữ 1 connection cho advisory lock suốt thời gian chạy DDL trong
  khi vẫn cần connection khác cho từng op — treo `pool timed out while waiting for an open
  connection` với tenant chưa đăng ký (`Schema` strategy, `pool_for` trả thẳng `shared_pool`). Sửa
  bằng cách nâng `max_connections` lên 5 riêng cho subcommand này.

**Chưa verify (nợ lại, ngoài phạm vi session này):**
- Chưa chạy trên `../metap-demo-crm` với `crm.customers` thật như tiêu chí chấp nhận gốc yêu cầu —
  session code phần này không có quyền truy cập repo đó. Cơ chế đã verify tương đương (entity
  tổng hợp, DB thật, cả đường copy lẫn đường resume) nhưng chưa phải đúng entity/repo nêu trong
  tiêu chí chấp nhận — người có quyền truy cập `../metap-demo-crm` cần chạy thật 1 lần trước khi
  đóng brief này thành `done`.
- Chưa verify workflow_events/outbox "không gián đoạn" theo đúng nghĩa tiêu chí (record tạo trước
  lúc dừng service vẫn còn nguyên sau migrate) trong bối cảnh có outbox/RabbitMQ thật chạy song
  song — batch-copy chỉ đụng `records`/bảng đích, không đụng `outbox_events`/`workflow_events`, nên
  về lý thuyết không ảnh hưởng, nhưng chưa có test riêng khẳng định điều đó.
