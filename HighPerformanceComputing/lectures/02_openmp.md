# Chapter 2 — OpenMP (Introduction + Advanced)

> Source: `OpenMP Introduction.pdf` (35 slides — shared-memory model, directives, work sharing, data environment, reductions) and `Advanced_OpenMP_Synchronization_Memory_and_Tasks.pdf` (16 slides — synchronization, race conditions, false sharing, SIMD, tasking).

## Bird's eye view

- **OpenMP** is a **directive-based** language extension for C/C++/Fortran that adds **shared-memory multithreading** via compiler `#pragma`s — orthogonal to functionality (ignored by non-OpenMP compilers, code still runs serially).
- **Target machine model**: SMP / multicore with **shared main memory** and **per-core private L1/L2 caches**. Threads of a process share the same address space.
- **Execution model = Fork–Join**: a serial **master thread** hits `#pragma omp parallel`, **forks** a team of workers, all run the region in parallel, **join** back into the master at the closing brace.
- **Five categories of constructs**: parallel regions, **work-sharing** (`for`, `sections`, `single`), **data-sharing clauses** (`shared`/`private`/...), **synchronization** (`critical`/`atomic`/`barrier`/`ordered`), and **runtime API + env vars**.
- **Data sharing is syntactic**: variables visible to >1 thread are shared by default (globals, outer locals); variables declared inside the parallel block are private.
- **Race conditions are the programmer's responsibility** — OpenMP assumes parallel regions are independent. Unprotected `sum += ...` is the canonical bug; fix with `reduction`, `atomic`, or `critical`.
- **Synchronization hierarchy by cost**: `atomic` (hardware CAS) < `reduction` (private + merge) < `critical` (OS lock) < `barrier` (whole team waits).
- **False sharing** kills scalability: independent variables that land on the *same 64-byte cache line* ping-pong between cores via the coherence protocol — fix via **padding** or **local accumulation in registers**.
- **SIMD ≠ threading**: SIMD = **DLP** (one instruction stream, wide registers, 128–512 bits); threading = **TLP**. Combine with `#pragma omp parallel for simd`.
- **Tasking model** decouples *work generation* from *work execution* — solves what `parallel for` cannot: recursion, linked lists, `while` loops, irregular workloads. Runtime uses **work stealing** across a global task queue.
- **Compile** with `gcc -fopenmp file.c`; control thread count with `OMP_NUM_THREADS` env var or `omp_set_num_threads(n)`.
- **Engineering motto**: parallel programming is about *managing resources* — correctness (atomic/critical/reduction), performance (nowait, false-sharing avoidance), hardware utilization (SIMD, alignment), and complexity (task vs for).

---

## 1. Shared-memory model & motivation

- **Shared-memory (SMP)**: all processors/cores see the **same physical memory**; threads in a process share the address space. Communication is implicit — through shared variables.
- Contrasts with **distributed memory** (MPI) where each node has private RAM and communication is explicit message passing.
- **Why a high-level model?** Raw threading (`pthread_create`, `CreateThread`) requires manual thread creation, mutexes, and *different code for serial vs parallel versions*. OpenMP unifies the two.
- **Key property — incremental parallelization**: write & debug serial code first, then add pragmas; the same source compiles correctly serial (compiler just ignores unknown `#pragma`s).
- **Industry standard** (Intel, GNU, Microsoft, IBM, …) with some implementation-defined behavior. C/C++ and Fortran bindings.

---

## 2. Execution model — Fork–Join

- Program starts as a single **master thread** (id 0).
- `#pragma omp parallel` **forks** a team of *N* threads (master + N−1 workers); each executes the structured block **redundantly**.
- At the closing `}` of the region, threads **synchronize at an implicit barrier** and **join** — only the master continues.
- Multiple parallel regions are allowed; teams may be reused by the runtime (optimization: threads not destroyed between consecutive regions).
- Inside a parallel region, threads identify themselves via `omp_get_thread_num()` and the team size via `omp_get_num_threads()`.

```math
\text{Speedup}(N) \;=\; \frac{T_{serial}}{T_{parallel}(N)} \;\le\; \frac{1}{(1-p) + p/N}\quad\text{(Amdahl)}
```

The serialized fraction `1 − p` includes any time spent inside `critical`, `atomic`, or barriers.

---

## 3. Syntax & directive categories

- All directives use the form `#pragma omp construct [clause [clause] …]` and apply to a **structured block** (one entry, one exit).
- Five categories:
  1. **Parallel regions** — `parallel`
  2. **Work sharing** — `for`, `sections`/`section`, `single`
  3. **Data environment** — clauses `shared`, `private`, `firstprivate`, `lastprivate`, `default`, `reduction`
  4. **Synchronization** — `critical`, `atomic`, `barrier`, `ordered`, `flush`, `nowait`
  5. **Runtime library + env vars** — `omp_*` functions and `OMP_*` variables

Compile: `gcc -fopenmp prog.c -o prog`.

---

## 4. Work-sharing constructs

| Construct | What it does | Implicit barrier at end? |
|---|---|---|
| `#pragma omp for` | Splits a `for` loop's iterations across the team | **Yes** (unless `nowait`) |
| `#pragma omp sections` / `section` | Distributes independent code blocks one-per-thread | **Yes** |
| `#pragma omp single` | Exactly **one** thread (the first to arrive) runs the block; others wait | **Yes** |
| `#pragma omp master` | The **master thread (id 0)** runs the block; others **skip** | **No** |

- `#pragma omp parallel for` is the common combined form (spawn team + share the loop).
- **Loop form restrictions**: a single signed integer index, comparison `<,<=,>,>=` against a loop-invariant bound, increment by a loop-invariant step.
- **`single` vs `master`** — both single out one thread, but only `single` has an implicit barrier. With `master`, the other threads race ahead — manual `barrier` often needed.

---

## 5. Loop scheduling — `schedule(kind [, chunk])`

| Kind | Behaviour | When to use |
|---|---|---|
| `static` | Iterations split into equal chunks at compile/region entry, assigned round-robin | Uniform work per iteration — lowest overhead |
| `dynamic` | Threads grab chunks (default size 1) from a queue as they finish | Highly variable / unpredictable work per iteration |
| `guided` | Like dynamic but chunk size starts large and **decays exponentially** down to `chunk` | Unknown distribution; good compromise of overhead vs balance |
| `auto` | Implementation chooses | Let the runtime decide |
| `runtime` | Read from `OMP_SCHEDULE` env var | Tune without recompiling |

- **Load-balancing vs granularity trade-off**: small chunks balance load but raise threading overhead; large chunks reduce overhead but may starve idle threads.

---

## 6. Data environment — sharing clauses

- **Default rules**: globals & file-scope variables are **shared**; automatic variables declared *inside* a parallel block are **private**; automatics declared *outside* and visible inside are **shared**.
- **`default(none)`** forces every variable's data-sharing attribute to be declared explicitly — strongly recommended for safety.

| Clause | Behaviour |
|---|---|
| `shared(x)` | All threads see the **same** `x`; programmer ensures race-free access |
| `private(x)` | Each thread gets its own **uninitialized** copy; original is untouched outside |
| `firstprivate(x)` | Private copy **initialized** from the master's value at region entry |
| `lastprivate(x)` | Private copy; the value from the **logically last iteration / last section** is copied back |
| `default(none\|shared)` | Set default; `none` requires explicit attributes for every variable |
| `reduction(op:list)` | Each thread gets a private copy initialised to op's identity; copies are **merged** with `op` at region end |

---

## 7. Reduction clause

- Pattern: parallelize `acc = acc op f(i)` without lock contention.
- Each thread accumulates into a **private copy** (initialised to the operator identity: `0` for `+`, `1` for `*`, all-ones for `&`, `0` for `|`/`^`, etc.); copies are combined with `op` at the join.
- Syntax: `reduction(op : var1, var2, ...)`.
- Supported associative operators: `+`, `-`, `*`, `&`, `|`, `^`, `&&`, `||`, `min`, `max`.

```math
\text{acc}_{\text{final}} \;=\; \text{acc}_{\text{init}} \;\text{op}\; \bigoplus_{t=1}^{T} \text{acc}^{(t)}_{\text{local}}
```

- *Why prefer reduction over `critical`*: serializes only once per thread at the merge, not once per iteration.

---

## 8. Runtime API & environment

| Function | Purpose |
|---|---|
| `omp_get_thread_num()` | Calling thread's id (0 .. N−1) |
| `omp_get_num_threads()` | Size of current team |
| `omp_get_max_threads()` | Upper bound for future regions |
| `omp_set_num_threads(n)` | Set default team size (must be called from serial code) |
| `omp_get_num_procs()` | Hardware cores available |
| `omp_in_parallel()` | Are we inside a parallel region? |
| `omp_get_wtime()` | Portable wall-clock timer |
| `omp_set_nested(int)` / `omp_get_nested()` | Enable nested parallelism |
| `omp_init_lock` / `omp_set_lock` / `omp_unset_lock` / `omp_destroy_lock` | Explicit locks (passable as variables, unlike `critical`) |

- **Environment variable** `OMP_NUM_THREADS=8` sets the default team size at launch.
- **Granularity controls**: `parallel if(expr)` disables parallelism when `expr` is false; `num_threads(expr)` overrides team size for one region.

---

## 9. Synchronization constructs (Advanced)

### 9.1 The hazard

- A **race condition** = ≥2 threads accessing the same memory, ≥1 is a write, no ordering enforced → **undefined behavior** (corruption, segfaults, silent wrong answers).
- Classic broken example: `int sum=0; #pragma omp parallel for; for (i…) sum += i;` — three runs can give 499500 (correct), 482331 (lost updates), 12039 (massive loss).
- **Trade-off (Amdahl)**: synchronization restores correctness but serializes — high contention destroys scalability.

### 9.2 Constructs catalogue

| Construct | Scope | Cost / mechanism |
|---|---|---|
| `#pragma omp critical [name]` | Arbitrary code block | High — OS lock |
| `#pragma omp atomic` | **Single** memory update (`x op= expr`, `x++`, …) | Low — hardware CAS / LOCK prefix |
| `#pragma omp barrier` | Sync point — all team threads wait | Medium — collective |
| `#pragma omp ordered` (inside `for ordered`) | Runs statement in sequential iteration order | High |
| `#pragma omp flush` | Memory consistency point | Low |
| `nowait` clause | Removes the **implicit** end-of-construct barrier | Negative cost (saves wait) |
| Explicit `omp_*_lock` | Lock variable; pass around / use across functions | Like critical |

### 9.3 `atomic` vs `critical`

| Feature | `atomic` | `critical` |
|---|---|---|
| Scope | One scalar update statement | Any structured block |
| Allowed ops | `+ - * / & ^ \| << >>`, `++`, `--` | Anything |
| Overhead | Low — hardware (CAS) | High — OS-level lock |
| Right of expr evaluated atomically? | **No** — only the update | Whole block atomic |
| Use when… | Counter, accumulator, single-var update | Multi-statement, non-thread-safe library, complex section |

### 9.4 Named vs anonymous `critical`

| | Unnamed `critical` | `critical(name)` |
|---|---|---|
| Mutual exclusion partner | **All** other unnamed critical regions in the program | Only regions with **the same name** |
| Use case | Protect one global resource everywhere | Multiple independent resources can be locked concurrently |
| Risk | Over-serialization — every unnamed CS contends globally | Mis-naming → unintended races |

### 9.5 `single` vs `master` (redux)

| Feature | `single` | `master` |
|---|---|---|
| Who runs it? | First thread to arrive | Thread 0 only |
| Implicit barrier after? | **Yes** | **No** |
| Use case | Thread-safe I/O, one-time init inside parallel region | MPI-style call, lightweight logging |
| Trap | — | Other threads race ahead — usually need manual `#pragma omp barrier` |

### 9.6 Barriers & `nowait`

- Implicit barriers at the **end** of `parallel`, `for`, `sections`, `single` (not `master`).
- Explicit: `#pragma omp barrier` — all team threads must arrive before any proceeds.
- **`nowait`** removes the implicit barrier of a work-sharing construct — fast threads start the next loop while slow ones finish. **Risk**: if the next region reads what the previous one wrote → race.

---

## 10. Race conditions & false sharing

### 10.1 The reduction mistake

- `sum += array[i]` inside `#pragma omp parallel for` without `reduction(+:sum)` is a **read-modify-write race** — non-deterministic, undefined results.
- Fixes (preferred → discouraged): `reduction(+:sum)` > local accumulator + single `atomic` merge > `#pragma omp atomic` per iter > `#pragma omp critical` per iter.

### 10.2 False sharing — the silent performance killer

- CPUs fetch memory in **64-byte cache lines**. If thread A writes `sums[0]` and thread B writes `sums[1]`, the cache-coherence protocol (MESI) **invalidates** the line on the other core — a write-write **ping-pong** even though the data is logically independent.
- Symptom: parallel code is **slower than serial**, scales negatively with thread count.
- **Mitigation 1 — Padding**: pad each entry to a full cache line.

```c
struct AlignedData { int value; char pad[60]; }; // 64-byte cell
struct AlignedData sums[T];
```

- **Mitigation 2 — Local accumulation**: keep a register-resident `int local_sum` inside each thread, do one `atomic` merge at the end. Equivalent to a hand-rolled `reduction`.

---

## 11. SIMD & vectorization

- **TLP (Thread-Level Parallelism)** = multiple instruction streams on multiple cores.
- **DLP (Data-Level Parallelism / SIMD)** = **one** instruction stream operating on **wide registers** (128 b SSE → 256 b AVX2 → 512 b AVX-512) — e.g. AVX-512 processes 16 single-precision floats per instruction.
- TLP and DLP are **orthogonal and composable**.

| Directive | Effect |
|---|---|
| `#pragma omp simd` | Tells compiler the loop is safe to vectorize (ignore assumed deps) |
| `#pragma omp parallel for simd` | Distribute across threads **and** vectorize each thread's chunk |
| `simd reduction(+:dot)` | Horizontal reduction across vector lanes |
| `simd aligned(a, b, c : 64)` | Promise that pointers are 64-byte-aligned — enables aligned vector loads |

- **Alignment matters**: misaligned data forces unaligned loads (slow) or scalar fallback / segfaults.

---

## 12. Tasking model

### 12.1 Why tasks

- `parallel for` requires a **counted loop with known trip count**. It fails for:
  - **`while` loops** (linked lists, streaming input)
  - **Recursion** (trees, quicksort, Fibonacci)
  - **Highly unbalanced** workloads
- **Tasks decouple** "describe work" from "execute work". A `task` directive enqueues a deferred work item; the team's threads pull from the queue.
- **Work stealing**: idle threads steal queued tasks from busy threads' deques → automatic load balancing.

### 12.2 Directives

| Directive | Purpose |
|---|---|
| `#pragma omp task [clauses]` | Generate a deferred task |
| `#pragma omp taskwait` | Suspend current task until **direct children** complete (not grandchildren) |
| `#pragma omp taskloop [grainsize(N)]` | Turn a loop into a set of tasks (vs work-sharing `for`) |
| `task depend(in: x)` / `depend(out: y)` / `depend(inout: z)` | Build a **DAG**: runtime executes tasks in data-availability order, not source order |

### 12.3 Task synchronization & DAG dependencies

- **`taskwait`** is *control flow*: "pause here until my children finish".
- **`depend()`** is *data flow*: declared `in`/`out`/`inout` lists let the runtime infer a Directed Acyclic Graph of producer→consumer relations and schedule tasks accordingly — no manual barriers needed.

### 12.4 Granularity — the Goldilocks problem

- **Too fine** (1 op per task): scheduling overhead > work done → slower than serial.
- **Too coarse** (whole loop = 1 task): load imbalance, idle cores.
- **Optimal**: tune `grainsize(N)` on `taskloop` so each task does enough work to amortise scheduling but small enough to balance across cores.

---

## Glossary

- **Directive / pragma** — compiler hint (`#pragma omp …`) attached to a structured block.
- **Structured block** — code with a single entry and single exit point (required by every OpenMP construct).
- **Team** — the master + worker threads executing one parallel region.
- **Fork–Join** — master spawns workers at `parallel`, all rejoin at the closing brace.
- **Work sharing** — splitting iterations or sections among the existing team (no new threads).
- **Shared variable** — single memory location visible to all threads; races possible.
- **Private variable** — per-thread copy; no race, no value transfer across threads.
- **Reduction** — built-in pattern: private copies merged with an associative op at region end.
- **Implicit barrier** — automatic synchronization at the end of `parallel`/`for`/`sections`/`single`.
- **`nowait`** — clause that removes the implicit barrier of a work-sharing construct.
- **Critical section** — code region executed by ≤1 thread at a time (mutex-style).
- **Atomic** — hardware-level (CAS / LOCK-prefixed) single-update protection — far cheaper than critical.
- **Race condition** — concurrent access to shared memory with ≥1 write, unordered → undefined behavior.
- **False sharing** — independent variables on the same cache line causing coherence ping-pong.
- **SIMD / DLP** — one instruction stream, wide vector registers (e.g. AVX-512 = 512 b = 16 floats).
- **Task** — deferred unit of work pushed onto a runtime-managed queue.
- **Work stealing** — idle threads pull queued tasks from busy threads' deques.

---

## Likely exam targets

1. **Define OpenMP**, describe its **shared-memory** assumption, and explain *why* the same source code stays correct when compiled without OpenMP support.
2. **Draw the fork–join model** with a parallel region; label master, team, fork, parallel work, implicit barrier, join.
3. **List the five categories of OpenMP constructs** and give one directive per category.
4. **Default data-sharing rules** + meaning of `shared`, `private`, `firstprivate`, `lastprivate`, `reduction`, `default(none)`. Predict the value of a variable after a region.
5. **Reduction**: explain how it's implemented (private copies + merge), give the identity values for `+, *, &, |`, and write the directive for `sum += a[i]*b[i]`.
6. **Schedule comparison**: when to use `static`, `dynamic`, `guided` — give one workload example each, and the load-balance / overhead trade-off.
7. **`critical` vs `atomic` vs `reduction`**: build a table by scope, overhead, allowed operations, when to use each.
8. **`single` vs `master`**: behavior, implicit barrier presence, when the `master` trap bites.
9. **False sharing**: explain the cache-line mechanism, write the broken `sums[T]` loop, and give the **padding** fix (`char pad[60];`) with byte-count justification.
10. **Tasking**: explain why `parallel for` is insufficient for recursion / linked lists; sketch `fib(n)` with `task` + `taskwait`; explain **work stealing** in one sentence.

---

## Pitfalls

- **Forgetting `reduction` on accumulators** — `sum += …` without it is *the* most common race bug. Compiles & runs, gives different answers every run.
- **`master` has no implicit barrier** — workers race past while the master is still working. Add `#pragma omp barrier` manually when subsequent code depends on `master`'s output.
- **Unnamed `critical` regions are globally mutually exclusive** — even unrelated critical sections serialize against each other. Use named critical to recover concurrency.
- **`atomic` does not protect the right-hand expression** — `x = x + f()` is atomic on the write to `x` but `f()` itself can race.
- **`nowait` introduces races** when the next region reads what the previous wrote — only safe if there's no cross-region dependency.
- **False sharing on hot per-thread counters** — a length-T array of `int` (4 B each) all live in the same 64 B cache line → expect catastrophic slowdown. Pad or use registers.
- **Unaligned SIMD loads** — promising `aligned(p:64)` when `p` is not actually 64-byte-aligned → segfault. Use `posix_memalign` / `aligned_alloc`.
- **`taskwait` only waits for direct children**, not transitive descendants. For full-subtree sync use `taskgroup` (OpenMP 4.0+) or a manual barrier outside.
