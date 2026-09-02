# Entity variant kiểu polymorphic/discriminated-union

- **Trạng thái:** proposed — chưa có trigger, **rủi ro cao nhất trong toàn bộ backlog "chưa có
  trigger"**
- **Người đề xuất:** ghi lại từ thảo luận kiến trúc, `docs/team-charter.md` ("Định hướng đang ghi
  nhận, chưa có trigger" #11)
- **Track sở hữu:** Backend Core
- **Phase roadmap liên quan:** không thuộc phase nào

## Vấn đề / động lực

Một entity logic chứa nhiều "hình dạng" record khác nhau trong cùng 1 logical collection (kiểu
MongoDB discriminated union/polymorphic document) — vd `crm.activities` có thể là `call`/`email`/
`meeting`, mỗi loại field khác nhau nhưng cùng list view "Activities".

## Phạm vi

**Trong phạm vi (nếu được kích hoạt):** chưa thiết kế.

**Ngoài phạm vi:** không phải `FieldKind::Enum` (đã có, cho 1 field có giá trị cố định) — đây là
toàn bộ *shape* record khác nhau theo variant, không phải 1 field.

## Tiêu chí chấp nhận

<Chưa xác định — chưa có trigger.>

## Ranh giới kiến trúc bị đụng tới — vì sao đây là ý rủi ro cao nhất

`EntityDefinition.fields` hôm nay là **1 danh sách phẳng** dùng chung cho:
- Validation (`MetadataCompiler::validate`, `crates/metap-metadata/src/compiler.rs`)
- List view / filter (`EntityListView.fields`)
- OpenAPI generator (`crates/metap-metadata/src/openapi.rs`)
- Codegen type phía FE (`platform-ui/src/metadata/generated-types.ts`)

Thêm variant nghĩa là lồng thêm 1 tầng `variant → fields` ở **tất cả** các chỗ trên cùng lúc,
không chỉ thêm 1 cột `variant` vào `records`. Mỗi chỗ trong 4 chỗ trên đều cần trả lời riêng:
validation theo variant nào, list view hiển thị field nào khi record thuộc variant khác nhau,
OpenAPI/TypeScript type có union type hay không.

## Rủi ro / phụ thuộc

- **Trigger: chưa có** — chưa entity nào trong repo hôm nay cần nhiều schema khác nhau trong cùng
  1 logical collection.
- **Rủi ro thiết kế cao nhất trong backlog này** — nên yêu cầu ADR + review kỹ trước khi bắt đầu
  dù trigger có xuất hiện, vì blast radius chạm toàn bộ pipeline metadata (không chỉ 1 crate).
- Cân nhắc trước: liệu 2 entity riêng biệt (thay vì 1 entity nhiều variant) có giải quyết được nhu
  cầu thật không — nếu có, effort thấp hơn nhiều so với xây polymorphic entity thật.
