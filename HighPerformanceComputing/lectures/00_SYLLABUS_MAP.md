# HPC — Syllabus Skeleton (Phase 0)

> Course: High Performance Computing — ENSIA, 4th year AI, 2025-2026.
> Sources: 13 lecture PDFs (Lecture 1 + Patterns + 2× OpenMP + 3× OpenMPI + 3× CUDA + 3× linear-algebra case studies).
> Out of scope here: labs (separate).
> Use this as a mental scaffold. **Active recall** does the real work — re-derive the formulas, sketch the diagrams from memory.

---

## Ch 1 — Introduction to HPC + Parallel Patterns
- **HPC = parallel computing**: clusters of multi-core nodes + GPUs, ranked by **TOP500** using **LINPACK / HPL** (FLOP/s); we are at **Exa-scale** (~10^18 FLOP/s)
- **Why parallel** = Moore's Law continues but **Dennard scaling died ~2005** at ~3 GHz → more cores, not faster ones. "Free lunch is over"
- Two motivations: **need for speed** + **need for memory** (LLM training, climate, the "AI memory wall")
- Dally's slogan: **Performance = parallelism, Efficiency = locality**
- **Flynn's taxonomy**: SISD / SIMD / MISD / MIMD
- **Memory architectures**: shared (SMP) vs distributed (clusters) vs hybrid (modern reality)
- **Programming models** map to memory model: OpenMP (shared) / MPI (distributed) / CUDA (GPU)
- **Amdahl's Law** (fixed problem):
  ```math
  S(p) = \frac{1}{(1-f) + f/p}, \qquad \lim_{p \to \infty} S = \frac{1}{1-f}
  ```
- **Gustafson's Law** (scaled problem): `S(p) = (1-f) + p·f` — more optimistic, justifies large-scale HPC
- **Strong vs weak scaling**: strong = same problem, vary `p`; weak = grow problem with `p`
- **Parallel patterns** = general solutions to recurring problems (not ready-made code)
- **McCool classification**: structural (Map, Partition, Pipeline, Fork-Join, Master-Worker, Geometric Decomposition, Recursive Splitting) vs computational (Map, Reduce, Scan, Stencil, Gather, Scatter, Pack)
- **6 core HPC patterns**: Embarrassingly Parallel, Partition, Master-Worker, Stencil, Reduce, Scan
- **Berkeley 7 Dwarfs / 13 Motifs** (Colella 2004 → 2006): Dense LA, Sparse LA, FFT, N-Body, Structured Grids, Unstructured Grids, Monte Carlo (+ Combinational Logic, Graph Traversal, FSM, DP, Backtrack/B&B, Graphical Models)

## Ch 2 — OpenMP (Intro + Advanced)
- **Directive-based** shared-memory multithreading via `#pragma` (C/C++/Fortran); ignored by non-OpenMP compilers → code still runs serially
- **Fork-join execution model**: master thread → `#pragma omp parallel` forks team → workers → join at closing brace
- **5 categories**: parallel regions, work-sharing (`for`, `sections`, `single`, `master`), data-sharing clauses, synchronization, runtime API + env
- **Data-sharing clauses**: `shared` (default for globals/outer locals), `private`, `firstprivate`, `lastprivate`, `default(none|shared)`, `reduction(op:list)`
- **Work-sharing**: `#pragma omp for` (loop), `#pragma omp sections` (heterogeneous tasks), `#pragma omp single` (one thread + implicit barrier), `#pragma omp master` (thread 0, no barrier)
- **Loop scheduling**: `schedule(static|dynamic|guided|auto, chunk)`
  - **static**: predictable workload, lowest overhead
  - **dynamic**: load-imbalanced workloads
  - **guided**: large chunks decreasing in size
- **Synchronization hierarchy by cost** (low → high): **atomic** (hardware CAS) → **reduction** (private + merge) → **critical** (OS lock) → **barrier** (whole team waits)
- **Race conditions**: programmer's responsibility; unprotected `sum += ...` = **undefined behaviour**. Fix with `reduction(+:sum)`
- **False sharing**: independent variables on the **same 64-byte cache line** → coherence ping-pong → fix via **padding** (`char pad[60]`) or local accumulation
- **SIMD vs threading**: SIMD = **DLP** (one stream, wide registers, 128-512 bits); threading = **TLP**. Combine with `#pragma omp parallel for simd`. `aligned(a,b,c:64)` clause + AVX-512 (16 floats/cycle)
- **Tasking model**: `#pragma omp task`, `taskwait`, `taskloop`, `task depend(in|out|inout: x)`. Decouples **work generation** from **work execution**. **Work stealing** runtime. Use for irregular workloads, recursion, linked lists, while loops
- **Compile**: `gcc -fopenmp -O2 -march=native code.c -o exec`

## Ch 3 — MPI / OpenMPI (1 + 2 + 3)
- **Distributed memory** model: each rank has its **own RAM**, must exchange data via **explicit messages**
- **SPMD** execution: one binary, many ranks, branch on `rank` within a **communicator**
- **Core skeleton**: `MPI_Init` → `MPI_Comm_rank` / `MPI_Comm_size` → work → `MPI_Finalize`
- **Default communicator**: `MPI_COMM_WORLD`
- **Point-to-point**: `MPI_Send(buf, count, datatype, dest, tag, comm)` + `MPI_Recv(buf, count, datatype, source, tag, comm, status)`
- **Send modes**: standard / buffered / synchronous / ready
- **MPI datatypes** (`MPI_INT`, `MPI_DOUBLE`, ...) abstract over heterogeneous endianness
- **Circular sends deadlock** unless ranks order their send/recv pairs (e.g., even/odd interleave)
- **Collectives** = all ranks in communicator participate; MPI uses **tree algorithms (O(log p))** internally:
  | Pattern | Function | Direction |
  |---|---|---|
  | Sync | `MPI_Barrier` | — |
  | 1 → all | `MPI_Bcast` | broadcast |
  | 1 → all chunks | `MPI_Scatter` | distribute |
  | all → 1 chunks | `MPI_Gather` | collect |
  | all → all chunks | `MPI_Allgather`, `MPI_Alltoall` | exchange |
  | Reduction | `MPI_Reduce`, `MPI_Allreduce` | combine via op |
- **Reduction ops**: `MPI_SUM`, `MPI_PROD`, `MPI_MIN`, `MPI_MAX`, `MPI_LAND`, `MPI_BAND`, ...
- **Blocking blocks CPU**: non-blocking `MPI_Isend` / `MPI_Irecv` return an `MPI_Request` immediately; finish with `MPI_Wait` or poll with `MPI_Test`. Enables **comm/comp overlap**
- **AI scaling wall**: gradient sync (`MPI_Allreduce`) dominates step time in distributed DL training
- **3D parallelism** for DL: **Data Parallel + Tensor Parallel + Pipeline Parallel**, each with its own sub-communicator
- **Physical network topologies**: bus, ring, mesh, **torus**, **fat-tree**, **dragonfly**
- **Logical/virtual topologies**: `MPI_Cart_create`, `MPI_Cart_coords`, `MPI_Cart_shift`, `MPI_Cart_rank` — periodic vs non-periodic
- **Communicator management**: `MPI_Comm_split` (carve by color), `MPI_Comm_dup`, `MPI_Comm_split_type(MPI_COMM_TYPE_SHARED)` for intra-node
- **Hierarchical All-Reduce**: split per-node → local reduce → leader-only inter-node all-reduce → local broadcast (keeps 90% of traffic on the fast intra-node bus)

## Ch 4 — CUDA (Intro + How It Works + Streams)
- **NVIDIA's GPU computing platform**; CUDA C/C++ = standard C/C++ + extensions + API (compiled with `nvcc`)
- **Heterogeneous model**: host (CPU + RAM) + device (GPU + VRAM); explicit transfers via `cudaMemcpy(H2D | D2H | D2D)`
- **3-step flow**: (1) `cudaMemcpy` H→D, (2) launch kernel `kernel<<<grid, block>>>(args)`, (3) `cudaMemcpy` D→H
- **Thread hierarchy**: thread → **warp (32 threads)** → block → grid. Built-ins: `threadIdx`, `blockIdx`, `blockDim`, `gridDim` (all up to 3D)
- **Global index**: `int i = threadIdx.x + blockIdx.x * blockDim.x`
- **Function qualifiers**: `__global__` (device entry, callable from host), `__device__` (device-only), `__host__` (default)
- **SIMT** (Single Instruction, Multiple Threads): warp = vector unit; lockstep execution; **branch divergence within a warp serializes**
- **Memory hierarchy** (fastest → slowest):
  | Memory | Scope | Lifetime | Speed |
  |---|---|---|---|
  | **Register** | thread | thread | fastest |
  | **Shared** (`__shared__`) | block | block | ~L1 |
  | **Constant** | grid (read-only) | app | cached |
  | **Texture** | grid (read-only) | app | cached |
  | **Local** | thread | thread | DRAM (slow) |
  | **Global** | grid | app | DRAM (slowest) |
- **DRAM physics is the constraint**: A100 has 19.5 FP32 TFLOP/s but only 1.55 TB/s memory bandwidth → bandwidth-derived FP64 ceiling ~194 GFLOP/s (**50× below peak**)
- **Memory coalescing**: 32 warp threads issue addresses together → contiguous = 1 wide transaction; strided/random = wasted bandwidth
- **Latency hiding via oversubscription**: GPUs don't rely on big caches — they **oversubscribe** warps and swap active warp on every stall
- **Occupancy** = (active warps per SM) / (max warps per SM). Limited by per-SM resources: threads (≤2048), registers (65536), shared memory (160 kB on A100). **Higher occupancy = more latency hiding = more perf** (biggest tuning lever)
- **Block independence**: no order guarantee, no inter-block communication during a kernel → allows runtime to distribute blocks freely across SMs and scale across GPUs
- **Synchronization**: `__syncthreads()` (block-level barrier), `cudaDeviceSynchronize()` (host waits for device)
- **Async engines**: GPU has separate kernel-exec + copy-H2D + copy-D2H engines → **all 3 can run in parallel** if work is on different streams
- **Streams** = ordered queues of GPU operations. Different streams run concurrently. Fermi 2.0+ supports up to 16 concurrent kernels + 2 async copies (different directions) + CPU work
- **Default stream (stream 0)** is implicitly synchronizing → kills concurrency. Use explicit streams for overlap
- **Pinned (page-locked) host memory** (`cudaMallocHost`) is **required** for `cudaMemcpyAsync` to actually overlap with compute
- **Events** (`cudaEventCreate`, `cudaEventRecord`, `cudaEventSynchronize`, `cudaEventElapsedTime`) → fine-grained timing + cross-stream sync via `cudaStreamWaitEvent`

## Ch 5 — Linear Algebra Case Studies (Dense GEMM + Sparse SpMV)
- **GEMM** `C = A·B` (`A, B, C ∈ ℝ^{n×n}`): `≈ 2n³` flops on `≈ 3n²` data → **arithmetic intensity ~ n** → **compute-bound** for large `n`
- **Naive 1D row partition** forces each proc to load **all of B** → comm scales like `O(n³)`. Bad
- **2D block partition** on `√p × √p` grid → memory per proc `O(n²/p)`, comm `O(n²/√p)`
- **Cannon's algorithm (1969)** — canonical 2D systolic GEMM:
  1. **Initial skew**: shift row `i` of `A` left by `i`; shift col `j` of `B` up by `j`
  2. **Main loop** (√p iterations): local `C += A·B` contribution, then shift `A` left by 1 and `B` up by 1
  - Total time: `O(n³/p + τ√p + μ·n²/√p)` — **memory-optimal**, matches Hong-Kung lower bound
  - **Constraint**: `p` must be a **perfect square** and matrices must be square
- **SUMMA (1997)** — replaces shifts with **row/column broadcasts of panels of width `b`**; works on any `p_r × p_c` grid; the parallel form of **outer-product GEMM**; used in **ScaLAPACK / Elemental**
- **Sparse matrices**: `nnz ≪ n²`. Markowitz (1990 Nobel) coined "sparse" in the 1950s. Dense algorithms waste memory and flops on zeros
- **Storage formats** (memorize this table cold):
  | Format | Arrays | When |
  |---|---|---|
  | **COO** | `(row, col, val)` | building / transposing |
  | **CSR** | `row_ptr, col_idx, val` | the default; row-major access |
  | **CSC** | `col_ptr, row_idx, val` | column ops |
  | **ELL** | dense pad to max-nnz row | GPU SIMD; wastes if rows uneven |
  | **DIA** | diagonals + offsets | stencils, banded |
  | **HYB** | ELL + COO outliers | irregular but mostly regular |
- **SpMV** `y = A·x`: only 2 flops per nonzero (`*` then `+=`) against ~12-16 bytes loaded → **memory-bound**, lives on the bandwidth-limited side of the **Roofline** model
- **Parallel SpMV bottlenecks**:
  1. **Load imbalance** — rows have wildly different `nnz`
  2. **Indirect access** `x[ind[j]]` — destroys cache locality
  3. **Reductions** when partitioning by columns
- **GPU SpMV**: CSR with **warp-per-row** or **vector-per-row** (irregular); ELL/DIA for regular structures (stencils); HYB for mixed

---

## Cross-cutting themes (likely exam targets)

### The three programming models — pick by memory architecture
| Model | Memory | Communication | Sync | Scale |
|---|---|---|---|---|
| **OpenMP** | Shared | implicit via reads/writes | `critical`/`atomic`/`barrier`/`reduction` | 1 node, ≤ ~100 cores |
| **MPI** | Distributed | explicit messages | implicit in send/recv, `Barrier` | many nodes, > 1M cores |
| **CUDA** | Hybrid (host + device) | `cudaMemcpy` + streams | `__syncthreads()`, `cudaDeviceSynchronize` | 1+ GPU per node |

Real codes combine all three: **MPI for inter-node + OpenMP for intra-node + CUDA for GPU kernels**.

### The scaling laws ladder
- **Amdahl's Law** (fixed problem): `S(p) = 1 / ((1-f) + f/p)` → speedup capped by **1/(1-f)**
- **Gustafson's Law** (scaled problem): `S(p) = (1-f) + p·f` → more optimistic, justifies massive parallelism
- **Strong scaling**: fixed problem, vary `p`. Amdahl's regime
- **Weak scaling**: grow problem with `p`. Gustafson's regime

### The synchronization-cost hierarchy (low → high)
1. **SIMD / vectorization** (no sync — within one core)
2. **Atomic** (hardware CAS)
3. **Reduction** (private + merge)
4. **Critical** / lock (OS-level)
5. **Barrier** (all threads wait)
6. **All-reduce** (across nodes, the AI scaling wall)

### The "where does the data sit?" ladder
- **Register** (CUDA) / **stack** (CPU) — fastest, smallest, per-thread
- **L1 / shared memory** — per-core or per-block
- **L2 / L3 cache** — per-socket
- **Main memory / DRAM** — per-node (the "memory wall")
- **Inter-node network** — slowest, biggest, the MPI domain

**Arithmetic intensity** = flops per byte. **Roofline model** = `min(peak FLOP/s, intensity × peak BW)`. GEMM lives on the FLOP roof; SpMV lives on the bandwidth roof.

### The "structural pattern → algorithmic example" map
| Pattern | Example | Programming model |
|---|---|---|
| Map | `axpy`, element-wise ops | OpenMP `parallel for`, CUDA kernels |
| Reduce | sum, max | OpenMP `reduction`, `MPI_Reduce`, CUDA shuffle-reduce |
| Scan | prefix sum | parallel-scan algorithms |
| Stencil | Jacobi, FDM, conv layers | OpenMP loops, CUDA shared-mem tiling |
| Geometric decomp | 2D block GEMM, Cannon | MPI Cartesian topology |
| Pipeline | streaming inference, image processing | MPI + non-blocking, CUDA streams |
| Fork-Join | recursive `fib`, divide-and-conquer | OpenMP tasks |
| Master-Worker | load-balancing | MPI rank 0 dispatcher |

---

## High-yield cheatsheet (memorize cold)

### Five formulas
```math
\text{Amdahl: } S(p) = \frac{1}{(1-f) + f/p}, \quad S_\infty = \frac{1}{1-f}
```
```math
\text{Gustafson: } S(p) = (1-f) + p \cdot f
```
```math
\text{Efficiency: } E(p) = \frac{S(p)}{p}
```
```math
\text{Roofline: } \text{Perf}(I) = \min(\pi, \, \beta \cdot I) \quad I = \text{flops/byte}, \pi = \text{peak FLOP/s}, \beta = \text{peak BW}
```
```math
\text{Cannon comm: } T_{\text{comm}} = 2\sqrt{p} \cdot \left(\tau + \mu \cdot \frac{n^2}{p}\right)
```

### Six numbers
- **Cache line** = 64 bytes (the false-sharing unit)
- **Warp size** = 32 threads
- **A100 memory bandwidth** ≈ 1.55 TB/s
- **A100 FP32 peak** ≈ 19.5 TFLOP/s
- **A100 max threads per SM** = 2048
- **A100 max registers per SM** = 65 536

### Architectures to be able to sketch
1. **Multicore node**: cores → private L1/L2 → shared L3 → DRAM
2. **MPI ring/torus/fat-tree** topology
3. **MPI Cartesian grid** with periodic / non-periodic boundaries
4. **GPU**: host → PCIe → device DRAM → L2 → SMs → registers + shared memory
5. **CUDA streams + events** lane diagram (compute + copy lanes overlapping)
6. **Cannon's 2D systolic shifts** on a `√p × √p` grid

---

## Pitfalls (the trap questions)

- **Amdahl ≠ Gustafson** — they answer **different questions** (fixed vs scaled problem). Don't blame Amdahl for "low scaling efficiency" on a weak-scaling benchmark.
- **OpenMP `reduction` vs manual sum** — manual `sum += a[i]` in a parallel loop is **undefined behaviour**, not "slow". Fix with `reduction(+:sum)`.
- **False sharing** is invisible in code review — the bug is in **memory layout**, not logic. Pad to cache-line boundaries.
- **`single` has an implicit barrier**, **`master` does not** — order of `single` blocks is *not* guaranteed.
- **MPI deadlock from blocking sends** — circular `MPI_Send` pairs deadlock. Either alternate even/odd ordering or use `MPI_Sendrecv`.
- **`MPI_Bcast` is collective** — every rank must call it, not just the root. Same for `MPI_Reduce` / `MPI_Allreduce`.
- **Non-blocking calls don't transfer immediately** — the buffer is not safe to reuse until `MPI_Wait` returns.
- **CUDA default stream synchronizes implicitly** — kills concurrency. Use explicit streams when you want overlap.
- **`cudaMemcpyAsync` requires pinned host memory** — otherwise it falls back to synchronous behaviour.
- **Branch divergence in a warp** serializes the diverging paths — write divergence-free kernels when possible.
- **Coalescing is per-warp**, not per-block — strided patterns are the silent perf killer in CUDA.
- **Cannon requires p to be a perfect square** — for arbitrary `p`, use SUMMA.
- **SpMV is bandwidth-bound, not flop-bound** — peak GFLOP/s is irrelevant; effective bandwidth is.
- **Bigger blocks ≠ better occupancy** — registers/threads/shared-mem may limit you before you fill the SM.
- **`atomic` is faster than `critical`** but only works for single ops (`+=`, `*=`, etc.). For arbitrary logic, `critical`.

---

## Study plan from here

1. ✅ **Phase 0 — Syllabus map** (this file)
2. ⏳ **Phase 1 — Per chapter**: read each `.md`, sketch the architecture diagrams from memory, recite the synchronization-cost hierarchy
3. ⏳ **Phase 2 — Interleaved drills**:
   - Pick a pattern (Reduce / Stencil / Pipeline) → describe its OpenMP, MPI, CUDA implementations
   - Pick a memory issue (false sharing / coalescing / load imbalance) → which model is each one specific to?
   - Derive Amdahl + Gustafson and explain when each applies
   - Walk through Cannon's algorithm on a 2×2 grid with `n = 4`
4. ⏳ **Phase 3 — Code muscle memory** (only if labs are in scope):
   - 5-line OpenMP parallel for + reduction
   - MPI ring exchange (Send + Recv with mod arithmetic)
   - CUDA SAXPY kernel + launch
