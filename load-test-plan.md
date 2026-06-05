# Load Test Plan — Vision Moderation

## 1. Tool choice — **k6**

k6 gives scripted scenarios with native ramping VUs/arrival-rate executors and
built-in p95/p99 + custom thresholds that fail CI on breach. Scriptable traffic
mix (sync vs batch) in one file beats wrk2's fixed-rate-only model.

## 2. Test phases

Target peak = **500 RPS** (the spike SLA), driven by a `ramping-arrival-rate`
executor so we hold a constant request *rate*, not a constant VU count.

| Phase | Duration | Rate profile |
|---|---|---|
| Warmup | 2 min | 0 → 50 RPS (fill caches, JIT, pull model warm) |
| Ramp | 5 min | 50 → 300 RPS linear |
| Sustained peak | 20 min | hold 300 RPS (sustained SLA) |
| Spike | 5 min | 300 → 500 RPS, hold 500 (exercise autoscale) |
| Soak | 60 min | hold 300 RPS (catch leaks / slow drift) |
| Ramp-down | 2 min | 300 → 0 RPS |

## 3. Traffic shape

- **Sync vs batch ratio:** 90% sync (`/moderate`), 10% batch (`/moderate:batch`).
- **Payload size distribution:** images log-normal — p50 ≈ 150 KB, p95 ≈ 1.2 MB,
  hard cap 5 MB (rejected with 413). Batch requests carry 8–16 images each.
- **Concurrency model:** open-model load (arrival-rate executor) so a slow server
  causes queue growth we can observe, instead of closed-loop back-pressure hiding
  it. Separate scenarios/tags for `sync` and `batch` so thresholds are per-path.

## 4. Pass/fail criteria (k6 thresholds — any breach fails the run)

| Metric | Threshold |
|---|---|
| Sync end-to-end p95 | ≤ 250 ms |
| Sync end-to-end p99 | ≤ 500 ms (partner refund line) |
| Error rate (5xx) | ≤ 0.1% over the run |
| HTTP 429 / shed rate at 300 RPS sustained | = 0% (must not shed below spike) |
| Batch p95 | ≤ 3 s |
| Autoscale recovery | sync p95 back ≤ 250 ms within 90 s of spike onset |

## 5. Bottleneck checklist (inspect per replica during the run)

- **CPU%** per replica (saturation > 80% sustained = add replicas / wrong sizing).
- **Memory / RSS** trend across the 60-min soak (leak detection).
- **GPU util** — N/A (CPU deployment), confirm no accidental GPU pods.
- **Redis** p95 lookup latency and connection-pool saturation (feature lookup).
- **Inference queue depth** and per-stage latency (pre / infer / post split).
- **HPA events** — scale-up lag, pod cold-start time, warm-pool hit rate.
- **Gateway** 429/shed counters and upstream connection errors.
- **Container restarts / OOMKills** during soak.
