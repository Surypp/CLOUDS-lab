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
| Naive attention, ms (`attention.naive_ms`) | <!--answer:attn_naive_ms-->464.546<!--/answer--> |
| Vectorized attention, ms (`attention.vectorized_ms`) | <!--answer:attn_vectorized_ms-->3.128<!--/answer--> |
| CPU training, ms for 100 steps (`devices.cpu_ms`) | <!--answer:cpu_ms-->461183.3<!--/answer--> |
| GPU training, ms for 100 steps (`devices.gpu_ms`) | <!--answer:gpu_ms-->16090.8<!--/answer--> |
| Final training loss (`training.final_loss`) | <!--answer:final_loss-->1.3992<!--/answer--> |

**The thread sweep.** Give your four timings from `threads` — 1, 2, 4 and 8 threads — and say
which count was fastest. (verbatim: the numbers, in order)

<!--answer:thread_sweep-->
1 thread: 2406.632 ms, 2 threads: 2865.359 ms, 4 threads: 2894.274 ms, 8 threads: 2356.522 ms. Fastest: 8 threads, but only 2% faster than 1 thread.
<!--/answer-->

---

## 2. Intra-node parallelism, three ways

**Why vectorizing wins.** Both attention implementations do the same arithmetic and produce the
same result (the unit test checks that). Explain where the naive version's time actually goes,
and what the vectorized one does differently — in terms of the machine, not the Python.
(~70 words)

<!--answer:vectorization_why-->
The naive version runs 2 x 256 x 256 = 131,072 Python loop iterations, and each one calls NumPy on a single 64-element vector. Most of the time will g into the interpreter and the overhead of each small call, and not into the arithmetic part at all. The vectorized version makes one call for Q @ K.T and one for weights @ V. These run in compiled BLAS code that uses AVX SIMD instructions and reuses data in cache. Here are the results: 464.546 ms vs 3.128 ms (about 148x faster).
<!--/answer-->

**Where the thread curve flattens, and why.** Using your sweep: more threads stopped helping —
or started hurting — at some point. Say where, and give the reason. Your Colab runtime's vCPU
count is part of the answer; so is what the threads have to coordinate on. (~80 words)

<!--answer:threading_plateau-->
The curve is flat from the start. 2 threads (2865.359 ms) and 4 threads (2894.274 ms) were about 19-20% slower than 1 thread (2406.632 ms), and 8 threads (2356.522 ms) were 2% faster, which is within noise. I checked on Colab, the runtime has only 2 vCPUs, so there is almost no extra compute to use. The threads also have to synchronise after every operation and share memory bandwidth, and with 4 or 8 threads the CPU is overloaded.
<!--/answer-->

**What made the GPU faster.** The model, the data and the step count were identical; only the
device changed. Say what about a transformer training step suits a GPU, and name one workload
that would *not* see this speed-up. (~70 words)

<!--answer:gpu_why-->
A training step is composed of large dense matrix multiplies (Q/K/V projections, attention scores, MLP). They are done in batches of 32 x 256 tokens (forward and backward). Each output element can be computed independently, so the T4 runs them in parallel on thousands of CUDA cores with high memory bandwidth. 100 steps took 461183.3 ms on CPU and 16090.8 ms on GPU, about 29x faster. A workload that would not benefit: generating text one token at a time with batch size 1, where each kernel is tiny and launch overhead and host-device transfers dominate (because batch size one favorizes latency).
<!--/answer-->

---

## 3. The question

**Where does parallelism stop helping, and why?** Pull the three experiments together. Your
answer should distinguish the *kinds* of limit you met — work that cannot be divided, hardware
you do not have, coordination that costs more than it saves, data that has to move — and say
which one bound each experiment. One concrete number from section 1 per claim. (~150 words)

<!--answer:parallelism_limit-->
Each experiment was limited by something different. 

First, in the attention test, the naive version was limited by serial work: Python runs the 131,072 loop steps one by one and does not use SIMD. Using whole matrices with BLAS fixed this (464.546 ms to 3.128 ms). 

Then, In the thread test, the limit was the hardware and the coordination between threads. Colab has only 2 vCPUs, so 2 threads were slower than 1 (2865.359 vs 2406.632 ms), and 8 threads were only 2% faster. So the threads spent more time waiting for each other and sharing memory than doing useful work. On the GPU, the limit was moving data and having too little work per step. The GPU was 29x faster (461183.3 vs 16090.8 ms), not 50x or more.

It is worth mentionning that the model is small (4,935,680 parameters, batch 32), so starting GPU operations and copying each batch to the GPU take a big part of each step.
<!--/answer-->

**What you would measure next.** One experiment this phase did not run that would sharpen the
answer, and what result would change your mind. (~50 words)

<!--answer:next_experiment-->
I would run the CPU/GPU comparison but with larger batch sizes (32, 128, 512) and compare the time per token. If the GPU speedup grows well above 29x with bigger batches, the GPU was starved by the small batch. If no, the problem is elsewere (maybe bandwidth??)
<!--/answer-->
