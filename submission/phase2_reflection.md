# Phase 2 — Reflection: where does parallelism stop helping?

Copy this file to `submission/phase2_reflection.md`, fill in every answer slot, and commit it:

```bash
cp phase-2-tiny-gpt/reflection_template.md submission/phase2_reflection.md
```

**How to fill it in.** Each answer sits between a pair of `<!--answer:...-->` markers. Replace
the `TODO` line with your answer — **leave the markers themselves untouched**, they are how the
grader finds your answers. Anything you write outside the markers is ignored, so add extra
prose, tables or your threading plot freely.

Numbers in section 1 must **match `submission/phase2_report.json`** — copy them across, do not
retype from memory; they are checked against it.

---

## 1. Your measurements

| | Value |
|---|---|
| Naive attention, ms (`attention.naive_ms`) | <!--answer:attn_naive_ms-->374.647<!--/answer--> |
| Vectorized attention, ms (`attention.vectorized_ms`) | <!--answer:attn_vectorized_ms-->1.156<!--/answer--> |
| CPU training, ms for 100 steps (`devices.cpu_ms`) | <!--answer:cpu_ms-->481900.8<!--/answer--> |
| GPU training, ms for 100 steps (`devices.gpu_ms`) | <!--answer:gpu_ms-->13257.1<!--/answer--> |
| Final training loss (`training.final_loss`) | <!--answer:final_loss-->1.4307<!--/answer--> |

**The thread sweep.** Give your four timings from `threads` — 1, 2, 4 and 8 threads — and say
which count was fastest. (verbatim: the numbers, in order)

<!--answer:thread_sweep-->
1 thread: 2545.818 ms
2 threads: 2523.683 ms
4 threads: 2485.339 ms
8 threads: 3008.299 ms
Fastest: 4 threads (2485.339 ms)
<!--/answer-->

---

## 2. Intra-node parallelism, three ways

**Why vectorizing wins.** Both attention implementations do the same arithmetic and produce the
same result (the unit test checks that). Explain where the naive version's time actually goes,
and what the vectorized one does differently — in terms of the machine, not the Python.
(~70 words)

<!--answer:vectorization_why-->
The naive version spends most of its time on pointer chasing, cache misses, and scalar instructions that process one float at a time. The vectorized implementation stores data contiguously in memory and delegates dense matrix multiplications to an optimized BLAS library. This allows the CPU to use SIMD (AVX2) instructions, processing 8 single-precision floats per cycle while keeping execution pipelines saturated and minimizing memory access latency.
<!--/answer-->

**Where the thread curve flattens, and why.** Using your sweep: more threads stopped helping —
or started hurting — at some point. Say where, and give the reason. Your Colab runtime's vCPU
count is part of the answer; so is what the threads have to coordinate on. (~80 words)

<!--answer:threading_plateau-->
The curve flattened immediately and performance severely degraded at 8 threads (increasing from 2485.339 ms at 4 threads to 3008.299 ms). The standard Colab runtime allocates only 2 physical vCPUs, so spawning 8 threads introduces heavy CPU context switching and scheduler contention. Furthermore, multi-threaded operations on shared memory suffer from synchronization barriers, cache line invalidation (false sharing), and memory bus bandwidth saturation, adding coordination overhead that outweighs any parallel computational gain.
<!--/answer-->

**What made the GPU faster.** The model, the data and the step count were identical; only the
device changed. Say what about a transformer training step suits a GPU, and name one workload
that would *not* see this speed-up. (~70 words)

<!--answer:gpu_why-->
Transformer steps are dominated by massive, independent matrix multiplications with high arithmetic intensity, perfectly matching the GPU's thousands of SIMT CUDA cores and high-bandwidth memory (HBM). A workload that would not see this speed-up is autoregressive generation with batch size 1, where sequential token dependencies and small matrix-vector operations starve the GPU cores and leave execution latency-bound.
<!--/answer-->

---

## 3. The question

**Where does parallelism stop helping, and why?** Pull the three experiments together. Your
answer should distinguish the *kinds* of limit you met — work that cannot be divided, hardware
you do not have, coordination that costs more than it saves, data that has to move — and say
which one bound each experiment. One concrete number from section 1 per claim. (~150 words)

<!--answer:parallelism_limit-->
Parallelism stops helping when coordination, hardware saturation, or memory bandwidth dominate raw computation.
First, vectorization achieved a massive 324.09x speedup (1.156 ms vs 374.647 ms) because dense matrix multiplication is embarrassingly parallel and fits modern SIMD registers.
Second, CPU threading hit hardware limits and coordination costs: going from 1 to 4 threads barely reduced latency (2545.818 ms to 2485.339 ms), and 8 threads degraded to 3008.299 ms. The Colab instance provides only 2 vCPUs, meaning extra threads cause CPU context-switching overhead and saturate memory bus bandwidth.
Third, the GPU trained 100 steps in 13257.1 ms compared to 481900.8 ms on CPU (~36x speedup) by leveraging thousands of SIMT cores for batch matrix operations. However, according to Amdahl's law, acceleration stops helping when workloads include serial bottlenecks, kernel launch latency, or excessive host-to-device PCI-e transfers that leave cores idle.
<!--/answer-->

**What you would measure next.** One experiment this phase did not run that would sharpen the
answer, and what result would change your mind. (~50 words)

<!--answer:next_experiment-->
I would measure GPU step time across varying batch sizes (from 1 to 128). If batch size 1 showed no speedup over a multi-threaded CPU due to kernel launch overhead, it would confirm that high parallelism requires sufficient batch concurrency to amortize transfer and dispatch costs.
<!--/answer-->
