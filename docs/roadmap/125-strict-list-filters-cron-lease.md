# 125. Bộ lọc list nghiêm ngặt, lease cho lượt chạy cron, job Direct chạy nền (2026-11-28)

Trạng thái (2026-11-28): đã sửa, test và merge (`metap`, `waap`, `platform-ui`).

## 1. Bộ lọc không hợp lệ bị bỏ qua

`PlanList` và `PlanAggregate` bỏ qua im lặng bộ lọc trên trường mà list view không khai trong `filters`. Bỏ một bộ lọc
làm kết quả **rộng hơn**. Code phía server lọc theo `domainId`, `customerTenantId`... rồi tin kết quả, nên đã thao tác
trên mọi bản ghi. Trong `waap`, đã tái hiện bằng test trên bản cũ:

| Chỗ gọi | Trường bị bỏ | Hậu quả |
|---|---|---|
| verify DNS của domain, cascade xuống zone | `zones.domainId` | Chứng minh **một** domain là mọi zone của tenant (tối đa 100) thành `verified`: vượt được kiểm tra quyền sở hữu |
| `PlanSource.Resolve` | `subscriptions.customerTenantId` | Khách chưa đăng ký nhận gói của khách khác |
| `MyPlanRequests` (đọc bằng quyền operator) | `planChanges.customerTenantId` | Khách thấy yêu cầu đổi gói của khách khác |
| kiểm tra yêu cầu đang chờ | như trên | Yêu cầu của khách này chặn khách khác |
| tài khoản mặc định | `accounts.slug` | Zone mới rơi vào tài khoản bất kỳ |
| đóng tài khoản | `zones.accountId` | Tài khoản không có zone chạy vẫn không đóng được |

**Sửa ở `metap`:** bộ lọc không được phép trả `400 validation_failed`, `fieldErrors` ghi tên trường (cả list lẫn
aggregate). **Ở `waap`:** khai thêm `domainId`, `accountId` (zones), `slug` (accounts), `customerTenantId`
(subscriptions, plan changes). **Ở `platform-ui`:** chỉ nhận từ URL những tham số là bộ lọc của list view; ô tham chiếu
chỉ lọc theo trường hiển thị khi entity đích khai trường đó, nếu không thì giữ trang đầu.

**Thay đổi phá vỡ:** client nào gửi bộ lọc chưa khai (kể cả tham số lạ trên URL của `GET /api/{entity}`) giờ nhận 400.
Job cron `bulk_query_action`/`workflow_transition` có bộ lọc chưa khai giờ **thất bại**; trước đây nó chạy trên mọi bản
ghi. Cần rà job đang có trước khi deploy. `metap-lowcode`, `metap-demo-jira` chưa được rà.

**Dữ liệu cũ:** trên môi trường đã chạy bản cũ, cần tìm zone `verified` mà domain của nó chưa `verified`:

```sql
SELECT z."hostname", d."apexDomain" FROM waf.waf_zones z LEFT JOIN waf.waf_domains d ON d.id = z."domainId"
WHERE NOT z.deleted AND z."verificationStatus" = 'verified'
  AND (d.id IS NULL OR d.deleted OR d."verificationStatus" IS DISTINCT FROM 'verified');
```

DB dev: 0 dòng.

## 2. Lượt chạy cron có lease

`StartRun` chỉ ghi `started_at`; việc "đã chạy chưa" là đọc trạng thái rồi mới chạy. Hai executor, hoặc một lần giao
lại trong lúc lượt đầu còn chạy, đều chạy job. `ClaimDueRetries` còn coi retry chạy quá 300 giây là chết và phát lại.

Thiết kế (migration `0061`: `lease_token`, `lease_expires_at`; status vẫn `enqueued` khi đang chạy):

- `cron.ClaimRun`: một câu `UPDATE` có điều kiện; giờ lấy từ database. Kết quả: nhận được, đã xong (hoặc đang chờ
  `wait_event`, hoặc đã xoá), đang có người giữ.
- Executor gia hạn mỗi 1/3 lease (mặc định 1 phút) trong lúc job chạy. Mất lease (người khác nhận, hoặc không gia hạn
  được suốt một lease) thì huỷ context của job.
- Chỉ người giữ token ghi được kết quả; kết quả và retry nằm trong một transaction.
- Message của lượt đang có người giữ, hoặc khi claim/ghi kết quả lỗi, được `Defer` 15 giây rồi giao lại, không ack bỏ.
  `Defer` không đếm lượt: vòng lặp tự dừng vì lượt chạy sẽ xong, hoặc lease hết hạn và lần giao đó nhận được việc.
- `ClaimDueRetries` không đụng retry còn lease.

**Đảm bảo là ít nhất một lần, không phải đúng một lần.** Worker chết sau khi gọi đích nhưng trước khi ghi kết quả thì
đích bị gọi lại; đích cần chịu được lời gọi lặp cho cùng `runId`. Huỷ context không dừng được email SMTP đang gửi.

## 3. Job Direct và chain được resume chạy nền

Trước đây chúng chạy đồng bộ trong consumer trigger (và ticker): một webhook chậm làm nghẽn mọi trigger phía sau.
Giờ chúng chạy trong goroutine, tối đa `CRON_DIRECT_CONCURRENCY` (mặc định 8) trên cả tiến trình; hết slot thì vòng lặp
đang gọi chờ. Khi shutdown, mỗi vòng lặp chờ việc của nó ghi xong kết quả.

- Direct vẫn là fire-and-forget: tiến trình chết thì lượt đầu không được chạy lại (retry thì được ticker nhận lại).
  Việc không được phép mất dùng `outbox`.
- Chain được resume đã rời trạng thái `waiting`; tiến trình chết giữa chừng thì nó kẹt ở `running` (như trước).
- Đổi hành vi: resume chạy **trước** job Direct của cùng sự kiện, nên chain do sự kiện đó tạo ra không còn bị chính
  sự kiện đó đánh thức.

## 4. Deploy

Chạy migration `0061` trước khi khởi động scheduler mới. Queue mới `cron.executor.retry.1` (queue chính giữ nguyên
tham số). Trong lúc rolling, scheduler cũ không tôn trọng lease, nên dừng hết bản cũ trước khi bật bản mới. `waap`,
`platform-ui` cần đi cùng `metap`: `waap` cũ chạy trên `metap` mới sẽ nhận 400 ở các chỗ gọi ở §1.
