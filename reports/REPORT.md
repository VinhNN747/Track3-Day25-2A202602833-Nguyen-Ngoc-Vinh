# Lab 25B — Report

**Họ tên / nhóm:** Nguyễn Ngọc Vĩnh · **Commit:** e2636f9f4872d98d5cc3b59ee2f3fda1ba069432 · **Điểm tự động (`make grade`):** 84 / 88

## 1. Kết quả Game Day

| Băng | Client | availability | p95 (s) | amplification | retry ratio | cost/turn | recovery (s) | PASS? |
|---|---|---:|---:|---:|---:|---:|---:|:---:|
| G1 | naive | 0.7194 | 7.157 | 7.738 | 6.738 | 2.163 | 4 | — |
| G1 | resilient | 1.0000 | 4.780 | 1.063 | 0.001 | 2.104 | 0 | PASS |
| G2 | naive | 0.4560 | 15.034 | 9.904 | 8.904 | 4.627 | 155 | — |
| G2 | resilient | 1.0000 | 10.362 | 1.036 | 0.005 | 2.108 | 0 | PASS |
| G3 | naive | 1.0000 | 4.768 | 1.164 | 0.164 | 2.328 | 0 | — |
| G3 | resilient | 1.0000 | 4.668 | 1.059 | 0.030 | 2.143 | 0 | PASS |
| G4 | naive | 1.0000 | 13.764 | 1.004 | 0.004 | 2.008 | 0 | — |
| G4 | resilient | 1.0000 | 5.626 | 1.014 | 0.002 | 2.072 | 0 | PASS |
| G5 | naive | 0.3699 | 5.359 | 17.030 | 16.030 | 2.124 | — | — |
| G5 | resilient | 0.8179 | 5.477 | 1.003 | 0.002 | 2.504 | — | PASS |
| G6 | naive | 0.5170 | 4.836 | 12.344 | 11.344 | 2.009 | 0 | — |
| G6 | resilient | 1.0000 | 6.025 | 1.056 | 0.019 | 2.188 | 0 | PASS |

ResilientClient đạt 6/6 tape. Ở G5, 413 request malformed được trả `client_error` đúng cách; availability 81.79% phản ánh giới hạn không thể sửa request sai ở phía client.

## 2. Cấu hình client và lý do

| Tham số | Giá trị | Lý do |
|---|---:|---|
| attempt_timeout_s | 8.0 s | Đủ cho p99 bình thường nhưng dành thời gian cho failover trước deadline 30 s. |
| max_attempts_per_provider | 2 | Tối đa một retry cùng provider, tránh khuếch đại; amplification cao nhất quan sát được chỉ 1.063. |
| backoff_base_s / cap_s | 0.25 / 4.0 s | Full jitter, cap ngắn để không đốt deadline; tôn trọng `retry_after` cho rate limit. |
| retry_budget_ratio | 0.1 | Giới hạn retry khoảng 10% traffic; G3 đạt retry ratio 0.030. |
| breaker.window_size / min_calls | 20 / 10 | Đủ mẫu để tránh trip vì vài lỗi lẻ, vẫn phản ứng nhanh với outage. |
| breaker.failure_rate_threshold | 0.5 | Mở khi ít nhất một nửa kết quả gần đây là lỗi health-related. |
| breaker.slow_call_s | 5.0 s | Phát hiện G4 brown-out trước khi timeout tích tụ. |
| breaker.open_duration_s | 10.0 s | Cooldown ngắn và HALF-OPEN probe có giới hạn 2 request. |

Quyết định `on_error`:

```text
400/401/403/404/422 -> release breaker -> GiveUp(client_error)
429 insufficient_quota -> release -> cooldown dài -> failover
429 rate_limit_exceeded -> release -> cooldown theo retry_after -> failover
503/529/5xx/timeout -> on_failure breaker
  nếu còn attempt, đủ retry budget, đủ deadline -> retry cùng provider với full jitter
  nếu không -> provider tiếp theo trong ladder
tool -> chỉ alpha, alpha-eu, beta; chat -> thêm gamma-mini
```

## 3. Ba thí nghiệm

1. Retry budget là lớp bảo vệ cuối cùng. Khi breaker bị cấu hình không trip và số attempt tối đa tăng, bật budget giữ retry ratio thấp; tắt budget làm amplification tăng nhanh vì mỗi lỗi được nhân lên ở nhiều lớp. Breaker vẫn cần thiết vì nó ngăn gửi request mới tới provider đang hỏng, còn budget bảo vệ hệ thống trong thời gian breaker chưa đủ mẫu hoặc khi lỗi phân tán.

2. Timeout 8 s cho G6 giúp chuyển sang provider khác sớm; naive chỉ đạt 51.70% availability và amplification 12.344. Timeout 30 s sẽ giữ request treo lâu hơn, làm giảm thời gian còn lại để failover và tăng lỗi deadline. G4 có lỗi ít nhưng response chậm, nên timeout dài không giải quyết được p95; slow-call breaker và route sang vùng khác mới đưa p95 resilient xuống 5.626 s.

3. Lọc capability trong `ladder()` ngăn tool call tới `gamma-mini`, model không hỗ trợ tool calling. Trong G6, resilient có `incompatible_routes = 0`; với product manager, điều này nghĩa là fallback nhanh nhưng vẫn giữ đúng chức năng, không biến lỗi provider thành câu trả lời “thành công” nhưng không thể thực thi hành động.

## 4. Workflow hoàn tiền exactly-once

| Executor | completion | crashes | duplicate refunds | duplicate emails | lost refunds | overpaid | wasted LLM / crash |
|---|---:|---:|---:|---:|---:|---:|---:|
| naive | 1.0000 | 81 | 128 | 81 | 0 | 1,496,700¢ | 1.123 |
| yours | 1.0000 | 66 | 0 | 0 | 0 | 0¢ | 0.303 |

Idempotency key một mình chưa đủ vì worker có thể crash trước khi ghi nhận checkpoint, khiến bước trước đó phải chạy lại; checkpoint một mình cũng chưa đủ vì retry payment với key mới vẫn tạo giao dịch thứ hai. Thứ tự đúng là chạy step trước, sau khi thành công mới ghi checkpoint; ghi trước payment sẽ tạo “checkpoint giả”, crash hoặc lỗi sau đó có thể làm mất refund thật.

## 5. Alerting và error budget

Burn-rate alerter đạt public SLO dataset: 3 page, 1 ticket, 0 false page. Kết quả theo nhãn: `checkout_bug` page sau 27 phút, `provider_outage` sau 4 phút, `slow_leak` ticket sau 1337 phút, `degraded_evening` page sau 291 phút; deploy blip, night spike và small bump không page.

Error budget 30 ngày:

```text
requests       = 25,901,755
allowed_bad    = 129,508.8
bad            = 55,618
availability   = 0.997853
budget used    = 42.95%
remaining      = 57.05%
freeze release = false
```

`small_bump` chỉ kéo dài 20 phút ở mức 12% lỗi không nên đánh thức on-call lúc 11 giờ trưa: cửa sổ ngắn không đủ chứng minh burn rate dài hạn, và đánh thức sai làm tăng alert fatigue.

## 6. Postmortem — G2 với client naive

Hình: [G2_overload_storm.png](gameday/G2_overload_storm.png). Dữ liệu: `gameday/naive_G2_overload_storm_timeseries.csv`.

**Tóm tắt:** G2 làm alpha mất 75% capacity trong 180 giây. NaiveClient tiếp tục retry cùng alpha với nhiều tầng retry, tạo 34,475 attempt cho 3,481 logical calls và chỉ 45.60% user turn thành công. Sau khi alpha hồi phục, hàng đợi do retry vẫn kéo recovery tới 155 giây.

**Ảnh hưởng:** 1,285/2,362 turn thất bại; amplification 9.904, cost/turn tốt 4.627 và recovery 155 s. Nếu tháng chỉ có incident này, error budget 0.5% bị vượt rất lớn: 1,285 lỗi trên 2,362 turn tương đương khoảng 54.4% lỗi trong population của tape, cao hơn budget khoảng 108.8 lần.

**Timeline:**

| t | Sự kiện | Bằng chứng |
|---:|---|---|
| 120 s | Alpha mất 75% capacity | Scenario G2 capacity factor 0.25 |
| 120–300 s | Queue overload, 529/timeout và retry chồng lớp | 3,194 `overloaded_error`, 2,737 timeout, 28,381 retry-after violations |
| 300 s | Alpha hồi phục phía provider | Scenario restore point |
| 300–455 s | Hệ thống vẫn suy giảm vì request cũ tiếp tục chiếm slot | recovery_s = 155 s |
| 455 s | 95% cửa sổ 30 s thành công | metrics `recovery_s` |

**Trigger vs root cause:** Trigger là capacity alpha giảm. Root cause hệ thống là retry storm: app, gateway và SDK cùng retry, không breaker, không retry budget và không tôn trọng `retry_after`; các request client timeout vẫn tiếp tục chạy ở server, tạo vòng lặp tự duy trì. Provider hồi phục nhưng slot vẫn bị lấp bởi work cũ nên hệ thống không hồi phục ngay.

**Điều giúp / điều làm tệ hơn:** Giới hạn attempt, full jitter, breaker theo slow/error rate, cooldown và fallback theo capability giúp giảm tải alpha. Retry cố định nhiều tầng, timeout dài và retry sớm trước `retry-after` làm tệ hơn.

| # | Hành động | Loại | Owner | Cách kiểm chứng |
|---|---|---|---|---|
| 1 | Chuẩn hóa retry thành một tầng, full jitter + retry budget | prevent | Agent platform | G1/G2 amplification ≤ 1.6 |
| 2 | Bật sliding-window breaker và giới hạn HALF-OPEN probe | mitigate | Runtime | recovery ≤ 30 s trên G2 |
| 3 | Dashboard retry-after violations, queue wait và primary share | detect | SRE | alert khi violation > 0 hoặc queue tăng |

## 7. Một điểm yếu còn lại

ResilientClient vẫn có p95 G2 là 10.362 s, cao hơn đáng kể so với bình thường, và G5 tốn 2.504 cost/turn vì beta phải gánh traffic sau quota. Bước cải thiện tiếp theo là admission control theo tenant/provider và một static degraded response cho các request không cần side effect, đồng thời đo riêng latency của failover để tránh che khuất tail latency.

## 8. Cache — Part G

| Client | availability | faq_availability | cache_hit_ratio | stale_served | wrong_answers | outdated_answers | cost/turn | PASS? |
|---|---:|---:|---:|---:|---:|---:|---:|:---:|
| resilient (không cache) | 0.8373 | 0.8424 | 0.000 | 0 | 0 | 0 | 1.386 | — |
| cached | 0.9124 | 0.9645 | 0.652 | 157 | 0 | 3 | 0.915 | PASS |

| Tham số `cache:` | Giá trị | Vì sao |
|---|---:|---|
| similarity_threshold | 0.93 | Loại near-miss khoảng 0.86 nhưng vẫn nhận paraphrase cùng intent; C1 có 0 wrong answer. |
| ttl_s | 60 | Giới hạn câu trả lời cũ, đặc biệt silent shipping update; C1 chỉ có 3 outdated answer. |
| max_stale_s | 300 | Bao phủ outage 240–360 s để FAQ vẫn có thể trả lời stale-if-error; đạt 157 stale served và FAQ availability 96.45%. |

Ở threshold 0.80, cache tạo 181 `other_question` wrong answers; threshold 0.93 loại lỗi này. TTL ngắn là cơ chế duy nhất giới hạn silent update vì không có webhook. Với `max_stale_s = 0`, FAQ availability trong outage giảm về mức provider-only; stale-if-error là phần giúp dashboard vẫn phục vụ FAQ nhưng vẫn cần theo dõi provider health riêng. Personal question không được cache vì embedding gần nhau giữa user khác nhau và câu trả lời chứa dữ liệu tài khoản; scope theo user vẫn dễ sai nếu key/embedding bị dùng nhầm, nên allowlist loại hoàn toàn.

Trong outage, dashboard nên hiển thị hai tín hiệu: `faq_availability/cache_hit_ratio` cho trải nghiệm người dùng và provider error rate/`stale_served` cho sức khỏe backend. Cache làm UX xanh không có nghĩa provider đang khỏe.

