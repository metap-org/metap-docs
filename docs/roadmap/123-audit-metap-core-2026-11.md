# 123. Audit metap core (2026-11-27)

Đợt audit đầu sau roadmap 122, tập trung `metap` core: quy ước, bảo mật bề mặt, tính đúng đắn (atomicity, audit, outbox, consumer
event, rollback). Mỗi mục dưới đây có test hồi quy; trạng thái ghi theo từng mục.

**Trạng thái (2026-11-27): đã sửa 4 lỗi trong metap; còn 3 mục nhỏ và 1 mục của `waap` (§3).**

## 1. Đã sửa (metap)

| # | Lỗi | Cách sửa | Test |
|---|---|---|---|
| 1 | `/metrics` công khai gắn nhãn `route` bằng URL path thô khi không khớp route: quét URL tạo series vô hạn (50 đường dẫn → 50 series) | request không khớp route (404, 405) dùng chung `route="unmatched"` (`httpmw.UnmatchedRoute`) | `TestUnknownPathsShareOneSeries` |
| 2 | `ResetMfa`, `Invite`, `CreateServiceAccount` đổi dữ liệu trước rồi ghi audit best-effort, bỏ qua lỗi: lỗi ghi để lại thay đổi không dấu vết | chạy qua `Change`/`ChangeNew`; hoàn tác member bằng `undoMember` (ghi vào trail, log nếu hoàn tác lỗi) | `TestMembershipChangesLeaveTheirAuditEntry` (chỉ chống hồi quy, chưa chứng minh atomic bằng audit store lỗi) |
| 3 | `Sink.Atomic`/`AtomicShared` chỉ rollback khi `work` trả lỗi; `work` panic để transaction mở, rò một kết nối pool và khoá | `defer unit.Rollback(context.WithoutCancel(ctx))` (no-op sau Commit) | `TestAPanickingStepReleasesItsConnection` (treo trước khi sửa) |
| 4 | Event không có id ổn định; `fireMatchedJobs` tạo run mới mỗi lần nhận event, nên event giao lại (worker chết trước ack, ack lỗi, batch outbox publish lại) bắn job hai lần | id dòng outbox làm AMQP `MessageId` (`eventbus.WithMessageID`, giữ khi retry); `ConsumedEvent.MessageID()`; cột `cron_job_runs.trigger_event_id` + unique `(job_id, trigger_event_id)` (migration 0060); `ON CONFLICT DO NOTHING` | `TestARedeliveredEventDoesNotFireTheJobAgain` |

Lưu ý về #4: job `Direct` mà lần giao đầu chết trước khi chạy sẽ không được chạy ở lần giao lại (chế độ này vốn là fire-and-forget); job
`Outbox` không mất vì event `cron.job.due` ghi cùng transaction. `MessageId` đi qua RabbitMQ (publish → consume → retry) chưa kiểm chứng
đầu-cuối vì repo chưa có test chạy với broker; phần cơ sở dữ liệu đã có test.

## 2. Đã kiểm tra, không phải lỗi

Không còn `json.NewDecoder(r.Body)` ngoài `httpx`, không có `chi`; mọi `io.ReadAll` ở đường mạng có giới hạn; SQL ghép chuỗi chỉ dùng
định danh đã quote/validate (`attachments` validate tập trung); route nhạy cảm đều qua `authGuard`; `/auth/mfa/setup/confirm` có giới hạn
số lần thử theo challenge; 54 chỗ `Begin` đều có rollback (hoặc là hàm bọc trả `tx` cho caller đã có).

## 3. Còn mở

- **`ResendInvitation`, `ssoconfig.VerifyDomain`** vẫn ghi audit sau khi đổi dữ liệu (việc gồm gửi email hoặc tra DNS, bọc vào transaction
  sẽ giữ kết nối DB khi gọi ra ngoài); cần chọn cách tách.
- **`StartRun`** chỉ `UPDATE started_at`, không chiếm quyền chạy theo điều kiện: cùng một message giao lại khi lần đầu còn chạy có thể chạy
  song song. Chiếm quyền có điều kiện đổi sang at-most-once nếu worker chết giữa chừng, nên cần thêm thời hạn chiếm quyền hết hạn.
- **Job `Direct`** chạy ngay trong handler consumer: webhook chậm chặn hàng đợi.
- **`waap` `e-edge`** (`CLIENT_IP_HEADER=X-Forwarded-For` lấy phần tử đầu, client tự khai được): sửa bằng `TRUSTED_PROXIES` + `httpx.ClientIP`;
  chờ chủ dự án quyết.
- Nit: `_ = ConsumeChallenge` trong `signin/mfa.go`; `_ = delivery.Nack` ở `rabbit.go`; nhánh `fallback` của `kernel.Change` không còn
  đường nào dùng.
