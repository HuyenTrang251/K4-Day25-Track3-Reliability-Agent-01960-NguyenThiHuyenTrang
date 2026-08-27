# Báo cáo cuối kỳ Reliability Agent

## 1. Tổng quan kiến trúc

Gateway kiểm tra cache trước khi gọi provider. Nếu cache không có kết quả, request đi qua circuit breaker của primary provider, sau đó đến backup provider. Nếu cả hai đều không dùng được, gateway trả về thông báo hệ thống đang suy giảm.

```text
User Request
    |
    v
[Reliability Gateway] --> [Memory Cache] --> hit: trả về response đã lưu
    |
    v miss
[Circuit Breaker: primary] --> primary provider
    |
    v provider lỗi hoặc circuit open
[Circuit Breaker: backup] --> backup provider
    |
    v provider lỗi hoặc circuit open
[Static fallback response]
```

## 2. Cấu hình

| Tham số | Giá trị | Lý do |
|---|---:|---|
| failure_threshold | 3 | Mở circuit sau ba lỗi liên tiếp, tránh gọi lặp lại provider đang không khỏe. |
| reset_timeout_seconds | 2 | Chờ một khoảng cooldown ngắn trước khi half-open probe kiểm tra provider phục hồi. |
| success_threshold | 1 | Một probe thành công đủ để khôi phục traffic nhanh trong workload mô phỏng. |
| cache TTL | 300 giây | Tái sử dụng response ổn định trong năm phút và hạn chế dữ liệu cũ. |
| similarity_threshold | 0.92 | Ngưỡng bảo thủ giảm false hit; query có số bốn chữ số khác nhau cũng bị chặn. |
| load_test requests | 100 mỗi scenario | Ba scenario tạo ra tổng cộng 300 request. |

## 3. Định nghĩa SLO

| SLI | Mục tiêu SLO | Giá trị thực tế | Đạt? |
|---|---|---:|---|
| Availability | >= 99% | 99.00% | Có |
| Latency P95 | < 2500 ms | 314.28 ms | Có |
| Fallback success rate | >= 95% | 96.05% | Có |
| Cache hit rate | >= 10% | 62.33% | Có |
| Recovery time | < 5000 ms | 2256.48 ms | Có |

## 4. Metrics

Số liệu dưới đây được sinh từ lần chạy cache-enabled trong `reports/metrics.json`.

| Metric | Giá trị |
|---|---:|
| total_requests | 300 |
| availability | 99.00% |
| error_rate | 1.00% |
| latency_p50_ms | 277.03 |
| latency_p95_ms | 314.28 |
| latency_p99_ms | 318.21 |
| fallback_success_rate | 96.05% |
| cache_hit_rate | 62.33% |
| estimated_cost | $0.046724 |
| estimated_cost_saved | $0.187000 |
| circuit_open_count | 9 |
| recovery_time_ms | 2256.48 |

## 5. So sánh cache

Hai lần chạy dùng cùng ba chaos scenario và 100 request mỗi scenario: một lần với `cache.enabled: true`, một lần với `cache.enabled: false`. Cache hit trả kết quả ngay tại gateway, nhưng danh sách percentile hiện chỉ ghi latency của provider call. Vì vậy P50/P95 thể hiện miss-path latency, chưa phải end-to-end latency.

| Metric | Không cache | Có cache | Chênh lệch (có - không) |
|---|---:|---:|---:|
| latency_p50_ms | 276.76 | 277.03 | +0.27 ms |
| latency_p95_ms | 315.55 | 314.28 | -1.27 ms |
| estimated_cost | $0.123772 | $0.046724 | -$0.077048 |
| cache_hit_rate | 0.00% | 62.33% | +62.33 pp |
| circuit_open_count | 23 | 9 | -14 |

Cache giảm estimated provider cost khoảng 62.3% và giảm số lần circuit open. Trong production nên ghi riêng end-to-end latency cho cache hit.

## 6. Redis shared cache

In-memory cache bị tách riêng theo từng gateway process, vì vậy các instance sau load balancer không dùng lại được cache của nhau. `SharedRedisCache` lưu query hash, query và response trong Redis kèm TTL; các instance dùng cùng prefix có thể thấy cùng một state. Các query nhạy cảm và false hit do số bốn chữ số khác nhau vẫn bị chặn.

Docker không có sẵn trên máy local nên chưa thu thập được bằng chứng shared-state và Redis CLI output. Cần xác minh trên môi trường có Docker bằng lệnh `docker compose up -d` và `pytest tests/test_redis_cache.py -v`.

## 7. Chaos scenarios

| Scenario | Hành vi kỳ vọng | Hành vi quan sát | Pass/Fail |
|---|---|---|---|
| primary_timeout_100 | Primary fail, backup xử lý traffic và primary circuit open. | Scenario pass trong lần chạy có thể tái lập. | Pass |
| primary_flaky_50 | Có cả primary failure và backup routing; circuit có thể open và recover. | Scenario pass trong lần chạy có thể tái lập. | Pass |
| all_healthy | Provider khỏe xử lý traffic, không có outage kéo dài. | Scenario pass trong lần chạy có thể tái lập. | Pass |

Lần chạy cache-enabled ghi nhận 9 lần circuit open và recovery time trung bình 2256.48 ms.

## 8. Phân tích failure

Lần fault-injection hiện tại đạt các SLO đã đặt ra, nhưng vẫn còn điểm yếu: circuit state nằm trong từng process và fallback provider vẫn có thể fail. Trước khi production, cần chia sẻ health signal của circuit hoặc dùng centralized observability, thêm retry có giới hạn kèm backoff, và cảnh báo khi fallback success thấp hơn SLO.

## 9. Bước tiếp theo

1. Chạy và lưu bằng chứng Redis integration test trên môi trường có Docker.
2. Ghi end-to-end latency riêng cho cache hit và provider call.
3. Thêm scenario-level metrics và SLO gate chặt chẽ hơn cho availability và fallback success.
