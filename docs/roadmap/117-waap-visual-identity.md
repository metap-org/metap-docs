# 117. Nhận diện thị giác WAAP: "control room"

**Trạng thái:** làm xong 2026-11-21 (design-system, metap-themes, platform-ui, m-web). **Chưa ai xem trên trình duyệt**: theo quy ước, chủ dự án tự kiểm; mọi thứ dưới đây mới qua typecheck, lint, test và build.

## Vấn đề

Chủ dự án: giao diện quá đơn giản, nhìn vào không nhận ra thương hiệu. Khảo sát code (2026-11-21) cho thấy lý do cụ thể:

- Token của `@metap/ui` là **bộ mặc định của shadcn**: primary xanh đen gần như đen, nền trắng, neutral xám xanh, bo góc 0.5rem. Không có màu thương hiệu, không có màu nhấn.
- Theme đang dùng (`@metap/theme-enterprise`) chỉ đổi sang navy/slate và góc vuông hơn; vẫn là "giao diện doanh nghiệp chung chung". Nó còn khai báo `--font-sans` nhưng **không có gì áp font đó** (không có `fontFamily` trong preset, không có `font-family` ở base), nên font chưa bao giờ đổi.
- Không có màu trạng thái trong token: `success`/`warning` của Badge là `emerald-600`/`amber-500` viết cứng, StatTile cũng vậy, biểu đồ dùng `--primary`. Với sản phẩm bảo mật, "chặn" và "cho qua" là thông tin chính mà lại không có thang màu riêng.
- Thanh bên là `bg-muted/30`: không có tương phản với vùng làm việc, không có dấu "đang ở đây" khi thu gọn còn icon.
- Đăng nhập là một form `max-w-xs` giữa trang trắng; thương hiệu là chữ `WAF Portal` cỡ `text-sm`; không có logo, favicon, tiêu đề tab là `WAF Portal`.

## Hướng đã chọn (chủ dự án, 2026-11-21): "control room tối"

Thanh bên mực tối bên cạnh vùng làm việc sáng; một màu tín hiệu cyan; bo 0.375rem; mono cho dữ liệu máy. Lý do chọn: gợi cảm giác phòng giám sát/SOC, khác dashboard SaaS thông thường, và thanh bên tối cho tương phản mạnh nhất với nội dung mà không đổi nền trang.

## Đã làm

**design-system (`@metap/ui`)** — chỉ thêm, không đổi giá trị mặc định (theme không khai báo token mới trông như cũ):
- Token mới: `success`/`warning`/`info` (+ foreground), `sidebar`, `sidebar-foreground`, `sidebar-accent`, `sidebar-accent-foreground`, `sidebar-primary`, `sidebar-border`, `chart-1..5`, `--font-sans`, `--font-mono`; sinh vào `globals.css` bằng `pnpm tokens:generate`, có bản tối. Preset Tailwind thêm màu `success`/`warning`/`info`/`sidebar`/`chart` và `fontFamily.sans`/`mono`; base áp `font-family: var(--font-sans)`.
- `Badge`: `success`/`warning` dùng token; thêm `info` và các biến thể "soft" (`success-soft`, `warning-soft`, `destructive-soft`, `info-soft`) cho bảng dày. `warning-soft` dùng màu mực vì chữ vàng trên nền sáng không đọc được.
- `Sidebar`: dùng token sidebar; mục đang chọn có vạch dọc bên trái (dấu vẫn thấy khi thu gọn chỉ còn icon). `StatTile` tone `success` và `BarChart`/`TimeSeries` (`--chart-1`) dùng token. Test cập nhật, 48 file test qua.

**metap-themes** — package mới `@metap/theme-waap`: bản sáng và tối, mọi cặp màu chính đạt ≥ 4.5:1 (đã tính, hai cặp cố tình để lớn-chữ: chữ `destructive` nhỏ trên nền tối 4.3:1). Màu tín hiệu (cyan) chỉ dùng cho dấu "đang ở đây", focus ring và hành động chính, **không bao giờ cho trạng thái**; trạng thái có màu riêng nên "chặn" đỏ và "cho qua" xanh không tranh với thương hiệu. Số dùng `tabular-nums`.

**platform-ui** — `AppShellLayout`: `brand` nhận `ReactNode` (logo + tên), thêm `brandLabel` cho nhãn truy cập; màu chữ của brand và nút trên thanh bên theo token sidebar (trước đây `text-foreground` sẽ là chữ tối trên nền tối); header top bar `bg-card` để vùng làm việc nổi lên so với trang.

**m-web** — logo (khiên + đường lưu lượng + điểm kiểm tra, `src/brand/Logo.tsx`), trang đăng nhập hai cột (panel mực có lưới mờ và vệt sáng cyan, tiêu đề, bốn trụ cột sản phẩm; dưới `lg` chỉ còn logo nhỏ trên form), favicon SVG, tiêu đề tab `WAAP Console`, chuyển từ theme enterprise sang `waap`, chuỗi i18n `waf.brand.*` (en, vi). Font: ngăn xếp hệ thống có `Inter`/`JetBrains Mono` đứng đầu; **không tải webfont** (không thêm phụ thuộc CDN, và ngăn xếp tự rơi về font hệ thống).

## Chỉnh chi tiết component: bớt vuông vắn (cùng ngày, theo phản hồi của chủ dự án)

Nguyên nhân "vuông vắn", tìm thấy trong code:
- **Popover, menu, tooltip, dialog hiện ra không có chuyển động.** Chúng dùng `transition` giữa hai trạng thái, nhưng một phần tử Radix được mount sẵn ở trạng thái mở nên không có trạng thái "trước" để chuyển; đóng thì Radix gỡ ngay vì không có animation để chờ. Giờ dùng keyframe (`pop-in`/`pop-out`, `dialog-in`/`dialog-out`, `fade-*`, `sheet-<cạnh>-in/out`) dùng thuộc tính `scale`/`translate` độc lập (không đè phép dịch căn giữa của dialog); popover, menu, tooltip và select phóng ra từ đúng điểm neo (`--radix-*-content-transform-origin`); lớp phủ mờ nhẹ (`backdrop-blur-[2px]`). `prefers-reduced-motion` tắt hết chuyển động.
- **Vòng focus cứng:** `ring-2` kèm `ring-offset-2` (khe trắng giữa phần tử và vòng). Đổi thành vầng sáng mềm `ring-[3px] ring-ring/30` và đường viền chuyển sang màu ring (`focus-visible:border-ring`) ở input, textarea, select; ô lỗi dùng `destructive/30`. Áp ở 17 component.
- **Bóng mặc định của Tailwind** (một lớp, đen). Thay bằng bóng nhiều lớp nhuốm màu mực qua `--shadow-color` (token mới, có bản tối): `shadow-xs/sm/md/lg`. Nút chính và nút nguy hiểm có `shadow-sm`, nổi lên `shadow-md` khi hover; input có `shadow-xs`; thẻ dùng `shadow-sm`.
- **Bo góc:** theme `waap` từ 0.375rem lên **0.625rem** (nút, input 8px, popover/menu/tab 10px, thẻ và dialog 14px); thẻ, dialog, command palette dùng `rounded-xl`, popover/menu/select `rounded-lg`, mục menu và tab `rounded-md`, mục thanh bên `rounded-lg`.
- **Tương tác:** nút có `transition` trên màu, viền, bóng và `transform`, ấn xuống dịch 1px; input hover đổi viền; công tắc bỏ bóng quá nặng của núm.

Làm nốt cùng ngày: `Tag` dùng token (info/success/warning/destructive/chart-2) thay `bg-*-100`; `Skeleton` quét sáng (`shimmer`) thay nhấp nháy; toast trượt vào (`toast-in`); đầu bảng có nền nhạt. `Chip` và `Alert` vốn đã dùng token (không có màu cứng như ghi trước đó). Cũng như phần trên, chưa ai xem trên trình duyệt: nên kiểm các lớp phủ (dialog, sheet, popover) khi mở và đóng, và vầng focus bằng bàn phím.

## Chưa làm / việc tiếp theo

- **Chưa xem bằng mắt.** Điều đáng nhìn kỹ nhất: tương phản thanh bên ở trạng thái hover/active, trang đăng nhập ở cỡ `lg` ngay ngưỡng, biểu đồ với `--chart-1` cyan.
- Dùng `font-mono` cho dữ liệu máy trong các trang (IP, id rule, request id, path, hostname): token và theme đã có, chưa áp vào từng bảng.
- Webfont `Inter`/`JetBrains Mono` tự host nếu muốn đúng chữ trên mọi máy.
- Mỗi trang của m-web chưa tự áp `font-mono`; cách đúng là một gợi ý hiển thị theo field (như `widget`) để danh sách sinh tự động tự chọn mono cho IP/id/path, chưa làm.
- Minh họa/trạng thái rỗng (`EmptyState`), màn hình lỗi và email giao dịch chưa mang nhận diện.
- Các theme khác trong metap-themes (`enterprise`, `fintech`, ...) chưa khai báo token mới; chúng chạy bằng giá trị mặc định, nên chỉ thanh bên và trạng thái trông như cũ.
