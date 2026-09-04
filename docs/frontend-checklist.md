# Frontend Build Checklist

Checklist chi tiết ở mức UI/UX cho `@metap/platform-ui` (repo `../platform-ui`) + `@metap/ui`
(repo `../design-system`), mịn hơn cấp độ phase của `docs/roadmap.md`. Rà soát lần gần nhất:
2026-09-04 (lần trước 2026-08-20, còn nhắc path cũ `packages/platform-react`/`apps/crm-fe` — cả
hai đã dời hẳn ra khỏi repo `metap` từ 2026-08-28/31, xem `metap`'s `CLAUDE.md` mục "No example
apps in this repo"/"Frontend library moved out of this repo"). Không có consumer app nào tên
`metap-demo-crm`/`metap-demo-jira` checkout cục bộ lúc rà soát lần này — mọi mục dưới đây kiểm
tra ở tầng **thư viện UI dùng lại được** (`platform-ui`), không phải cách 1 app cụ thể dùng nó.
Mục đã `[x]` có chú thích file/dòng liên quan để trace ngược; mục `[ ]` là gap thật, chưa có gì
trong sản phẩm — không phải lúc nào cũng là việc cần làm ngay, xem "Chưa lên kế hoạch" ở cuối file.

Đừng nhầm các nhóm bên dưới (1-6) với "Phase N" của `docs/roadmap.md` — hai hệ đánh số độc lập.
Tầm nhìn kiến trúc dài hạn hơn checklist này (Jira/Confluence-style app trên metadata, durable
workflow runtime kiểu Temporal/Cadence) không nằm ở đây — xem `docs/team-charter.md`'s "Định
hướng đang ghi nhận, chưa có trigger".

---

## 1. Dynamic Table

- [x] Dynamic columns — `GeneratedList.tsx:261-282`, đọc từ `EntityListView`
- [x] Column metadata — từ `EntityListView`
- [x] Pagination — cursor-based (`useApiInfiniteQuery`)
- [x] Cursor pagination — keyset, opaque cursor (`GeneratedList.tsx:131-134`)
- [x] Sorting — `toggleSort` (`GeneratedList.tsx:179-195`), giới hạn theo metadata (`QueryPlanner`)
- [x] Filtering — per-field `Input`/`Select` ở filter row (`GeneratedList.tsx:290-330`), giới
      hạn theo metadata
- [x] Search — per-field ILIKE mặc định/FTS opt-in qua `searchMode`
      (`LowCodeEntitiesAdminPage.tsx:80-235` cấu hình, không phải 1 ô search toàn cục)
- [ ] Column visibility — không có toggle/menu nào, `listView.fields` render không điều kiện.
      `@metap/ui`'s `Table` cũng chỉ là wrapper `<table>` thuần, chưa có primitive cho việc này.
- [ ] Column ordering — thứ tự cột hardcode theo mảng `listView.fields`, không có drag/reorder
- [ ] Column resizing — `table-fixed`, độ rộng suy ra từ header, không có resize handle/state
- [ ] Row selection — không có cột checkbox; `@metap/ui` có sẵn atom `Checkbox` nhưng chưa dùng
      cho mục đích này
- [ ] Bulk actions — phụ thuộc row selection, chưa có
- [x] Row actions — cột action View/Delete theo dòng (`GeneratedList.tsx:387-406`)
- [ ] Export — không có nút export/CSV/JSON download nào trong `platform-ui/src`
- [x] Refresh — **2026-09-04**: `IconButton` cạnh nút "New" gọi `refetch()`
      (`GeneratedList.tsx:230-251`), disable + xoay icon khi `isFetching`
- [ ] Saved views — chưa persist filter/sort/column state ở đâu cả
- [x] Virtualization — `@tanstack/react-virtual`, `useVirtualizer` (`GeneratedList.tsx:144-161`)

---

## 2. Generic CRUD

Frontend đã hỗ trợ generic CRUD cho entity Metap bất kỳ qua
`GeneratedList`/`GeneratedForm`/`RecordDetail` (`platform-ui/src/list|form|detail/`).

### List

- [x] Entity list page
- [x] Dynamic columns
- [x] Search
- [x] Filter
- [x] Sort
- [x] Pagination
- [ ] Bulk selection — xem mục 1
- [ ] Bulk action
- [ ] Export
- [ ] Saved view

### Create

- [x] Dynamic create form — `GeneratedForm.tsx`
- [x] Metadata-driven validation — field-metadata validator, không phải Zod schema riêng
- [ ] Default values — `formData` khởi tạo `{}` lúc create (`GeneratedForm.tsx:47-51`); chỉ có
      giá trị khi edit từ `existing.data`. Chưa có khái niệm default value per-field trong
      `metadata/types.ts`.
- [ ] Conditional fields — mọi field non-`id` render không điều kiện
      (`GeneratedForm.tsx:133-145`); `metadata/types.ts` chưa có `visibleWhen`/`dependsOn`.
- [x] Submit
- [ ] Submit & create another — chỉ có 1 đường `handleSubmit` → 1 callback `onSaved`
      (`GeneratedForm.tsx:82-116, 146-148`)

### Detail

- [x] Dynamic detail page — `RecordDetail.tsx`
- [ ] Field groups — field render flat trong grid 2 cột `<dl>` (`RecordDetail.tsx:213-234`);
      `entityLayout.ts` chỉ có hint `span: 1|2` per-field, không có named group/section
- [ ] Tabs — `RecordDetail.tsx` không dùng `Tabs`; `@metap/ui` có atom `Tabs` (Radix-based) nhưng
      mới chỉ dùng ở `admin/PoliciesAdminPage.tsx`, chưa ở Detail
- [x] Related entities — **`detail/RelatedRecordsPanel.tsx`**: build query gộp theo từng
      `entity.relatedViews`, render 1 card/related view (`RelatedRecordsPanel.tsx:14-98`), mount
      từ `RecordDetail.tsx:236-238`
- [ ] Activity — chưa có activity feed/timeline nào
- [ ] Audit information — `workflow_events` append-only log tồn tại ở backend (Phase 5), 0 tham
      chiếu trong `platform-ui/src`, chưa lộ ra UI

### Update

- [x] Dynamic edit form
- [ ] Dirty state — `GeneratedForm` chưa có. Tiền lệ cô lập: `admin/policies/PermissionMatrix.tsx:264,342-349`
      có track `dirty` boolean + label "You have unsaved changes" + disable Save, nhưng chỉ ở màn
      đó, chưa nhân rộng cho generic form
- [ ] Unsaved changes guard — không có `beforeunload`/router-blocker nào; label ở
      `PermissionMatrix.tsx` chỉ thông báo thụ động, không chặn điều hướng
- [ ] Partial update — **PARTIAL**: `updateMutation` gọi đúng `PATCH /api/{entity}/{id}`
      (`GeneratedForm.tsx:60-63`), nhưng payload gửi cả object `formData` dựng lại từ mọi key
      (`GeneratedForm.tsx:86-99, 102-104`) chứ không phải diff chỉ field đã đổi — "partial" mới
      chỉ đúng ở method HTTP, chưa đúng ở dữ liệu gửi đi
- [ ] Optimistic update — cả delete lẫn submit đều `await` mutation rồi mới refetch/navigate;
      `useApiMutation.ts` chưa có `onMutate`/rollback

### Delete

- [x] Confirmation — `window.confirm`
- [x] Soft delete — backend (`CrudService.delete()`)
- [ ] Restore — không có action/endpoint restore nào; `recordCapabilities.ts` chưa có
      `canRestore`/`deletedAt`
- [ ] Bulk delete — phụ thuộc row selection, xem mục 1

---

## 3. Workflow Builder

**Sửa path**: builder không nằm ở `src/workflow/WorkflowBuilder.tsx` như checklist bản cũ giả
định — đó là 1 component private bên trong `admin/LowCodeEntitiesAdminPage.tsx:740` (hàm
`WorkflowBuilder`). `src/workflow/` chỉ chứa UI transition ở tầng record
(`WorkflowActionBar.tsx`/`WorkflowStepper.tsx`/`WorkflowDiagram.tsx`/`TransitionButtons.tsx`).

- [x] Trigger configuration — chọn `stateField` (`LowCodeEntitiesAdminPage.tsx:740-832`)
- [ ] Condition builder — guard vẫn là JSON thô trong `Textarea` (`guardText`, parse qua
      `JSON.parse` — `LowCodeEntitiesAdminPage.tsx:579-611, 730-732`), cùng pattern
      `PolicyCondition` editor của `PoliciesAdminPage`, chưa có UI builder có cấu trúc
- [x] Action configuration — `TransitionRowEditor`, transition action/from/to/label
- [x] Workflow validation — `handleSaveDraft` validate phía client trước khi save (`:975-1007`)
- [x] Workflow publish — `publish`/`previewPublish` (`:916-917, 1083-1088`), hot-reload không
      cần restart
- [x] Workflow version — `LowCodeVersionHistory` + `rollback` (`:862-911`), version history
      append-only + preview publish
- [ ] Execution history — chưa có execution log ở tầng workflow-transition. `CronJobRuns`
      (`admin/CronJobsAdminPage.tsx:32-64`) là run history của job-scheduler, khác khái niệm này
- [ ] Execution detail — cùng lý do trên
- [ ] Retry execution — kể cả `CronJobRuns` (analog gần nhất) cũng chỉ đọc, không có nút retry
- [ ] Cancel execution — chưa có action cancel nào

Basic model:

```
Trigger
   |
   v
Condition
 ┌─┴─┐
YES  NO
 │    │
 v    v
Action End
```

---

## 4. Admin

- [ ] Tenant / workspace — backend đã có (`POST /platform/tenants`, Phase 16), `adminApi.ts` chỉ
      mang `tenantId` như 1 field scoping trên DTO có sẵn, chưa có màn CRUD tenant/workspace nào
- [x] Users — `UsersAdminPage.tsx`
- [x] Roles — assign/revoke role qua `UsersAdminPage.tsx`
- [x] Permissions — `PoliciesAdminPage.tsx` + `admin/policies/*`
- [x] Entities — `LowCodeEntitiesAdminPage.tsx`
- [x] Fields — field builder trong `LowCodeEntitiesAdminPage.tsx`
- [ ] Relations — chỉ có config field kind `reference` (`refEntity`/`refDisplayField` —
      `LowCodeEntitiesAdminPage.tsx:45, 88-89, 133, 251`), chưa có UI riêng cho relation
- [x] Views — list-view builder
- [x] Workflows — sub-component `WorkflowBuilder` trong cùng `LowCodeEntitiesAdminPage.tsx`
      (không phải file riêng, xem mục 3)
- [ ] Integrations — không có file/component/API call nào
- [ ] API keys — không có
- [ ] Webhooks — không có
- [ ] Audit logs — không có
- [ ] System settings — không có
- [ ] Feature flags — không có

---

## 5. Developer / Platform UI

- [ ] Metadata explorer — một phần qua `LowCodeEntitiesAdminPage.tsx`, chưa phải explorer riêng
- [ ] Entity explorer — chưa có
- [ ] API explorer — `GET /metadata/openapi.json` public sẵn; repo chỉ có
      `metadata/generated-types.ts` (build artifact từ `openapi-typescript`, dùng cho type-check
      lúc build), không phải UI duyệt API lúc runtime
- [ ] Event explorer — chưa có
- [ ] Webhook manager — chưa có
- [ ] Integration manager — chưa có
- [x] Job monitor — `CronJobsAdminPage.tsx`, `CronJobRuns` hiện status/scheduledFor/finishedAt/
      error theo từng job (`:32-64`), chỉ đọc (không nhầm với "Workflow execution monitor" bên
      dưới — 2 khái niệm khác nhau)
- [ ] Workflow execution monitor — chưa có, xem mục 3
- [ ] System health — `/health` chỉ xuất hiện như 1 route type trong
      `generated-types.ts:1838`, chưa có component nào tiêu thụ
- [ ] Feature flag manager — chưa có
- [ ] Configuration viewer — chưa có (backend có `metap-config`/`/admin/config`, chưa lộ UI)

---

## 6. UX Infrastructure

- [ ] Keyboard shortcuts — 0 hit cho hotkey/cmdk/keydown-meta pattern
- [ ] Command palette — chưa có
- [ ] URL state — filter/sort trong `GeneratedList` sống trong `useState`
      (`GeneratedList.tsx:78-81`), chưa sync qua `useSearchParams`
- [ ] Deep linking — record ID là route param (routing cơ bản hoạt động), nhưng filter/sort/tab
      state chưa deep-link được
- [x] Browser back/forward — mặc định của `react-router`, qua
      `ReactRouterNavigationProvider.tsx`
- [ ] State persistence — locale là preference **duy nhất** được persist, và persist ở
      **server-side** qua `GET/PUT /preferences` (`i18n/LocaleProvider.tsx:33-58`), không phải cả
      `localStorage`
- [ ] Local storage abstraction — 0 tham chiếu `localStorage`/`sessionStorage` trong toàn bộ
      `platform-ui/src`
- [ ] Unsaved changes guard — xem mục 2 (Update)
- [ ] Global loading state — loading xử lý per-screen (`Spinner` riêng từng component), chưa có
      provider/indicator cấp app
- [ ] Global error handling — `AppShellLayout.tsx` chưa có `ErrorBoundary`; lỗi xử lý cục bộ per
      screen qua `ApiErrorMessage`
- [ ] Error recovery — cùng lý do trên, chưa có retry/reset boundary
- [ ] Accessibility — **PARTIAL**: `aria-*`/`role=` trực tiếp trong `platform-ui/src` thưa (19
      chỗ/11 file, chủ yếu `aria-label` trên icon button). `@metap/ui` build nhiều atom tương tác
      trên Radix (`react-dialog`/`-dropdown-menu`/`-popover`/`-tabs`/`-tooltip`/`-accordion`/
      `-alert-dialog` — `design-system/package.json:73-80`) nên có sẵn keyboard/focus/ARIA ở tầng
      atom miễn phí, nhưng chưa có audit a11y riêng, skip-link, hay `aria-live` cho list/form
      async state
- [ ] Responsive behavior — **PARTIAL, gần như chưa có**: chỉ 3 chỗ dùng breakpoint Tailwind
      (`sm:`/`md:`/`lg:`) trong toàn `platform-ui/src` (cả 2 ở `RecordDetail.tsx`/
      `RelatedRecordsPanel.tsx`). Table cố định width của `GeneratedList` và header nav của
      `AppShellLayout` chưa có xử lý mobile (không hamburger/collapse)

---

## Nhóm ưu tiên tiếp theo

### Security / Platform

- [x] Permission / ABAC — `PermissionService` (Phase 3)
- [x] Audit — `workflow_events` (Phase 5), chưa có UI (xem mục 2 Detail)
- [ ] File — chưa có `FieldKind` `"file"`, chưa wiring upload trong
      `FieldInput.tsx`/`fieldKindConfig.ts`. `@metap/ui` **đã có sẵn atom `FileUpload`**
      (`design-system/src/components/file-upload/file-upload.tsx`) nhưng `platform-ui` chưa dùng
      — gap thấp công sức hơn các mục cần dựng atom mới từ đầu
- [ ] Notification — vẫn `[ ]` ở tầng **kênh backend thật** (outbox stub + `notification-worker`
      chỉ log stdout, Phase 5, chưa phải email/SMS/webhook) — nhưng **2026-09-04**: tầng feedback
      trong-app đã có, `AppShellLayout` mount `ToastProvider`, `GeneratedForm`/`GeneratedList`
      toast khi tạo/sửa/xoá thành công (`shell/AppShellLayout.tsx`, `form/GeneratedForm.tsx:105-112`,
      `list/GeneratedList.tsx:210-212`) — 2 khái niệm khác nhau, đừng nhầm "toast báo kết quả 1
      action FE" với "kênh notification thật gửi ra ngoài hệ thống"
- [ ] Realtime — 0 WebSocket/SSE/socket client trong `platform-ui/src`
- [x] Admin — xem mục 4 ở trên

### Advanced

- [ ] Workflow builder nâng cao — phần cơ bản đã có (mục 3), chưa "advanced" (condition builder
      có cấu trúc, execution monitor)
- [ ] Dashboard builder — **chưa từng được ghi nhận ở roadmap hay team-charter**; cần một
      feature brief riêng trong `docs/features/` nếu muốn theo đuổi, theo đúng kỷ luật
      trigger-based hiện tại
- [ ] Developer tools
- [ ] Integrations
- [ ] Advanced customization

### Production

- [ ] Unit tests FE — **0, không phải "tối thiểu"**: `platform-ui` không có 1 file
      `*.test.*`/`*.spec.*` nào, không có config vitest/jest trong toàn repo. Ngược lại,
      `design-system` có test khá đầy đủ ở tầng atom (25+ file `*.test.tsx` dưới
      `design-system/src/components/*/`, script `vitest`/`vitest run` trong `package.json`) —
      atom UI có test, logic CRUD/workflow/admin của `platform-ui` thì không
- [ ] Integration tests FE — chưa có
- [ ] E2E tests FE — có chạy Playwright thủ công một lần cho Phase 15 (verify live), không phải
      suite tự động trong CI. Không tìm thấy config Playwright/Cypress hay spec file thật nào ở
      cả 2 repo lúc rà soát này
- [x] Performance — load test cho list/query path, backend only (Phase 8, 2026-08-17)
- [ ] Accessibility — xem mục 6
- [ ] Error monitoring — không có Sentry/error-tracking SDK hay `window.onerror` hook nào
- [x] Production hardening — phần lớn xong ở backend (Phase 8: CORS/CSP/rate limit/structured
      logging/Docker non-root/CI gate); secret manager, production deployment topology, và
      hardening riêng cho FE vẫn còn treo

---

## Definition of Done

Ghi chú: bản gốc yêu cầu unit/integration/E2E test cho mọi feature — mâu thuẫn với hướng test
tối giản, có mục tiêu đang áp dụng cho dự án hiện nay. Giữ checklist tham khảo bên dưới nhưng áp
dụng theo phạm vi thực tế của từng feature, không áp dụng máy móc từng dòng cho mọi PR.

Một feature được coi là hoàn thành chỉ khi:

- [ ] UI implemented
- [ ] Loading state implemented
- [ ] Empty state implemented
- [ ] Error state implemented
- [ ] Permission handling implemented
- [ ] API integration implemented
- [ ] Validation implemented
- [ ] Responsive behavior implemented
- [ ] Accessibility considered
- [ ] Unit tests added — chỉ khi có logic thật sự cần bảo vệ, không phải mặc định cho mọi feature
- [ ] Integration tests added where applicable
- [ ] E2E test added for critical flows
- [ ] No duplicated business logic
- [ ] No hard-coded metadata that should come from backend
- [ ] Documentation updated

---

## Critical Architecture Principle

Metap FE là một UI platform, không phải một tập hợp màn hình ERP.

Implementation nên theo hướng:

```
                METAP BACKEND
                     |
                     | Metadata
                     v
             +---------------+
             | Metadata      |
             | Engine        |
             +-------+-------+
                     |
                     v
             +---------------+
             | UI Engine     |
             +---------------+
             | Field         |
             | Form          |
             | Table         |
             | Detail        |
             | Filter        |
             | View          |
             +-------+-------+
                     |
          +----------+----------+
          |          |          |
          v          v          v
        CRUD      Workflow   Dashboard
          |          |          |
          +----------+----------+
                     |
                     v
                APPLICATION
```

Nhánh "Dashboard" trong sơ đồ trên hiện chưa có gì trong roadmap (xem "Dashboard builder" ở
"Nhóm ưu tiên tiếp theo" bên trên) — giữ trong sơ đồ như định hướng dài hạn, không phải cam kết
đã lên kế hoạch.

Tiêu chí thành công chính:

"Thêm một entity hoặc field type mới nên chỉ cần configuration/metadata và một extension nhỏ,
biệt lập, thay vì tạo thêm một implementation frontend hard-code hoàn toàn mới."

## Chưa lên kế hoạch

Các mục `[ ]` ở trên không tự động là backlog theo thứ tự ưu tiên — dự án đang theo kỷ luật
trigger-based (`docs/architectures/02-constraints/00-index.md`), không xây trước khi có nhu cầu cụ thể.
Trước khi bắt đầu bất kỳ mục nào trong nhóm "Admin" (Integrations/API keys/Webhooks/Audit
logs/System settings/Feature flags), "Developer / Platform UI", hay "Dashboard builder", nên có
một feature brief trong `docs/features/` nêu rõ trigger, theo đúng cách các mục tương tự đã được
ghi nhận ở `docs/team-charter.md`.

Gợi ý theo công sức khi cần chọn việc trong nhóm "làm được ngay" (mục 1-3, 6): những gap có sẵn
atom ở `design-system` nhưng chưa wiring vào `platform-ui` rẻ hơn những gap chưa có atom nào ở
tầng nào cả (column resize/reorder, saved views, command palette, realtime). **Đã làm 2026-09-04**:
`Toast` (notification, xem mục "Nhóm ưu tiên tiếp theo") và `Refresh` (không cần atom mới, chỉ lộ
`refetch()` có sẵn ra UI). Còn lại trong nhóm này: `Checkbox` cho row selection (cần quyết định
select-all nghĩa là gì khi list dùng infinite scroll + virtualization — chỉ chọn record đã load
hay cả trang chưa fetch — và tự nó chưa có ý nghĩa nếu không có bulk action đi kèm, nên gộp 2 việc
này lại khi làm), `FileUpload` cho field kind file (cần thêm `FieldKind` mới ở backend
`metap-metadata` trước — không phải việc chỉ ở `platform-ui`), `Tabs` cho Detail field groups (cần
khái niệm "group" mới trong `entityLayout.ts`, hiện chỉ có `span: 1|2` per-field, là quyết định
thiết kế chứ không chỉ wiring).
