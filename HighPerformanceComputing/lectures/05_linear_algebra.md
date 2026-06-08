# Chapter 5 — Linear Algebra Case Studies (Dense GEMM + Sparse SpMV)

> Source: `Dense MxM.pdf` (44 pages — parallelization patterns, systolic shifts, SUMMA), `HPC GEMM.pdf` (29 pages — formal Cannon's algorithm lecture, ENSIA April 2026), `SpMV.pdf` (24 pages — sparse formats, serial/parallel SpMV).

## Bird's eye view

- **GEMM (General Matrix-Matrix multiply)** `C = A·B` with `A,B,C ∈ ℝ^{n×n}` is the workhorse of HPC: `≈ 2n³` flops on `≈ 3n²` data → **arithmetic intensity ~ n** → **compute-bound** as `n` grows. This is what makes it parallelize well.
- A **naive 1D row partition** forces each processor to receive **the full matrix B** (`O(n²)` data) — total communication scales like `O(n³)`, the same as compute. **Bad ratio.**
- A **2D block partition** on a `√p × √p` grid gives each processor an `n/√p × n/√p` block. Memory per proc drops to `O(n²/p)` and communication to `O(n²/√p)`.
- **Cannon's algorithm (1969)** is the canonical 2D systolic GEMM: an **initial skew** of `A` (left by row index) and `B` (up by column index), then `√p` rounds of **local multiply + cyclic shift** (A left by 1, B up by 1). One copy of every element exists at any time.
- Cannon achieves total time `O(n³/p + τ√p + μ·n²/√p)` — **memory-optimal**, matches the Hong–Kung communication lower bound. Constraint: `p` must be a **perfect square** and `A,B` must be square.
- **SUMMA (van de Geijn & Watts 1997)** replaces Cannon's shifts with **row/column broadcasts of panels** of width `b`. No skew, runs on any `p_r × p_c` grid, used in **ScaLAPACK / Elemental**.
- The **inner-product view** computes `n²` dot products; the **outer-product view** writes `C = Σ_k A[:,k]·B[k,:]` — SUMMA is the parallel realization of outer-product GEMM.
- **Sparse matrices** (`nnz ≪ n²`) make dense storage and dense algorithms wasteful: storing zeros wastes memory and flops. Markowitz (1990 Nobel) coined "sparse" in the 1950s.
- **Storage formats** — COO (build), CSR (default), CSC (column ops), ELL (SIMD/GPU), DIA (stencils), HYB (ELL + COO outliers) — each trades index overhead vs access regularity.
- **SpMV** `y = A·x` is **memory-bound**: only 2 flops per nonzero (`*` and `+=`) against ~12–16 bytes loaded (value + col index + part of x). **Roofline** lives on the bandwidth-limited side.
- **Parallel SpMV bottlenecks**: (i) **load imbalance** when rows have wildly different `nnz`, (ii) **indirect access** `x[ind[j]]` destroys cache locality, (iii) reductions when partitioning columns.
- **GPU SpMV**: CSR with **warp-per-row** or **vector-per-row** for irregular matrices, ELL/DIA for regular structures (stencils, banded), HYB for mixed.

---

## 0. Outline

- Part 1 — Dense MxM: from naive partition to 2D systolic shift
- Part 2 — Cannon's algorithm (formal): skew, compute-and-shift, analysis, SUMMA comparison
- Part 3 — Sparse storage formats and SpMV (serial + parallel)

---

## 1. Dense MxM: why parallelization patterns matter

### 1.1. The problem and its arithmetic intensity

```math
C_{ij} = \sum_{k=0}^{n-1} A_{ik} B_{kj}, \qquad 0 \le i,j \le n-1
```

- Sequential cost: `≈ 2n³` flops on `O(n²)` data.
- For `n = 10 000` on a 1 TFLOP/s core: `t ≈ 2·10¹² / 10¹² = 2000 s` — single-core is hopeless, hence parallelism.

```math
\text{Sequential complexity: } O(n^3) \qquad \text{Data: } O(n^2)
```

### 1.2. Why naive 1D partition is bad

- Assign each `C_{ij}` (or a row of C) to one processor.
- To compute its share, each processor needs **full row `i` of A** (`n` elements) and **full column `j` of B** (`n` elements), or for a row-partition the **entire B** (`n²` elements).
- Total communication: `O(n³)` — same order as the computation. Bandwidth, not flops, becomes the bottleneck.

### 1.3. The 2D block partition

- Arrange `p = q²` processors as a `q × q` (= `√p × √p`) **2D grid**.
- Partition each of `A, B, C` into `q × q` blocks of size `n/q × n/q`.
- Processor `P(i,j)` owns `A_{ij}, B_{ij}, C_{ij}`. To compute `C_{ij}` it needs the **block row i of A** and the **block column j of B** — far less data than naive 1D.

### 1.4. Restriction: one copy + cyclic shift insight

- "**One instance of each element** of A, B across all processors" — no replication allowed.
- Therefore: at each step, A blocks must **flow left one hop at a time** along their processor row, and B blocks must **flow north one hop at a time** along their processor column. A classic **systolic** pattern.
- In each round, every processor must hold a **matching (A block, B block) pair** that contributes to its own `C_{ij}`. The skew + shift schedule guarantees this matching.

---

## 2. Cannon's algorithm (the formal GEMM treatment)

### 2.1. Motivation

- Same `O(n³/p)` compute as naive 2D parallel MxM.
- **Memory per processor**: only `O(n²/p)` — one A block, one B block, one C accumulator. No replication of A or B.
- **One copy of each element** of A and B exists across the system at any time.

### 2.2. Setup — 2D torus mesh

- `p = q²` processors arranged as a **q × q torus** (`q = √p`). Each `P(i,j)` has four neighbours with wrap-around:
  - Left `P(i, (j-1) mod q)`, Right `P(i, (j+1) mod q)`, Up `P((i-1) mod q, j)`, Down `P((i+1) mod q, j)`.
- The torus wrap-around removes boundary edge cases — every cyclic shift is well-defined.

### 2.3. Three phases

**Phase 1 — Initial Skew (alignment).** Cyclically shift:
- Row `i` of A **left by `i` positions** (for `i = 0, 1, …, q-1`).
- Column `j` of B **up by `j` positions** (for `j = 0, 1, …, q-1`).

After the skew, processor `P(i,j)` holds:

```math
A_{i,\,(i+j) \bmod q} \quad \text{and} \quad B_{(i+j) \bmod q,\,j}
```

The inner index `(i+j) mod q` **matches**: this is exactly the first term `A_{ik} B_{kj}` with `k = (i+j) mod q` needed for `C_{ij}`.

**Phase 2 — Compute and Shift (repeat `q` times).** For step `s = 0, 1, …, q-1`:

```math
C_{ij} \mathrel{+}= A^{\text{loc}}_{ij} \times B^{\text{loc}}_{ij}
```

then shift A blocks **left by 1** and B blocks **up by 1** (circular on the torus). After step `s`:

```math
P(i,j) \text{ holds } A_{i,\,(i+j+s) \bmod q} \text{ and } B_{(i+j+s) \bmod q,\,j}
```

**Phase 3 — Result.** After `q` rounds, the index `(i+j+s) mod q` has visited every `k ∈ {0,…,q-1}` exactly once, so:

```math
C_{ij} = \sum_{s=0}^{q-1} A_{i,\,(i+j+s) \bmod q}\, B_{(i+j+s) \bmod q,\,j} \;=\; \sum_{k=0}^{q-1} A_{ik} B_{kj}
```

No final gather — every processor already owns its complete `C_{ij}` block.

### 2.4. Step-by-step 2×2 example (be ready to reproduce)

Setup: `q = 2`, `n = 4`, block size `2 × 2`. Each processor `P(i,j)` initially holds `A_{ij}` and `B_{ij}`.

**Skew A** (shift row `i` left by `i`):
- Row 0: no shift (so `P(0,0)` keeps `A_{00}`, `P(0,1)` keeps `A_{01}`).
- Row 1: shift left by 1 (so `P(1,0)` now holds `A_{11}`, `P(1,1)` holds `A_{10}`).

**Skew B** (shift col `j` up by `j`):
- Col 0: no shift.
- Col 1: shift up by 1 (so `P(0,1)` now holds `B_{11}`, `P(1,1)` holds `B_{01}`).

**Step 1 — local multiplies:**

| Proc | A block | B block | C update |
|---|---|---|---|
| `P(0,0)` | `A_{00}` | `B_{00}` | `C_{00} += A_{00} B_{00}` |
| `P(0,1)` | `A_{01}` | `B_{11}` | `C_{01} += A_{01} B_{11}` |
| `P(1,0)` | `A_{11}` | `B_{10}` | `C_{10} += A_{11} B_{10}` |
| `P(1,1)` | `A_{10}` | `B_{01}` | `C_{11} += A_{10} B_{01}` |

**Step 2 — shift A left by 1, B up by 1, then multiply:**

| Proc | A block | B block | C update |
|---|---|---|---|
| `P(0,0)` | `A_{01}` | `B_{10}` | `C_{00} += A_{01} B_{10}` |
| `P(0,1)` | `A_{00}` | `B_{01}` | `C_{01} += A_{00} B_{01}` |
| `P(1,0)` | `A_{10}` | `B_{00}` | `C_{10} += A_{10} B_{00}` |
| `P(1,1)` | `A_{11}` | `B_{11}` | `C_{11} += A_{11} B_{11}` |

Verification at `P(0,0)`: `C_{00} = A_{00}B_{00} + A_{01}B_{10}` — exactly the block formula for the top-left block of `A·B`.

### 2.5. Analysis

Per-processor work, with bandwidth term `μ` (per element) and latency term `τ` (per message):

```math
\text{Compute: } O\!\left(\sqrt{p}\,\cdot\,\Big(\tfrac{n}{\sqrt{p}}\Big)^{3}\right) = O\!\left(\tfrac{n^3}{p}\right)
```

```math
\text{Alignment comm: } O\!\left(\tau\sqrt{p} + \mu\,\tfrac{n^2}{p}\sqrt{p}\right)
```

```math
\text{Compute+shift comm: } \sqrt{p} \text{ rounds of } O\!\left(\tau + \mu\,\tfrac{n^2}{p}\right) \text{ shifts}
```

```math
\boxed{T_{\text{Cannon}} = O\!\left(\tfrac{n^3}{p} + \tau\sqrt{p} + \mu\,\tfrac{n^2}{\sqrt{p}}\right)}
```

```math
\text{Memory per processor: } \tfrac{3n^2}{p} \;=\; O\!\left(\tfrac{n^2}{p}\right) \quad (\text{A block + B block + C block})
```

- More **messages** than broadcast-based schemes (one shift per step), but Cannon is **memory-optimal** and matches the **Hong–Kung lower bound** `Ω(n²/√p)` on communication volume for parallel GEMM.

### 2.6. Limitations

- Requires `p` to be a **perfect square** (`p = q²`).
- Requires `A, B` (and so `C`) to be **square**, and `n` divisible by `q`.
- Non-square shapes need padding or a different algorithm (SUMMA).

### 2.7. SUMMA — broadcast-based alternative

SUMMA = **Scalable Universal Matrix Multiply Algorithm** (van de Geijn & Watts 1997). Used in **ScaLAPACK, PLAPACK, Elemental, PBLAS**.

- Based on the **outer-product** view: `C = Σ_{k} A[:,k] · B[k,:]`, generalized to width-`b` panels: `C = Σ_ℓ A[:, ℓb:(ℓ+1)b] · B[ℓb:(ℓ+1)b, :]`.
- At each of `n/b` steps:
  1. Processor column owning the current A panel **broadcasts it along its processor row**.
  2. Processor row owning the current B panel **broadcasts it down its processor column**.
  3. Each processor performs a local `b`-wide multiply-add into its `C_{ij}` block.

**Cannon vs SUMMA comparison:**

| Property | Cannon | SUMMA |
|---|---|---|
| Grid shape | `q × q` (square only) | `p_r × p_c` (any) |
| Initial setup | Skew phase required | None |
| Communication | Cyclic shifts (A left, B up) | Row/column broadcasts |
| Steps | `q = √p` | `n/b` |
| Block width | Fixed `n/q` | Tunable `b` |
| Library usage | Teaching, torus HPC | ScaLAPACK, Elemental |
| Comm/compute overlap | Natural (shift + compute) | Possible with non-blocking |

Both compute the same partial products — only the **mechanism for routing them** differs.

---

## 3. Sparse Matrix-Vector multiply (SpMV)

### 3.1. Motivation and applications

> "Most of the coefficients in our matrices were zero — the nonzeros were sparse." — Harry Markowitz, 1990 Nobel Prize in Economics (modelling work from the 1950s).

- A **sparse matrix** has `nnz ≪ n²` (typically `>> 90%` zeros).
- Examples: web adjacency (PageRank), social graphs, FEM stiffness matrices, recommendation matrices, transportation networks, optimization constraints.
- Storing a sparse matrix in a dense `n × n` array wastes **memory** (`O(n²)` for `O(nnz) ~ O(n)` information) and **flops** (multiplying by zero).

### 3.2. Computational pattern

```math
y = A x, \qquad y_i = \sum_{j : A_{ij} \neq 0} A_{ij}\, x_j
```

Only nonzeros contribute. Per nonzero: **1 multiply + 1 add = 2 flops**.

### 3.3. Storage formats — the cheat sheet

Let `n` = matrix dimension, `nnz` = number of nonzeros, `d` = max nonzeros per row (ELL), `nd` = number of stored diagonals (DIA). Sizes are in matrix entries (values + indices).

| Format | Arrays | Size (entries) | Best for | Access pattern |
|---|---|---|---|---|
| **COO** (Coordinate) | `row[nnz]`, `col[nnz]`, `val[nnz]` | `≈ 3·nnz` | Building / modifying matrices; easy to insert | Row order **not** enforced — slow row scans |
| **CSR** (Compressed Sparse Row) | `row_ptr[n+1]`, `col_idx[nnz]`, `val[nnz]` | `≈ 2·nnz + n + 1` | **Default**; fast row traversal; SpMV `y = Ax` | Sequential read of nonzeros of one row; `x[col_idx[j]]` is **indirect** |
| **CSC** (Compressed Sparse Column) | `col_ptr[n+1]`, `row_idx[nnz]`, `val[nnz]` | `≈ 2·nnz + n + 1` | Column ops; fast `A^T x`; sparse-direct solvers | Symmetric to CSR (rows ↔ columns) |
| **ELL** (ELLpack) | `val[n × d]`, `col_idx[n × d]` (padded) | `≈ 2·n·d` | **SIMD / GPU**; coalesced access | Each row padded to length `d` — wasteful if `nnz` per row varies wildly |
| **DIA** (Diagonal) | `val[n × nd]`, `offset[nd]` | `≈ n·nd + nd` | **Stencils**, banded matrices | Iterate diagonal-by-diagonal; very regular |
| **HYB** (Hybrid) | ELL part + COO part (outliers) | ≈ ELL(n·d_typ) + COO(rest) | Mixed: most rows short, a few rows long | Best of both — ELL coalescence, COO flexibility |

Common refinements: **register-blocked CSR** (small dense sub-blocks), **cache-blocked CSR** (large sparse blocks), **symmetric** storage (only upper/lower triangle).

```math
\text{Size}_{\text{CSR}} = \underbrace{nnz}_{\text{val}} + \underbrace{nnz}_{\text{col\_idx}} + \underbrace{n+1}_{\text{row\_ptr}}
```

### 3.4. Serial SpMV in CSR

```
for each row i in 0..n-1:
    for k in row_ptr[i] .. row_ptr[i+1] - 1:
        y[i] += val[k] * x[col_idx[k]]
```

- **No reuse in A** — every nonzero is touched exactly once.
- **Maximum reuse in y** — `y[i]` stays in a register across the inner loop (write-once).
- **Reuse in x is structure-dependent** — `x[col_idx[k]]` is an **indirect load**; locality depends on how the nonzeros of row `i` cluster in column space. Diagonal-heavy matrices reuse `x` well; random sparsity does not.

### 3.5. Why SpMV is memory-bound — arithmetic intensity

Per nonzero loaded from CSR you need: `val` (8 B), `col_idx` (4 B), and one `x` element (8 B, if not cached). That's ~20 B per 2 flops worst-case, or ~12 B when `x` is reused.

```math
\text{Arithmetic intensity (SpMV)} \approx \frac{2 \text{ flops}}{12\text{–}20 \text{ bytes}} \approx 0.1\text{–}0.17 \text{ flop/byte}
```

Compare GEMM at `~n` flop/byte. SpMV lives on the **bandwidth-limited side of the roofline** — peak flops are unreachable; performance equals `BW × intensity`.

### 3.6. Parallel SpMV

**Row partitioning (most common):** assign blocks of rows to threads. Each thread writes its own `y[i]` (no reduction needed).

```
parallel_for (i = 0 .. m-1):
    for (j = row_ptr[i] .. row_ptr[i+1] - 1):
        y[i] += val[j] * x[col_idx[j]]
```

**Challenges:**
- **Load imbalance** — rows differ in `nnz`. A naive `n/p` split gives some threads much more work. Mitigations: **balanced partitioning by `nnz`**, dynamic scheduling, or 2D nonzero partitioning.
- **Indirect access `x[col_idx[j]]`** — pulls cold cache lines; reordering rows/columns (RCM, AMD, partitioning) improves locality.
- **`x` is read by all threads** — need to broadcast/replicate or use a 2D partition.
- **Column partitioning** would require an atomic reduction on `y[i]`. Usually avoided unless using **segmented scan** (a parallel-primitive approach that handles the segmented sum across the flattened nonzero stream).

**GPU SpMV:**
- **CSR + scalar (thread-per-row)**: simple, suffers when rows are short — underutilized warps and uncoalesced loads of `val`/`col_idx`.
- **CSR + vector (warp-per-row)**: a whole warp cooperates on one row, then a warp-level reduction. Good when most rows have ≥ 32 nonzeros.
- **ELL**: column-major padded storage gives **perfectly coalesced** loads — fastest when `nnz` per row is uniform.
- **DIA**: ideal for stencils — no indirection, contiguous diagonals.
- **HYB**: ELL for the dense head + COO for long-tail rows — most production GPU SpMV (e.g., cuSPARSE) uses variants of this.

---

## 4. Glossary

- **GEMM** — General Matrix-Matrix Multiply: `C ← α·A·B + β·C`. The BLAS Level-3 primitive.
- **2D block partition** — split a matrix into `q×q` tiles assigned to a `q×q` processor grid.
- **Torus mesh** — 2D processor grid with wrap-around connections in both dimensions; supports cyclic shifts without boundary cases.
- **Skew / alignment** — Cannon's Phase 1: shift row `i` of A left by `i`, column `j` of B up by `j`.
- **Cyclic / toroidal shift** — communication primitive sending each block to its neighbour, wrapping around at the edge.
- **Cannon's algorithm** — square-grid systolic GEMM: skew, then `√p` rounds of local multiply + 1-hop shift.
- **SUMMA** — broadcast-based parallel GEMM on arbitrary `p_r × p_c` grids; underlies ScaLAPACK.
- **Outer-product view** — `C = Σ_k A[:,k] · B[k,:]`; the algorithmic basis for SUMMA.
- **Hong–Kung lower bound** — `Ω(n²/√p)` words must be moved per processor in any parallel GEMM.
- **Arithmetic intensity** — flops per byte of memory traffic; the x-axis of the roofline model.
- **Sparse matrix** — matrix with `nnz ≪ n²` (typically > 90% zeros).
- **COO / CSR / CSC / ELL / DIA / HYB** — sparse storage formats; see table above.
- **`nnz`** — number of nonzeros in a sparse matrix.
- **SpMV** — Sparse Matrix-Vector multiply, `y = A·x`.
- **Indirect load** — `x[col_idx[j]]`: data address known only at runtime → cache-unfriendly.
- **Roofline model** — performance ceiling = `min(peak flops, BW × arithmetic intensity)`.

---

## 5. Likely exam targets

1. **Why is naive 1D partition bad for GEMM?** Show that each processor needs `O(n²)` data → total comm `O(n³)`, same order as compute. Contrast with 2D partition `O(n²/√p)` per proc.
2. **State Cannon's three phases** (skew → compute-and-shift → result) with the formal expression `A_{i,(i+j+s) mod q}`, `B_{(i+j+s) mod q, j}`. Explain *why* the skew makes inner indices match.
3. **Walk through the 2×2 Cannon worked example** — fill in which A and B block each processor holds at each step and accumulate `C_{00}`. The trick: row 1 of A shifts left by 1, col 1 of B shifts up by 1.
4. **Derive Cannon's complexity:** show how `√p` rounds × `O((n/√p)³)` local multiply gives `O(n³/p)`, and the comm sums to `O(τ√p + μ n²/√p)`. State the memory `O(n²/p)`.
5. **Why is Cannon memory-optimal?** One copy of each A and B element across all processors; matches the Hong–Kung lower bound on communication.
6. **Cannon vs SUMMA** — fill the comparison table from memory: grid shape, skew vs no skew, shifts vs broadcasts, where each is used.
7. **List Cannon's limitations** and explain how SUMMA fixes them.
8. **Write CSR and explain it** — three arrays (`row_ptr`, `col_idx`, `val`), sizes `n+1`, `nnz`, `nnz`. Show how to access row `i`.
9. **Compare storage formats** — fill the COO/CSR/CSC/ELL/DIA table, picking the best format for: a stencil (DIA), a build-from-stream (COO), a GPU SpMV on uniform-`nnz` rows (ELL).
10. **Explain why SpMV is memory-bound** — compute arithmetic intensity `≈ 0.1 flop/byte`, place it on the roofline, identify the bandwidth ceiling.

---

## 6. Pitfalls

- **Cannon needs `p = q²` and square `A, B`.** A common mistake is to claim Cannon works on any grid — it does not. Use SUMMA when shapes don't match.
- **The skew is not just for show.** Without the initial skew, the inner index `k` on each processor at step `s = 0` would be `j` and `i` respectively — those don't match, so the local product would not contribute to `C_{ij}`. The skew aligns them at `(i+j) mod q`.
- **More shifts ≠ slower than SUMMA.** Cannon sends `√p` small shifts; SUMMA sends `n/b` broadcasts. Latency `τ` term differs but the volume term `μ n²/√p` is the same and meets the Hong–Kung bound for both.
- **CSR is not symmetric in cost** — `y = Ax` is fast in CSR; `y = A^T x` is slow (needs CSC or atomics).
- **ELL wastes memory when row lengths vary.** A single very long row forces padding for *all* rows. HYB or hybrid CSR-ELL fixes this.
- **SpMV's bottleneck is not flops.** Doubling FPU width won't help — you need higher bandwidth, fewer bytes per nonzero (compressed indices, register blocking), or reuse-friendly reordering (RCM, blocking).
- **Indirect loads `x[col_idx[j]]` cannot be vectorized straightforwardly.** GPU/SIMD performance depends entirely on how clustered the column indices are; pure random sparsity is a worst case.
