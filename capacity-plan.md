# Capacity Plan — Vision Moderation

Serving target: **300 RPS sustained, 500 RPS 5-min spikes.** Sync path = 90% of
traffic (p95 budget 250 ms), batch path = 10%. Compute budget **$4,000/mo**.

All RPS figures below are **requests per second** (RPS), consistent throughout.
All latency figures are **milliseconds** (ms) end-to-end unless flagged.

---

## 1. Latency budget breakdown (synchronous path)

The 250 ms p95 budget, allocated per stage. Per-stage numbers are typical
(median-to-p95) costs; **Headroom** absorbs tail variance, queueing, and GC
pauses so the *end-to-end* p95 lands under 250 ms.

| Stage | Budget (ms) | Notes |
|---|---|---|
| Network in | 5 | half of the ~10 ms in+out RTT |
| Auth + routing | 2 | edge/gateway |
| Payload parse | 8 | decode image bytes |
| Feature lookup (Redis) | 8 | given p95 ~8 ms |
| Pre-processing | 10 | resize/normalize (CPU) |
| Model inference | 75 | **CPU**, single-core median (see §2) |
| Post-processing | 5 | threshold/label mapping |
| Serialization | 5 | JSON response build |
| Network out | 5 | other half of in+out RTT |
| **Headroom** | **127** | tail + queue + scheduling slack |
| **Total** | **250** | |

Check: `5+2+8+8+10+75+5+5+5 = 123`; `123 + 127 = 250`. **Headroom = +127 ms,
positive.** Even if inference hits its own p95 (~1.5× median ≈ 113 ms, +38 ms)
and Redis spikes, we stay well inside 250 ms.

---

## 2. CPU vs GPU decision — **CPU**

| | CPU `c6i.2xlarge` (8 vCPU) | GPU `g4dn.xlarge` (T4) |
|---|---|---|
| Inference | 75 ms median | 22 ms |
| On-demand price | ~$0.34/hr → **~$248/mo** | ~$0.526/hr → **~$384/mo** |
| Per-replica throughput (derated) | ~56 RPS | ~40 RPS |

Pricing: AWS on-demand, us-east-1, within ±30% (source: AWS EC2 pricing page,
c6i / g4dn families). Throughput is derated ~70% from the naive `1000/inference_ms`
to leave cores for pre/post-processing and OS, and for queueing contention.

**Decision: CPU.** The 75 ms CPU inference fits comfortably inside the 250 ms p95
budget (127 ms headroom) — latency is *not* tight enough to need a GPU. CPU is
~35% cheaper per instance **and** wins on throughput-per-dollar here:
56 RPS / $248 = **0.226 RPS/$** vs 40 RPS / $384 = 0.104 RPS/$. GPU would only
win if the latency budget were sub-150 ms, or if heavy dynamic batching pushed
T4 utilization far above its no-batch rate. Neither holds for the sync path.

---

## 3. Replica sizing (sync path = 90% of traffic)

Effective per-replica throughput: **56 RPS** (derated, from §2). Replicas needed
include **30% headroom**, then `ceil`.

| Case | Total RPS | Sync RPS (90%) | Per-replica | Raw replicas | +30% → replicas | Monthly cost |
|---|---|---|---|---|---|---|
| Sustained | 300 | 270 | 56 | 4.8 | **7** | 7 × $248 = **$1,736** |
| Spike (5 min) | 500 | 450 | 56 | 8.0 | **12** | 12 × $248 = **$2,976** |

Batch path (10%): 30 RPS sustained / 50 RPS spike, separate deployment, ~2–3
replicas (~$500–750/mo). **Worst-case combined still < $4,000/mo budget.**

**Spike strategy — warm baseline + autoscale, with shed as backstop.** Keep **7
warm** replicas (sustained capacity). Autoscale via HPA on RPS/CPU up to **12**
for spikes. Because spikes are short (5 min) and container+180 MB model cold-start
is ~30–60 s, set aggressive scale-up (stabilization window 0 s up) and keep a
small warm pool (2 pre-pulled, pre-loaded pods) to cover the cold-start gap. If
scaling still lags the leading edge of a spike, **load-shed** lowest-priority /
batch traffic at the gateway to protect sync p95.

---

## 4. Batching decision

**Sync path: no dynamic batching (or batch=1).** The single-request CPU inference
is 75 ms and the headroom is 127 ms — but a dynamic-batch wait window directly
eats that headroom for *every* request, and ONNX CPU batching gives only modest
throughput gains (no massive parallel-tensor speedup like a GPU). A 10–20 ms
wait window to assemble a batch would trade 127 ms of safety margin for a small
throughput bump we don't need, since horizontal replicas already meet 450 RPS
spike. Keeping batch=1 keeps p95 predictable and tail-latency tight.

**Batch endpoint: yes — max batch 16, max wait 50 ms.** Partner batch traffic is
latency-tolerant (uploads aggregated client-side) and throughput-sensitive.
Here batching amortizes per-request Python/pre-processing overhead and improves
instances-per-dollar. A 50 ms wait window is invisible against a multi-second
batch SLA, and capping batch at 16 bounds memory and keeps any single batch's
inference under ~1.2 s (16 × 75 ms) so one slow batch can't starve the queue.
This isolates the throughput-vs-latency trade-off to the path that actually
wants throughput, leaving the sync path lean.
