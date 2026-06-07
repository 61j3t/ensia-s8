# Chapter 4 — Dynamic Programming

## Bird's eye view

- **Dynamic Programming (DP)** = a collection of algorithms that compute **optimal policies** given a **perfect model** of the environment as a (finite) **MDP**.
- DP requires the dynamics `p(s', r | s, a)` to be **known** — it is the **planning** end of the spectrum (Ch. 1), not learning.
- Core idea: use **Bellman equations** as iterative update rules — turn the recursive value equations into a fixed-point computation.
- Two fundamental subproblems: **policy evaluation** (predict `vπ`) and **policy control** (find `π*`). Evaluation is the necessary stepping stone for control.
- Building blocks: **Iterative Policy Evaluation** → **Policy Improvement Theorem** → **Policy Iteration** → **Value Iteration**, all unified under **Generalized Policy Iteration (GPI)**.
- **Bootstrapping**: each update reuses the *current estimate* of `V(s')` to update `V(s)` — DP doesn't wait for a final return.
- Two **flavours of sweep**: synchronous (full sweep of `S` per iteration) vs **asynchronous DP** (any order, selective).
- **Policy Iteration** converges in a finite number of iterations on a finite MDP (only finitely many deterministic policies); **Value Iteration** truncates evaluation to a single sweep then takes the max.
- GPI is the *master pattern* of RL — almost every later method (MC, TD, Q-learning, actor-critic) is an instance.

---

## 1. Introduction — Dynamic Programming

### 1.1. What is DP?

- The term **dynamic programming (DP)** refers to a **collection of algorithms** that can be used to compute **optimal policies** given a **perfect model** of the environment as a Markov decision process (MDP).
- We usually assume the environment is a **finite MDP**: state set `𝒮`, action set `𝒜`, reward set `ℛ` are all finite.
- Dynamics are given by `p(s', r | s, a)` for all `s ∈ 𝒮`, `a ∈ 𝒜(s)`, `r ∈ ℛ`, `s' ∈ 𝒮⁺` (where `𝒮⁺ = 𝒮 ∪ {terminal}` if the problem is episodic).
- **Key idea of DP (and of RL more generally)**: use **value functions** to organize and structure the **search for good policies**.

### 1.2. Evaluation vs Control

- DP algorithms use the **Bellman equations** to define **iterative algorithms** for both **policy evaluation** and **control**.

| Task | Question | Output |
|---|---|---|
| **Policy evaluation (prediction)** | How good is policy `π`? | `vπ(s)` for all `s` |
| **Policy control** | Which policy obtains the most reward? | `π*` that maximises `vπ` |

- **Control is the ultimate goal of RL**, but **policy evaluation is usually a necessary first step**.
- *Rationale*: "It's hard to improve our policy if we don't have a way to assess how good it is."

---

## 2. Iterative Policy Evaluation

### 2.1. The Bellman equation, used as an update rule

Recall the Bellman equation for `vπ`:

```math
v_\pi(s) = \sum_a \pi(a \mid s) \sum_{s'} \sum_r p(s', r \mid s, a)\,[\,r + \gamma\, v_\pi(s')\,]
```

DP turns this into an **iterative assignment**:

```math
v_{k+1}(s) \;\leftarrow\; \sum_a \pi(a \mid s) \sum_{s'} \sum_r p(s', r \mid s, a)\,[\,r + \gamma\, v_k(s')\,]
```

- `v_k` is the current estimate; `v_{k+1}` the updated one.
- The update is a **full backup**: it sums over **all** actions, next states, and rewards weighted by their probabilities.
- The sequence `{v_k}` converges to `vπ` as `k → ∞` (guaranteed under `γ < 1`, or under proper termination for episodic tasks).
- This is **bootstrapping** — each update uses current estimates `v_k(s')` to update `v_k(s)`.

### 2.2. Algorithm — Iterative Policy Evaluation

```
Iterative Policy Evaluation, for estimating V ≈ vπ
Input π, the policy to be evaluated
V ← 0,  V' ← 0
Loop:
    Δ ← 0
    Loop for each s ∈ 𝒮:
        V'(s) ← Σ_a π(a|s) Σ_{s',r} p(s', r | s, a) [r + γ V(s')]
        Δ ← max(Δ, |V'(s) − V(s)|)
    V ← V'
until Δ < θ            (θ a small positive number)
Output V ≈ vπ
```

Notes:
- Uses **two arrays** `V` and `V'` — this is the **synchronous** version (all states updated from the *old* `V` before swapping). An in-place version (single array) also works and often converges *faster*.
- `Δ` tracks the **maximum change** across all states in this sweep. The loop terminates when `Δ < θ`.
- One iteration = one **sweep** of the full state set `𝒮`.

### 2.3. Worked example — `4 × 4` Gridworld

Setup (slides 10-21):
- A `4 × 4` grid with two **terminal states** at the top-left and bottom-right corners (shaded grey).
- Reward `R = −1` on every transition (until termination); `γ = 1`.
- Policy `π` = uniform random: each of `{N, E, S, W}` has probability `0.25`.
- Actions that would leave the grid bounce back (the agent stays in place).

Update applied at each cell (using the equiprobable random policy):

```math
V'(s) = 0.25 \times (-1 + V(N)) + 0.25 \times (-1 + V(E)) + 0.25 \times (-1 + V(S)) + 0.25 \times (-1 + V(W))
```

| Iteration | Notable values | Comment |
|---|---|---|
| `k = 0` | All `0` | Initialised to zero. |
| `k = 1` | Every non-terminal cell becomes `−1` | One step costs `−1`. |
| `k = 2` | Corners-adjacent: `−1.7`; far cells: `−2` | Information propagates one step from terminals. |
| `k = 3` | Mid-corners: `−2.4` … `−3.0` | Value spreads further. |
| `k = 10` (e.g.) | Symmetric pattern, larger magnitudes | Converging towards `vπ`. |
| Until `Δ < 0.001` | `0, −14, −20, −22; −14, −18, −20, −20; …` | Final estimate `V ≈ vπ`. |

- **The terminal states stay at 0** (absorbing — no further reward).
- Convergence point: when one full sweep changes every cell by less than `θ = 0.001`.

---

## 3. Policy Improvement

### 3.1. The greedy step

The Bellman **optimality** equation defines the optimal policy as:

```math
\pi_*(s) = \arg\max_a \sum_{s'} \sum_r p(s', r \mid s, a)\,[\,r + \gamma\, v_*(s')\,]
```

That `arg max` is the **greedy action**. Apply the same construction to `vπ` (the value we just computed) to get a new policy `π'`:

```math
\pi'(s) = \arg\max_a \sum_{s'} \sum_r p(s', r \mid s, a)\,[\,r + \gamma\, v_\pi(s')\,] \quad \text{for all } s \in \mathcal{S}
```

If `vπ` already obeys the Bellman *optimality* equation, then `π` is already optimal.

### 3.2. Policy Improvement Theorem

For any pair of deterministic policies `π` and `π'`:

```math
q_\pi(s, \pi'(s)) \;\geq\; q_\pi(s, \pi(s)) \quad \text{for all } s \in \mathcal{S} \;\;\Rightarrow\;\; \pi' \geq \pi
```

with **strict improvement** somewhere if the inequality is strict at any state:

```math
q_\pi(s, \pi'(s)) \;>\; q_\pi(s, \pi(s)) \quad \text{for at least one } s \in \mathcal{S} \;\;\Rightarrow\;\; \pi' > \pi
```

In words: if acting `π'` for one step then following `π` is at least as good as `π` everywhere, then `π' ≥ π` *as a whole policy*; if it is strictly better anywhere, `π' > π`.

**Construction**: choosing `π'(s) = arg max_a q_π(s, a)` is **greedy w.r.t. `vπ`** and satisfies the hypothesis by definition of `arg max`, so the new policy is **always at least as good** as the old one.

### 3.3. Worked example — gridworld policy improvement

From the converged `vπ` of section 2.3:

| Top row | Middle rows | Bottom row |
|---|---|---|
| `0, −14, −20, −22` | `−14, −18, −20, −20` and `−20, −20, −18, −14` | `−22, −20, −14, 0` |

Apply `π'(s) = arg max_a Σ p(s', r | s, a)[r + γ vπ(s')]` cell by cell — the new arrows point **towards the nearest terminal** (the cells with higher value). The slides show: top-row arrows leftward, bottom-row arrows rightward, etc.

> **`π' > π`** — the greedy policy is **strictly better** than the uniform random one (since `π` was not yet optimal).

---

## 4. Policy Iteration

### 4.1. The alternating sequence

Once `π` has been improved using `vπ` to yield `π'`, we can compute `v_{π'}` and improve again. This gives a sequence of **monotonically improving** policies and value functions:

```math
\pi_0 \xrightarrow{E} v_{\pi_0} \xrightarrow{I} \pi_1 \xrightarrow{E} v_{\pi_1} \xrightarrow{I} \pi_2 \xrightarrow{E} v_{\pi_2} \xrightarrow{I} \pi_3 \xrightarrow{E} \cdots \xrightarrow{I} \pi_* \xrightarrow{E} v_* \xrightarrow{I} \pi_*
```

where `→E` is policy **Evaluation** and `→I` is policy **Improvement**.

- Each step is a **strict improvement** (unless already optimal).
- A **finite MDP** has only a **finite number of deterministic policies**, so this process **must converge** to the optimal policy and optimal value function in a **finite number of iterations**.
- This algorithm is called **Policy Iteration**.

### 4.2. Geometric intuition (the two lines)

The "dance" between value and policy can be drawn as zig-zag steps converging to the corner `(v*, π*)`:

- One line: `v = vπ` (evaluation hits this line when V is fully consistent with the current policy).
- Other line: `π = greedy(v)` (improvement hits this line when the policy is greedy w.r.t. current V).
- Each evaluation projects vertically onto the first line; each improvement projects onto the second.

### 4.3. Algorithm — Policy Iteration

```
Policy Iteration (using iterative policy evaluation) for estimating π ≈ π*

1. Initialization
   V(s) ∈ ℝ  and  π(s) ∈ 𝒜(s)  arbitrarily for all s ∈ 𝒮

2. Policy Evaluation
   Loop:
       Δ ← 0
       Loop for each s ∈ 𝒮:
           v ← V(s)
           V(s) ← Σ_{s', r} p(s', r | s, π(s)) [r + γ V(s')]
           Δ ← max(Δ, |v − V(s)|)
   until Δ < θ           (θ a small positive number)

3. Policy Improvement
   policy-stable ← true
   For each s ∈ 𝒮:
       old-action ← π(s)
       π(s) ← arg max_a Σ_{s', r} p(s', r | s, a) [r + γ V(s')]
       If old-action ≠ π(s), then policy-stable ← false
   If policy-stable, then stop and return V ≈ v* and π ≈ π*; else go to 2
```

Notes:
- Evaluation here uses the *current* deterministic `π(s)` (the inner sum has no `Σ_a π(a|s)` — just the chosen action).
- **Termination flag**: `policy-stable` — if no action changed during improvement, we're at the fixed point of GPI.
- In-place V updates are used (single array `V`).

### 4.4. Worked example — gridworld with penalties

A more elaborate `4 × 4` grid (slides 30-43):
- Two terminals (top-left, bottom-right).
- Some cells are **blue** with reward `R = −10`; the rest have `R = −1`. `γ = 1`.

Process traced on the slides:
- **Initial `π`** = arbitrary uniform; **Initial `V`** = all 0 with the arrows shown in every direction.
- **Evaluation step 1** converges to e.g. `0, −137.5, −209.6, −239.4` on the top row — large magnitudes from the `−10` penalty cells and the random policy bouncing around them.
- **Improvement step 1** turns most arrows to point upward / leftward (toward the top-left terminal).
- After several rounds of (eval → improve → eval → improve …), values collapse to small magnitudes (e.g. top row `0, −1, −4, −5`) and arrows form a clean directed path to a terminal.
- The algorithm halts when the **improvement step changes no actions** (`policy-stable = true`).

---

## 5. Generalized Policy Iteration (GPI)

### 5.1. The general idea

- **Generalized Policy Iteration (GPI)** refers to the **general idea** of letting the policy-evaluation and policy-improvement processes **interact**, **independent** of the granularity and other details of the two processes.
- **Almost all reinforcement-learning methods are well described as GPI** — they all maintain identifiable policy and value functions, with the policy being improved with respect to the value function and the value function driven towards the value function for the policy.

### 5.2. The fixed point

Both processes have a clear fixed-point condition:

| Process | Stabilises when … |
|---|---|
| **Evaluation** | `V` is **consistent with the current policy** (`V = vπ`) |
| **Improvement** | `π` is **greedy with respect to the current value function** (`π = greedy(V)`) |

Both conditions together ⟺ the Bellman optimality equation holds ⟺ `(π, V) = (π*, v*)`.

### 5.3. Policy Iteration vs GPI

- **Policy iteration** is the *strict* version: run each step **all the way to completion** before switching (eval to convergence, then full greedification, then eval again, …).
- **GPI** loosens this: each evaluation brings `V` *a little closer* to `vπ`, not all the way; each improvement makes `π` *a little more greedy*, not totally greedy. Intuitively the process still makes progress towards `(π*, v*)`.
- This relaxation is what makes Value Iteration (and later, MC and TD) possible.

### 5.4. Value Iteration

**Value Iteration** is the simplest GPI algorithm: collapse policy evaluation into a **single sweep** (one update per state) and combine it with the improvement step via the **`max` operator**.

```
Value Iteration, for estimating π ≈ π*

Algorithm parameter: a small threshold θ > 0 determining accuracy of estimation
Initialize V(s), for all s ∈ 𝒮⁺, arbitrarily except that V(terminal) = 0

Loop:
    Δ ← 0
    Loop for each s ∈ 𝒮:
        v ← V(s)
        V(s) ← max_a Σ_{s', r} p(s', r | s, a) [r + γ V(s')]
        Δ ← max(Δ, |v − V(s)|)
until Δ < θ

Output a deterministic policy π ≈ π*, such that
    π(s) = arg max_a Σ_{s', r} p(s', r | s, a) [r + γ V(s')]
```

Key points:
- Update rule = Bellman **optimality** backup applied iteratively:

```math
v_{k+1}(s) \;\leftarrow\; \max_a \sum_{s'} \sum_r p(s', r \mid s, a)\,[\,r + \gamma\, v_k(s')\,]
```

- **No explicit policy** is stored during the loop — it falls out at the end as the `arg max`.
- Equivalent to "one sweep of policy evaluation + one greedification" per iteration.
- Same convergence guarantee as policy iteration (finite MDP, suitable `γ` or termination).

### 5.5. Asynchronous Dynamic Programming

- A major **drawback** of the DP methods above is that they require **sweeps of the entire state set** of the MDP.
- If `𝒮` is very large, even a **single sweep** can be **prohibitively expensive** — e.g., **backgammon has over `10²⁰` states**.
- **Asynchronous DP** algorithms update the values of states **in any order**, using whatever values of other states happen to be available. They do **not perform systematic sweeps**.
- They may update a given state **many times** before another state is updated even once.
- **Convergence guarantee**: to converge, asynchronous algorithms must **continue to update the values of all states** (cannot ignore any state forever).
- **Advantage**: can **propagate value information quickly** through **selective updates** — focus on states that matter (e.g., near the agent's current position, or with high TD-error). Sometimes more efficient than systematic sweeps.

---

## 6. Putting it together — DP method comparison

| Algorithm | Update rule | Per-iteration cost | Stops when |
|---|---|---|---|
| **Iterative Policy Evaluation** | Bellman expectation (with current `π`) | `O(|𝒮|² · |𝒜|)` per sweep | `Δ < θ` |
| **Policy Iteration** | Eval (to `θ`) + greedification | many sweeps per outer step | `policy-stable = true` |
| **Value Iteration** | Bellman optimality (`max_a`) | one sweep per outer iteration | `Δ < θ` |
| **Asynchronous DP** | Any of the above, but state-by-state | depends on order | application-specific |

Where they sit in the GPI picture:

| Method | Evaluation done to … | Improvement done to … |
|---|---|---|
| Policy Iteration | full convergence | full greedification each round |
| Value Iteration | one sweep only | implicit `max` (fully greedy at each backup) |
| GPI (general) | partial | partial |

---

## Key terms (glossary)

- **Dynamic Programming (DP)** — family of algorithms that compute optimal policies given a perfect MDP model.
- **Perfect model** — full knowledge of `p(s', r | s, a)`.
- **Bellman equation (expectation form)** — recursive definition of `vπ` used as an update rule in evaluation.
- **Bellman optimality equation** — recursive definition of `v*` used as an update rule in value iteration.
- **Bootstrapping** — updating an estimate using other current estimates (not a final return).
- **Sweep** — one pass through every state in `𝒮`.
- **Synchronous update** — uses two arrays so all states are updated from the *old* `V` (then swap).
- **In-place / asynchronous update** — single array, uses freshly updated neighbours immediately.
- **Iterative Policy Evaluation** — the DP algorithm to compute `vπ` for a fixed `π`.
- **Policy Improvement** — replacing `π` by the greedy policy w.r.t. `vπ`.
- **Policy Improvement Theorem** — if `q_π(s, π'(s)) ≥ q_π(s, π(s))` for all `s` then `π' ≥ π`.
- **Policy Iteration** — alternate evaluation and improvement until policy is stable.
- **Value Iteration** — one Bellman-optimality backup per state per sweep.
- **Generalized Policy Iteration (GPI)** — the umbrella idea: interleaving any partial evaluation and any partial improvement.
- **Asynchronous DP** — update states in any order, no full sweeps required.
- **Policy-stable** — termination flag for policy iteration; `true` iff no greedy action changed.

---

## Exam targets (likely written-exam questions)

1. **Define dynamic programming** in the RL context and state its two prerequisites (finite MDP + perfect model).
2. **Write the iterative policy evaluation update** for `v_{k+1}(s)` from the Bellman expectation equation; explain bootstrapping.
3. **Algorithm**: write the full pseudocode of *Iterative Policy Evaluation*, including the role of `Δ`, `θ`, and the two-array vs in-place choice.
4. **Gridworld**: given `R = −1`, `γ = 1`, and a uniform random policy on a `4×4` gridworld with two terminals, **compute the first 2 iterations** of policy evaluation by hand.
5. **State and prove (sketch) the Policy Improvement Theorem**; explain why the greedy step is always an improvement (unless already optimal).
6. **Write the Policy Iteration algorithm** with all three blocks (init, evaluation, improvement) and explain why it converges in a finite number of iterations on a finite MDP.
7. **Write the Value Iteration algorithm** and contrast it with Policy Iteration — what is collapsed and why it still converges.
8. **Define Generalized Policy Iteration (GPI)** and explain its fixed-point conditions; classify policy iteration and value iteration as special cases of GPI.
9. **Asynchronous DP**: motivation (large state spaces like backgammon ~ `10²⁰`), how it differs from synchronous DP, and the convergence requirement.

### Pitfalls

- **DP requires a known model** `p(s', r | s, a)`. If you have to *learn* the dynamics by sampling, you are doing MC / TD, not DP.
- **Iterative Policy Evaluation evaluates one fixed policy** — there is no `max` in the inner sum, just `Σ_a π(a|s)`. The `max` only appears in *Value Iteration* and the *Improvement* step.
- **Don't confuse the two Bellman equations**: expectation form (with `Σ_a π(a|s)`) is for evaluation; optimality form (with `max_a`) is for value iteration / optimal policies.
- **Terminal states have `V(terminal) = 0`** by definition — keep them fixed during updates.
- **Policy iteration ≠ value iteration**: PI runs evaluation to convergence; VI does **one sweep** then takes the max. Mixing them up is a classic exam mistake.
- **`policy-stable = false` does NOT mean policy iteration failed** — it means we need another round of evaluation + improvement. Only `policy-stable = true` after a full improvement sweep is the termination condition.
- **Bootstrapping ≠ Monte Carlo** — DP uses the *expected* one-step lookahead (over all next states/rewards) using the model, not sampled trajectories.
- **Convergence of PI is in a finite number of *iterations*** (because there are finitely many deterministic policies) — but each iteration involves an evaluation that itself converges only **in the limit**.
- **Asynchronous DP must still update every state** infinitely often — "any order" doesn't mean "skip some states forever".
- **γ = 1 is fine here** because the gridworld is episodic (terminal states absorb) — in non-episodic settings always discount.
