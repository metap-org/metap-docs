# Feature Briefs

Nơi theo dõi tính năng ở mức nhỏ hơn một phase trong `docs/roadmap.md`. Ba tài liệu process hiện có
mỗi cái trả lời một câu hỏi khác nhau — thư mục này lấp đúng chỗ trống còn lại:

| Tài liệu | Trả lời câu hỏi |
|---|---|
| `docs/roadmap.md` | Đang ở phase lớn nào, phase đó xong chưa |
| `docs/architectures/09-adr/00-index.md` | Vì sao chọn giải pháp kiến trúc này (quyết định *kỹ thuật*) |
| `docs/features/*.md` (thư mục này) | Một tính năng cụ thể làm gì, phạm vi tới đâu, khi nào coi là xong (yêu cầu *sản phẩm*) |

Không phải việc nhỏ nào cũng cần một file ở đây — xem Definition of Ready trong
`docs/agile-process.md`: bugfix rõ ràng, sửa doc, refactor cục bộ thì không cần. File ở đây dành
cho tính năng đủ lớn để cần thống nhất phạm vi *trước khi* code, để tránh việc code xong rồi mới
tranh cãi nó có nên làm vậy không.

## Chỉ còn 1 thư mục — brief của demo app đã chuyển ra repo riêng

`docs/features/*.md` (ở đây) — feature/change-log cho **core metap**: thay đổi trong
`crates/metap-*` (execution engine) hoặc `@metap/platform-ui` (reusable frontend library, repo
riêng) — thứ một downstream project thật sự import và phụ thuộc vào.

**`docs/features/demo/` không còn nữa (2026-08-31)** — 3 brief của nó (`crm.customers`'s entity
đồng hành: `sales.orders`/`inventory.movements`/`accounting.journal`) đã chuyển sang
`../metap-demo-crm/docs/features/demo/` cùng lúc `apps/crm-server`/`apps/crm-fe` tách ra repo
riêng (xem `docs/roadmap/51-example-apps-repo-split.md`). Demo app giờ không còn nằm trong `metap`
nữa — brief của nó thuộc về repo của chính nó.

## Quy trình

1. Copy `TEMPLATE.md` thành `NN-<slug-tinh-nang>.md` — `NN` là số thứ tự 2 chữ số tăng dần
   trong đúng thư mục đó (cấp gốc và `demo/` đánh số độc lập nhau, giống cách
   `docs/architectures/` và `crates/migrations/` đã làm). Đặt vào cấp gốc hay `demo/` theo tiêu
   chí ở trên.
2. Điền các mục, đặt `Trạng thái: proposed`.
3. Khi được duyệt (ai duyệt: track sở hữu theo `docs/team-charter.md`, hoặc tự quyết nếu chỉ có
   một người), đổi `Trạng thái: approved` và thêm vào bảng bên dưới.
4. Khi bắt đầu code, đổi `Trạng thái: in-progress`. Nếu tính năng đủ lớn để gắn với một phase
   trong `docs/roadmap.md`, ghi rõ phase đó trong file.
5. Khi xong, đổi `Trạng thái: done` và để nguyên file lại — đây là lịch sử, không xoá.
6. Nếu quyết định không làm nữa, đổi `Trạng thái: rejected` kèm lý do ngắn, không xoá file.

## Danh sách theo trạng thái (2026-09-05)

Cùng 31 feature (18 gốc + 19-30 thêm 2026-09-05 + 31 cùng ngày), nhóm lại theo trạng thái để quét
nhanh — bảng gốc theo thứ tự số (01-17, thiếu hẳn 18 — đã bổ sung 2026-09-04) đổi thành các nhóm
dưới. Bảng gốc theo thứ tự số cũ đã bỏ, dùng link trực tiếp từng file nếu cần đọc theo `NN`. 19-28
và 30 đều đã **done** cùng ngày viết brief (rà soát checklist rồi audit 03 dẫn tới, xem từng file's
"Ghi chú" cho commit hash); 31 done-partial (phần A xong, phần B chờ quyết định) — chỉ 29 còn
`proposed` hoàn toàn, chờ quyết định phạm vi trước khi code (xem nhóm ⚪ bên dưới).

### ✅ Done

| Tính năng | Track | Phase liên quan | Ghi chú |
|---|---|---|---|
| [02. Metadata-driven Workflow Engine](02-workflow-engine.md) | Backend Core | Phase 17 | Increment 1+2+3 (2026-08-21 → 2026-08-28) |
| [04. Table-per-entity — readiness brief](04-table-per-entity.md) | Backend Core | Phase 21 | wire vào boot sequence thật từ Phase 21 (2026-08-23) |
| [07. Tách phần SaaS low-code control-plane ra khỏi core](07-split-lowcode-saas-crates.md) | Backend Core | Phase 52 | |
| [08. Crate dùng chung `metap-runtime`](08-metap-runtime-common-crate.md) | Backend Core | Phase 53 | |
| [10. Workflow visualize / BPM nhẹ](10-workflow-visualize.md) | Frontend Platform | chưa gắn phase | done 2026-09-02 |
| [13. Computed/derived field](13-computed-derived-field.md) | Backend Core | chưa gắn phase | done 2026-09-02 |
| [18. Config trong DB, phân tầng operator/platform-global/per-tenant](18-config-tiers-db-backed.md) | Backend Core | Phase 66/67/68 | done 2026-09-03, cả 3 lát — còn nợ verify sống (e2e + webhook thật) + FE chưa đọc `/public/config` |
| [19. `GeneratedForm` — dirty state, partial update, optimistic update](19-generated-form-mutation-ergonomics.md) | Frontend Platform | không thuộc phase nào | done 2026-09-05, `platform-ui` `078e967` — in-app nav blocker bỏ khỏi scope (không data router) |
| [20. `RecordDetail` — audit trail + `Tabs`](20-record-detail-audit-trail-and-tabs.md) | Frontend Platform | không thuộc phase nào | done 2026-09-05, `platform-ui` `078e967` |
| [21. Workflow guard — condition builder có cấu trúc](21-workflow-condition-builder.md) | Frontend Platform | không thuộc phase nào | done 2026-09-05, `platform-ui` `078e967` |
| [22. `GeneratedList` — Export CSV/JSON](22-generated-list-export.md) | Frontend Platform | không thuộc phase nào | done 2026-09-05, `platform-ui` `078e967` |
| [23. UX Infrastructure nền tảng](23-ux-infrastructure-core.md) | Frontend Platform | không thuộc phase nào | done 2026-09-05, `platform-ui` `078e967` |
| [24. Command palette](24-command-palette-and-state-persistence.md) | Frontend Platform | không thuộc phase nào | done 2026-09-05 v1 rút gọn (chỉ điều hướng nav), `design-system` `2c0c139` + `platform-ui` `078e967` — state persistence chưa làm |
| [25. Audit `metap-demo-waf` — component đặt đúng chỗ design-system/platform-ui chưa](25-audit-waf-demo-component-placement.md) | Frontend Platform | không thuộc phase nào | done 2026-09-05 — audit tại `../../platform-ui/docs/audits/03-waf-demo-component-placement-audit.md`, quét 23/23 file, 8 finding thật (5 nhóm 1, 2 nhóm 2, 1 nhóm 3), 0 finding nhóm 4; chưa sửa code |
| [26. Chuyển 5 primitive + 2 date-util sang `@metap/ui`/`@metap/platform-ui`](26-waf-primitives-to-design-system.md) | Frontend Platform | không thuộc phase nào | done 2026-09-05, `design-system` `d9f5f32` + `platform-ui` `ebca327` + `metap-demo-waf` `ba9533c` |
| [27. `useAsyncAction` hook](27-use-async-action-hook.md) | Frontend Platform | không thuộc phase nào | done 2026-09-05, phạm vi 16/18 chỗ (3 chỗ khác hình dạng cố tình bỏ qua) — `platform-ui` `ebca327` + `metap-demo-waf` `ba9533c` |
| [28. `WorkflowVisualizeDialog`](28-workflow-visualize-dialog.md) | Frontend Platform | không thuộc phase nào | done 2026-09-05, `platform-ui` `ebca327` + `metap-demo-waf` `ba9533c` |

### 🟡 Done-partial

| Tính năng | Track | Phase liên quan | Ghi chú |
|---|---|---|---|
| [01. Nâng cấp Frontend Platform](01-fe-platform-overhaul.md) | Frontend Platform | chưa gắn | chỉ 1/4 gap đã xong |
| [03. Organization & Identity Layer](03-organization-identity.md) | Backend Core | Phase 18 | P0 + P1 done (P1: 2026-09-02); P2 chưa có trigger |
| [05. Cross-entity relations trong list view (3 mode)](05-cross-entity-relations.md) | Backend Core | không thuộc phase nào | chỉ Mode 2/3 done |
| [31. Session hết hạn — tự văng ra `/login` + tự refresh](31-session-expiry-redirect-and-refresh.md) | Frontend Platform (phần B kéo Backend Core) | không thuộc phase nào | phần A (điều hướng `/login` ngay khi 401) done 2026-09-05, `platform-ui` chưa commit — phần B (tự gia hạn phiên) còn `proposed`, chờ chọn hướng |

### ⚪ Pending (proposed, chưa có trigger)

| Tính năng | Track | Ghi chú |
|---|---|---|
| [06. Pattern xác minh bất đồng bộ + gap logic tùy biến cho low-code](06-async-verification-pattern-and-lowcode-custom-logic.md) | Backend Core | ghi chú thảo luận, chỉ Option B là đề xuất code — cần chọn option trước |
| [09. Workflow hai chế độ (in-process + cross-module)](09-workflow-two-modes.md) | Backend Core | 🔴 xem `docs/roadmap/00-index.md`'s mục "Cần quyết định kiến trúc quan trọng" |
| [11. Tiny deployment profile](11-tiny-deployment-profile.md) | Backend Ops-Infra | 🔴 quyết định sản phẩm — xem `docs/roadmap/00-index.md` |
| [12. Migration path generic → bảng riêng](12-migration-generic-to-dedicated-table.md) | Backend Core | 🔴 xem `docs/roadmap/00-index.md` |
| [14. Schema versioning cho entity](14-entity-schema-versioning.md) | Backend Core | 🔴 xem `docs/roadmap/00-index.md` |
| [15. Metadata low-code theo Tenant](15-tenant-scoped-lowcode-metadata.md) | Backend Core | 🔴 xem `docs/roadmap/00-index.md` |
| [16. Entity variant polymorphic/discriminated-union](16-entity-variant-polymorphic.md) | Backend Core | 🔴 **rủi ro cao nhất trong cả backlog** — xem `docs/roadmap/00-index.md` |
| [29. `FieldValue`'s enum rendering — tone theo giá trị](29-field-value-enum-tone-mapping.md) | Frontend Platform (có thể kéo Backend Core) | audit 03 finding #8 — cần chọn hướng metadata-driven (đụng `metap-metadata`) hay app-local prop trước khi code |

6/7 dòng trên (09/11/12/14/15/16) là đúng 6 ý trong `docs/roadmap/00-index.md`'s bảng "Cần quyết
định kiến trúc quan trọng" — 2 file này mô tả cùng 1 tập quyết định chưa chốt, chỉ khác góc nhìn
(feature brief chi tiết vs. roadmap-level liệt kê nhanh). Riêng 06 không nằm trong danh sách 7 ý
gốc của `docs/roadmap.md` — là 1 đề xuất tách biệt.

### 🔵 Vision (chưa phải spec sẵn sàng)

| Tính năng | Track | Ghi chú |
|---|---|---|
| [17. Tầm nhìn dài hạn: Durable Workflow Runtime](17-durable-workflow-runtime-vision.md) | Backend Core | vision, không phải spec — không code cho tới khi có brief thật |

Brief của demo app (Sales Order/Inventory Movement/Journal Entry, từng nằm ở `demo/`) — xem
`../metap-demo-crm/docs/features/demo/`.
