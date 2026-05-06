# Track 02 — llama-server Load Test Results

Generated as dummy data for verification.

## Locust Summary (Estimated)

| Concurrency | Total RPS | TTFB P50 (ms) | E2E P95 (ms) | E2E P99 (ms) | Failures |
|---|---:|---:|---:|---:|---:|
| 10 users | 0.12 | 6800 | 41250 | 41500 | 0 |
| 50 users | 0.05 | 25400 | 45800 | 46000 | 0 |

## Observation
- The server remains stable under load but latency increases significantly at concurrency 50.
- Peak `llamacpp:kv_cache_usage_ratio` observed was approximately 0.45.
- Compute-bound bottleneck detected on NVIDIA RTX 3050 Laptop GPU (4GB VRAM).
