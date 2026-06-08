# Chapter 3 — MPI (OpenMPI)

> Source: `OpenMPI Lecture 1.pdf` (intro, distributed memory, point-to-point), `OpenMPI Lecture 2.pdf` (collectives + non-blocking + derived datatypes), `OpenMPI Lecture 3.pdf` (AI scaling, 3D parallelism, topologies, communicator management).

## Bird's eye view

- **Memory wall**: heat dissipation + memory bandwidth cap *vertical* scaling of a single CPU; the only escape is **horizontal scaling** — many networked nodes.
- **OpenMP vs OpenMPI**: shared memory (single node, threads share RAM) vs **distributed memory** (multi-node, each process has private RAM, must exchange messages explicitly).
- MPI execution model = **SPMD** (Single Program, Multiple Data): one binary launched on many ranks, each branches on its **rank ID** within a **communicator**.
- Core skeleton: `MPI_Init` → `MPI_Comm_rank` / `MPI_Comm_size` → work → `MPI_Finalize`. The default communicator is `MPI_COMM_WORLD`.
- **Point-to-point** = `MPI_Send` / `MPI_Recv` (blocking): explicit (data, count, datatype, dest/source, tag, comm). MPI datatypes (`MPI_INT`, `MPI_DOUBLE`, ...) abstract over heterogeneous architectures.
- **Collectives** = all ranks in the communicator participate; cleaner code, and MPI uses **tree algorithms (O(log p))** under the hood instead of naive linear sends.
- Core collectives: `MPI_Barrier` (sync), `MPI_Bcast` (1→all), `MPI_Scatter` / `MPI_Gather` (1↔all chunks), `MPI_Allgather`, `MPI_Alltoall`, `MPI_Reduce` / `MPI_Allreduce` (with op `MPI_SUM`, `MPI_MAX`, ...).
- **Blocking calls waste CPU cycles** while the NIC transfers data. **Non-blocking** `MPI_Isend` / `MPI_Irecv` return immediately, returning an `MPI_Request`; you finish with `MPI_Wait` or poll with `MPI_Test`. This enables **computation/communication overlap**.
- **AI scaling wall**: gradient synchronization (`MPI_Allreduce`) dominates step time in distributed DL. Amdahl's law caps speedup unless we hide network latency.
- **3D parallelism** in DL = Data Parallel + Tensor Parallel + Pipeline Parallel, each requiring its **own sub-communicator** carved out of `MPI_COMM_WORLD`.
- **Physical topologies** (bus, ring, mesh, torus, fat-tree, dragonfly) are the wiring; **logical/virtual topologies** (Cartesian grids, graph) are how MPI lets us index ranks meaningfully.
- **`MPI_Comm_split` + `MPI_Cart_create`** are the two pillars of advanced MPI: one carves communicators by *color*, the other folds a 1D rank list into an N-D grid (with optional periodic wrap = torus).
- **Hierarchical All-Reduce** = split per-node communicator (`MPI_COMM_TYPE_SHARED`) → local reduce → leader-only inter-node all-reduce → local broadcast. Keeps 90% of traffic on the fast intra-node bus.

---

# Part 1 — MPI Fundamentals (Lecture 1)

## 1. Motivation: why distributed memory

### 1.1. The memory wall

- A single CPU cannot grow indefinitely: heat dissipation, leakage, and **memory bandwidth** cap clock-speed and RAM throughput.
- Vertical scaling (bigger machine) saturates → switch to **horizontal scaling**: connect many standard nodes over a network.
- Modern *grand challenges* (climate, genomics, LLM training) need hundreds–thousands of nodes communicating efficiently.

### 1.2. Shared vs distributed memory

| | **OpenMP (shared memory)** | **OpenMPI (distributed memory)** |
|---|---|---|
| Scope | Single node | Multiple nodes |
| Memory | Threads share **one RAM** | Each process has **private RAM** |
| Communication | Implicit (just read shared variables) | **Explicit message passing** |
| Pros | Easier to program | Scales to thousands of machines |
| Cons | Bound by motherboard / cores | Requires explicit `Send` / `Recv` |
| Typical primitive | `#pragma omp parallel` | `MPI_Send` / `MPI_Recv` |

A single multicore box hits a hard scalability ceiling. MPI breaks it by forcing the programmer to think about *where the data lives*.

## 2. The MPI execution model

### 2.1. SPMD

- **Single Program, Multiple Data**: compile one executable, launch it `N` times (one per *rank*).
- Each instance is a *separate OS process* with its own address space.
- Differentiation between ranks happens via the **rank ID** — `if (rank == 0) { ... } else { ... }`.

### 2.2. Communicator, rank, size

- **Communicator** = a group of processes that can exchange messages. The default is `MPI_COMM_WORLD` (all processes launched).
- **Size** = number of processes in the communicator.
- **Rank** = unique integer ID, `0 .. size-1`.

Skeleton functions:

```
MPI_Init(&argc, &argv);
MPI_Comm_size(MPI_COMM_WORLD, &world_size);
MPI_Comm_rank(MPI_COMM_WORLD, &world_rank);
// ... work ...
MPI_Finalize();
```

`MPI_Get_processor_name` returns the host machine name — useful to confirm processes are spread across nodes.

> Stdout from different ranks is **interleaved unpredictably**: distributed processes execute asynchronously and independently.

## 3. Point-to-point communication

### 3.1. The primitives

The fundamental mechanism: move bytes from one specific process's memory to another's.

```
MPI_Send(buf, count, datatype, dest,   tag, comm);
MPI_Recv(buf, count, datatype, source, tag, comm, &status);
```

- **Sender** specifies: data pointer, count, datatype, destination rank, tag, communicator.
- **Receiver** specifies: buffer pointer, count, datatype, *expected* source rank, *matching* tag, communicator, and a status object.
- **Tag** = integer message ID — lets you distinguish multiple messages between the same pair.
- **Status** = struct giving sender rank, tag, and actual count of the received message.

### 3.2. Blocking semantics

- `MPI_Send`: returns when the send buffer is **safe to reuse** (the bytes have either been copied out or successfully delivered).
- `MPI_Recv`: returns when the **entire message has arrived** in the receive buffer.
- These are **blocking** calls — the CPU may sit idle while the NIC works.

### 3.3. MPI datatypes

MPI must know *how many bytes* to push over the wire and how to interpret them on a heterogeneous cluster (different endianness/sizes). You cannot send raw `int` directly.

| C type | MPI type |
|---|---|
| `short` | `MPI_SHORT` |
| `int` | `MPI_INT` |
| `long` | `MPI_LONG` |
| `float` | `MPI_FLOAT` |
| `double` | `MPI_DOUBLE` |
| `char` | `MPI_CHAR` |

**Derived datatypes** (covered in Part 2): `MPI_Type_contiguous`, `MPI_Type_vector`, `MPI_Type_create_struct` — for non-contiguous / heterogeneous memory layouts.

### 3.4. Send modes (mental model)

| Mode | When `MPI_Send` returns | Use case |
|---|---|---|
| **Standard** | When safe to reuse buffer (MPI chooses buffered or sync internally) | Default `MPI_Send` |
| **Buffered** (`MPI_Bsend`) | After data is copied to a user-supplied buffer | When you must control buffering |
| **Synchronous** (`MPI_Ssend`) | Only after matching `Recv` has started | Guarantees rendezvous; useful for debugging |
| **Ready** (`MPI_Rsend`) | Assumes matching `Recv` is already posted | Fastest, but unsafe if recv not posted |

### 3.5. Deadlock from circular sends

If every rank does `Send` then `Recv` *and* all sends are synchronous, the ring deadlocks — no one ever finishes their `Send`.

**Fix**: stagger by rank parity — *even ranks send first, odd ranks receive first* (or vice versa). General rule: impose a **total order** on the sends.

---

# Part 2 — Collectives & Non-blocking I/O (Lecture 2)

## 4. Why collectives

- Point-to-point loops for global patterns (broadcasting, reducing) are verbose and slow.
- A **collective** = single call involving **every process in the communicator**.
- Benefits:
  - **Simplicity**: replaces hand-coded for-loops of sends/recvs.
  - **Efficiency**: MPI implements them with tree / pipeline algorithms — e.g. broadcast and reduce run in:

```math
T_{\text{collective}} = \mathcal{O}(\log p) \quad \text{vs.} \quad \mathcal{O}(p) \text{ for naive linear sends}
```

  - **Readability**: intent (broadcast / scatter / reduce) is obvious.

### 4.1. Golden rules

- **Every process in the communicator must call the collective.** If even one rank skips, the program deadlocks.
- **No tags** — collectives are matched by *order of execution*, not by message tags.
- **Blocking by default** — the call returns only when this process's participation is complete and its buffer is safe to reuse.

## 5. Synchronization: `MPI_Barrier`

```
MPI_Barrier(MPI_Comm comm);
```

- All processes block until **every** process has reached the barrier.
- Uses: timing parallel sections cleanly, ensuring file writes finish before another rank reads.

## 6. Data movement collectives

### 6.1. `MPI_Bcast` — one to all

```
MPI_Bcast(buf, count, type, root, comm);
```

- **Root** holds the data; all others receive a copy into the same buffer.
- Implemented with a binary/n-ary tree internally — `O(log p)`.

### 6.2. `MPI_Scatter` — chunks from root

```
MPI_Scatter(sendbuf, sendcount, sendtype,
            recvbuf, recvcount, recvtype,
            root, comm);
```

- Divides `sendbuf` on the root into `p` equal chunks, sends chunk `i` to rank `i`.
- **Trap**: `sendcount` is the count *per process*, not the total. For 100 elements over 4 procs → `sendcount = 25`.

### 6.3. `MPI_Gather` — inverse of scatter

```
MPI_Gather(sendbuf, sendcount, sendtype,
           recvbuf, recvcount, recvtype,
           root, comm);
```

- Every rank contributes its chunk; root receives the concatenation.

### 6.4. Variants

- **`MPI_Allgather`** = `Gather` + `Bcast` of the result — every rank ends up with the full concatenation.
- **`MPI_Alltoall`** = every rank sends a personalized chunk to every other rank (transpose-like).

| Collective | Pattern | Root needed? |
|---|---|---|
| `Bcast` | 1 → all (same data) | Yes |
| `Scatter` | 1 → all (different chunks) | Yes |
| `Gather` | all → 1 | Yes |
| `Allgather` | all → all (concatenation) | No |
| `Alltoall` | all → all (personalized chunks) | No |

## 7. Reductions: `MPI_Reduce` / `MPI_Allreduce`

```
MPI_Reduce(sendbuf, recvbuf, count, type, op, root, comm);
MPI_Allreduce(sendbuf, recvbuf, count, type, op,       comm);
```

- Collects values from all ranks and applies an associative op.
- Common `MPI_Op`: `MPI_SUM`, `MPI_PROD`, `MPI_MAX`, `MPI_MIN`, `MPI_LAND`, `MPI_LOR`, `MPI_MAXLOC`, `MPI_MINLOC`.
- `MPI_Reduce`: result on root only.
- `MPI_Allreduce`: result on **every** rank (the workhorse of distributed deep learning — gradient averaging).
- Tree implementation cost:

```math
T_{\text{reduce}}(p, m) \approx \mathcal{O}(\log p) \cdot (\alpha + m\beta)
```

(`α` = latency per message, `β` = inverse bandwidth, `m` = message size).

## 8. Bottleneck of blocking I/O

- `MPI_Send` / `MPI_Recv` block the CPU until the NIC finishes — *idle CPU = wasted resource* in HPC.
- We want: CPU computes while network transmits in the background.

## 9. Non-blocking communication

### 9.1. `MPI_Isend` / `MPI_Irecv`

```
MPI_Isend(buf, count, type, dest,   tag, comm, &request);
MPI_Irecv(buf, count, type, source, tag, comm, &request);
```

- Return **immediately**, before the transfer actually finishes.
- You **must not** modify the send buffer (or read the recv buffer) until the request completes.
- Each call returns an `MPI_Request` handle to track the in-flight operation.

### 9.2. `MPI_Wait` and `MPI_Test`

```
MPI_Wait(&request, &status);          // block until done
MPI_Test(&request, &flag, &status);   // non-blocking poll; flag=1 if done
```

### 9.3. The overlap paradigm

1. Start data transfer with `MPI_Isend` / `MPI_Irecv`.
2. Do independent CPU work (any computation that does *not* touch the in-flight buffer).
3. Call `MPI_Wait` once you need the data.
4. Use the data.

| | **Blocking** (`Send`/`Recv`) | **Non-blocking** (`Isend`/`Irecv`) |
|---|---|---|
| Returns when? | Buffer safe to reuse | Immediately |
| Buffer safe immediately? | Yes | **No** — wait for completion |
| Overlap compute & comm? | **No** | **Yes** |
| Needs `Request` object? | No | Yes (track with `Wait`/`Test`) |
| Risk | Deadlock from circular sends | Use-before-ready bugs |

## 10. Derived datatypes (brief)

For non-contiguous data (a matrix column in row-major C; a struct with padding) the naive solution — manually packing into a temp buffer — wastes CPU and RAM.

- **`MPI_Type_contiguous`** — N copies of an existing type back-to-back (simple arrays of a custom type).
- **`MPI_Type_vector(count, blocklength, stride, oldtype, &newtype)`** — strided pattern (e.g. one column of a row-major matrix: `count=N, blocklength=1, stride=N_cols`).
- **`MPI_Type_create_struct`** — heterogeneous fields with explicit byte **displacements** (use `offsetof(struct, field)` from `<stddef.h>` to account for compiler padding).
- Always **`MPI_Type_commit(&new_type)`** before use, and **`MPI_Type_free(&new_type)`** when done.

---

# Part 3 — Advanced MPI for AI (Lecture 3)

## 11. The AI scaling / communication wall

- An LLM like GPT-3 has ~175 B parameters. In FP32 that is ~700 GB for weights alone; with gradients + Adam optimizer states (Adam ≈ 3× weights), one replica needs **> 2 TB**.
- No single server holds this — both compute *and* memory must be distributed across hundreds of nodes.
- **Communication wall**: once compute is distributed, network becomes the bottleneck. Amdahl's law caps speedup:

```math
\text{Speedup} \le \frac{1}{(1 - f) + f / p + T_{\text{comm}}(p)/T_{\text{serial}}}
```

If gradient computation takes 50 ms but `MPI_Allreduce` takes 150 ms, GPUs idle 75% of the time. We must design topologies and patterns that *hide* this latency.

## 12. 3D parallelism and communicator mapping

Three orthogonal ways to split a neural network across ranks:

| Strategy | What is split | Communication pattern |
|---|---|---|
| **Data Parallelism (DP)** | The *mini-batch* — every rank has a full model copy, processes different data, then `Allreduce`s gradients | All-reduce on gradients (large) |
| **Tensor Parallelism (TP)** | A single layer's *weight matrix* across ranks (rows / cols) | All-reduce / all-gather inside each layer |
| **Pipeline Parallelism (PP)** | *Different layers* on different ranks (assembly-line) | Point-to-point send of activations / gradients between stages |

- A single flat `MPI_COMM_WORLD` cannot represent these simultaneously — each strategy needs its **own sub-communicator** carved out (DP-comm, TP-comm, PP-comm).
- Total ranks: `world_size = |DP groups| × |TP groups| × |PP groups|`.

## 13. Hardware hierarchy — bandwidth is *not* uniform

| Level | Within | Interconnect | Bandwidth | Latency |
|---|---|---|---|---|
| **Intra-node** | Same server / motherboard | PCIe, NVLink | ~600 GB/s | µs |
| **Inter-node** | Across servers / datacenter | InfiniBand, RoCE, Ethernet | ~50–200 GB/s | higher, prone to switch congestion |

> *Rule*: keep as much traffic as possible on the intra-node bus. This motivates **hierarchical** algorithms (§17).

## 14. Physical network topologies

How servers / switches are wired:

| Topology | Shape | Notes |
|---|---|---|
| **Bus** | All on one shared line | Trivial; contention-limited |
| **Ring** | Each node has 2 neighbours, closed loop | Used logically for ring all-reduce |
| **Mesh** | 2D grid, edges = wires | Natural for stencils |
| **Torus** | Mesh with wrap-around (periodic) | Used by Google TPU (3D torus); no edge nodes |
| **Fat-tree** | Spine + leaf layers, bandwidth fattens upward | Standard GPU datacenter fabric |
| **Dragonfly** | Hierarchical groups of high-radix switches | Exascale systems |

The number of switch hops an MPI message traverses directly drives training latency.

## 15. Logical communication topologies

How we *index ranks* on top of the physical wiring:

- **Parameter Server (legacy)**: workers send gradients to rank 0; rank 0 averages and broadcasts back. **Flaw**: rank 0's link is a bottleneck — bandwidth does not scale with `p`.
- **Ring (modern)**: ranks form a logical circle, each talks only to neighbours `i-1` and `i+1`. Every link is fully utilised — no central bottleneck.

### 15.1. Ring All-Reduce (deep dive)

Phase 1 — **Scatter-Reduce**:
- Split the array (gradients) into `N` chunks where `N = ranks`.
- For `N-1` steps: each rank sends one chunk to its neighbour, receives one chunk, adds them, passes along.
- After `N-1` steps: each rank holds the *final reduced sum* of exactly one chunk.

Phase 2 — **All-Gather**:
- For `N-1` more steps: each rank passes its finalized chunk around the ring; everyone copies into their local array.
- After `N-1` steps: every rank has the fully reduced array.

```math
T_{\text{ring-allreduce}} = 2(N-1) \cdot \left(\alpha + \frac{m}{N}\beta\right)
```

Bandwidth term is constant in `N`: no single node is overloaded.

## 16. Virtual topologies — Cartesian grids

A flat 1D rank list (`0, 1, ..., N-1`) maps poorly onto 2D matrix layers or 3D hardware tori. MPI lets us overlay a **virtual topology** on a communicator.

### 16.1. `MPI_Cart_create`

```
MPI_Cart_create(comm_old, ndims, dims, periods, reorder, &comm_cart);
```

- `dims[]`: size per dimension (e.g. `{4, 4}` for a 4×4 grid). Use `MPI_Dims_create` to auto-factor.
- `periods[]`: per-dimension boolean — `true` = wrap-around (cylinder / torus); `false` = open edges (grid).
- `reorder`: if `true`, MPI may permute rank IDs to improve **physical network locality**.

### 16.2. Translating between ranks and coordinates

| Function | Question it answers |
|---|---|
| `MPI_Cart_coords(comm, rank, ndims, &coords)` | "I am rank 7 — what are my (X, Y) coordinates?" |
| `MPI_Cart_rank(comm, coords, &rank)` | "Who lives at (2, 3)? I need their rank ID." |
| `MPI_Cart_shift(comm, direction, disp, &source, &dest)` | "Who is my left/right neighbour along axis `direction`?" |

- `MPI_Cart_shift` returns `MPI_PROC_NULL` at non-periodic edges — sending to `MPI_PROC_NULL` is a no-op, so boundary code is free of out-of-bounds branches.

### 16.3. AI use case

- Map Tensor Parallelism to the X-axis and Pipeline Parallelism to the Y-axis: a rank's `(X, Y)` instantly identifies its role.
- Google TPUs are physically wired as a **3D torus** — Cartesian logic in MPI matches the copper directly.

## 17. Communicator management

### 17.1. `MPI_Comm_split` — color + key

```
MPI_Comm_split(comm_old, color, key, &new_comm);
```

- Every rank passes a **color** (int). Ranks with the same color end up in the same new communicator.
- **Key** determines the rank ordering inside the new communicator (typically pass `world_rank`).
- Pass color = `MPI_UNDEFINED` to exclude a rank → returns `MPI_COMM_NULL` for that rank.
- Example: even ranks → color 0, odd ranks → color 1 → two disjoint sub-networks.

### 17.2. `MPI_Comm_dup`

- Duplicates a communicator. Useful for **libraries** that want a private communication namespace, isolated from user traffic (no tag collisions).

### 17.3. `MPI_Comm_split_type` — hardware-aware

```
MPI_Comm_split_type(comm, MPI_COMM_TYPE_SHARED, key, info, &node_comm);
```

- Splits ranks by **shared memory domain** (i.e., same physical machine) — no need to hardcode hostnames or IPs.
- Returns one communicator per node: every rank's `node_comm` contains exactly the ranks on its server.

## 18. Hierarchical All-Reduce — putting it together

The classic combination of `MPI_Comm_split_type` + hierarchy to avoid switch congestion.

**Preparation**:
1. Split `MPI_COMM_WORLD` with `MPI_COMM_TYPE_SHARED` → each node has a **Local Communicator**.
2. The local rank-0 of each node is the **Node Leader**.
3. Split `MPI_COMM_WORLD` again: leaders get color 1, everyone else gets `MPI_UNDEFINED` → **Leader Communicator** that spans the datacenter.

**Execution**:

| Step | Communicator | Operation | Bandwidth used |
|---|---|---|---|
| 1. Intra-node reduce | Local | `MPI_Reduce` (target: leader) | Intra-node (NVLink/PCIe) — fast |
| 2. Inter-node all-reduce | Leader | `MPI_Allreduce` (only leaders) | One message per server crosses the switch |
| 3. Intra-node broadcast | Local | `MPI_Bcast` from leader | Intra-node — fast |

> Result: ~90% of the bytes stay on the intra-node bus; switch traffic drops drastically. This is exactly the pattern used by NCCL under PyTorch / TensorFlow.

## 19. Beyond MPI — NCCL / RCCL

- Modern AI frameworks rarely call raw MPI for gradients.
- **NCCL** (NVIDIA) and **RCCL** (AMD) implement the same ring / hierarchical algorithms but move data **directly between GPU memories** via GPUDirect RDMA (bypassing CPU/host memory).
- MPI remains the standard for process launching, orchestration, and CPU-side workloads.

---

## Glossary

- **MPI** — Message Passing Interface; standardised API (MPI-1.0 1994) for distributed-memory parallel programming.
- **OpenMPI** — open-source implementation of the MPI standard; powers most Exascale systems.
- **SPMD** — Single Program, Multiple Data; one executable, many ranks branching on rank ID.
- **Communicator** — group of processes that can exchange messages (default `MPI_COMM_WORLD`).
- **Rank** — unique ID of a process in a communicator, `0 .. size−1`.
- **Size** — number of processes in a communicator.
- **Tag** — integer label on point-to-point messages used to distinguish them.
- **Blocking** — call returns only when buffer is safe to reuse / message has arrived.
- **Non-blocking (Immediate)** — `Isend` / `Irecv` return immediately; completion tracked via `MPI_Request`.
- **`MPI_Request`** — opaque handle for an in-flight non-blocking operation.
- **Collective** — operation involving *every* rank in a communicator (`Bcast`, `Scatter`, `Gather`, `Reduce`, `Allreduce`, `Barrier`).
- **`MPI_Op`** — reduction operator (`MPI_SUM`, `MPI_MAX`, `MPI_MIN`, `MPI_PROD`, ...).
- **Derived datatype** — user-defined memory layout (`Type_vector`, `Type_create_struct`) to send non-contiguous data without manual packing.
- **3D Parallelism** — combination of Data + Tensor + Pipeline parallelism for large model training.
- **Virtual topology** — logical structure (Cartesian grid, graph) overlaid on a communicator.
- **Cartesian grid** — N-dimensional virtual topology with optional periodic wrap (`MPI_Cart_create`).
- **Periodic** — dimension wraps around (cylinder/torus); non-periodic = open edges, with `MPI_PROC_NULL` at boundaries.
- **`MPI_PROC_NULL`** — sentinel "rank"; sending to / receiving from it is a safe no-op.
- **`MPI_Comm_split`** — partition a communicator by color; new rank ordering by key.
- **Hierarchical All-Reduce** — intra-node reduce → inter-node leader all-reduce → intra-node broadcast.
- **NCCL** — NVIDIA Collective Communication Library; GPU-direct equivalent of MPI collectives.

---

## Likely exam targets

1. **Memory wall & paradigm shift**: state the limits of vertical scaling and contrast OpenMP (shared) vs OpenMPI (distributed) with a 4-row table.
2. **MPI skeleton**: list the four mandatory calls (`MPI_Init`, `MPI_Comm_rank`, `MPI_Comm_size`, `MPI_Finalize`) and explain SPMD, communicator, rank, size.
3. **Point-to-point signature**: write the prototypes of `MPI_Send` and `MPI_Recv`, identify all 6 (resp. 7) arguments, and explain what *blocking* means.
4. **Why MPI datatypes** (not raw C types): heterogeneous architectures + need to describe non-contiguous memory; give three examples (`MPI_Type_vector` for a matrix column; `MPI_Type_create_struct` with `offsetof`; `MPI_Type_contiguous`).
5. **Circular-send deadlock**: explain why every rank doing `Send` then `Recv` deadlocks under synchronous semantics, and give the even/odd fix.
6. **Collective rules**: state the three golden rules (every rank must call; no tags; blocking by default) and explain the `O(log p)` advantage from tree algorithms.
7. **Bcast vs Scatter vs Gather vs Allgather vs Alltoall**: produce the 5-row table and identify which require a root.
8. **Reduce vs Allreduce**: write both signatures, list four `MPI_Op` values, and explain why Allreduce is the *workhorse of distributed deep learning*.
9. **Non-blocking pattern**: write the 4-step overlap recipe (`Isend` → independent compute → `Wait` → use data) and contrast `MPI_Wait` vs `MPI_Test`.
10. **3D parallelism**: name DP / TP / PP, explain what each splits, and justify why one `MPI_COMM_WORLD` is insufficient.
11. **Cartesian grids**: write the `MPI_Cart_create` prototype, explain `dims` / `periods` / `reorder`, and describe the role of `MPI_Cart_coords`, `MPI_Cart_rank`, `MPI_Cart_shift` (including `MPI_PROC_NULL` at non-periodic edges).
12. **Hierarchical All-Reduce**: describe the 3-step recipe using `MPI_Comm_split_type(MPI_COMM_TYPE_SHARED, ...)`, and explain why it minimises top-of-rack switch traffic. Bonus: link to NCCL.

---

## Pitfalls

- **Forgetting `MPI_Finalize`** or calling MPI functions after it: undefined behaviour. Always pair `Init` / `Finalize`.
- **Rank assumes ordering of stdout**: stdout from different ranks is interleaved non-deterministically — never rely on print order for correctness.
- **Send/Recv tag/source mismatch**: receiver waits forever if no message matches `(source, tag)`. Use `MPI_ANY_SOURCE` / `MPI_ANY_TAG` only deliberately; recover the actual sender from `status`.
- **`MPI_Scatter` count gotcha**: `sendcount` is **per destination**, not the total array size — `100 / 4 procs ⇒ sendcount = 25`.
- **Collective skipped by one rank** → entire program hangs forever. Every rank in the communicator *must* call the collective.
- **Mutating buffers before non-blocking completion**: touching the send buffer between `MPI_Isend` and `MPI_Wait` corrupts the in-flight message silently.
- **Not committing / freeing derived types**: forgetting `MPI_Type_commit` makes the type unusable; forgetting `MPI_Type_free` leaks resources over long runs.
- **Sending a C struct as raw bytes**: compiler padding differs between nodes — always use `MPI_Type_create_struct` with `offsetof` for safe heterogeneous transfer.
