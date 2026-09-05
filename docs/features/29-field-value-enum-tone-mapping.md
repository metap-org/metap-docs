# `FieldValue`'s enum rendering — tone theo giá trị thay vì luôn `variant="secondary"`

- **Trạng thái:** proposed — **chưa scope xong, không code trong đợt này**
- **Người đề xuất:** `platform-ui/docs/audits/03-waf-demo-component-placement-audit.md` (2026-09-05),
  finding #8 (phần "gap 2 chiều").
- **Track sở hữu:** Frontend Platform (có thể kéo theo Backend Core — xem "Rủi ro")
- **Phase roadmap liên quan:** không thuộc phase nào

## Vấn đề / động lực

`metap-demo-waf` tự viết `StatusBadge`/`TONES` (giữ nguyên tại app — data là từ vựng enum WAF thật,
audit đã xác nhận đúng khi giữ) để làm điều `platform-ui/src/field/FieldValue.tsx:75-77` chưa làm:

```tsx
if (field.kind === "enum") {
  return <Badge variant="secondary">{formatted}</Badge>;
}
```

Mọi giá trị enum — `"active"` hay `"failed"` hay `"critical"` — đều ra `variant="secondary"` như
nhau, không map theo ngữ nghĩa giá trị. **Cơ chế** "tone theo giá trị enum" là 1 khả năng chung
đáng có ở tầng `platform-ui` (không chỉ WAF cần) — nhưng đây không phải bug rõ ràng, là 1 khả năng
còn thiếu, cần quyết định thiết kế trước khi code, đúng tinh thần audit: *"không tự quyết định nó
có đáng làm hay không"*.

## Phạm vi

**Chưa chốt — cần quyết định trước khi code:**
- Khai báo tone-mapping ở đâu? Ứng viên: 1 `FieldDisplayHint` mới kiểu
  `{ field, enumTones: Record<string, BadgeVariant> }` (metadata do backend/low-code khai báo) —
  nhưng đây là thay đổi contract `EntitySummary`/`metap-metadata`, **đụng cả Backend Core**
  (`crates/metap-metadata/src/entity.rs`, `openapi.rs`'s hand-written JSON Schema), không chỉ FE.
  Cần sign-off track chéo theo `docs/team-charter.md` nếu đi hướng này.
- Hướng thay thế nhẹ hơn: 1 prop tuỳ chọn trên `FieldValue`/`GeneratedList` cho phép caller (app)
  tự truyền `enumTones` mà không cần đổi backend/metadata — giữ contract hiện tại nguyên vẹn, đổi
  ít hơn, nhưng "khai báo được từ metadata" (đúng triết lý platform metadata-driven của dự án) yếu
  hơn.
- Convention đặt tên giá trị→tone dùng chung nào (nếu có) giữa các app khác nhau, hay mỗi app tự
  định nghĩa hoàn toàn độc lập như `TONES` của WAF hiện tại?

**Ngoài phạm vi (chắc chắn):**
- Di chuyển `StatusBadge`/`TONES` của WAF sang `platform-ui` — audit đã kết luận rõ KHÔNG, data là
  của WAF.

## Tiêu chí chấp nhận

Chưa có — sẽ điền khi chuyển `proposed` → `approved`, sau khi chốt hướng ở "Phạm vi".

## Ranh giới kiến trúc bị đụng tới

Tối thiểu `platform-ui/src/field/FieldValue.tsx`. Nếu chọn hướng metadata-driven (`FieldDisplayHint`
mới), lan sang `crates/metap-metadata` (Backend Core) + `platform-ui/src/metadata/generated-types.ts`
(cần regenerate) — cần ADR nếu đổi contract `EntitySummary` ra ngoài.

## Rủi ro / phụ thuộc

Rủi ro chính: đây là quyết định thiết kế thật (không phải wiring), không nên tự chọn hướng và code
luôn trong 1 lượt xử lý nhanh — khác hẳn #26/#27/#28 (thuần di chuyển/dọn code, không đổi contract
nào). Không phụ thuộc feature khác, nhưng nên chờ quyết định trước khi bắt đầu.
