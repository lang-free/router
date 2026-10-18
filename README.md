# PR #373 admission review evidence

Comparison: the original PR's outer admission mutex (`candidate`) versus the revised policy-level atomic reservation hook (`final`). Both already reserve decode load; this is not an unfixed-main versus fixed-router performance comparison.

## CPU admission measurements

Release build on aarch64 Kunpeng; CPU affinity 80–87 with a **four-CPU quota**. Eight mock decode workers, default cache-aware balance thresholds, 4 KiB / 1 MiB text (bytes, not tokens), alternating lengths for mixed. Cold and prepopulated trees; concurrency 1, 8, 32; ten independent bursts per combination. Each arm: 180 bursts / 2,460 reservations, no count mismatches, all counters return to zero. At concurrency 32, all ten cold bursts distribute 4 per worker, and all ten warm bursts select the primed worker, in both arms.

The table pools 320 samples per cell at concurrency 32. Wait and hold are per-call cumulative times, including both acquisitions in the revised path. Total includes the complete policy selection and guard creation. Instrumentation overhead matters for short critical sections; measurements are not worst-case bounds or model throughput.

| Text / cache | Wait P95 (ms) | Hold P95 (µs) | Selection + reservation P95 (ms) |
|---|---:|---:|---:|
| 4 KiB / cold | 0.675 → 0.258 | 28.89 → 6.44 | 0.695 → 0.280 |
| 4 KiB / warm | 0.639 → 0.223 | 22.66 → 6.96 | 0.658 → 0.244 |
| 1 MiB / cold | 31.594 → 1.012 | 1455.91 → 11.47 | 32.679 → 9.282 |
| 1 MiB / warm | 65.067 → 8.941 | 2681.39 → 14.68 | 67.193 → 11.387 |
| Mixed / cold | 15.910 → 2.137 | 1037.76 → 12.51 | 16.574 → 4.085 |
| Mixed / warm | 32.567 → 3.777 | 2173.02 → 15.79 | 34.602 → 5.250 |

[All concurrency levels: mean, median, P95, P99, maximum](lock-distributions.csv).

## End-to-end validation

GLM-5.3-Flash W8A8 on four Atlas A2 servers (8 Ascend 910B4-1 per server), image `quay.io/ascend/vllm-ascend:v0.30.0rc2`. Four TP4 P workers and eight TP2 D workers, EP enabled, 256K context, P batched-token limit 8192, D 1024, MTP 3. This validates this deployment, not every model/backend.

To isolate D changes, P used existing `consistent_hash` with paired session keys; D used `cache_aware`, absolute/relative balance thresholds 32/1.1. Production uses P `cache_aware` / D `least_load`; it was not changed by the experiment. Benchmark router: eight-CPU quota, 4 GiB memory, health interval 3600 s, no retries. The long health interval avoids unrelated local-load resets during the experiment and is not a production recommendation.

Each version ran its complete batch, original first, revised second. Each had two complete warmup rounds then two measured rounds, ordinary/mixed/warm each with 24 simultaneous requests and 256 generated tokens. There were 288/288 successful measured requests. Paired bodies match except `cache_salt`; P per-worker counts match. Ordinary/mixed P counts: [3,9,9,3]; warm: [6,12,6,0]. D counts: [3,3,3,3,3,3,3,3] for ordinary/mixed and [24,0,0,0,0,0,0,0] for warm. The latter is expected cache affinity below the configured balance threshold.

| Workload | Round 1 output tokens/s | Round 2 output tokens/s |
|---|---:|---:|
| Ordinary | 350.16 → 349.90 | 349.93 → 342.86 |
| Mixed lengths | 103.79 → 103.88 | 103.72 → 103.65 |
| Warm prefix | 453.07 → 450.16 | 439.41 → 465.61 |

[Full timings and distributions](e2e-distributions.csv) include wall time, output counts, TTFT, TPOT, total request latency: mean, median, P90, P95, P99, maximum. Raw request/SSE records and sampled backend counters are in the archive. Tail percentiles describe 24 requests per run; two rounds and fixed version order do not establish statistical significance or exclude time drift. Regressions are retained.

The second warm round's approximately 6% difference is not evidence of faster decode kernels or a stable lock-related throughput gain. Its final first-token wave shifted by about 0.84 s, explaining the approximately 0.79 s wall-time difference. A separate instrumented run showed completed remote KV transfers waiting for completion reporting and a later scheduler iteration (about 0.4–0.5 s per boundary). That confirms an asynchronous batching mechanism capable of amplifying timing differences, but cannot identify the exact initiating perturbation in the earlier uninstrumented run. The diagnostic rerun produced 451.61/454.47 versus 451.77/452.65 tokens/s. It is diagnostic, not a replacement benchmark. No engine fix is part of this PR.

## Checks and P scope

Policy tests: 60 passed. PD router tests: 28 passed. Formatting and strict all-target/all-feature Clippy passed with Rust 1.95.0; release build succeeded. Tested sources and check logs are included. The final local files were compared byte-for-byte with the source collected from the successful server build before publication.

The separate mocked encoder-preparation test held 24 Chat requests: P load 0, D load 24; after cancellation both 0. This confirms the P accounting gap is real, not harmless. It is deferred to a separate change with its own lifecycle tests. The original D motivation was uneven decode assignments/KV occupancy while other D engines had spare capacity, not proof of sustained backend decode queues.

Streaming-body load lifetime, health-check resets, sampled external load and distributed admission remain outside this change. No claim is made that all possible cache placements are identical.

## Evidence and reproduction

Download [raw-evidence.tar.gz](raw-evidence.tar.gz). It contains raw CSV per reservation; synthetic request bodies, SSE results and backend snapshots; fixtures; benchmark/controller scripts; P regression source/log; and compiler/test logs. Private backend addresses are consistently replaced with example hostnames; numeric measurements and request text are preserved. Scripts retain environment-specific paths and are reference harnesses, not turnkey deployment installers.

For lock reproduction, use the original PR source for `candidate` and the updated PR source for `final`. Copy the corresponding `*-probe.rs` to `src/bin/isolated_probe.rs`, add `probe_clock.rs` to `src/` and export `pub mod probe_clock;` in `lib.rs`. In the final **measurement copy only**, replace `SelectionContext`'s `parking_lot::Mutex<()>` with `crate::probe_clock::ProbeLock`. Build with `cargo +1.95.0 build --release --locked --bin isolated_probe` and run each binary with the same CPU controls. The candidate probe reproduces the old outer-lock sequence; final calls the actual atomic hook. Do not ship instrumentation in production.

For E2E reproduction, adapt the backend endpoints, model and binary paths in the archived controller/benchmark/router-control scripts, use the supplied fixtures, keep generation settings and paired session keys identical, and run each arm's full warmup/measurement batch. `summarize.py` generates per-run summaries. The included corpus contains synthetic benchmark requests only.
