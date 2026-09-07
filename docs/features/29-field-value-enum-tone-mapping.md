# `FieldValue`'s enum rendering — tone theo giá trị thay vì luôn `variant="secondary"`

- **Trạng thái:** done, 2026-09-06 — chọn hướng metadata-driven (`FieldDisplayHint.enumTones`)
- **Người đề xuất:** `platform-ui/docs/audits/03-waf-demo-component-placement-audit.md` (2026-09-05),
  finding #8 (phần "gap 2 chiều").
- **Track sở hữu:** Frontend Platform + Backend Core (contract `FieldDisplayHint` đụng cả hai, tự
  duyệt — team một người, theo `docs/team-charter.md`)
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

**Đã chốt (2026-09-06): hướng metadata-driven** — `FieldDisplayHint` (đã tồn tại sẵn cho
`resolveVia: "users"`, `crates/metap-metadata/src/entity.rs`) có thêm field thứ hai, độc lập với
`resolveVia`: `enumTones: Option<HashMap<String, String>>` (camelCase `enumTones` trên wire),
key là giá trị enum, value là tên `Badge` variant. Cả `resolveVia` lẫn `enumTones` giờ optional
trên struct — một hint chỉ cần khai đúng cái nó dùng, không bắt buộc cả hai như `resolveVia`
từng là required field duy nhất.

**Lý do chọn hướng 1 thay vì prop-based**: `FieldDisplayHint` đã là đúng cơ chế "backend/low-code
khai báo, generic renderer tự đọc" cho `resolveVia` — thêm `enumTones` vào cùng struct là tái
dùng một cơ chế đã có, không phải xây mới; tách riêng thành 2 khái niệm (1 hint metadata-driven
cho id-resolution, 1 prop-based cho tone) sẽ tạo 2 cách khai báo "display hint" khác nhau trên
cùng 1 field cho 2 nhu cầu tương tự.

**Giá trị variant là `String` thô, không phải enum Rust** — cùng lý do `resolve_via` luôn là
`String`: một giá trị lạ là vấn đề của FE tự fallback an toàn (`FieldValue.tsx`'s `asBadgeVariant`,
fallback về `"secondary"` — đúng hành vi cũ khi không có hint), không phải lý do fail
`MetadataRegistry::register_all_submitted` hay bắt recompile `metap-metadata` mỗi khi
`@metap/ui`'s `Badge` thêm variant mới.

**Ngoài phạm vi (chắc chắn, không đổi)**:
- `StatusBadge`/`TONES` của WAF (`data-plane/web/src/components/primitives.tsx`) **giữ nguyên tại
  app** — audit đã kết luận đúng, đây là từ vựng enum của WAF, WAF không migrate sang cơ chế này
  trong đợt này (không ai yêu cầu, ngoài scope brief).
- Convention đặt tên tone dùng chung giữa nhiều app — vẫn để mỗi app tự quyết như trước, cơ chế
  chỉ cho phép khai báo, không áp đặt vocabulary.

## Tiêu chí chấp nhận

- `FieldDisplayHint.enum_tones: Option<HashMap<String, String>>` thêm vào `entity.rs`, serde
  `rename_all = "camelCase"` → `enumTones`, `skip_serializing_if = "Option::is_none"`.
  `resolve_via` đổi từ `String` bắt buộc sang `Option<String>` (thay đổi tương thích ngược trên
  wire — response cũ có `resolveVia` vẫn đọc được, response mới thiếu nó vẫn hợp lệ).
- `openapi.rs`'s `field_display_hint_json_schema()` cập nhật: `enumTones` là
  `{"type": "object", "additionalProperties": {"type": "string"}}`, `required` chỉ còn `["field"]`.
  Verify sống: `GET /metadata/openapi.json` từ `crm-server` đang chạy phản ánh đúng schema mới
  (đọc trực tiếp qua `curl`, không chỉ đọc code).
- `platform-ui`'s `generated-types.ts` regenerate qua `pnpm generate:types` (chạy thật, không tay
  sửa — đúng quy tắc "never hand-edit generated-types.ts" của `../metap/CLAUDE.md`).
- `FieldValue.tsx`'s nhánh `field.kind === "enum"` đọc `displayHint.enumTones?.[String(value)]`,
  fallback `"secondary"` nếu hint vắng mặt hoặc giá trị không khớp tên variant nào của `@metap/ui`.
- 2 call site cũ dựng `FieldDisplayHint` bằng struct literal (`metap-metadata/src/registry.rs`'s
  test, `metap-demo-waf`'s `incident_entity.rs`) cập nhật theo field mới — cả hai vẫn chỉ dùng
  `resolveVia`, không migrate sang `enumTones` (ngoài scope, xem "Ngoài phạm vi").
- `cargo test -p metap-metadata` (61 test, gồm 2 test mới cho `enum_tones` round-trip + hint chỉ
  có `field`), `cargo build --workspace` ở cả `metap` và `metap-demo-waf/data-plane` xanh;
  `tsc --noEmit`/`prettier --check` sạch trên `platform-ui`.

## Ranh giới kiến trúc bị đụng tới

`crates/metap-metadata/src/entity.rs` + `openapi.rs` (Backend Core), `platform-ui/src/field/FieldValue.tsx`
+ `src/metadata/generated-types.ts` (Frontend Platform), 2 call site có sẵn ở `metap`/`metap-demo-waf`
cập nhật theo struct mới (không đổi hành vi của chúng). Không đổi `EntitySummary`'s field khác,
không cần ADR mới — cùng tiền lệ `resolveVia` đã có, chỉ mở rộng struct đã tồn tại.

## Rủi ro / phụ thuộc

Không phụ thuộc feature khác. Chưa browser-test (đúng frontend verification policy — viết code +
`tsc`/`lint`/`format`, để user tự kiểm bằng mắt).

**Cập nhật 2026-09-06 — có consumer thật đầu tiên:** `metap-demo-waf`'s `zone_entity.rs`
(`zones-service`) khai `FieldDisplayHint.enumTones` cho `waf.zones`'s `status`, mirror đúng
`primitives.tsx`'s `TONES` zone-status mapping (`active→default`, `pending→secondary`,
`paused→outline`, `suspended→destructive`) — không phải để thay `StatusBadge`/`TONES` (vẫn giữ
nguyên tại app, đúng kết luận audit 03), mà để màn hình generic `/records/waf.zones` (escape-hatch
CRUD qua `GeneratedList`) không còn hiện `status` toàn màu xám mặc định, khác màu với chính
`ZonesPage`'s `StatusBadge` đang hiện. Verify sống qua `GET /metadata/entities` (token thật,
`zones-service` chạy trong `docker-compose.dev.yml`'s `cargo watch`) — `fieldDisplayHints` trả
đúng `enumTones` như khai. Không entity nào khác trong repo dùng `enumTones` — vẫn còn hầu hết là
cơ chế sẵn sàng, chưa phải phổ biến.
