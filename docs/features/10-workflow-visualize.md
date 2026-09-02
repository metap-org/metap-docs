# Workflow visualize được / hướng BPM nhẹ

- **Trạng thái:** proposed — chưa có trigger
- **Người đề xuất:** ghi lại từ thảo luận kiến trúc, `docs/team-charter.md` ("Định hướng đang ghi
  nhận, chưa có trigger" #2)
- **Track sở hữu:** Frontend Platform (đọc metadata backend đã có sẵn, không cần core mới)
- **Phase roadmap liên quan:** không thuộc phase nào

## Vấn đề / động lực

Một entity có workflow (`EntityWorkflow`/`WorkflowTransition` — state, transition, guard) hôm nay
chỉ hiển thị được qua metadata JSON thô hoặc list trạng thái/action trên UI form — không có cách
nhìn trực quan "state nào chuyển được sang state nào, điều kiện gì". Với low-code platform, đây là
giá trị sản phẩm hợp lý: admin/operator định nghĩa workflow xong muốn thấy ngay sơ đồ, không phải
đọc JSON.

**Không nhầm với việc build 1 workflow engine mới** — engine (`metap-cron`'s
`TargetType::Steps`/`WaitEvent`, Phase 17) đã xong. Đây thuần là 1 UI đọc metadata có sẵn
(`WorkflowTransition` đã có đủ `from`/`to`/`action`/`guard`) và vẽ ra, không cần backend mới.

## Phạm vi

**Trong phạm vi (đề xuất sơ bộ, chưa chốt):**
- Đọc `EntityWorkflow`/`WorkflowTransition` từ `GET /metadata/entities/{name}` (đã có đủ dữ liệu).
- Render dạng graph (state = node, transition = edge có label = action) — chỉ đọc, không edit qua
  UI này (workflow vẫn định nghĩa qua code hoặc low-code builder như hôm nay).

**Ngoài phạm vi:**
- Không phải BPM editor cho phép kéo-thả định nghĩa workflow mới — chỉ visualize cái đã có.
- Không visualize `metap-cron`'s `TargetType::Steps`/`WaitEvent` chain (đó là 1 tầng khác, activity
  chain, không phải state machine của entity) — nếu cần, nên là 1 brief riêng.

## Tiêu chí chấp nhận

<Chưa xác định — chưa có yêu cầu cụ thể từ 1 entity/nghiệp vụ thật nào trong repo.>

## Ranh giới kiến trúc bị đụng tới

Không đụng backend/core — thuần đọc `GET /metadata/entities/{name}` đã public. Thuộc
`platform-ui`/`@metap/ui` (repo riêng), không phải `metap`.

## Rủi ro / phụ thuộc

- Chưa có yêu cầu cụ thể — brief này tồn tại để không quên ý, không phải sẵn sàng code.
- Cần chọn thư viện graph-rendering (không có sẵn trong `@metap/ui` hôm nay) — 1 dependency mới,
  nên cân nhắc kỹ trước khi thêm.
