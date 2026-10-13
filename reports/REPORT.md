# Lab 25B — Report

**Họ tên / nhóm:** Ngụy Quang Hùng (MSSV: 2A202602998)  **Commit:** `b3ed9a4`  **Điểm tự động (`make grade`):** 84 / 88

> Mọi nhận định trong báo cáo đều được trích xuất trực tiếp từ các file số liệu thực nghiệm: `reports/gameday/*.json`, `reports/workflows.json`, `reports/slo.json`, `reports/cache/*.json` và `reports/gameday/naive_G2_overload_storm_timeseries.csv`.

---

## 1. Kết quả Game Day (naive vs của bạn)

| Băng | Client | availability | p95 (s) | amplification | retry_ratio | cost/turn | recovery (s) |  PASS?  |
| ----- | ------ | -----------: | ------: | ------------: | ----------: | --------: | -----------: | :-----: |
| G1    | naive  |       0.7194 |   7.157 |         7.738 |       6.738 |     2.163 |          4.0 | ❌ FAIL |
| G1    | yours  |       1.0000 |   4.776 |         1.062 |       0.002 |     2.101 |          0.0 | ✅ PASS |
| G2    | naive  |       0.4560 |  15.034 |         9.904 |       8.904 |     4.627 |        155.0 | ❌ FAIL |
| G2    | yours  |       1.0000 |  11.217 |         1.044 |       0.005 |     2.129 |          0.0 | ✅ PASS |
| G3    | naive  |       1.0000 |   4.768 |         1.164 |       0.164 |     2.328 |          0.0 | ❌ FAIL |
| G3    | yours  |       1.0000 |   4.718 |         1.065 |       0.035 |     2.146 |          0.0 | ✅ PASS |
| G4    | naive  |       1.0000 |  13.764 |         1.004 |       0.004 |     2.008 |          0.0 | ❌ FAIL |
| G4    | yours  |       1.0000 |   5.684 |         1.013 |       0.002 |     2.069 |          0.0 | ✅ PASS |
| G5    | naive  |       0.3699 |   5.359 |        17.030 |      16.030 |     2.124 |         None | ❌ FAIL |
| G5    | yours  |       0.8179 |   5.477 |         1.003 |       0.002 |     2.504 |         None | ✅ PASS |
| G6    | naive  |       0.5170 |   4.836 |        12.344 |      11.344 |     2.009 |          0.0 | ❌ FAIL |
| G6    | yours  |       1.0000 |   5.958 |         1.053 |       0.019 |     2.203 |          0.0 | ✅ PASS |

---

## 2. Cấu hình client và lý do

| Tham số                                |  Giá trị | Vì sao (dựa trên băng nào, số liệu nào)                                                                                                                                                                            |
| --------------------------------------- | ---------: | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `attempt_timeout_s`                   |        8.0 | Median latency của alpha là 1.2s, p99 bình thường ~3.5s. Đặt 8.0s giúp phát hiện treo ở G6 (alpha-eu hang) mà vẫn đủ dư thời gian để failover sang provider khác trước deadline 30s của turn.       |
| `max_attempts_per_provider`           |          2 | Tối đa 1 lần thử lại trên cùng provider. Ở G2 và G3, giới hạn này ngăn việc spam thử lại cục bộ khi provider đang suy kiệt.                                                                            |
| `backoff_base_s` / `cap_s`          | 0.25 / 4.0 | Base 0.25s giúp retry nhanh đối với transient error; cap 4.0s bảo đảm thời gian retry không ngốn hết deadline còn lại của turn.                                                                              |
| `retry_budget_ratio`                  |        0.1 | Khống chế tổng số retry không vượt quá 10% lưu lượng ban đầu. Giúp vượt qua check`retry_ratio_max: 0.10` ở băng G3 (`retry_ratio` thực tế là 0.035).                                              |
| `breaker.window_size` / `min_calls` |    20 / 10 | Cửa sổ trượt 20 calls với tối thiểu 10 calls giúp breaker có đủ ý nghĩa thống kê, tránh nhảy trạng thái sai khi lưu lượng thưa thớt vào ban đêm.                                                |
| `breaker.failure_rate_threshold`      |        0.5 | Ngắt mạch khi tỷ lệ lỗi$\ge 50\%$. Ở G3 tỷ lệ lỗi của alpha là 30% (< 50%) nên breaker không ngắt nhầm (để retry budget xử lý); ở G2 và G6 tỷ lệ lỗi cao ngắt mạch kịp thời.                 |
| `breaker.slow_call_s`                 |        5.0 | Ở băng G4, alpha bị chậm thêm 5.0s (tổng latency ~6.5s-7.0s). Ngưỡng 5.0s giúp breaker phát hiện suy giảm hiệu năng không có mã lỗi và chuyển mạch sang alpha-eu (p95 giảm từ 13.76s xuống 5.68s). |
| `breaker.open_duration_s`             |       10.0 | Sau khi mở, chờ 10s cho provider hạ nhiệt trước khi gửi probe. Ở G2 và G6 giúp`recovery_s = 0.0s` ngay khi restore.                                                                                            |
| `breaker.half_open_max_probes`        |          2 | Chỉ cho phép 2 request thăm dò đồng thời ở trạng thái HALF_OPEN, loại bỏ hiện tượng thundering herd đè sập lại provider vừa gượng dậy.                                                              |

### Sơ đồ quyết định của `on_error`

```
                         [ Nhận ProviderError ]
                                   │
                         Phân loại: classify(err)
                                   │
         ┌─────────────────────────┼─────────────────────────┐
         ▼                         ▼                         ▼
  CLIENT_ERROR             QUOTA_EXHAUSTED /           OVERLOADED /
  (400, 401, 422)            RATE_LIMITED           SERVER_ERROR / TIMEOUT
         │                         │                         │
  Breaker release           Breaker release          Breaker on_failure
         │                         │                         │
   GiveUp ngay           Cooldown provider:          Kiểm tra điều kiện retry:
(Không retry,            • QUOTA: 300s               • tries < max_attempts (2)
 không failover)         • RATE: retry_after         • Breaker != OPEN
                                   │                 • Fits deadline (now+delay+1 <= dl)
                          Failover ngay lập tức      • Budget try_acquire_retry == True
                          (delay = 0 sang ladder)            │
                                                      ┌──────┴──────┐
                                                    ĐỦ              THIẾU
                                                     │               │
                                              Retry SAME      Failover ngay
                                              với backoff     (chuyển index,
                                              delay           delay = 0)
```

---

## 3. Ba thí nghiệm (mỗi cái 3–5 câu + số)

1. **Retry budget là lớp phòng thủ cuối:**

   - Khi giả lập breaker cấu hình sai (`min_calls: 1_000_000_000`) và `max_attempts_per_provider: 5`:
     - **Băng G2:** Khi có retry budget, `amplification = 1.382`, `retry_ratio = 0.058`, `recovery_s = 0.0s`, `availability = 99.96%`. Khi tắt retry budget, bão retry lập tức bùng nổ: `amplification = 1.672`, `retry_ratio = 0.491` (gần 50% traffic là retry vô vọng), `recovery_s = 30.0s`, và `availability` tụt xuống `96.49%`.
     - **Băng G3:** Có budget giữ `retry_ratio = 0.067` ($\le 10\%$). Tắt budget khiến `retry_ratio = 0.162` (vi phạm SLA).
   - *Kết luận:* Ngay cả khi circuit breaker bị vô hiệu hóa hoặc cấu hình sai, retry budget vẫn đóng vai trò là chốt chặn an toàn cuối cùng khống chế lượng request khuếch đại không bao giờ vượt quá tỷ lệ định sẵn (10%). Do đó trong thực tế luôn cần kết hợp cả hai lớp: breaker để ngắt mạch chuyển hướng, budget để giới hạn tải khuếch đại.
2. **`attempt_timeout_s: 30`:**

   - Chạy trên **G6** (alpha-eu bị treo hoàn toàn): `availability` giảm từ `1.0` xuống `0.9488`. Nguyên nhân thể hiện ở `give_up_reasons = {'deadline': 108, 'no_provider_available': 14}` — có 108 request bị treo trọn 30s ở alpha-eu và cạn kiệt deadline của turn người dùng mà không còn thời gian failover sang beta. Trong khi đó, `latency_p95_s` chỉ đạt `5.298s` (nhìn có vẻ "nhanh" hơn 5.958s của timeout=8s) do survivorship bias: các lượt treo 30s bị fail đã bị loại khỏi tập tính p95.
   - Chạy trên **G4**: `availability = 1.0` và `p95 = 5.683s` gần như không đổi so với cấu hình chuẩn (5.684s), vì ở G4 alpha chỉ chậm +5s (latency ~6.5s) và circuit breaker đã phát hiện qua `slow_call_s: 5.0` để cắt mạch chuyển sang alpha-eu từ sớm, không phụ thuộc vào timeout 30s.
   - *Kết luận:* Timeout chuẩn không được đặt theo deadline của user mà phải tính bằng: `p99 bình thường + buffer` sao cho đủ nhỏ để hệ thống kịp failover qua ít nhất 2 provider dự phòng trước khi chạm deadline tổng.
3. **Bỏ lọc tool calling khỏi `ladder()`:**

   - Chạy trên **G6**: Xuất hiện `incompatible_routes = 244` và `availability` sụt giảm từ `1.0` xuống `0.8977`.
   - *Góc nhìn Product Manager:* "Khi hạ tầng chính của Provider A gặp sự cố, hệ thống định tuyến thiếu thông minh đã chuyển nhầm 244 yêu cầu cần thao tác nghiệp vụ (như kiểm tra đơn hàng, kích hoạt hoàn tiền) sang mô hình giá rẻ `gamma-mini` — một mô hình hoàn toàn không có khả năng gọi tool. Kết quả là 244 khách hàng nhận được câu trả lời lỗi hoặc chatbot ảo tưởng trả lời lòng vòng mà không thực thi được hành động thực tế nào, làm hỏng trực tiếp tỷ lệ hoàn thành tác vụ của sản phẩm."

---

## 4. Workflow hoàn tiền

| Executor | completion | crashes | duplicate refunds | duplicate emails | lost refunds |   overpaid | wasted LLM / crash |
| -------- | ---------: | ------: | ----------------: | ---------------: | -----------: | ---------: | -----------------: |
| naive    |        1.0 |      81 |               128 |               81 |            0 | $14,967.00 |              1.123 |
| yours    |        1.0 |      66 |                 0 |                0 |            0 |      $0.00 |              0.303 |

- **Vì sao chỉ idempotency key là chưa đủ, và chỉ checkpoint cũng chưa đủ?**
  - Nếu chỉ có *idempotency key* mà không có *checkpoint*: Khi worker crash, tiến trình mới khởi động lại từ đầu không nhớ tiến độ đã đi đến đâu, buộc phải gọi lại toàn bộ các bước LLM phân loại và tóm tắt, gây lãng phí chi phí token (`wasted_llm_per_crash` cao) và tăng độ trễ.
  - Nếu chỉ có *checkpoint* mà không có *idempotency key*: Nếu worker crash *sau khi* API thanh toán đã trừ tiền nhưng *trước khi* kịp ghi checkpoint vào database, worker mới chạy lại sẽ thấy bước thanh toán chưa có checkpoint và gọi thanh toán lại $\rightarrow$ gây hoàn tiền trùng lặp (duplicate refund).
  - Cần cả hai để tạo ra ngữ nghĩa **Exactly-Once Effects**: Idempotency key bảo vệ tầng side-effect ở provider; Checkpoint bảo vệ tiến trình nội bộ và giảm thiểu re-computation.
- **Điều gì xảy ra nếu ghi checkpoint *trước* khi gọi payment?**
  - Nếu ghi checkpoint trước khi gọi payment, trong trường hợp worker bị crash ngay lúc chuẩn bị gọi payment (hoặc mạng đứt lúc gửi request), worker mới khởi động lại sẽ đọc thấy checkpoint và tin rằng tiền đã được hoàn, từ đó nhảy cóc qua bước gửi email $\rightarrow$ dẫn đến **Lost Refund** (khách hàng nhận thông báo hoàn tiền nhưng tài khoản thực tế không nhận được đồng nào).

---

## 5. Alerting & error budget

| Sự cố          | Mong đợi | Naive alerter        | Burn-rate alerter     | Độ trễ phát hiện |
| ---------------- | ---------- | -------------------- | --------------------- | --------------------: |
| checkout_bug     | page       | paged (1m)           | paged (start: 3507)   |                27 min |
| provider_outage  | page       | paged (0m)           | paged (start: 8764)   |                 4 min |
| deploy_blip      | none       | **false page** | không page           |                    — |
| slow_leak        | ticket     | không phát hiện   | ticket (start: 22037) |             1,337 min |
| degraded_evening | page       | paged (214m)         | paged (start: 30291)  |               291 min |
| night_spike      | none       | **false page** | không page           |                    — |
| small_bump       | none       | **false page** | không page           |                    — |

- **Error budget 30 ngày:**
  - Tổng số requests: 25,901,755
  - Lỗi cho phép (0.5% ở SLO 99.5%): 129,508.8 requests
  - Số lỗi thực tế: 55,618 requests
  - Ngân sách lỗi đã tiêu: **42.95%**; Ngân sách còn lại: **57.05%**.
  - Quyết định: **Không đóng băng release** (`freeze_releases = False`) vì ngân sách còn hơn 57%.
- **Vì sao `small_bump` (20 phút ở 12% lỗi) không nên đánh thức ai lúc 11 giờ trưa?**
  - Mặc dù tỷ lệ lỗi 12% nghe có vẻ cao trong tích tắc, nhưng nó chỉ kéo dài vỏn vẹn 20 phút. Tổng lượng lỗi tích lũy trong 20 phút này chỉ tiêu tốn chưa đầy $0.5\% - 1\%$ tổng ngân sách lỗi của cả tháng 30 ngày.
  - Đánh thức kỹ sư on-call cho một đợt tăng lỗi nhỏ tự hết sẽ gây mệt mỏi cảnh báo (alert fatigue). Burn-rate alerter với cơ chế đa cửa sổ (yêu cầu cả cửa sổ dài 1h/6h đạt ngưỡng) đã lọc bỏ thành công sự cố này, bảo vệ sự tập trung của đội ngũ kỹ thuật.

---

## 6. Postmortem — Băng G2 với client naive (blameless)

![G2 Overload Storm Timeseries](file:///d:/AI20K/Lab25/Track3-Day25-2A202602998-NguyQuangHung/reports/gameday/G2_overload_storm.png)

### Tóm tắt (3 câu)

Vào thời điểm t=120s trong bài kiểm tra Game Day G2, nhà cung cấp mô hình chính (alpha) bất ngờ suy giảm 75% năng lực xử lý trong vòng 3 phút do sự cố phần cứng GPU. Do client naive sử dụng chiến lược retry mù quáng không giới hạn tải và không có circuit breaker, hệ thống đã tự tạo ra một cơn bão retry cục bộ làm quá tải hoàn toàn hàng đợi của alpha. Hậu quả là dù alpha đã khôi phục 100% công suất phần cứng tại t=300s, toàn bộ dịch vụ vẫn bị sập kéo dài thêm 155 giây và chỉ hồi phục sau khi hàng đợi cũ được xả cạn.

### Ảnh hưởng

- **Số lượt người dùng thất bại:** 1,285 trên tổng số 2,362 lượt (tỷ lệ khả dụng tụt xuống thảm hại **45.60%**).
- **Thời gian gián đoạn:** Kéo dài tổng cộng 335 giây (180 giây sự cố phần cứng + 155 giây thời gian tự phục hồi).
- **Tiêu hao ngân sách lỗi:** Nếu tính trên cửa sổ 1 giờ chứa sự cố này, lượng lỗi 1,285 turns đã thiêu rụi hơn **400%** ngân sách lỗi cho phép của cả tuần chỉ trong 5 phút.

### Timeline (trích xuất từ `naive_G2_overload_storm_timeseries.csv`)

| t (s) | Sự kiện                                      | Bằng chứng định lượng                                                                              |
| ----: | ---------------------------------------------- | -------------------------------------------------------------------------------------------------------- |
|   118 | Trạng thái bình thường trước sự cố    | `calls_ok = 10`, `attempts_alpha = 5`, 0 lỗi, latency ổn định                                    |
|   120 | alpha mất 75% GPU (bắt đầu sự cố)        | `calls_ok` tụt giảm còn 7, bắt đầu xuất hiện hàng đợi                                       |
|   136 | Bão retry bắt đầu hình thành             | `calls_ok = 3`, `attempts_alpha` vọt lên **53 req/s**                                        |
|   143 | Đỉnh điểm bão retry đè bẹp hệ thống  | `calls_ok = 0`, `attempts_alpha` đạt đỉnh **254 req/s** (gấp 25 lần bình thường)      |
|   180 | Hệ thống tê liệt hoàn toàn               | `calls_ok = 0`, `attempts_alpha` duy trì 138-195 req/s, tràn ngập lỗi `overloaded_error` (529) |
|   300 | alpha khôi phục 100% công suất GPU         | `calls_ok = 0`, `attempts_alpha` duy trì 115 req/s, hệ thống **vẫn sập hoàn toàn**      |
|   350 | Hàng đợi backlog kẹt cứng                 | `calls_ok = 0`, `attempts_alpha = 13`, các request cũ đã quá hạn vẫn chiếm hàng đợi       |
|   453 | Bắt đầu xử lý giải tỏa xong backlog     | `calls_ok` xuất hiện trở lại: đạt 4 calls thành công                                           |
|   455 | Hệ thống chính thức hồi phục hoàn toàn | `calls_ok = 12`, `attempts_alpha = 13`, recovery kết thúc sau **155.0s** kể từ t=300       |

### Trigger vs Root Cause

- **Trigger:** Nhà cung cấp alpha mất 75% cụm máy chủ GPU từ t=120s đến t=300s.
- **Root Cause:** Kiến trúc client naive thiếu hoàn toàn cơ chế backpressure:
  1. Không có circuit breaker để cắt tải và chuyển mạch sang provider dự phòng (`alpha-eu`, `beta`).
  2. Retry tuyến tính không có full jitter dẫn đến các đợt thử lại đồng pha.
  3. Không có retry budget, khiến mỗi request thất bại sinh ra hàng loạt request thử lại đè nặng lên một hệ thống đang ngạt thở.
- **Tại sao provider đã hồi phục tại t=300 mà hệ thống vẫn sập đến t=455 (Metastable Failure)?**
  Hàng đợi của provider bị nghẽn bởi hàng ngàn request mà client thực tế đã timeout hoặc bỏ cuộc (abandoned requests). Khi GPU phục vụ trở lại lúc t=300s, chúng phải tốn tài nguyên xử lý các request rác này trước. Đồng thời, các turn mới tiếp tục retry dồn dập vào hàng đợi, tạo thành một vòng lặp tự duy trì (self-sustaining overload loop) khiến hệ thống không thể tự thoát ra cho đến khi toàn bộ hàng đợi quá hạn được dọn sạch sau 155 giây.

### Điều gì giúp / Điều gì làm tệ hơn

- **Làm tệ hơn:** Thử lại ngay lập tức không có jitter; cố chấp gửi vào alpha duy nhất; giữ timeout quá dài khiến tài nguyên bị giữ chân.
- **Giúp ích:** Circuit breaker (ngắt ngay khi thấy failure rate $\ge 50\%$), chuyển mạch sang `alpha-eu` và `beta`, và Token Bucket Retry Budget (chặn đứng khuếch đại tải $\le 10\%$).

### Action Items

| # | Hành động                                                                 | Loại    | Owner            | Cách kiểm chứng                                                    |
| - | ---------------------------------------------------------------------------- | -------- | ---------------- | --------------------------------------------------------------------- |
| 1 | Tích hợp`SlidingWindowBreaker` cho mọi client gọi LLM                  | Prevent  | Reliability Team | Chạy CI mô phỏng G2, kiểm tra`recovery_s <= 5s`                 |
| 2 | Áp dụng`RetryBudget` tỷ lệ 10% và Full Jitter Exponential Backoff     | Prevent  | Core Platform    | CI test không cho phép`amplification > 1.2` trên mọi băng lỗi |
| 3 | Triển khai Multi-Window Burn-Rate Alerting phát hiện sập nguồn GPU      | Detect   | SRE On-call      | Bắn alert page trong vòng$\le 5$ phút khi có fast burn          |
| 4 | Cấu hình tự động Failover sang vùng EU (`alpha-eu`) khi US suy giảm | Mitigate | Infrastructure   | Băng G2 đạt`availability >= 99%` trong môi trường staging     |

---

## 7. Một điểm yếu còn lại

**Điểm yếu hiện tại:** Khi sự cố xảy ra và client chuyển mạch sang `beta` (như ở băng G5, G6), chi phí trên mỗi lượt (`cost_per_good_turn`) bị đội lên đáng kể (từ 1.0 lên 1.4 cho mỗi LLM call, khiến chi phí turn ở G5 tăng lên 2.504 so với mức chuẩn 2.0). Ngoài ra, latency p95 ở băng G2 vẫn chạm ngưỡng 11.2s do các turn phải trải qua một chu kỳ chờ retry backoff trước khi failover.

**Cách khắc phục:**

1. **Dynamic Cost-Aware Routing:** Thay vì danh sách fallback cứng nhắc, tính toán điểm số định tuyến kết hợp giữa trạng thái breaker, latency p50 hiện tại và trọng số chi phí.
2. **Hedged Requests (Tail-Tolerance):** Nếu sau $p95$ thông thường (khoảng 2.5s) mà provider chính chưa phản hồi, tự động bắn một request song song sang provider phụ và lấy kết quả của bên nào về trước, huỷ bên còn lại. Điều này sẽ kéo p95 ở kịch bản quá tải xuống dưới 5s mà chỉ tốn thêm một lượng nhỏ chi phí cho đuôi latency.

---

## 8. Cache (Part G)

### So sánh Resilient (không cache) vs Cached (của bạn) trên băng C1

| Client                   | availability | faq_availability | cache_hit_ratio | stale_served | wrong_answers | outdated_answers | cost/turn |  PASS?  |
| ------------------------ | -----------: | ---------------: | --------------: | -----------: | ------------: | ---------------: | --------: | :-----: |
| resilient (không cache) |       0.8373 |           0.8424 |           0.000 |            0 |             0 |                0 |     1.386 | ❌ FAIL |
| cached (của bạn)       |       0.9124 |           0.9645 |           0.592 |          192 |             0 |                2 |     0.955 | ✅ PASS |

### Cấu hình `cache:` và lý do

| Tham số`cache:`       | Giá trị | Vì sao (số liệu nào)                                                                                                                                                                                                                                                          |
| ------------------------ | --------: | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `similarity_threshold` |      0.90 | Ngưỡng 0.80 quá lỏng gây ra 181 câu trả lời sai (`wrong_answers = 181` vì nhận nhầm câu hỏi khác). Tăng lên 0.90 triệt tiêu hoàn toàn lỗi này (`wrong_answers = 0`) trong khi vẫn giữ `cache_hit_ratio = 59.2%` ($\ge 50\%$).                     |
| `ttl_s`                |        30 | Khi carrier đổi phí vận chuyển âm thầm tại t=480s mà không có webhook, TTL ngắn 30s giúp entry hết hạn nhanh, khống chế số câu trả lời cũ chỉ ở mức 2 câu (`outdated_answers = 2 <= 15`). Nếu để TTL 600s, con số này sẽ là 80-95 câu (FAIL). |
| `max_stale_s`          |       300 | Cho phép phục vụ câu trả lời cũ tối đa 300s khi toàn bộ provider sập (240s-360s, kéo dài 120s). Nhờ đó phục vụ 192 lượt stale, kéo`faq_availability` từ 85.29% lên 96.45% ($\ge 93\%$).                                                               |

### Ba thí nghiệm chuyên sâu

1. **Thử nghiệm ngưỡng tương đồng (`similarity_threshold`):**
   - Chạy với `threshold = 0.80`: `cache_hit_ratio = 0.610`, xuất hiện tới **181 câu trả lời sai** (`wrong_answers_by_reason = {'other_question': 181}`), `cost/turn = 1.016`.
   - Chạy với `threshold = 0.90` (của bạn): `cache_hit_ratio = 0.592`, **0 câu trả lời sai**, `cost/turn = 0.955`.
   - Chạy với `threshold = 0.97`: `cache_hit_ratio = 0.499` (tụt dưới mức tối thiểu 50% yêu cầu), `wrong_answers = 0`, `cost/turn = 1.018`.
   - *Cặp câu hỏi bị nhầm ở 0.80:* Các câu hỏi có vector ngữ nghĩa gần nhau nhưng ý nghĩa nghiệp vụ khác biệt hoàn toàn (ví dụ: *"Chính sách hoàn tiền thế nào?"* bị nhầm với *"Thời gian nhận tiền hoàn là bao lâu?"* hoặc *"Quy định đổi trả hàng"*).
2. **Thử nghiệm thời gian sống (`ttl_s`):**
   - Với `ttl_s = 30`: `outdated_answers = 2`, `hit_ratio = 0.592`, `faq_availability = 0.9645`.
   - Với `ttl_s = 60`: `outdated_answers = 3`, `hit_ratio = 0.652`.
   - Với `ttl_s = 300`: `outdated_answers = 95` (vi phạm nghiêm trọng trần 15 câu).
   - *Vì sao webhook không giúp được gì cho `outdated_answers`?* Vì sự kiện tại t=480s là `silent_updates` (đối tác đổi giá nhưng không bắn webhook báo tin). Không có webhook thì hàm `invalidate()` không bao giờ được kích hoạt. Lúc này cơ chế TTL (hết hạn theo thời gian) là cơ chế phòng vệ **duy nhất** buộc cache phải revalidate lại từ nguồn.
3. **Thử nghiệm Stale-if-error (`max_stale_s`):**
   - Khi tắt stale (`max_stale_s: 0`): `stale_served = 0`, `faq_availability = 0.8529` (FAIL do dưới 93%).
   - Khi bật stale (`max_stale_s: 300`): phục vụ được **192 lượt stale**, đẩy `faq_availability` lên **0.9645** (96.45%, PASS rực rỡ).

### Trả lời câu hỏi lý thuyết:

- **Vì sao câu hỏi cá nhân ("Đơn #48213 của tôi đang ở đâu?") không được cache, kể cả khi cache theo từng user?**
  1. *Rủi ro rò rỉ dữ liệu nhạy cảm (Privacy/PII Leakage):* Ngay cả khi định danh theo user, sai sót trong logic hashing hoặc chia sẻ phiên có thể khiến thông tin cá nhân của người này lọt sang người khác.
  2. *Tính biến động cao (High Volatility):* Trạng thái đơn hàng thay đổi theo thời gian thực (đang giao, đã giao, huỷ). Nếu cache lại, người dùng sẽ nhận thông tin trạng thái cũ sai lệch.
  3. *Tỷ lệ tái sử dụng cực thấp (Low Cache Utility):* Người dùng hiếm khi hỏi lặp lại câu hỏi cùng một mã đơn hàng nhiều lần với cùng một trạng thái.
- **Trong lúc mọi provider sập, dashboard vẫn xanh nhờ cache. Bạn sẽ alert dựa trên số nào để on-call vẫn biết có sự cố?**
  - Tuyệt đối không chỉ nhìn vào `overall_availability` hay mã HTTP 200 trả về cho client.
  - Cần đặt cảnh báo dựa trên:
    1. **Tỷ lệ phục vụ dữ liệu cũ (`stale_served_ratio` hoặc `stale_rate`):** Nếu `stale_served > 0` kéo dài vượt quá 1-2 phút, phải kích hoạt cảnh báo on-call ngay vì cache đang phải "gánh team" che giấu sự cố backend.
    2. **Backend Error Rate / Upstream Provider Health:** Giám sát trực tiếp số lượng request đi ra ngoài LLM providers (`provider_5xx_rate` hoặc breaker state chuyển sang OPEN).
