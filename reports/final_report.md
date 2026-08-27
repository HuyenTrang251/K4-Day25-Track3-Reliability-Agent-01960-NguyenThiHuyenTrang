# Reliability Agent Final Report

## 1. Architecture Summary

The gateway checks the cache before calling providers. Cache misses are sent through a circuit breaker for the primary provider, then a circuit breaker for the backup provider. If both paths are unavailable, the gateway returns a static degraded response.

```text
User Request
    |
    v
[Reliability Gateway] --> [Memory Cache] --> hit: return cached response
    |
    v miss
[Circuit Breaker: primary] --> primary provider
    |
    v failure or open circuit
[Circuit Breaker: backup] --> backup provider
    |
    v failure or open circuit
[Static fallback response]
```

## 2. Configuration

| Setting | Value | Reason |
|---|---:|---|
| failure_threshold | 3 | Opens a circuit after three consecutive failures to avoid repeatedly calling an unhealthy provider. |
| reset_timeout_seconds | 2 | Allows a short cooldown before a half-open probe tests recovery. |
| success_threshold | 1 | A successful half-open probe restores traffic quickly in this simulated workload. |
| cache TTL | 300 seconds | Reuses stable responses for five minutes while limiting staleness. |
| similarity_threshold | 0.92 | Conservative threshold reduces semantic false hits; differing four-digit values are also blocked. |
| load_test requests | 100 per scenario | Three scenarios produce 300 requests total. |

## 3. SLO Definitions

| SLI | SLO target | Actual value | Met? |
|---|---|---:|---|
| Availability | >= 99% | 99.00% | Yes |
| Latency P95 | < 2500 ms | 314.28 ms | Yes |
| Fallback success rate | >= 95% | 96.05% | Yes |
| Cache hit rate | >= 10% | 62.33% | Yes |
| Recovery time | < 5000 ms | 2256.48 ms | Yes |

## 4. Metrics

Metrics below are from the cache-enabled run in `reports/metrics.json`.

| Metric | Value |
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

## 5. Cache Comparison

Both runs use 300 requests and the same provider and chaos settings: one with `cache.enabled: true`, one with `cache.enabled: false`. Cache-hit responses have zero gateway latency, but the current latency percentile list intentionally records provider calls only, so the P50/P95 columns measure miss-path latency rather than end-to-end latency.

| Metric | Without cache | With cache | Delta (with - without) |
|---|---:|---:|---:|
| latency_p50_ms | 276.76 | 277.03 | +0.27 ms |
| latency_p95_ms | 315.55 | 314.28 | -1.27 ms |
| estimated_cost | $0.123772 | $0.046724 | -$0.077048 |
| cache_hit_rate | 0.00% | 62.33% | +62.33 pp |
| circuit_open_count | 23 | 9 | -14 |

The cache reduced estimated provider cost by about 62.3% and reduced circuit openings. End-to-end cache latency should be recorded separately in a production metrics implementation.

## 6. Redis Shared Cache

In-memory cache is isolated to one gateway process, so instances behind a load balancer cannot reuse each other's cached responses. `SharedRedisCache` stores query hashes, queries, and responses in Redis with a TTL, allowing instances with the same prefix to share state while retaining the privacy and false-hit checks.

Redis was not available on the local machine because Docker is not installed. Therefore Redis shared-state evidence and Redis CLI `KEYS` output were not collected locally. The Redis implementation is present in `src/reliability_lab/cache.py`; it should be verified with `pytest tests/test_redis_cache.py -v` in an environment where Redis is running.

## 7. Chaos Scenarios

| Scenario | Expected behavior | Observed behavior | Pass/Fail |
|---|---|---|---|
| primary_timeout_100 | Primary fails; backup handles traffic and the primary circuit opens. | Scenario reported pass in the reproducible run. | Pass |
| primary_flaky_50 | Mixture of primary failures and backup routing; circuit may open and recover. | Scenario reported pass in the reproducible run. | Pass |
| all_healthy | Requests use healthy providers with no sustained outage. | Scenario reported pass in the reproducible run. | Pass |

The combined cache-enabled run recorded 9 circuit openings and an average recovery time of 2256.48 ms.

## 8. Failure Analysis

The current fault-injection run met the stated SLOs, but a remaining weakness is that each process has its own circuit state and the fallback provider can also fail. Before production, share circuit health signals or use centralized observability, add bounded retries with backoff, and alert when fallback success falls below its SLO.

## 9. Next Steps

1. Run Redis integration tests in a Docker-enabled environment and include shared-state evidence.
2. Record end-to-end latency for both cache hits and provider calls, then define an end-to-end latency SLO.
3. Add scenario-level metrics and stricter pass criteria for availability and fallback success.
