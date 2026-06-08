# Chapter 4 — CUDA (Intro + How It Works + Streams)

> Source: `CUDA Lecture 1.pdf` (Cyril Zeller, NVIDIA — CUDA C/C++ Basics, SC 2011); `How CUDA Programming Works.pdf` (Stephen Jones, NVIDIA — GTC 2022); `Streams and Concurrency Webinar.pdf` (Steve Rennich, NVIDIA).

## Bird's eye view

- **CUDA** = NVIDIA's parallel-computing platform that exposes the GPU for general-purpose computation while retaining performance. CUDA C/C++ is **standard C/C++ + a small set of extensions** plus an API (`nvcc` compiler).
- **Heterogeneous model**: `host` (CPU + host memory) and `device` (GPU + device memory) are separate. Data must be **explicitly transferred** across the PCI bus (`cudaMemcpy`).
- **Three-step processing flow**: (1) copy input H→D, (2) launch kernel, GPU caches on-chip and executes, (3) copy result D→H.
- **Thread hierarchy** = `thread → warp (32 threads) → block → grid`. Built-ins: `threadIdx`, `blockIdx`, `blockDim`, `gridDim` (all up to 3D). Global index: `int i = threadIdx.x + blockIdx.x * blockDim.x`.
- **Kernel launch**: `kernel<<<gridDim, blockDim>>>(args)`. The triple chevrons mark a host→device call. Function qualifiers: `__global__` (device, callable from host), `__device__` (device, callable from device), `__host__` (host).
- **SIMT** (Single Instruction, Multiple Threads): every thread in a kernel runs the **same program**; the warp (32 threads) is the **vector unit** of the GPU — they execute in lockstep. Branch divergence inside a warp serialises the diverging paths.
- **Why CUDA is the way it is = physics**: the GPU is a *throughput* engine, but **DRAM bandwidth**, not FLOPS, is the binding constraint. Ampere A100: 19.5 FP32 TFLOP/s, but only 1555 GB/s memory bandwidth → bandwidth-derived FP64 ceiling ≈ 194 GFLOP/s, **50× less than the peak**.
- **Bandwidth amplification via coalescing**: DRAM is row-addressable — reading one byte costs almost as much as reading a whole row (page). The 32 threads of a warp issue addresses **together**, so contiguous accesses combine into one wide transaction. Strided/random accesses waste bandwidth.
- **Latency hiding via massive parallelism**: a memory load takes hundreds of cycles. The GPU does **not** rely on caches — it **oversubscribes** the SM with many warps and swaps the active warp on every stall. As long as enough warps are resident, the math units stay busy.
- **Occupancy** = (active warps per SM) / (max warps per SM). Limited by the three per-SM resources: **threads (≤2048), registers (65 536), shared memory (160 kB)** on A100. Higher occupancy → more latency hiding → more performance (often the single biggest tuning lever).
- **Block independence**: CUDA does **not** guarantee the order of execution of blocks and provides **no inter-block communication** during a kernel — that's what lets the runtime distribute blocks freely across SMs and scale across GPUs.
- **Shared memory** (`__shared__`) = small, fast, on-chip scratchpad shared between threads of a block. `__syncthreads()` is a **barrier within the block** preventing RAW/WAR/WAW hazards.
- **Asynchronous engines**: kernel launches return immediately; `cudaMemcpyAsync` does not block. The GPU has separate engines for **kernel execution** and **copy (each direction)** — they can run **in parallel** if the work is on different streams.
- **Streams** = ordered queues of GPU operations. Different streams can run **concurrently** (Fermi 2.0+: up to 16 kernels + 2 async copies in opposite directions + CPU work). The **default stream (stream 0)** is implicitly synchronising — using it kills concurrency.
- **Pinned (page-locked) host memory** (`cudaMallocHost`) is **required** for `cudaMemcpyAsync` to actually overlap with compute. Events (`cudaEvent_t`) provide fine-grained timing and cross-stream synchronisation (`cudaStreamWaitEvent`).

---

## 0. Course context

- HPC course, ENSIA 4th year. Chapter 4 brings together three NVIDIA decks: an introductory tutorial (Zeller), a deep "why does this exist" architecture talk (Jones, GTC 2022), and a hands-on streams webinar (Rennich).
- The progression is **what** (Zeller) → **why** (Jones) → **how to overlap I/O and compute** (Rennich). Exam answers should be able to move freely between code-level API and bandwidth/occupancy reasoning.

---

# Part 1 — CUDA C/C++ Basics (Zeller deck)

## 1. What CUDA is

- **CUDA Architecture**: NVIDIA's hardware/software design exposing the GPU for **general-purpose computing** while keeping graphics-grade performance.
- **CUDA C/C++**: industry-standard C/C++ + a small set of extensions for heterogeneous programming, with straightforward APIs to manage devices and memory.
- Compiled with **`nvcc`**, which splits source into:
  - **Device code** (`__global__`, `__device__` functions) → NVIDIA compiler.
  - **Host code** (everything else) → standard compiler (`gcc`, `cl.exe`).
- A program with **no device code** can still be compiled by `nvcc` — it just behaves like ordinary C.

## 2. Heterogeneous computing

| Term | Meaning |
|---|---|
| **Host** | The CPU and host memory (DRAM on the motherboard). |
| **Device** | The GPU and device memory (on-board DRAM, e.g. HBM). |
| **PCI bus** | Bridge between host and device — slow compared to on-chip bandwidth. |

**Three-step processing flow** for a kernel:

1. Copy input data from CPU memory to GPU memory.
2. Load the GPU program and execute, caching data on-chip for performance.
3. Copy the result from GPU memory back to CPU memory.

Host pointers must not be dereferenced in device code, and device pointers must not be dereferenced in host code — they live in disjoint address spaces.

## 3. Kernels, qualifiers, and launch syntax

- **Kernel** = a function that runs on the device, written with `__global__`:

  ```
  __global__ void add(int *a, int *b, int *c) { *c = *a + *b; }
  ```

- **Launch syntax**: `kernel<<<gridDim, blockDim>>>(args);` — the triple angle brackets mark a host→device call. `gridDim` blocks of `blockDim` threads are spawned.
- **Function qualifiers**:

| Qualifier | Runs on | Callable from |
|---|---|---|
| `__global__` | Device | Host (the "kernel") |
| `__device__` | Device | Device |
| `__host__` | Host | Host (default) |

## 4. Thread hierarchy

The kernel is launched as a **grid of blocks of threads**. `blockIdx`, `threadIdx`, `blockDim`, `gridDim` are 3D built-in variables (only `.x` shown in the simple cases).

| Level | Description | Built-in |
|---|---|---|
| **Thread** | Individual lane of execution; runs the kernel. | `threadIdx` |
| **Warp** | Group of **32 threads** executed in lockstep (SIMT vector unit). Not a CUDA language concept but a hardware reality. | — |
| **Block** | Group of threads that share memory and can synchronise. Always runs on a single SM. | `blockIdx`, `blockDim` |
| **Grid** | All blocks of a single kernel launch. | `gridDim` |

**Standard global-index pattern** (1D):

```math
\text{idx} = \text{threadIdx.x} + \text{blockIdx.x} \cdot \text{blockDim.x}
```

For arbitrary `N`: launch `(N + M − 1) / M` blocks of `M` threads and guard with `if (idx < N)`.

## 5. Memory management API

- Host and device memory are **separate**. CUDA mirrors the C memory API:

| Standard C | CUDA equivalent | Purpose |
|---|---|---|
| `malloc` | `cudaMalloc` | Allocate device memory |
| `free` | `cudaFree` | Free device memory |
| `memcpy` | `cudaMemcpy(dst, src, size, kind)` | Copy between host/device |

- `kind` is one of `cudaMemcpyHostToDevice`, `cudaMemcpyDeviceToHost`, `cudaMemcpyDeviceToDevice`.
- `cudaMemcpy` **blocks** the CPU until the copy is complete (and waits for prior CUDA work to finish).

## 6. Why threads? Cooperation via shared memory

- Threads in a block can **communicate** and **synchronise** — blocks cannot. This is the whole reason threads exist as a level distinct from blocks.
- **`__shared__`** declares an on-chip scratchpad allocated **per block**:
  - Extremely fast (orders of magnitude faster than global memory).
  - Acts like a **user-managed L1 cache**.
  - Not visible to threads in other blocks.
- Classic use: **1D stencil**. Each block loads `blockDim.x + 2·radius` elements (block + left/right halo) into shared memory once, then every thread reads from shared memory `2·radius+1` times to compute its output.

## 7. `__syncthreads()` and data hazards

- **`__syncthreads()`** is a **barrier within the block** — all threads must reach it before any can proceed.
- Used to prevent **RAW / WAR / WAW** hazards on shared memory (e.g. after the cooperative load in a stencil, before reading neighbours).
- Conditional `__syncthreads()` is unsafe: the condition must be **uniform across the block** or the kernel deadlocks.

## 8. Asynchronicity, errors, devices

- **Kernel launches are asynchronous** — control returns to the host immediately. The host **must synchronise** before reading results.

| API | Behaviour |
|---|---|
| `cudaMemcpy` | Blocks CPU until copy completes (waits for prior CUDA calls). |
| `cudaMemcpyAsync` | Does not block CPU. |
| `cudaDeviceSynchronize` | Blocks host until **all** preceding CUDA calls finish. |

- Every CUDA API call returns `cudaError_t`. `cudaGetLastError()` and `cudaGetErrorString()` retrieve the last error — errors can come from the call itself **or from an earlier asynchronous launch**.
- **Device management**: `cudaGetDeviceCount`, `cudaSetDevice(i)`, `cudaGetDeviceProperties` — a single host thread may manage multiple GPUs.

## 9. Compute capability and primitives skipped

- **Compute capability** = version number describing what the device supports (number of registers, sizes of memories, features).

| Capability | Selected features |
|---|---|
| 1.0 | Fundamental CUDA support |
| 1.3 | Double precision, atomics, improved memory accesses |
| 2.0 (Fermi) | Caches, FMA, 3D grids, ECC, P2P, **concurrent kernels/copies**, function pointers, recursion |
| 3.5+ (Kepler) | Hyper-Q, dynamic parallelism |

- **Textures** (briefly): read-only objects with a dedicated cache, hardware-accelerated linear/bilinear/trilinear filtering, 1D/2D/3D addressing, configurable boundary handling (wrap/clamp).

---

# Part 2 — How CUDA Programming Works (Jones, GTC 2022)

> Thesis: **You use a GPU for performance, performance is limited by physics, so CUDA is the way it is because of physics.** The two physical truths that shape everything are (a) DRAM is bandwidth-limited and row-addressed, and (b) memory latency is hidden by oversubscribing parallelism.

## 10. Bandwidth, not FLOPS, is the constraint

- Ampere A100 spec sheet: 108 SMs, 221 184 threads, **19.5 TFLOP/s FP32**, **9.7 TFLOP/s FP64 (non-tensor)**, **1555 GB/s** HBM2 memory bandwidth, 40 MB L2.
- An SM can load 64 bytes per clock cycle. Peak **requested** rate:

```math
\text{Peak request} = 64\,\text{B} \times 108\,\text{SMs} \times 1410\,\text{MHz} = 9750\,\text{GB/s}
```

- HBM2 provides only **1555 GB/s** ⇒ request:supply ratio ≈ **6.3×**. The cores can ask for **6× more** than memory can deliver.
- **Bandwidth-limited FP64 ceiling** (one FP64 = 8 bytes loaded): `1555 / 8 ≈ 194 GFLOP/s`, i.e. **50× below** the 9.7 TFLOP/s peak. So in many kernels you cannot reach the FLOP peak no matter what — you are **memory-bound**.

## 11. How DRAM physically works → memory pages

- DRAM cell = one capacitor + one transistor on a wordline/bitline grid. Reads are **destructive**: activating the row drains capacitors into sense amplifiers, then the row has to be rewritten.
- An address splits into **(row, column)**. To read any single bit:
  1. Decode row → **activate** the entire row into sense amplifiers (slow, destroys row).
  2. Decode column → tap the column from the amps (fast, non-destructive).
  3. Subsequent reads to the **same row** are cheap; switching rows is expensive.
- Consequence: the smallest practical unit of transfer is a **page** (typically ~1024 bytes), not a byte. **Reading one byte costs as much bandwidth as reading the whole page.**

## 12. Warps, coalescing, and bandwidth amplification

- A thread block is broken up into **warps of 32 threads** — the warp is the GPU's **vector element**.
- A single instruction in a warp issues **32 lane requests at once**. If each thread loads an 8-byte `float2`:

```math
\text{warp traffic} = 32 \times 8 = 256\,\text{B per warp}, \quad 4\,\text{warps} \times 256 = 1024\,\text{B} = \text{one page}
```

- The **coalescer** combines the 32 lane addresses into the minimum number of memory transactions. If threads access **contiguous** addresses → **one wide transaction, full bandwidth**. If addresses are strided/random → multiple transactions, each fetches a whole page but only uses a few bytes → bandwidth wasted.
- A100 measured throughput **collapses from ~1700 GB/s at stride 8 (coalesced) to ~100 GB/s at stride 1024+** — a factor of ~17×.
- Key reframing (Jones): what looks like **random reads from one thread's view** is actually **adjacent reads of whole pages** when you remember 32 threads issue together.

## 13. Execution hierarchy revisited

Pictorially: the workload is a **grid of blocks of threads**.

| Step | What CUDA does |
|---|---|
| 1. Define grid of work | Split the problem into equal-sized chunks. |
| 2. Each block is independent | No inter-block ordering, no inter-block data exchange during a kernel. |
| 3. Block → single SM | A block is **placed on one SM** for its entire lifetime; threads of a block run together. |
| 4. SM runs many blocks | Each SM keeps several resident blocks; their warps interleave on the warp schedulers. |

Why? Block independence is what lets the scheduler distribute blocks across **any number of SMs**, on **any future GPU**, without the programmer re-coding. It is the source of CUDA's scalability.

## 14. Resources per block, resources per SM

A block consumes three per-SM resources:

1. **Block size** — threads that must be co-resident.
2. **Shared memory** — `__shared__` declarations, per block.
3. **Registers** — per thread; total `threads_per_block × registers_per_thread`.

A100 SM budget:

| Resource | A100 limit |
|---|---|
| Max threads per SM | **2048** |
| Max warps per SM (= 2048 / 32) | **64** |
| Concurrent warps active (issued per cycle) | 4 |
| Total registers per SM | **65 536** |
| Shared memory per SM | **160 kB** |
| Max L1 cache per SM | 192 kB |
| Threads per warp | 32 |

**How many blocks fit?** A block needing 256 threads + 64 regs/thread + 48 kB shared:

- Threads: `2048 / 256 = 8` block-slots.
- Registers: `65 536 / 16 384 = 4` block-slots.
- Shared: `160 / 48 ≈ 3` block-slots.

The **minimum** of the three (here **3 blocks/SM, shared-memory-limited**) is the **occupancy**.

## 15. Occupancy and latency hiding

```math
\text{Occupancy} = \frac{\text{active warps per SM}}{\text{max warps per SM}}
```

- A DRAM load takes **hundreds of cycles**. Instead of relying on caches, the GPU keeps **many warps in flight** per SM and the warp scheduler **switches to a ready warp** every time the current one stalls on memory.
- Required (rough) parallelism to hide latency (Little's Law):

```math
\text{required\_in\_flight\_bytes} \approx \text{bandwidth} \times \text{latency}
```

- More resident warps ⇒ more chances to find a ready warp ⇒ more latency hidden ⇒ math units stay busy.
- Jones's slogan: **"Occupancy is the most powerful tool for tuning a program."** Reducing shared-memory or registers per block to go from 3 → 4 blocks/SM gave a **33% speed-up** in the example.

## 16. Filling in the gaps

- Different kernels can have **different occupancy bottlenecks** — one kernel may be shared-memory-bound, another register-bound.
- **Cooperative kernel design**: a "blue grid" (256 threads, 64 regs, 48 kB shared, register-limited at 4 blocks) and a "green grid" (512 threads, 32 regs, 0 shared) can coexist on the same SM, because the green kernel uses resources the blue kernel didn't use. Concurrent kernels (Fermi 2.0+ / Hyper-Q) exploit exactly this gap-filling.

---

# Part 3 — Streams and Concurrency (Rennich)

## 17. What concurrency means here

- **Concurrency** = performing multiple CUDA operations *simultaneously*, **beyond the multi-thread parallelism inside a single kernel**:
  - CUDA kernels (`<<<…>>>`)
  - `cudaMemcpyAsync(H→D)` and `cudaMemcpyAsync(D→H)`
  - Work running on the CPU
- **Fermi 2.0+** supports simultaneously: up to **16 kernels on the GPU**, **2 async copies (one per direction)**, and CPU computation.

## 18. Streams

- A **stream** is **a sequence of operations that execute in issue order on the GPU**. Different streams may run **concurrently** and may be **interleaved**.
- Streams are the programming model that exposes the GPU's separate copy and execute engines.

**Concurrency levels** (typical speed-ups):

| Pattern | Speedup |
|---|---|
| Serial (H→D, kernel, D→H, no streams) | 1× |
| 2-way (overlap kernel with one copy direction) | up to **2×** |
| 3-way (overlap kernel with both copies) | up to **3×** |
| 4-way (above + CPU work) | **3×+** |
| 4+ way with split kernels & multi-stage pipeline | depends on tiling |

Real example (tiled DGEMM, Westmere CPU + C2070 GPU): serial 125 Gflops → 2-way 177 → 3-way 262 → 4-way 282 Gflops. Up to **6.6× CPU baseline**; all communication is hidden.

## 19. The default stream (stream 0) — the gotcha

- The default stream is used when **no stream is specified**.
- It is **completely synchronous with respect to host and device**: as if `cudaDeviceSynchronize()` were inserted before **and after** every CUDA operation in stream 0.
- Therefore any operation issued to the default stream **kills concurrency** with all other streams.
- Exceptions (asynchronous w.r.t. host even in stream 0): kernel launches in the default stream, `cudaMemcpy*Async`, `cudaMemset*Async`, intra-device `cudaMemcpy`, and `H→D cudaMemcpy` of ≤ 64 kB.

## 20. Requirements for actual overlap

1. CUDA operations must be in **different, non-zero streams**.
2. `cudaMemcpyAsync` requires the host buffer to be **page-locked (pinned)** memory, allocated with `cudaMallocHost` or `cudaHostAlloc`. Pageable memory forces an internal staging copy and breaks async-ness.
3. Sufficient hardware resources must be available: copies must use different directions, and device resources (shared memory, registers, blocks) must fit.

## 21. Async API quick reference

| Operation | Function |
|---|---|
| Create stream | `cudaStreamCreate(&s)` |
| Destroy stream | `cudaStreamDestroy(s)` |
| Async copy | `cudaMemcpyAsync(dst, src, size, kind, stream)` |
| Pinned host alloc | `cudaMallocHost(&p, size)` / `cudaHostAlloc` |
| Async kernel | `kernel<<<grid, block, smemBytes, stream>>>(...)` |

**Idiomatic async-with-streams skeleton**:

```
cudaStreamCreate(&s1); cudaMallocHost(&host, size);
cudaMemcpyAsync(d_in, host, size, H2D, s1);
kernel<<<..., 0, s2>>>(d_in, d_out);
cudaMemcpyAsync(host, d_out, size, D2H, s3);
some_CPU_method();    // overlaps with everything above
```

## 22. Synchronisation — explicit and implicit

**Explicit synchronisation primitives**:

| Call | Effect |
|---|---|
| `cudaDeviceSynchronize()` | Blocks host until **all** CUDA work is complete (sledgehammer). |
| `cudaStreamSynchronize(s)` | Blocks host until all work in stream `s` is complete. |
| `cudaStreamWaitEvent(s, e)` | Makes stream `s` wait for event `e` (does **not** block the host). |
| `cudaEventSynchronize(e)` | Blocks host until `e` has been reached. |
| `cudaEventQuery(e)` | Non-blocking test: has `e` happened? |

**Events**: `cudaEventCreate`, `cudaEventRecord(e, stream)`, `cudaEventElapsedTime(&ms, start, stop)` — also used for **precise GPU-side timing**.

**Implicit synchronisation** (these silently serialise *all other* CUDA operations — easy to trigger by accident):

- Page-locked memory allocation (`cudaMallocHost`, `cudaHostAlloc`).
- Device memory allocation (`cudaMalloc`).
- Non-async memory ops (`cudaMemcpy`, `cudaMemset` without the `Async` suffix).
- L1 / shared-memory configuration changes (`cudaDeviceSetCacheConfig`).

**Rule of thumb**: do all allocations up-front (outside the timed/concurrent region), then issue only async ops on non-default streams during the hot path.

## 23. Hyper-Q (Kepler 3.5+) — multiple work queues

- Fermi had a **single hardware work queue** between host and GPU; even if you used many streams, false dependencies in that single queue could prevent concurrency.
- **Hyper-Q** introduces multiple (32) hardware work queues, so streams no longer block each other in the queue. The programmer model is unchanged — just more concurrency in practice.

---

## Glossary

- **CUDA** — NVIDIA's parallel computing platform (architecture + C/C++ extensions + runtime API).
- **Host / Device** — the CPU (+ its DRAM) / the GPU (+ its on-board DRAM).
- **`nvcc`** — NVIDIA's compiler driver; splits source into host and device parts.
- **Kernel** — `__global__` function that runs on the device, launched via `kernel<<<gridDim, blockDim>>>(args)`.
- **Thread / Warp / Block / Grid** — the four levels of CUDA's execution hierarchy. Warp = 32 threads in SIMT lockstep.
- **`blockIdx`, `threadIdx`, `blockDim`, `gridDim`** — built-in 3D index/dimension variables.
- **SM (Streaming Multiprocessor)** — physical compute unit of the GPU; runs whole blocks. A100 has 108 SMs.
- **SIMT** (Single Instruction, Multiple Threads) — execution model where all 32 threads of a warp execute the same instruction lockstep; divergent branches serialise.
- **Warp divergence** — threads in the same warp taking different control paths; reduces throughput proportionally.
- **Coalescing** — the hardware's combining of 32 lane addresses in a warp into the smallest number of memory transactions.
- **Memory page (DRAM row)** — the unit of physical DRAM transfer (~1024 B on A100), set by the row-activate cost.
- **Occupancy** — active warps per SM ÷ max warps per SM (= 64 on A100). The latency-hiding budget.
- **Latency hiding** — covering memory-load stalls by switching to other resident warps.
- **`__global__` / `__device__` / `__host__`** — function qualifiers: device-from-host / device-from-device / host-from-host.
- **`__shared__`** — qualifier for on-chip per-block scratchpad memory.
- **`__syncthreads()`** — block-level barrier; prevents RAW/WAR/WAW hazards on shared memory.
- **Global memory** — device DRAM; large, slow, shared by all threads of a kernel.
- **Shared memory / L1** — small, fast, per-SM on-chip memory; a user-managed cache.
- **Registers** — per-thread fastest storage; in finite supply (65 536/SM on A100).
- **Texture memory** — read-only memory path with dedicated cache, filtering hardware, and boundary modes.
- **Compute capability** — version number for a GPU's feature set (1.0 → 2.0 Fermi → 3.5 Kepler …).
- **Stream** — ordered queue of GPU operations; different streams can run concurrently.
- **Default stream (stream 0)** — implicit synchronising stream; using it kills concurrency.
- **`cudaMemcpyAsync`** — non-blocking memcpy; requires **pinned** host memory to overlap.
- **Pinned / page-locked memory** — host memory that cannot be swapped out (`cudaMallocHost`); needed for DMA-driven async copy.
- **Event** — handle for fine-grained GPU timing and cross-stream synchronisation.
- **Hyper-Q** — Kepler 3.5+ feature providing multiple hardware work queues so streams don't false-share a single queue.

---

## Memory hierarchy at a glance

| Space | Scope | Speed | Lifetime | Typical use |
|---|---|---|---|---|
| **Registers** | Per thread | Fastest | Thread | Local scalars, loop counters |
| **Local memory** | Per thread | Slow (lives in global) | Thread | Register-spill, large per-thread arrays |
| **Shared memory** | Per block | Very fast (on-chip) | Block | Cooperative loads, stencils, reduction |
| **L1 / L2 cache** | SM / device | Fast | Hardware-managed | Implicit caching of global accesses |
| **Global memory** | Whole device | Slow (DRAM) | Application | Input/output arrays; `cudaMalloc` |
| **Constant memory** | Whole device | Fast when broadcast | Application | Read-only kernel parameters |
| **Texture memory** | Whole device | Cached, filtered | Application | Image-like 1D/2D/3D read-only data |

---

## Likely exam targets

1. **Explain heterogeneous computing**: define host and device, draw the 3-step processing flow (H→D copy, kernel, D→H copy), explain why explicit transfers exist.
2. **Kernel launch and qualifiers**: write `kernel<<<gridDim, blockDim>>>(args)`, give the meaning of `__global__`, `__device__`, `__host__`, and the role of `nvcc`.
3. **Thread hierarchy and indexing**: list thread → warp → block → grid; give the formula `idx = threadIdx.x + blockIdx.x * blockDim.x` and explain the `(N+M−1)/M`-block launch with bounds check for arbitrary `N`.
4. **SIMT and warp divergence**: define warp (32 threads, lockstep), explain how a branch inside a warp serialises both paths, contrast with classical SIMD.
5. **Shared memory + `__syncthreads()`**: walk through the 1D stencil — cooperative load with halo, barrier, then compute. Justify why the barrier is mandatory and what hazards it prevents.
6. **Memory management API**: list `cudaMalloc`, `cudaMemcpy(kind)`, `cudaFree`; explain the `kind` enum values; contrast `cudaMemcpy` (blocking) with `cudaMemcpyAsync`.
7. **"Why is CUDA the way it is?"**: state the bandwidth-vs-FLOPS argument with the A100 numbers (19.5 TFLOP/s vs 1555 GB/s, 6.3× gap), and explain why DRAM physics forces page-sized transfers.
8. **Coalescing**: explain how 32 threads of a warp issue addresses together, why contiguous addresses → one wide transaction, and why a stride collapses bandwidth (cite the ~17× collapse on A100).
9. **Latency hiding & occupancy**: define occupancy, state that it is limited by **threads / registers / shared memory**, give the A100 budget (2048 / 65 536 / 160 kB), and explain how oversubscribing warps hides DRAM latency.
10. **Streams and overlap**: define a stream, state the requirements for overlap (different non-zero streams + pinned host memory + sufficient resources), draw the 3-way pipeline timeline (HD/Kernel/DH overlap), and explain the default-stream gotcha.
11. **Events and sync**: list `cudaDeviceSynchronize` / `cudaStreamSynchronize` / `cudaStreamWaitEvent`; explain `cudaEventRecord` + `cudaEventElapsedTime` for GPU timing.
12. **Implicit synchronisation**: list operations that silently serialise (`cudaMalloc`, `cudaMallocHost`, non-async memcpy, cache-config change) and explain why allocations should be hoisted out of the concurrent region.

---

## Pitfalls

- **Dereferencing a device pointer in host code (or vice versa)** silently breaks: pointers travel between spaces but the bytes don't.
- **Forgetting the bounds check** `if (idx < N)` when `N` is not a multiple of `blockDim.x` — last block reads/writes past the array.
- **Treating `__syncthreads()` as a global barrier** — it is **per block** only; there is no in-kernel inter-block sync (the only one is the kernel boundary itself).
- **Conditional `__syncthreads()` with non-uniform conditions** → deadlock; the condition must be uniform across the block.
- **Assuming kernels are synchronous** — they aren't; the host runs ahead. You must `cudaDeviceSynchronize()` (or copy back) before reading results.
- **Diagnosing a stale error**: a kernel error shows up on the **next** API call. Always check `cudaGetLastError()` right after the launch and after a `cudaDeviceSynchronize()`.
- **Misreading FLOP peak**: most real kernels are **bandwidth-bound**, not FLOP-bound. The bandwidth-derived ceiling is the right back-of-envelope number, not the FLOP peak.
- **Strided / random access patterns** waste bandwidth at the page granularity — even a single byte miss costs a whole row activation.
- **Using `cudaMemcpyAsync` from pageable memory** — it silently falls back to a blocking staged copy, killing concurrency.
- **Issuing anything in the default stream during the concurrent region** — implicit `cudaDeviceSynchronize()` before/after kills overlap.
- **Calling `cudaMalloc` in the hot loop** — it implicitly synchronises *all* other CUDA work. Allocate up front.
- **Tuning for FLOPs and ignoring occupancy** — reducing shared-memory or registers per block can raise occupancy from 3 → 4 blocks/SM and give a 33%+ speed-up without any algorithmic change.
