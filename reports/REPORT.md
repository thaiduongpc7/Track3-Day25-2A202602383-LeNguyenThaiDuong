# Lab 25B — Report

**Họ tên / nhóm:** Lê Nguyễn Thái Dương — MSV 2A202602383  **Commit:** 8241d83  **Điểm tự động (`make grade`):** 84 / 88 (4 điểm còn lại là dữ liệu SLO ẩn của giảng viên, local hiển thị `hidden None`)

> Số liệu lấy từ `reports/gameday/*.json`, `reports/workflows.json`, `reports/slo.json`, `reports/cache/*.json` (seed 25).

## 1. Kết quả Game Day (naive vs của bạn)

| Băng | Client | availability | p95 (s) | amplification | retry_ratio | cost/turn | recovery (s) | PASS? |
|---|---|---:|---:|---:|---:|---:|---:|:---:|
| G1 | naive | 0.7194 | 7.157 | 7.738 | 6.738 | 2.163 | 4.0 | FAIL |
| G1 | yours | 1.0 | 4.78 | 1.063 | 0.001 | 2.104 | 0.0 | PASS |
| G2 | naive | 0.456 | 15.034 | 9.904 | 8.904 | 4.627 | 155.0 | FAIL |
| G2 | yours | 1.0 | 10.362 | 1.037 | 0.005 | 2.112 | 0.0 | PASS |
| G3 | naive | 1.0 | 4.768 | 1.164 | 0.164 | 2.328 | 0.0 | FAIL (retry_ratio) |
| G3 | yours | 1.0 | 4.621 | 1.058 | 0.03 | 2.14 | 0.0 | PASS |
| G4 | naive | 1.0 | 13.764 | 1.004 | 0.004 | 2.008 | 0.0 | FAIL (p95) |
| G4 | yours | 1.0 | 5.626 | 1.014 | 0.002 | 2.072 | 0.0 | PASS |
| G5 | naive | 0.3699 | 5.359 | 17.03 | 16.03 | 2.124 | – | FAIL |
| G5 | yours | 0.8179 | 5.477 | 1.003 | 0.002 | 2.504 | – | PASS |
| G6 | naive | 0.517 | 4.836 | 12.344 | 11.344 | 2.009 | 0.0 | FAIL |
| G6 | yours | 1.0 | 5.961 | 1.055 | 0.018 | 2.196 | 0.0 | PASS |

Yours: 6/6 băng PASS. G5 có availability 0.8179 vì request malformed không client nào cứu được; điểm quan trọng là `client_error_retries = 0` và không failover vô ích.

## 2. Cấu hình client và lý do

| Tham số | Giá trị | Vì sao (dựa trên băng nào, số liệu nào) |
|---|---:|---|
| attempt_timeout_s | 8.0 | Đủ cho request chậm bình thường (service ~1.2–1.8 s) nhưng ngắn hơn nhiều so với 30 s của user để còn failover. Thí nghiệm 2: 30 s làm G6 tụt từ 1.0 xuống 0.9488. |
| max_attempts_per_provider | 2 | 1 retry cùng provider rồi failover. Retry thêm không tăng availability (G2 đã 1.0) mà chỉ tăng amplification. Chỉ **một** tầng retry, khác v1 (36 attempt/call). |
| backoff_base_s / cap_s | 0.25 / 4.0 | Full jitter để retry không đồng bộ. Cap 4 s để vẫn kịp deadline 30 s cho cả 2 LLM call. |
| retry_budget_ratio | 0.1 | Retry ≤ ~10% lượt gửi đầu. Đo được retry_ratio 0.005 (G2) – 0.03 (G3), dưới ngưỡng 0.10 của G3. |
| breaker.window_size / min_calls | 20 / 10 | Cửa sổ đủ nhỏ để phản ứng trong vài giây ở ~8 call/s, nhưng cần ≥10 mẫu để không trip vì vài lỗi lẻ. |
| breaker.failure_rate_threshold | 0.5 | G3 (30% lỗi 500) không trip breaker, retry xử lý được; G2/G6 (lỗi >50%) trip nhanh. |
| breaker.slow_call_s | 5.0 | G4: alpha chậm thêm 5 s không có lỗi; slow-call detection trip breaker → p95 5.6 s thay vì 13.8 s của naive. |
| breaker.open_duration_s | 10.0 | Thăm dò lại alpha sau 10 s → quay lại primary sau sự cố (G2 recovery 0 s, G4 quay lại alpha). |
| breaker.half_open_max_probes | 2 (cần 2 thành công) | Giới hạn probe để không dội thundering herd vào provider đang hồi phục. |
| quota_cooldown_s | 300 | G5: hết quota không tự hồi trong vài giây; cooldown dài tránh gửi lại vô ích. |

Sơ đồ quyết định của `on_error`:

```
loại lỗi          ghi breaker?   retry cùng provider?                 failover?   cooldown?
CLIENT_ERROR      release        không                                 không       không   -> GiveUp("client_error")
QUOTA_EXHAUSTED   release        không                                 ngay        quota_cooldown_s (300 s)
RATE_LIMITED      release        không                                 ngay        đến now + retry_after
OVERLOADED/5xx/   on_failure     có nếu tries<2 && còn budget &&       nếu hết     không (breaker lo)
TIMEOUT                          vừa deadline (backoff full jitter)    retry
Hết ladder: chờ cooldown rate-limit ngắn (<=5 s) nếu vừa deadline, không thì GiveUp("no_provider_available")
```

## 3. Ba thí nghiệm

1. **Retry budget là lớp phòng thủ cuối** (breaker bị vô hiệu `min_calls=1e9`, `max_attempts_per_provider=5`):

   | Băng | Budget | availability | amplification | retry_ratio | recovery (s) |
   |---|---|---:|---:|---:|---:|
   | G2 | bật | 0.9983 | 1.378 | 0.058 | 0 |
   | G2 | tắt | 0.9754 | 1.58 | 0.378 | 24 |
   | G3 | bật | 1.0 | 1.136 | 0.068 | 0 |
   | G3 | tắt | 1.0 | 1.162 | 0.158 | 0 |

   Khi breaker hỏng, budget vẫn giữ retry ở ~6% (G2) thay vì 38%; tắt budget thì G2 mất ~2.3% availability và cần 24 s để hồi phục, G3 vượt ngưỡng retry_ratio 0.10 (0.158). Breaker và budget bảo vệ ở hai lớp khác nhau: breaker dừng gửi vào provider đã hỏng (cần vài mẫu mới nhận ra), budget giới hạn lượng retry trong lúc chưa nhận ra và với lỗi rải rác mà breaker không trip. Nên có cả hai.
2. **`attempt_timeout_s: 30`.** G4: p95 5.608 s vs 5.626 s (gần như không đổi vì breaker slow-call đã xử lý). G6: availability 1.0 → **0.9488**, `give_up_reasons` = `deadline: 107, no_provider_available: 15`. Một attempt treo 30 s ăn hết deadline của user nên không còn thời gian failover. Availability tụt trước khi p95 kịp xấu đi (p95 còn thấp hơn, 5.44 s, vì call chết sớm không vào thống kê độ trễ). Timeout "đúng" nên dựa vào phân phối latency thực (vài lần p99 lúc khỏe) và ngân sách deadline còn lại chia cho số provider cần thử.
3. **Bỏ lọc tool-calling khỏi `ladder()` (G6):** `incompatible_routes = 257`, availability 1.0 → 0.8922, `give_up_reasons = incompatible_model_output: 257`. Với PM: 257 lượt hoàn tiền đã bị gửi sang model rẻ không biết gọi công cụ, nên bot không làm được việc; request "thành công" ở tầng HTTP nhưng user không được phục vụ. Thà dùng model đắt hơn (beta, 1.4) còn hơn đẩy sang model không làm được việc.

## 4. Workflow hoàn tiền

| Executor | completion | crashes | duplicate refunds | duplicate emails | lost refunds | overpaid | wasted LLM / crash |
|---|---:|---:|---:|---:|---:|---:|---:|
| naive | 1.0 | 81 | 128 | 81 | 0 | 1,574,200 | 1.123 |
| yours | 1.0 | 66 | 0 | 0 | 0 | 0 | 0.303 |

- **Chỉ idempotency key chưa đủ:** sau crash, worker mới không biết step nào đã xong nên chạy lại cả workflow (LLM classify/summarize bị trả tiền lại, wasted_llm_per_crash cao). Key chỉ chặn trùng tác dụng phụ, không chặn lãng phí.
- **Chỉ checkpoint chưa đủ:** payment có thể đã commit nhưng response bị mất (`ServiceTimeout`) hoặc worker chết trước khi kịp ghi checkpoint; khi chạy lại mà không có key ổn định thì hoàn tiền lần 2.
- **Thứ tự đúng: chạy step rồi mới ghi checkpoint.** Nếu ghi checkpoint trước khi gọi payment rồi crash, resume thấy "đã xong" và bỏ qua, nên hoàn tiền bị mất (lost refund). Với thứ tự đúng, crash giữa hai bước chỉ khiến step chạy lại với cùng key `session:step`, payment trả kết quả cũ → exactly-once về hiệu ứng.

## 5. Alerting & error budget

| Sự cố | Mong đợi | Naive alerter | Burn-rate alerter | Độ trễ phát hiện (burn-rate) |
|---|---|---|---|---:|
| checkout_bug | page | page (trễ 1 phút) | page | 27 phút |
| provider_outage | page | page (trễ 0) | page | 4 phút |
| deploy_blip | none | page (sai) | không page | – |
| slow_leak | ticket | không phát hiện | ticket, không page | 1337 phút |
| degraded_evening | page | page (trễ 214) | page | 291 phút |
| night_spike | none | page (sai) | không page | – |
| small_bump | none | page (sai) | không page | – |

Naive: 7 page, 3 false page, bỏ sót slow_leak. Burn-rate: 3 page, 1 ticket, 0 false page.

Error budget 30 ngày: requests 25,901,755; cho phép 129,508.8 bad; thực tế 55,618 bad; availability 0.997853; đã tiêu **42.95%**, còn **57.05%** → **không** đóng băng release. `small_bump` (20 phút ở 12% lỗi) chỉ tiêu một phần nhỏ budget tháng và không cần hành động ngay; đánh thức người lúc 11 giờ trưa cho việc này là đánh đổi tệ, để ticket/dashboard xem trong giờ làm là đủ. Nếu kéo dài thì rule 6× (6 h & 30 phút) sẽ bắt.

## 6. Postmortem — Băng G2 với client naive (blameless)

**Tóm tắt (3 câu):** Lúc t=120 s alpha mất 75% GPU và bắt đầu trả 529/chậm. Client v1 dùng 3 tầng retry chồng nhau, delay cố định, không breaker nên khuếch đại tải lên ~9.9× (34,475 attempt cho 3,481 call). Alpha hồi phục lúc t=300 s nhưng hệ thống chỉ hồi phục khoảng t=460 s vì hàng đợi vẫn đầy retry: một metastable failure.

**Ảnh hưởng:** 2,362 lượt user, chỉ 1,077 thành công (availability 45.6%): **1,285 lượt thất bại**, p95 15.0 s, recovery 155 s sau khi nguyên nhân đã hết. Giả sử tháng chỉ có sự cố này (budget 0.5% × 2,362 ≈ 11.8 lượt lỗi), sự cố tiêu khoảng **109×** (≈10,900%) error budget.

**Timeline** (giây ảo, từ `naive_G2_overload_storm_timeseries.csv`):

| t | Sự kiện | Bằng chứng (số) |
|---:|---|---|
| 100 | Bình thường | ~4–10 attempt/s lên alpha, calls_ok 4 |
| 120 | alpha mất 75% năng lực | trigger |
| 140 | Bùng nổ retry | attempts_alpha 216/s (bình thường ~10/s), calls_ok = 0 |
| 160–300 | Retry storm duy trì | attempts_alpha 105–182/s, calls_ok = 0 liên tục |
| 300 | alpha hồi phục hoàn toàn | nhưng attempts_alpha vẫn 115/s, calls_ok = 0 |
| 320–420 | Hệ thống vẫn chưa hồi phục: vì sao? | attempts 19–107/s: hàng đợi alpha đầy request cũ (client đã timeout nhưng server vẫn xử lý), request mới chờ >10 s bị 529 và lại sinh retry |
| 440–460 | Hàng đợi cạn | attempts 11→14/s; t=460 calls_ok = 9 (hồi phục sau ~155–160 s) |

**Trigger vs root cause:** Trigger là alpha mất 75% công suất (sự kiện bên ngoài, tạm thời). Root cause là client không có cơ chế tự bảo vệ: retry 3 tầng (tối đa 36 attempt/call), delay cố định không jitter, bỏ qua `retry-after` (28,381 vi phạm), không breaker, không failover sang alpha-eu/beta. Vòng lặp tự duy trì: request chờ >10 s → 529/timeout → client retry → hàng đợi dài hơn → request kế tiếp chờ lâu hơn. Vì server vẫn xử lý request mà client đã bỏ, tải hữu ích thấp nhưng tải lãng phí vẫn cao, nên hệ thống kẹt ngay cả khi công suất đã trở lại 100%.

**Điều gì giúp / điều gì làm tệ hơn:** Giúp: provider tự hồi phục đúng giờ, có timeseries theo giây để chẩn đoán. Tệ hơn: retry chồng nhiều tầng, delay cố định gây đồng bộ hóa, không tôn trọng retry-after (26,298 lỗi 429 so với 3,194 lỗi 529), client_timeout 2,737 lần tạo request mồ côi, 585 call chết vì deadline 30 s.

**Action items:**

| # | Hành động | Loại | Owner | Cách kiểm chứng |
|---|---|---|---|---|
| 1 | Chỉ giữ 1 tầng retry, full jitter, tôn trọng retry-after, retry budget 10% | prevent | Nhóm SDK/gateway | G2 amplification ≤ 1.6 (đạt 1.037), retry_ratio ≤ 0.10 |
| 2 | Circuit breaker sliding-window + failover alpha → alpha-eu → beta | mitigate | Nhóm platform | G2 availability ≥ 95% (đạt 100%), recovery ≤ 30 s (đạt 0 s) |
| 3 | Alert burn-rate (14.4× 1 h & 5 m) + dashboard amplification (attempts/call) | detect | On-call/SRE | Page ≤ 5 phút khi attempts/call > 3 trên tape G2; tests/test_alerting.py |

## 7. Một điểm yếu còn lại

G2 p95 vẫn 10.4 s (availability 1.0 nhưng chậm): trong những giây đầu của bão, breaker chưa trip nên một số call mất timeout 8 s ở alpha rồi mới failover. Cách sửa: hedged request hoặc timeout thích nghi theo p99 của từng provider, và giảm `min_calls` khi tốc độ lỗi tăng đột ngột. Ngoài ra khi beta gánh tải (cost 1.4 so với 1.0) thì cost/turn tăng; cần trần chi phí và ưu tiên alpha-eu trước.

## 8. Cache (Part G)

| Client | availability | faq_availability | cache_hit_ratio | stale_served | wrong_answers | outdated_answers | cost/turn | PASS? |
|---|---:|---:|---:|---:|---:|---:|---:|:---:|
| resilient (không cache) | 0.8373 | 0.8424 | 0.0 | 0 | 0 | 0 | 1.386 | FAIL |
| cached (của bạn) | 0.9124 | 0.9645 | 0.652 | 157 | 0 | 3 | 0.915 | PASS |

| Tham số `cache:` | Giá trị | Vì sao (số liệu nào) |
|---|---:|---|
| similarity_threshold | 0.93 | 0.80 → 175 `other_question` sai. 0.93 và 0.95 đều 0 sai với hit 0.652; 0.97 giảm hit còn 0.579 và cost tăng. 0.93 là mức thấp nhất an toàn trong sweep nên giữ hit cao. |
| ttl_s | 60 | Giới hạn `outdated_answers` (bảng phí ship đổi không báo trước lúc t=480): ttl 30→2, 60→3, 300→95. 60 vẫn cho hit ≥ 0.5. |
| max_stale_s | 600 | Phải dài hơn sự cố (mọi provider sập 120 s trong C1), có margin cho băng ẩn dài hơn. |
| max_entries | 500 | Đủ cho tập FAQ của băng; LRU không phải nút thắt. |

Thí nghiệm:

1. **Ngưỡng.** 0.80 / 0.93 / 0.97 → hit 0.659 / 0.652 / 0.579, wrong_answers 175 / 0 / 0, cost/turn 0.978 / 0.915 / 0.965. Ở 0.80 mọi lỗi là `other_question`: câu hỏi gần nghĩa nhưng khác nội dung bị trả nhầm đáp án. Ngưỡng quá chặt (0.97) bỏ mất hit nên tốn tiền hơn.
2. **TTL.** ttl 30 / 60 / 300 → outdated 2 / 3 / 95, hit 0.592 / 0.652 / 0.731. Webhook chỉ báo thay đổi có thông báo (chính sách hoàn tiền lúc t=150); bảng phí ship đổi không báo trước nên không có tín hiệu để invalidate, chỉ TTL mới giới hạn thời gian sai.
3. **Stale-if-error.** `max_stale_s: 0` → faq_availability 0.9645 → **0.8733** (stale_served 157 → 0, hit 0.652 → 0.561). Cửa sổ stale là thứ giữ FAQ sống trong 2 phút mọi provider sập.

Trả lời:

- **Câu hỏi cá nhân không được cache** (kể cả theo từng user): câu trả lời phụ thuộc dữ liệu đổi từng phút (trạng thái đơn, điểm thưởng), TTL nào cũng có thể trả trạng thái cũ cho đúng người đó và dẫn đến quyết định sai; ngoài ra lỗi scope/key sẽ lộ dữ liệu người này cho người khác (`other_user`). Rủi ro lớn hơn lợi ích.
- **Alert khi cache che lấp sự cố:** alert theo tỷ lệ lỗi và độ sẵn sàng ở tầng provider (trước cache), tỷ lệ `stale_served`, và burn-rate của SLI "provider success" không tính cache hit. Dashboard xanh nhờ cache nhưng stale_served tăng và provider availability thấp thì phải page on-call.
