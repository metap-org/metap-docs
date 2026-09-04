## Phase 72: `control-plane` + `edge-plane` — phần chặn request thật, lần đầu có code (2026-09-04)

Trigger: chủ dự án đọc mục "Còn lại" của Phase 71 ("`control-plane`/`edge-plane` vẫn 0 dòng code —
phần khiến sản phẩm là WAF thật chưa tồn tại") và bảo làm luôn. Cùng chế độ với Phase 70/71:
code-only, không build/test, chủ dự án tự kéo nhánh về kiểm tra.

Trước phase này `metap-demo-waf` là **portal cấu hình một thứ không tồn tại** — sửa `FirewallRule`
trên UI không có gì đọc, không có gì thực thi. Phase này đóng đúng khoảng đó.

## Quyết định kiến trúc đã chốt trong phiên (cần review kỹ)

**Telemetry `SecurityEvent` đi lên: chọn hướng 2 (edge → control-plane → data-plane).**
`data-plane/docs/04-architecture-boundary.md` ghi rõ "chưa chốt hướng nào" với 2 ứng viên, và
`CLAUDE.md` của repo dặn "đừng tự chọn, phải flag". Phiên này chủ dự án đã nói trước "không cần
hỏi quyết định, cứ như bro recommend", nên chọn hướng 2 và **flag ở đây + trong PR** thay vì chôn
trong code. Lý do: hướng 1 (edge gọi thẳng `metap-grpc`'s `RecordService.Create`) phá đúng nguyên
tắc "edge-plane không bao giờ nói chuyện trực tiếp với data-plane" ghi ở mọi chỗ khác trong thiết
kế, và đẩy N edge node × traffic thật thành write nhỏ lẻ vào DB của portal. Chính doc đó cũng đã
nghiêng về hướng 2. **Đảo lại rẻ**: edge chỉ biết đúng 1 URL ingest.

**Không dùng `metap-cache::RedisCache` để phân phối rule-set** (doc `04` có gợi ý dùng lại). Lý do
ghi trong `distribute.rs`: đó là *cache* — TTL cố định mỗi instance, key tự scope `{tenant_id}:{key}`
bởi chính crate. Cả hai đều sai ở đây. Dữ liệu này không phải cache: hết hạn = zone âm thầm mất bảo
vệ, và không có nguồn nào để đọc lại trên hot path. Còn key layout là **contract công bố** cho một
process cố ý không phụ thuộc `metap` — không thể là chi tiết nội bộ của 1 crate `metap`.

## `control-plane/` — `waf-config-distributor`

3 việc trong 1 process (chung 1 session `data-plane`, chung 1 pool Redis):

- **subscribe** — `run_resilient_consumer` (không tự viết reconnect loop), routing key
  `waf.*.record.*`, lọc 3 entity thật sự đổi hành vi edge. Đây là **đường nhanh, không phải đường
  đảm bảo**.
- **resync** — quét toàn bộ theo `RESYNC_INTERVAL_SECONDS`, cộng 1 lần lúc boot. Đây mới là thứ
  đảm bảo hội tụ: event luôn có thể mất (message vào DLQ, replica chết giữa chừng, ai đó sửa thẳng
  DB). Cũng là thứ duy nhất dọn được zone bị xoá mà event delete không tới.
- **ingest** — `POST /ingest/events`, buffer `mpsc` + batch, ghi xuống `data-plane` qua **đúng
  route CRUD thường** (`POST /api/waf.security_events`) nên validation/permission/outbox vẫn áp
  dụng đủ. Không phải cửa sau vòng qua `CrudService`.

Đọc config từ `data-plane` **qua REST API, không qua Postgres** — đọc thẳng DB sẽ khoá process này
vào layout `records`/JSONB của `metap` và bỏ qua permission. Auth bằng `ServiceTokenSource` (login
thật, tự refresh), không phải JWT mint tay — `metap` đã trả giá cho kiểu đó khi 1 token tĩnh hết
hạn giữa deployment đang chạy.

Compile quyết sẵn để edge khỏi quyết: zone `pending`/`suspended` không publish; rule/policy
disabled bị loại hẳn chứ không mang cờ; sort priority 1 lần; **rule có match condition không biểu
diễn được thì bị bỏ + log to**, không bao giờ publish với điều kiện yếu hơn (rule match rộng hơn
tác giả viết còn tệ hơn rule biến mất).

## `edge-plane/` — `waf-edge`

**Không có `metap` ở bất kỳ đâu trong cây này** — hyper 1.x trần, `ArcSwap` snapshot, dependency
list ngắn. Đường đi 1 request: Host → snapshot in-memory (không I/O) → clearance cookie → evaluate
(DDoS budget trước, rồi rule theo priority, first match wins) → block/challenge/log → allowed thì
proxy sang origin → non-allow thì queue SecurityEvent (không bao giờ chặn).

Điểm đáng soi:
- **Request không bao giờ chạm Redis.** Refresh chạy nền, chỉ fetch lại zone có `configVersion`
  đổi. Redis chết / payload lỗi / schema mới hơn build → **giữ snapshot cũ**, không bao giờ thành
  outage hay zone mất bảo vệ.
- **Monitor mode áp dụng ở bước cuối**: decision vẫn ghi lại verdict thật, request vẫn qua. Nếu
  control-plane rewrite action thành `Log` từ đầu thì monitor mode chỉ còn nghĩa là "tắt".
- **`CLIENT_IP_HEADER` mặc định KHÔNG bật.** Tin header client gửi lên = ai cũng giả được source
  IP, vượt mọi IP rule và mọi rate limit.
- **`X-Forwarded-For` gửi sang origin là ghi đè, không append** — append giá trị client gửi lên là
  cho phép giả mạo với mọi hệ thống phía sau tin header đó. Hop-by-hop header bị strip cả 2 chiều.
- Rate limit **fixed window, per-node** — sliding log là bộ nhớ không chặn do attacker điều khiển,
  đúng kiểu hỏng sai chỗ nhất cho component có việc là sống sót qua flood. Per-node nghĩa là fleet
  N node có budget thực tế ≈ N lần; ghi rõ trong README chứ không giả vờ chính xác.
- **Block/challenge page**: đóng luôn gap `docs/14-cloudflare-gap-analysis.md` đề xuất cho v1 (một
  trang mặc định, chưa customise per-zone).

## Đã verify (cập nhật 2026-09-04, phiên riêng — "bắt đầu viết test, build và verify các thay đổi")

Cả 5 chỗ tự nghi ở lần ghi đầu ("Chưa verify" gốc, giữ nguyên văn bên dưới để đối chiếu) đã có câu
trả lời thật:

1. **hyper 1.x/`hyper-util` legacy client, `proxy::forward`** — build sạch ngay lần đầu, không sửa
   gì. Đúng.
2. **`redis` 0.27 async API** — build sạch ngay lần đầu (cả `control-plane`'s `distribute.rs` lẫn
   `edge-plane`'s `cache.rs`). Đúng.
3. **`run_resilient_consumer`'s generic bounds** — build sạch; chỉ có 1 warning thật (unused
   import `EventBus`, không phải lỗi kiểu) do `clippy -D warnings` bắt được, đã xoá import thừa.
4. **`ip_in_cidr`** — viết 5 unit test riêng cho hàm này (`ip_in_cidr_matches_inside_the_prefix`,
   `_zero_prefix_matches_everything`, `_bare_address_is_exact_match`, `_malformed_cidr_never_matches`,
   `_ipv6_prefix_works`, `_mixed_families_never_match`) — tất cả pass, kể cả case gap đã biết
   (IPv4-mapped IPv6 không match rule IPv4 — assert đúng hành vi hiện tại, không phải baokhuất).
5. **Chưa từng `cargo build`** — giờ đã build/clippy/test sạch cả 2 workspace.

`cargo build --workspace`/`cargo clippy --workspace --all-targets -- -D warnings`/`cargo test
--workspace` cho cả `control-plane` và `edge-plane` đều xanh. Chi tiết lỗi/sửa thật tìm được:

- **`control-plane`**: 1 warning clippy (unused import `EventBus` trong `subscribe.rs`) — sửa.
  Thêm `.cargo/config.toml` nối vào target-dir chung + mold linker (đúng convention 5 repo Rust
  khác trong `metap-org`) — trước đó build vào `target/` cục bộ, không track được. Thêm 19 unit
  test cho `compile.rs`: `publishable` (chỉ `active`/`paused`), `action_from` (giá trị lạ luôn về
  `Log`, không bao giờ đoán thành `Block`), `parse_match` (predicate/all/any/not/in, và case không
  biểu diễn được → `None`), `compile_rule`/`compile_ddos` (rule/policy tắt bị loại hẳn),
  `compile_zone` (sort priority đúng, thiếu hostname thì fail).
- **`edge-plane`**: 5 warning `dead_code` (2 hàm thật sự không dùng — `body::empty()`,
  `proxy::error_body()`, xoá hẳn; field `host` của `RequestContext` trùng lặp với biến `main.rs`
  đã truyền riêng, xoá kèm comment giải thích; 4 field của `CompiledRule`/`CompiledZone` — dữ liệu
  thật trên wire nhưng chưa đọc ở logic hiện tại — giữ lại với `#[allow(dead_code)]` + comment
  thay vì xoá khỏi struct, vì vẫn cần cho `{:?}` log và tương thích ngược). Thêm `.cargo/config.toml`
  (chỉ mold linker, **cố ý không** trỏ shared target-dir — plane này gần như không overlap
  dependency với 5 repo kia). Thêm 35 unit test: 13 cho `evaluate()`/`eval_match`/`ip_in_cidr`
  (gồm case DDoS budget check trước rule, first-match-wins, rate-limit rule chỉ fire khi vượt
  ngưỡng, monitor mode hạ mọi action về `Log`), 6 cho `RateLimiter` (fixed-window reset, key độc
  lập), 9 cho `clearance` cookie (issue/verify round-trip, sai zone/ip/secret bị từ chối, giả mạo
  expiry bị từ chối nhờ expiry nằm trong phần đã ký, cookie hết hạn bị từ chối).

**Vẫn chưa verify**: chưa chạy thật qua HTTP/Redis/RabbitMQ sống (không có docker daemon trong môi
trường build này — thử `sudo service docker start` và `dockerd` đều bị chặn quyền), nên **chưa có
bằng chứng runtime nào** cho `subscribe.rs`/`resync.rs`/`ingest.rs`/`cache.rs`'s refresh loop hay
`main.rs`'s proxy pass-through thật — chỉ có build+clippy+unit test thuần logic. Test end-to-end
"đổi rule trên portal → 10-30s sau edge chặn thật" (mục tiêu cuối) vẫn chưa làm được vì lý do đó.

### Chưa verify (ghi lúc code, 2026-09-04 sớm hơn — giữ nguyên để đối chiếu với mục trên)

1. hyper 1.x + `hyper-util` legacy client: kiểu body (`BoxBody`/`Incoming`/`Full`) là chỗ dễ sai
   kiểu nhất trong cả phase, đặc biệt `proxy::forward`.
2. `redis` 0.27 async API (`smembers`/`get`/pipeline `query_async` turbofish) — viết theo trí nhớ
   API, chưa chạy.
3. `run_resilient_consumer`'s generic bounds với closure `connect` trả `RabbitEventBus` — kiểu
   closure ở đây khó chiều.
4. `ip_in_cidr` (tự viết, không dùng crate) — logic mask/prefix cần test thật; **IPv4-mapped IPv6
   không match rule IPv4**, đã ghi là gap thật.
5. Cả 2 workspace mới chưa từng `cargo build`; version dependency (`redis` 0.27, `hyper-util` 0.1
   feature set) chưa đối chiếu với registry.

## Còn lại

- Chưa có `docker-compose` cho 2 plane mới, chưa nối vào dev stack sẵn có.
- Chưa có Traefik/reverse-proxy thật (vẫn là việc còn treo từ Phase 61).
- Admin Portal backend (`Plan`/`Subscription`) vẫn chưa có.
- Origin health check định kỳ (gap thứ 2 mà `docs/14` đề xuất cho v1) chưa làm.
- Chưa có test end-to-end "đổi rule trên portal → 10-30s sau edge chặn thật" — đây là thứ chứng
  minh cả 3 plane nối đúng, và nên là việc đầu tiên sau khi build sạch. **Vẫn là việc quan trọng
  nhất còn lại** — build/unit-test sạch chỉ chứng minh logic từng phần đúng, không chứng minh 3
  plane nối với nhau đúng qua Redis/RabbitMQ/HTTP thật.
