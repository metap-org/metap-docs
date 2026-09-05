# Chuyển 5 primitive domain-free + 2 date-util từ `metap-demo-waf` sang `@metap/ui`/`@metap/platform-ui`

- **Trạng thái:** proposed
- **Người đề xuất:** `platform-ui/docs/audits/03-waf-demo-component-placement-audit.md` (2026-09-05),
  finding #1-5 + finding phụ (`shortDate`/`dayLabel`).
- **Track sở hữu:** Frontend Platform (đọc/sửa chéo sang `metap-demo-waf`)
- **Phase roadmap liên quan:** không thuộc phase nào — cùng tinh thần Phase 59

## Vấn đề / động lực

`metap-demo-waf/data-plane/web/src/components/primitives.tsx` tự ghi chú ngay trong doc-comment
đầu file: *"Anything here that turns out to be domain-free (`StatTile`, `TimeSeries`) is a
candidate to move later, once a second app wants it."* Audit 03 xác nhận **5 trong 6 component**
của file này hoàn toàn domain-free (không field nào biết `Zone`/`Incident`/WAF là gì), đã lặp lại
3-12 chỗ, và **chưa có** component tương đương ở `@metap/ui`/`@metap/platform-ui` (grep xác nhận
0 kết quả) — không phải trùng lặp, là gap thật:

- `PageHeader` (10 chỗ dùng), `EmptyState` (10), `SectionCard` (12), `StatTile` (4) — thuần layout,
  build trên `Card`/`CardContent`/`Skeleton` đã có sẵn.
- `TimeSeries` (3 chỗ) — line/area chart, tự nhận trong doc-comment là "cùng lý do & cách làm với
  `BarChart`" (`@metap/ui` đã có) — đúng 1 chart-kind design-system còn thiếu.
- `shortDate`/`dayLabel` (8+3 chỗ) — 2 hàm format ngày thuần, 0 phụ thuộc ngoài `Date` built-in.

`StatusBadge`/`TONES` (component thứ 6 trong file) **không** nằm trong phạm vi này — audit đã kết
luận rõ đây là từ vựng nghiệp vụ WAF thật, giữ nguyên tại app.

## Phạm vi

**Trong phạm vi:**
- Thêm 5 atom mới vào `design-system/src/components/`: `page-header`, `empty-state`,
  `section-card`, `stat-tile`, `time-series` — port gần như nguyên trạng logic/style, đổi
  `forwardRef` + `className` passthrough cho đúng convention design-system (theo `Card`/`BarChart`
  đã có).
- `TimeSeries` bỏ phụ thuộc `useTranslation()`/key i18n riêng của WAF (`waf.common.noDataInWindow`,
  `waf.common.timeSeries`) — đổi thành prop `emptyMessage`/`ariaLabel` với default tiếng Anh, giống
  `BarChart.ariaLabel` đã làm. Caller (WAF) tự truyền chuỗi đã dịch, không đổi UX hiển thị.
- `shortDate`/`dayLabel` chuyển sang `platform-ui/src/lib/dates.ts` (thuần function, không phải
  UI — theo audit gợi ý "cạnh `i18n/useEntityLabels.ts`").
- `metap-demo-waf/data-plane/web/src/components/primitives.tsx` xoá 5 export đã chuyển + 2 util,
  chỉ giữ lại `StatusBadge`/`TONES`; mọi import site (10+10+12+4+3+8+3 lượt) đổi sang
  `@metap/ui`/`@metap/platform-ui`.

**Ngoài phạm vi:**
- `StatusBadge`/`TONES` — giữ nguyên tại app (đã quyết định ở audit).
- Gap `FieldValue.tsx`'s enum-tone rendering mà audit nêu — brief riêng
  ([29](29-field-value-enum-tone-mapping.md)), cần quyết định thiết kế trước, không gộp vào đây.

## Tiêu chí chấp nhận

- `design-system`: `tsc`/`build`/`eslint` sạch; 5 atom mới export từ `src/index.ts`.
- `platform-ui`: `tsc`/`oxlint`/`prettier --check` sạch; `dates.ts` export từ `src/index.ts`.
- `metap-demo-waf/data-plane/web`: không còn import nào từ `./components/primitives` ngoại trừ
  `StatusBadge`/`TONES`; `tsc`/`oxlint`/`prettier`/`vite build` sạch.
- Không đổi hành vi/giao diện hiển thị nào (pixel-for-pixel port) — mọi call site giữ nguyên props
  truyền vào, chỉ đổi đường import.

## Ranh giới kiến trúc bị đụng tới

`design-system/src/components/*` (5 thư mục mới), `platform-ui/src/lib/dates.ts` (mới),
`metap-demo-waf/data-plane/web/src/components/primitives.tsx` + mọi file import từ đó (~19 file).
Đúng rule "UI → design-system, UI+logic/utility dùng chung → platform-ui". Không đụng backend,
không cần ADR.

## Rủi ro / phụ thuộc

`design-system` build ra `dist/` (tsup) — `platform-ui`/`metap-demo-waf` chỉ thấy atom mới sau khi
rebuild `dist/`, không tự động qua watch nguồn `src` (đã gặp ở phiên trước, biết cách xử lý). Không
phụ thuộc feature khác.
