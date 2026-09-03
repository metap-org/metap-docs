## Phase 66: `metap-config` — tunable nền tảng phân tầng, lưu trong Postgres (2026-09-03)

Trigger: chủ dự án hỏi khi vá audit 04 A#1 — "những config như này có thể set trong database, đầu
API admin/config hay gì đó - per tenant". Feature brief đầy đủ ở
`../features/18-config-tiers-db-backed.md`; đây là **lát 1/3**.

## Điểm quan trọng nhất: tầng là thuộc tính của khoá, không phải của người gọi

Câu hỏi ban đầu rất hợp lý, nhưng đúng hai config làm nó nảy sinh
(`CRON_WEBHOOK_ALLOWED_HOSTS`, `CRON_WEBHOOK_ALLOW_PRIVATE_TARGETS`) lại là ví dụ **không được
phép** per-tenant, thậm chí không được phép qua bất kỳ API nào. Toàn bộ giá trị bản vá A#1 nằm ở
chỗ danh sách host do **operator** kiểm soát; một API config tiện tay sửa được nó thì gỡ luôn bản
vá, kẻ tấn công không cần khai thác gì cả.

Nên `ConfigLevel::Operator` không ghi được bởi `/platform/config` (platform admin) lẫn
`/admin/config` (tenant admin, lát 2). Platform admin **không phải** là operator — người thứ hai
là người nắm env/shell của deployment, và các khoá bảo mật trả lời cho người đó.

Ba khoá đó **vẫn được khai báo** trong `keys::REGISTRY` dù giá trị không bao giờ đọc từ DB: khai
báo là thứ cho phép write surface từ chối **đúng tên** với 403 thay vì im lặng nhận một khoá lạ, và
biến ranh giới thành thứ test được thay vì ngầm hiểu qua sự vắng mặt. Có 2 test giữ nó — 1 unit
(`the_security_critical_keys_are_operator_level`) và 1 e2e qua HTTP thật
(`an_operator_key_is_refused_even_for_a_platform_admin`), vì route mới là chỗ dễ bị nới nhất về sau.

## Không phải bảng key-value tự do

Mỗi khoá khai báo tier + default + validator. Khoá lạ bị từ chối chứ không lưu. Lý do: nền tảng này
bán đúng luận điểm "validation sinh từ metadata đã khai báo" — ship kèm một bảng config không
validate gì sẽ mâu thuẫn với chính nó và thành bãi rác trong vài tháng.

Bound của validator không phải trang trí: mọi khoá ở đây đều hỏng nặng ở 0 (`burst_size = 0` từ
chối *mọi* request; TTL = 0 mint token hết hạn sẵn), còn trần quá cao thì vô hiệu hoá đúng cái
guardrail mà setting đó tồn tại để cung cấp.

## Additive tuyệt đối

Default của mỗi khoá **bằng đúng** giá trị hardcode nó thay thế (10/1000, 200/300, 3600). Bảng rỗng
= hành vi y như trước khi có bảng. `AppState.config` khởi tạo ở trạng thái `unloaded` (vì
`AppState::new` là sync còn đọc bảng thì không) — host binary gọi `state.config.reload().await` một
lần trước `build_router`; **quên gọi là an toàn**, chỉ mất khả năng chỉnh runtime chứ không hỏng gì.

## Đọc không chạm database

`ArcSwap` snapshot nạp lúc boot, swap lại sau mỗi lần ghi — cùng hình dạng hot-swap `MetadataRegistry`
đã dùng. Đây là thứ khiến việc đọc limit từ middleware chạy mỗi request là an toàn.

## A#7 và ngoại lệ `graphql-gateway`

`graphql-gateway` **không có Postgres pool** (BFF thuần: không entity, không `CrudService`, không
DB) nên không có gì để đọc bảng đó. Nó đọc `GRAPHQL_MAX_DEPTH`/`GRAPHQL_MAX_COMPLEXITY` từ env —
vừa là cơ chế nó thực sự với tới được, vừa đúng nguyên văn điều A#7 yêu cầu ("không chỉnh qua env").
Service *có* pool đọc cùng 2 khoá đó từ `platform_configs`.

Một điểm trung thực trong response: `PUT /platform/config` trả `appliesImmediately`. Hai khoá rate
limit là `false` — chúng bị nướng vào `GovernorLayer` lúc `build_router` chạy nên cần restart. Nói
rõ còn hơn để người dùng ngồi debug một thay đổi "không ăn".

## Xác minh

`cargo test --workspace` 93/93 suite ok (từ 90 — thêm `metap-config` và test file mới), `clippy
--workspace --all-targets -D warnings` sạch, `fmt --check` sạch. **5 e2e test mới chưa chạy** —
phiên cloud không có Docker/Postgres, cần `cargo test --workspace -- --ignored`.

Disk quota hết 1 lần giữa phiên (`No space left on device` lúc build workspace), `cargo clean` giải
phóng 30.3GB. Lần thứ 3 trong ngày — xem `65-audit-04-fixes.md`.

## Còn lại

Lát 2: `tenant_configs` + theme + endpoint public theo hostname (theme phải hiện trên màn hình
login, lúc đó chưa có session nên không đi qua `/admin/config` được).
Lát 3: `SecretStore::get_secret` + `secretRef` + nới `FORBIDDEN_HEADERS` cho webhook secret — lát
dễ sai nhất, làm sau cùng.
