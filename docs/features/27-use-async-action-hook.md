# `useAsyncAction` hook ở `@metap/platform-ui` — dọn 18 chỗ lặp busy+try/catch/toast

- **Trạng thái:** proposed
- **Người đề xuất:** `platform-ui/docs/audits/03-waf-demo-component-placement-audit.md` (2026-09-05),
  finding #6.
- **Track sở hữu:** Frontend Platform (đọc/sửa chéo sang `metap-demo-waf`)
- **Phase roadmap liên quan:** không thuộc phase nào

## Vấn đề / động lực

Audit 03 tìm thấy pattern sau lặp lại **18 lần trên 9 file** trong `metap-demo-waf/data-plane/web`,
byte-for-byte gần như giống hệt, không 1 chữ nào biết WAF là gì:

```tsx
setBusy(true);
try {
  await someAction(...);
  invalidate();
  toast(t("..."), { variant: "default" });
} catch (e) {
  toast(e instanceof Error ? e.message : String(e), { variant: "destructive" });
} finally {
  setBusy(false);
}
```

`useApiMutation` (React Query, đã có ở `platform-ui`) giải quyết đúng vấn đề này cho mutation qua
REST — nhưng **15/18 chỗ gọi hàm imperative trong `api/waf.ts`** (GraphQL qua `graphqlAuthed`,
không phải hook), nên không thể thay bằng `useApiMutation` mà không đổi kiến trúc data layer của
cả app. Đây là lý do thật khiến pattern không tự dùng lại được cái đã có, không phải do tác giả
không biết nó tồn tại.

## Phạm vi

**Trong phạm vi:**
- 1 hook mới `useAsyncAction()` ở `platform-ui/src/hooks/useAsyncAction.ts`: nhận 1 async function,
  trả về `{ run, busy }` — tự bọc try/catch/finally, tuỳ chọn toast thành công/lỗi qua tham số.
  Độc lập với `useApiMutation`/React Query — bổ sung, không thay thế.
- Áp dụng vào cả 18 chỗ đã liệt kê trong audit (9 file:
  `AlertingPage.tsx`×3, `SettingsPage.tsx`×2, `FindingsPage.tsx`×1, `IncidentsPage.tsx`×2,
  `IncidentDetailPage.tsx`×1, `ZoneDetailPage.tsx`×1, `ZoneDdosTab.tsx`×2, `ZoneRulesTab.tsx`×3,
  `ZoneScansTab.tsx`×2, `ZoneEventsTab.tsx`×1) + biến thể nhẹ hơn ở `OnboardingPage.tsx`'s `guard()`
  (dùng `error` state thay vì toast — audit gợi ý đổi theo cho nhất quán).

**Ngoài phạm vi:**
- Đổi transport GraphQL→REST hay wire `useApiMutation` trực tiếp cho các action này — không phải
  mục tiêu của brief này.

## Tiêu chí chấp nhận

- `useAsyncAction` type-safe, không ép buộc `toast` (tham số optional message/variant).
- Cả 18+1 chỗ trong `metap-demo-waf` refactor xong, hành vi UI giữ nguyên (nút vẫn disable lúc
  chạy, toast/error message y hệt trước).
- `platform-ui`: `tsc`/`oxlint`/`prettier` sạch. `metap-demo-waf/data-plane/web`: `tsc`/`oxlint`/
  `prettier`/`vite build` sạch.
- Giảm được tổng số dòng lặp thật (đo bằng diff -, không chỉ tuyên bố).

## Ranh giới kiến trúc bị đụng tới

`platform-ui/src/hooks/useAsyncAction.ts` (mới) + 9 file trong `metap-demo-waf/data-plane/web/src`.
Thuần UI+logic tái dùng, đúng chỗ `platform-ui`.

## Rủi ro / phụ thuộc

19 điểm sửa cùng lúc — rủi ro chính là bỏ sót hành vi khác biệt nhỏ giữa các chỗ dùng (ví dụ có chỗ
không có success-toast) khi rập khuôn theo 1 hook chung; cần đọc kỹ từng chỗ trước khi thay, không
chỉ tìm-thay hàng loạt. Không phụ thuộc feature khác.
