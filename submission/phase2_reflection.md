# Phase 2 — Reflection: where does parallelism stop helping?

## 1. Your measurements

| | Value |
|---|---|
| Naive attention, ms (`attention.naive_ms`) | <!--answer:attn_naive_ms-->694.589<!--/answer--> |
| Vectorized attention, ms (`attention.vectorized_ms`) | <!--answer:attn_vectorized_ms-->11.995<!--/answer--> |
| CPU training, ms for 100 steps (`devices.cpu_ms`) | <!--answer:cpu_ms-->487526.5<!--/answer--> |
| GPU training, ms for 100 steps (`devices.gpu_ms`) | <!--answer:gpu_ms-->13850.9<!--/answer--> |
| Final training loss (`training.final_loss`) | <!--answer:final_loss-->1.4811<!--/answer--> |

**The thread sweep.**

<!--answer:thread_sweep-->
1 thread: 2315.512 ms, 2 threads: 2713.093 ms, 4 threads: 2712.213 ms, 8 threads: 3040.988 ms. The fastest was 1 thread.
<!--/answer-->

---

## 2. Intra-node parallelism, three ways

**Why vectorizing wins.**

<!--answer:vectorization_why-->
The naive version walks over positions in a Python loop and computes one small dot product at a time. Each iteration pays interpreter overhead, touches memory in small pieces with poor cache reuse, and uses the CPU's wide vector units at most by accident. The vectorized version hands whole matrices to a compiled BLAS matrix multiply, which uses SIMD instructions (several floats per instruction), blocks the data for the cache and avoids the Python loop. Same arithmetic, 694.6 ms against 12.0 ms, a 57.9x speedup.
<!--/answer-->

**Where the thread curve flattens, and why.**

<!--answer:threading_plateau-->
More threads never helped in my sweep: 1 thread was fastest at 2315.5 ms, 2 threads took 2713.1 ms (about 17% slower), 4 threads 2712.2 ms and 8 threads 3041.0 ms (about 31% slower). A Colab CPU runtime has only a few vCPUs (typically 2; I did not check the exact count), so beyond that the threads share the same cores and compete for them. The model is also tiny (4.9M parameters), so each matrix multiply is small and the work per thread is too little to pay for the scheduling and synchronization at every operation. Coordination cost exceeded the gain.
<!--/answer-->

**What made the GPU faster.**

<!--answer:gpu_why-->
100 training steps took 487526.5 ms on the CPU and 13850.9 ms on the GPU, a 35.2x speedup. A transformer training step is dominated by large dense matrix multiplications (attention and the MLP) applied to a whole batch: the same operation on thousands of independent elements, which suits thousands of GPU cores and high memory bandwidth. A workload that would not see this speedup is one with a strict dependency chain or very small batches, such as a sequential recurrent loop at batch size 1, or a task dominated by data transfer.
<!--/answer-->

---

## 3. The question

<!--answer:parallelism_limit-->
Parallelism stops helping when the work cannot be divided, when the hardware is missing, when coordination costs more than it saves, or when data has to move, and each experiment hit a different limit. Vectorization (SIMD) gave a 57.9x speedup (694.6 ms to 12.0 ms) because attention is perfectly divisible, and it is bounded by vector width and memory bandwidth. Threading was bound by hardware I do not have and by coordination: 8 threads took 3041.0 ms against 2315.5 ms for 1 thread, about 31% slower, since a few vCPUs and tiny matrices make synchronization cost more than it saves. The GPU gave 35.2x (487526.5 ms to 13850.9 ms), but even there the 2000 training steps are sequential, because each step needs the weights from the previous one (final loss 1.4811). That part cannot be parallelized, and host-to-device transfers add a further cost.
<!--/answer-->

**What you would measure next.**

<!--answer:next_experiment-->
I would repeat the GPU-versus-CPU comparison at several batch sizes (1, 8, 64) and the thread sweep on a runtime with more vCPUs. If the GPU speedup shrinks toward 1x at small batches, data movement and launch overhead are the real limit; if threads help on 8 or more vCPUs, the missing hardware was what bounded my sweep.
<!--/answer-->
