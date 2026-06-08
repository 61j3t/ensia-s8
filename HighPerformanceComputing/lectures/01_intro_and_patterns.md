# Chapter 1 — Introduction to HPC + Parallel Patterns

> Source: `Lecture 1.pdf` (HPC Systems: Architectures, Parallelism, and the Scale of Modern AI) + `Patterns (1).pdf` (Parallel Programming Patterns — McCool et al., Ch. 3).

## Bird's eye view

- **HPC = parallel computing.** A modern HPC system is a cluster of nodes connected by a high-speed network; each node has multi-core CPUs and (almost always) GPUs. Performance is ranked by the **TOP500** list using the **LINPACK / HPL** benchmark, measured in **FLOP/s** (we are at the **Exa** = `10^18` scale: El Capitan ≈ 1.74 EFlop/s on ~11M cores).
- **Why parallelism is the only path forward**: Moore's Law (transistor density 2× every ~18 months) still holds, but **Dennard scaling and clock frequency hit a wall (~3 GHz) around 2004-2005**. Single-thread performance stalled — performance gains now come from **more cores**, not faster ones. "The free lunch is over."
- Two complementary motivations: **need for speed** (faster solution of a fixed problem) and **need for memory** (problems whose data won't fit on one node — climate, LLM training, the "AI memory wall"). Bill Dally's slogan: **Performance = parallelism, Efficiency = locality**.
- Hardware is classified by **Flynn's taxonomy** (SISD / SIMD / MISD / MIMD); memory organization splits into **shared** (SMP, easy programming, limited scalability) vs **distributed** (clusters, infinite scalability, explicit message passing). Modern HPC = hybrid: a distributed cluster of shared-memory nodes.
- Programming model maps to memory model: **OpenMP** (shared, `#pragma`-based, fork-join), **MPI** (distributed, SPMD, message passing), **CUDA / HIP** (GPU kernels). Real codes often combine all three.
- Two fundamental scaling laws bound the benefit of parallelism: **Amdahl's Law** (fixed problem, serial fraction caps speedup) and **Gustafson's Law** (problem grows with `P`, so scaling is more optimistic). Distinguish **strong** vs **weak** scaling.
- **A parallel pattern is a general solution to a recurring problem** — *not* a ready-made block of code. Same idea as Christopher Alexander's architectural patterns (you don't reinvent the bridge each time; you pick the right type).
- McCool et al. group patterns into **structural** (decomposition / how work is organized: Map, Partition, Pipeline, Fork-Join, Master-Worker, Geometric Decomposition, Recursive Splitting) and **computational** (what is computed: Map, Reduce, Scan, Stencil, Gather, Scatter, Pack, Search/Match).
- The classical **6-pattern HPC curriculum**: **Embarrassingly Parallel**, **Partition**, **Master-Worker**, **Stencil**, **Reduce**, **Scan**. Add **Pipeline**, **Fork-Join**, **Producer-Consumer**, **Speculative Execution** for completeness.
- The Berkeley **7 Dwarfs / 13 Motifs** (Colella 2004 → 2006 expansion) classify the recurring numerical kernels behind almost every HPC and AI workload: Dense LA, Sparse LA, FFT, N-Body, Structured Grids, Unstructured Grids, Monte Carlo (+ 6 commercial motifs: Combinational Logic, Graph Traversal, FSM, DP, Backtrack/B&B, Graphical Models). Understand the motif ⇒ understand the bottleneck ⇒ co-design the hardware (GPUs for LA, TPUs for AI).

---

# Deck A — Introduction to HPC

## 1. What HPC is and how it's measured

### 1.1. Definitions

- **HPC** = aggregating compute resources (cores, nodes, accelerators) to solve problems that a single workstation cannot solve in reasonable time or memory.
- **Supercomputer / HPC cluster** = many compute nodes + high-speed interconnect (e.g., 100 Gbps InfiniBand) + parallel file system + a login node + a job scheduler (batch queue).
- A user `ssh`s into the **login node**, submits a **batch job** to the **queue**, the scheduler runs it on compute nodes, output goes to a **parallel file system** (global scratch / long-term storage).

### 1.2. TOP500 and LINPACK

- **TOP500** ranks the 500 most powerful non-distributed systems twice a year (June / November).
- Benchmark = **LINPACK / HPL**: solve a dense linear system `Ax = b` of huge size.
- Metric = **FLOP/s** (floating-point operations per second). `Rmax` = maximal LINPACK performance achieved.
- Prefixes: **Giga** `10^9`, **Tera** `10^12`, **Peta** `10^15`, **Exa** `10^18`. We are at exascale (since 2022 with Frontier; El Capitan is the current #1).

### 1.3. Why we needed parallelism — the end of Dennard scaling

- **Moore's Law**: transistor count per chip ≈ doubles every 18 months. *Still true.*
- **Dennard scaling**: as transistors shrink, voltage scales down too, so power per area stays constant. *Failed around 2004-2005.*
- Consequence: pushing clocks higher would melt the chip → the **Power Wall** at ~3 GHz.
- **Moore's Law reinterpreted**: density still doubles, but the extra transistors go into **more cores**, not a faster clock. Performance gains require software to expose parallelism.
- The **AI memory wall**: transformer model size grows ~410× / 2 years; AI hardware memory only ~2× / 2 years. Models no longer fit on one device → distributed training is mandatory.

---

## 2. Hardware taxonomy

### 2.1. Flynn's taxonomy

| | Single Data | Multiple Data |
|---|---|---|
| **Single Instruction** | **SISD** — classic serial PC; 1 instruction on 1 datum | **SIMD** — 1 instruction on a vector; the **GPU / vector unit** model, critical for matrix math |
| **Multiple Instruction** | **MISD** — rare, fault-tolerant systems | **MIMD** — the **cluster** model; independent processors run different code on different data |

> Modern systems are mostly **MIMD made of SIMD nodes** (a cluster of CPUs+GPUs, each GPU itself doing SIMD/SIMT).

### 2.2. Memory architectures

| | **Shared memory (SMP)** | **Distributed memory (cluster)** | **Hybrid (real HPC)** |
|---|---|---|---|
| Layout | All procs see one global memory via bus/network | Each proc has its own local memory; nodes communicate over network | Distributed cluster of shared-memory nodes |
| Pro | Easy to program | Infinite scalability | Best of both |
| Con | Bus contention limits scaling | Explicit message passing required | Two programming models in one code |
| Tool | **OpenMP** | **MPI** | **MPI + OpenMP (+ CUDA)** |

### 2.3. Concurrency vs parallelism

- **Concurrency**: multiple tasks logically active, progress via context switching. Analogy: one person juggling three balls. Single core is enough.
- **Parallelism**: tasks physically execute at the same instant. Requires multiple hardware units. Analogy: three people each holding one ball.

### 2.4. Levels of parallelism (where it can be exploited)

1. **Bit-level** — wider ALUs operate on more bits per cycle.
2. **Instruction-level (ILP)** — pipelining, superscalar, out-of-order execution inside one core.
3. **Data-level** — same operation across many data items (SIMD, vectorization, GPU).
4. **Task-level** — different independent tasks running on different cores / nodes (MIMD).

---

## 3. Programming models (the software layer)

| Model | Target hardware | Mechanism | Use case |
|---|---|---|---|
| **OpenMP** | Shared memory (single node) | Compiler directives (`#pragma omp parallel for`) | Parallelize `for` loops; fork-join multithreading |
| **MPI** | Distributed memory (cluster) | Library calls (`MPI_Send`, `MPI_Recv`) — SPMD: every process runs the same executable, behavior differentiated by **rank** | Cross-node scaling, the "gold standard" for exascale |
| **CUDA / HIP** | GPU accelerators | Device kernels offloaded from a host CPU over PCIe / NVLink | Massive data-parallel kernels (matrix math, DL training) |

- **OpenMP execution model**: master thread → `parallel` region forks a **team of threads** → work distributed → implicit **barrier / join** → only master continues.
- **OpenMP memory model**: variables declared **outside** the parallel region are **shared by default** (risk of **data races**); variables declared **inside** are **private** to each thread.
- **MPI execution model (SPMD)**: every process runs the *exact same* executable. Logic differs by **rank** (`if (rank == 0) { distribute_data(); } else { compute_worker(); }`). Processes have private memory and must explicitly send/receive.
- **Heterogeneity**: CPU = optimized for logic / serial control flow; GPU = optimized for throughput (SIMT). CPU "host" offloads compute kernels to GPU "device".

---

## 4. The scaling laws

### 4.1. Speedup, efficiency, scaling regimes

- **Speedup** on `P` processors: `S(P) = T_serial / T_parallel(P)`.
- **Efficiency**: `E(P) = S(P) / P` (1.0 = perfect, usually lower because of communication & serial parts).
- **Strong scaling**: problem size fixed, `P` increases. Question: "does it solve faster?" Bounded by Amdahl.
- **Weak scaling**: problem size grows proportionally with `P`. Question: "can we solve a bigger problem in the same time?" Captured by Gustafson.

### 4.2. Amdahl's Law (fixed problem)

Let `s` = serial fraction of the workload, `1 − s` = parallelizable fraction:

```math
S(P) \;\le\; \frac{1}{s + \dfrac{1 - s}{P}}
```

In the limit `P → ∞`:

```math
S_\infty = \frac{1}{s}
```

> **Interpretation**: even with infinite processors, the serial part caps total speedup. If 10% of the code is serial, max speedup is **10×**. The serial fraction is what kills you.

### 4.3. Gustafson's Law (scaled problem)

Let `s` be the serial fraction *in the parallel run*. Then **scaled speedup**:

```math
S(P) \;=\; s + (1 - s)\,P \;=\; P - s\,(P - 1)
```

> **Interpretation**: if we grow the problem with `P` (more cores → bigger simulation), speedup is roughly *linear* in `P`. This is why exascale machines are useful in practice even though Amdahl looks pessimistic.

| | Amdahl | Gustafson |
|---|---|---|
| Problem size | fixed | grows with `P` |
| Verdict | pessimistic — bounded by `1/s` | optimistic — ~linear in `P` |
| Matches | strong scaling | weak scaling |

---

## 5. The Berkeley Motifs (HPC application landscape)

- 2004: Phillip **Colella** observed that scientific computing relies on just **7 numerical methods** (the "7 Dwarfs"). Idea: design future hardware for these, not for old SPEC benchmarks ("driving by rearview mirror").
- 2006: Berkeley Par Lab expanded to **13 Motifs** by adding 6 commercial workloads (games / DBs / AI).

| Group | Motifs |
|---|---|
| **7 Dwarfs (HPC)** | Dense Linear Algebra, Sparse Linear Algebra, Spectral Methods (FFT), N-Body, Structured Grids, Unstructured Grids, Monte Carlo |
| **+6 Commercial** | Combinational Logic, Graph Traversal, Finite State Machines, Dynamic Programming, Backtrack / Branch & Bound, Graphical Models |

Each motif has a characteristic **bottleneck** which dictates the right hardware:

| Motif | Pattern | Bottleneck |
|---|---|---|
| **Dense LA** | Matrix-matrix multiply | **Compute-bound** (high arithmetic intensity) → near-peak performance |
| **Sparse LA** | Matrix-vector with mostly zeros | **Memory-bound** (indirect addressing) → often <10% of peak |
| **Structured grids** | Regular arrays (weather) | Bandwidth-sensitive, but high spatial locality |
| **Unstructured grids** | Irregular meshes (CFD) | Latency + memory (pointer chasing), hard to parallelize |
| **FFT** | Time ↔ frequency domain | **Communication-bound** (global all-to-all shuffle) |
| **N-Body** | Pairwise interactions | Compute-intensive; naive `O(N²)`, Barnes-Hut `O(N log N)` |
| **Monte Carlo** | Repeated independent random trials | **None — embarrassingly parallel** (MapReduce is a generalization) |
| **Graph traversal** | BFS/DFS on social/web graphs | Memory latency (random jumps) |
| **FSM** | Pattern matching | "Embarrassingly **sequential**" — next state depends on current |
| **DP** | Sequence alignment, bioinformatics | Overlapping sub-problems, dependency chains |

> **Takeaway**: identify the motif ⇒ identify the bottleneck ⇒ co-design hardware. GPUs answered Dense LA. TPUs answered AI matmul. "We stopped chasing clock speed and started chasing algorithmic efficiency."

---

# Deck B — Parallel Programming Patterns

## 6. What is a pattern?

- "A **design pattern** is a general solution to a recurring engineering problem." (Christopher Alexander, 1977, architecture.)
- A pattern is **not** ready-made code — it is a *description of how a certain kind of problem can be solved*.
- Example (Alexander): you don't invent a new bridge for every river; you choose between *beam*, *arch*, *cable-stayed*, *suspension* depending on span, materials, traffic. Same for parallel algorithms.
- McCool et al. split parallel patterns into:
  - **Structural patterns** (how computation & data are *organized*): Partition, Pipeline, Fork-Join, Master-Worker, Map, Geometric Decomposition, Recursive Splitting, Producer-Consumer.
  - **Computational patterns** (what is *computed*): Map, Reduce, Scan, Stencil, Gather, Scatter, Pack, Search.
- A third axis (data management): **gather / scatter / pack** describe how data moves; **partition** how data is split.

---

## 7. The core parallel patterns

### 7.1. Embarrassingly Parallel (a.k.a. Map)

- **What**: computation decomposes into **independent tasks** with little or no communication. Same operation applied to every element of a collection: `c[i] = f(a[i], b[i])`.
- **When**: vector sum, Mandelbrot set, 3D rendering, brute-force password cracking, Monte Carlo trials, mini-batch over independent samples in DL.
- **Why it matters**: theoretical maximum speedup, near-zero synchronization overhead.

### 7.2. Partition (Geometric Decomposition)

- **What**: split the input *domain* into **disjoint partitions**; each processor works on one partition. Useful when the app has **locality of reference** (a processor mostly needs its own partition).
- **Variants**:
  - **Regular** partition (same shape/size — matrix-vector product) vs **irregular** (e.g., triangular mesh of a lake split by graph partitioning).
  - **1-D**: **Block** (each core gets a contiguous chunk; `global_index = local_index + block_id * BLKLEN`) vs **Cyclic** (round-robin).
  - **2-D**: Block-Block (tiles), Block-*, *-Block (strips), Cyclic-Cyclic (checkerboard).
- **Granularity trade-off**:

| Granularity | Fine-grained (many small partitions) | Coarse-grained (few large partitions) |
|---|---|---|
| Load balance | better (especially with master-worker) | worse |
| Communication / scheduling overhead | higher (comm. dominates if too fine) | lower |
| Comp/Comm ratio | low | high |

  → Optimal partition size minimizes wall-clock time and is system + application dependent.

### 7.3. Master-Worker (Process Farm, Work Pool)

- **What**: use a **fine-grained** decomposition with **#tasks ≫ #workers**. A *master* hands tasks to whichever *worker* is free.
- **Why**: handles **load imbalance** automatically (dynamic task allocation). Critical when task durations vary unpredictably (e.g., Mandelbrot: black pixels need `maxit` iterations, others much fewer — naive static partitioning leaves some cores idle).
- **Cost**: dynamic scheduling has higher overhead than static.

### 7.4. Stencil

- **What**: update each cell of a grid as a function of a **fixed neighborhood pattern** (the stencil). E.g., Gaussian smoothing of an image with a 5×5 weighted average; PDE solvers (heat equation, Jacobi).
- **Neighborhoods**:
  - 2D: 5-point (von Neumann / cross), 9-point 2-axis (extended cross), 9-point 1-plane (Moore / box).
  - 3D: 7-point 3-axis, 13-point 3-axis.
- **Two-domain trick**: read from `current`, write to `next`, swap at end of each step (avoids races, preserves semantics).
- **Ghost cells**: extend the domain with one (or more) extra layer to handle boundaries uniformly. Filled either with fixed boundary values or with the opposite edge for **periodic boundary conditions**.

### 7.5. Reduce

- **What**: combine an array `[x₀, …, xₙ₋₁]` into a single value using an **associative** binary operator `op` (sum, product, min, max, logical AND/OR). E.g., `sum = x₀ + x₁ + … + xₙ₋₁`.
- **Serial**: `O(n)` operations.
- **Parallel (tree reduction)**: pair up elements at each level, halve the array → `O(log n)` parallel steps with `n` processors.
- **Work-efficient**: total work = `n/2 + n/4 + … + 1 = O(n)` — same as serial, just spread over fewer time steps.
- Requires **associativity** of `+`. (Floating-point `+` is only *almost* associative — different reduction trees can give slightly different results.)

### 7.6. Scan (Prefix Sum)

- **What**: compute *all* prefixes of `[x₀, …, xₙ₋₁]` under an associative `op`. Two flavours:
  - **Inclusive**: `y_i = x₀ op x₁ op … op x_i`.
  - **Exclusive**: `y₀ = neutral`, `y_i = x₀ op … op x_{i−1}`.
- **Example**: `x = [1,−3,12,6,2,−3,7,−10]`, inclusive-sum scan = `[1,−2,10,16,18,15,22,12]`.
- **Serial**: `O(n)`. **Parallel** (Blelloch up-sweep + down-sweep): `O(log n)` time, `O(n)` work — work-efficient.
- **Why it's a fundamental pattern**: many problems reduce to scan — line-of-sight, stream compaction (pack), radix sort, lexical analysis, parallel allocation. "If you can phrase it as a scan, you can parallelize it."

### 7.7. Gather / Scatter

- **Scatter**: distribute pieces of one array from one process to many (`MPI_Scatter`).
- **Gather**: collect pieces from many processes back to one (`MPI_Gather`). Map is `Scatter → local op → Gather`.

### 7.8. Pack (Stream Compaction)

- **What**: from an array `x[]` with a boolean predicate, produce a contiguous array of elements where the predicate is true. Implemented via a scan on the predicate flags to compute output indices.

### 7.9. Pipeline

- **What**: a sequence of *stages*; each item flows through all stages. Stages run in parallel on different items (like CPU pipeline or assembly line). Throughput limited by the **slowest stage**.

### 7.10. Fork-Join

- **What**: a single thread *forks* into a team, threads work in parallel, then *join* back. The execution model of OpenMP, Cilk, Java's `ForkJoinPool`.

### 7.11. Recursive Splitting (Divide & Conquer)

- **What**: recursively split the problem into smaller independent subproblems, solve each in parallel, combine. Backbone of parallel sort (mergesort, quicksort), FFT, Barnes-Hut.

### 7.12. Producer-Consumer

- **What**: producers push items into a shared queue; consumers pull. Decouples task generation from execution. Needs synchronization on the queue.

### 7.13. Speculative Execution

- **What**: start a computation whose result *might* be needed (e.g., both branches of an `if`). If the prediction is right, you saved latency; if wrong, discard the work. Used at the hardware level (branch prediction) and the algorithmic level.

---

## 8. Load balancing

- Ideal: every processor does equal work. If tasks synchronize at the end (barrier), total time = time of the **slowest task**.
- **Sources of imbalance**: irregular domain, data-dependent task duration (Mandelbrot), heterogeneous hardware.
- **Cures**:
  - **Fine-grained partitioning** — many small tasks self-balance, but watch communication overhead.
  - **Master-worker** with a work pool — dynamic, robust to data-dependent variance.

---

## Glossary

- **FLOP/s** — floating-point operations per second; the HPC unit of speed (Giga `10⁹`, Tera `10¹²`, Peta `10¹⁵`, Exa `10¹⁸`).
- **LINPACK / HPL** — benchmark solving dense `Ax=b`; basis of the **TOP500** ranking.
- **Moore's Law** — transistor density doubles ≈ every 18 months; still alive.
- **Dennard scaling** — power per area stays constant as transistors shrink; **failed** ~2004, ending clock-frequency scaling.
- **Power Wall** — ~3 GHz practical clock limit caused by heat / power.
- **Flynn's taxonomy** — SISD / SIMD / MISD / MIMD classification of parallel hardware.
- **SMP / Shared memory** — multiple processors, one global memory; programmed with OpenMP.
- **Distributed memory** — each node has private memory; programmed with MPI.
- **SPMD** — Single Program, Multiple Data: every process runs the same executable, behaviour depends on **rank**.
- **Speedup `S(P)`** — `T_serial / T_parallel(P)`. **Efficiency** = `S(P)/P`.
- **Amdahl's Law** — `S(P) ≤ 1 / (s + (1−s)/P)`; serial fraction caps speedup.
- **Gustafson's Law** — `S(P) = P − s(P−1)`; scaled (weak) speedup is ~linear in `P`.
- **Strong vs weak scaling** — fixed-problem vs problem-grows-with-`P` regimes.
- **Pattern** — general solution to a recurring problem (Alexander); not ready-made code.
- **Embarrassingly parallel** — independent tasks, no communication.
- **Stencil** — update each cell from a fixed neighborhood pattern (PDEs, image filters).
- **Reduce / Scan** — combine an array via an associative `op`: reduce → single value; scan → all prefixes; both `O(log n)` parallel time, `O(n)` work.
- **Ghost cells** — extra boundary cells added to a partition so border points don't need special-case code.
- **Berkeley Motifs (7 Dwarfs + 6)** — recurring computational patterns that drive hardware/software co-design.

---

## Likely exam targets

1. **Define HPC** and explain why HPC ≡ parallel computing today. Mention Moore's Law + the end of Dennard scaling + the Power Wall.
2. **State Flynn's taxonomy** (SISD / SIMD / MISD / MIMD) and give one example architecture for each (PC / GPU / rare / cluster).
3. **Contrast shared-memory and distributed-memory architectures** (programming model, scalability, communication cost). Place OpenMP, MPI, CUDA in this map.
4. **State and derive Amdahl's Law** `S(P) ≤ 1 / (s + (1−s)/P)`. Compute max speedup for `s = 0.1, P → ∞` (answer: 10×). Explain why parallel hardware alone is not enough.
5. **State Gustafson's Law** and explain when it gives a more realistic estimate than Amdahl (weak scaling, growing problem sizes — exascale simulations).
6. **Distinguish strong vs weak scaling** and connect each to Amdahl / Gustafson.
7. **Define a parallel pattern** (Alexander definition) and give the structural-vs-computational classification (McCool). Name at least 6 patterns and one use case each.
8. **Reduce vs Scan**: definitions, parallel complexity (`O(log n)` time, `O(n)` work), why associativity matters, one application of each.
9. **Stencil**: definition, neighborhood shapes (5-point, 9-point, 3D 7-point), the two-domain (current/next) trick, ghost cells, periodic boundary conditions.
10. **Partition + Master-Worker for load balancing**: when does static block partitioning fail (Mandelbrot example), how does dynamic task allocation fix it, what are the trade-offs (overhead vs balance, granularity vs comm/comp ratio).

---

## Pitfalls

- **Moore's Law ≠ frequency scaling**. Moore's Law is about transistor *density*, which still doubles. What stalled is clock frequency (Dennard scaling broke). Don't conflate the two.
- **Amdahl's serial fraction `s`** refers to *the fraction of work that is inherently sequential*, not the time spent on one processor. Common mistake: plugging in walltime ratios.
- **Amdahl is fixed-problem, Gustafson is scaled-problem.** Saying "Amdahl is wrong, Gustafson is right" is a false dichotomy — they answer different questions (strong vs weak scaling).
- **Reduce/Scan require associativity.** Floating-point `+` is only approximately associative, so different reduction trees produce slightly different sums. Reproducibility matters in scientific computing.
- **OpenMP shared-by-default is dangerous.** Variables declared outside a parallel region are shared → data races unless you mark them `private` or use `reduction(+:x)`.
- **MPI is SPMD, not master-worker by default.** Every process runs the same executable; differentiation is via `rank`. A "master" exists only because you wrote `if (rank == 0) …`.
- **A pattern is not code.** "Use the stencil pattern" doesn't mean copy-paste a function; it means structure the computation as a per-cell update over a fixed neighborhood, then handle boundaries (ghost cells) and partition the grid.
- **Fine-grained partitioning is not always better.** It improves load balance but raises the communication/scheduling overhead. The optimal granularity is system- and application-dependent.
- **Embarrassingly parallel ≠ trivial to scale on a real cluster.** You still pay for data distribution, result aggregation, and I/O. "Embarrassingly parallel" = no inter-task communication, not zero overhead.
