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

## Chưa làm / việc tiếp theo

- **Chưa xem bằng mắt.** Điều đáng nhìn kỹ nhất: tương phản thanh bên ở trạng thái hover/active, trang đăng nhập ở cỡ `lg` ngay ngưỡng, biểu đồ với `--chart-1` cyan.
- Dùng `font-mono` cho dữ liệu máy trong các trang (IP, id rule, request id, path, hostname): token và theme đã có, chưa áp vào từng bảng.
- Đổi hết các chỗ màu cứng còn lại của design-system (`Tag` có palette riêng, một số `bg-*-100` trong `Alert`/`Chip`) sang token trạng thái.
- Webfont `Inter`/`JetBrains Mono` tự host nếu muốn đúng chữ trên mọi máy.
- Minh họa/trạng thái rỗng (`EmptyState`), màn hình lỗi và email giao dịch chưa mang nhận diện.
- Các theme khác trong metap-themes (`enterprise`, `fintech`, ...) chưa khai báo token mới; chúng chạy bằng giá trị mặc định, nên chỉ thanh bên và trạng thái trông như cũ.
