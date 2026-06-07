# Chapter 3 — Finite Markov Decision Processes

## Bird's eye view

- An **MDP** is the formal model behind RL: states `𝓢`, actions `𝓐`, rewards `𝓡`, and a one-step dynamics function `p(s', r | s, a)` that fully specifies the environment.
- The MDP is **Markov**: the present state contains all the information necessary to predict the future — past states/actions can be discarded.
- The agent's **goal** is to maximize the **expected return** `𝔼[Gₜ]`, not just immediate reward. Returns can be **stochastic** because dynamics are stochastic.
- Two task classes: **episodic** (each episode ends in a terminal state, finite `T`) vs **continuing** (no terminal state, infinite horizon — needs discounting to stay finite).
- The **discount factor** `γ ∈ [0, 1)` makes the infinite sum finite and trades short-sighted (`γ = 0`) vs far-sighted (`γ → 1`) behaviour. Returns obey the recursion `Gₜ = Rₜ₊₁ + γ Gₜ₊₁`.
- A **policy** `π(a | s)` maps states to action probabilities; deterministic policies are a special case.
- Value functions `vπ(s)` and `qπ(s, a)` are expected returns under `π`; they satisfy the **Bellman equations** — recursive linear relations between successor states/state-action pairs.
- A policy `π₁ ≥ π₂` iff `vπ₁(s) ≥ vπ₂(s) ∀s`. For any MDP there exists an **optimal policy** `π*` whose values are `v*` and `q*`.
- The **Bellman optimality equations** for `v*` and `q*` replace the policy expectation with a `max` — they are **non-linear** and have no closed form, but `π*` is greedy w.r.t. `v*` (or `q*`).
- Direct solution by linear-system inversion only works for **small** MDPs; large MDPs (e.g., chess: `~10⁴⁵` states) need iterative methods — Value Iteration, Policy Iteration, Q-learning, Sarsa.

---

## 1. The MDP Framework

### 1.1. The agent–environment interaction

At each time step `t = 0, 1, 2, …`:
- The agent observes state `Sₜ ∈ 𝓢` and reward `Rₜ ∈ 𝓡 ⊂ ℝ`.
- The agent picks an action `Aₜ ∈ 𝓐(Sₜ)`.
- The environment emits the next reward `Rₜ₊₁` and next state `Sₜ₊₁`.

The resulting trajectory:

```math
S_0, A_0, R_1, S_1, A_1, R_2, S_2, A_2, R_3, S_3, \dots
```

### 1.2. The dynamics function `p(s', r | s, a)`

The one-step environment dynamics is fully specified by:

```math
p(s', r \mid s, a) \;\doteq\; \Pr\{S_{t+1} = s', R_{t+1} = r \mid S_t = s, A_t = a\}
```

Signature: `p : 𝓢 × 𝓡 × 𝓢 × 𝓐 → [0, 1]`. It is a **proper probability distribution**, i.e.

```math
\sum_{s' \in \mathcal{S}} \sum_{r \in \mathcal{R}} p(s', r \mid s, a) = 1, \quad \forall s \in \mathcal{S}, \; a \in \mathcal{A}(s)
```

> **Markov property**: the present state `Sₜ` contains all the information necessary to predict the future. The dynamics depends only on `(s, a)`, **not** on the entire history.

### 1.3. Finite MDPs

A **finite** MDP has finite sets `𝓢`, `𝓐`, `𝓡`. This chapter restricts to that case.

---

## 2. Example: Recycling Robot

A canonical small MDP used to illustrate the formalism.

- **States**: `𝓢 = {high, low}` (battery level).
- **Actions**: `𝓐(high) = {search, wait}`, `𝓐(low) = {search, wait, recharge}` (action set is state-dependent).
- **Parameters**: `α` = prob. of staying `high` while searching from `high`; `β` = prob. of staying `low` while searching from `low`; `r_search`, `r_wait` are the per-step rewards.

| `s` | `a` | `s'` | `p(s' \| s, a)` | `r(s, a, s')` |
|---|---|---|---|---|
| high | search | high | `α` | `r_search` |
| high | search | low | `1 − α` | `r_search` |
| low | search | high | `1 − β` | `−3` (rescued, depleted) |
| low | search | low | `β` | `r_search` |
| high | wait | high | `1` | `r_wait` |
| high | wait | low | `0` | — |
| low | wait | high | `0` | — |
| low | wait | low | `1` | `r_wait` |
| low | recharge | high | `1` | `0` |
| low | recharge | low | `0` | — |

The same MDP can be drawn as a transition graph with nodes `{high, low}` and edges labelled `(prob, reward)`.

### 2.1. Generality of the formalism

The MDP framework is deliberately abstract — it formalizes a huge range of sequential decision problems:

| Element | Low-level | High-level |
|---|---|---|
| **States** | pixel values of a video frame | object descriptions, symbolic features |
| **Actions** | motor voltages, wheel speeds | "go to the charging station" |
| **Time steps** | one millisecond | one month |

### 2.2. Example: Robot arm pick-and-place

- **State**: joint angles & velocities.
- **Action**: voltage applied to each motor.
- **Reward**: `+100` per object successfully placed; `−1` per unit of energy consumed.

---

## 3. The Goal: Return `Gₜ`

### 3.1. Definition

The **return** at time `t` is the (possibly discounted) sum of future rewards:

```math
G_t \;\doteq\; R_{t+1} + R_{t+2} + R_{t+3} + \cdots
```

`Gₜ` is a **random variable**: many trajectories from the same state are possible because dynamics are stochastic. The agent maximizes the **expected return**:

```math
\mathbb{E}[G_t] = \mathbb{E}[R_{t+1} + R_{t+2} + \cdots + R_T]
```

For this to be well-defined, the sum of rewards must be **finite**.

### 3.2. Episodic vs continuing tasks

| | **Episodic** | **Continuing** |
|---|---|---|
| Trajectory | Breaks into independent episodes | Goes on forever |
| Terminal state | Yes (absorbing, value `0`) | No |
| Return | `Gₜ = Rₜ₊₁ + Rₜ₊₂ + … + R_T` (finite `T`) | `Gₜ = Rₜ₊₁ + Rₜ₊₂ + … = ∞` (without help) |
| Fix | None needed | **Discounting** |

> An episode ends in a **terminal state**. Episodes are independent — one task = many episodes.

### 3.3. Discounting

To keep `Gₜ` finite for continuing tasks, discount future rewards by `γ ∈ [0, 1)`:

```math
G_t \;\doteq\; R_{t+1} + \gamma R_{t+2} + \gamma^2 R_{t+3} + \cdots \;=\; \sum_{k=0}^{\infty} \gamma^k R_{t+k+1}
```

`γ` is the **discount rate**: a reward `k` steps in the future is worth `γ^{k−1}` (relative to receiving it now, in the indexing used in the slides) — i.e. `γᵏ` if you index from `Rₜ₊ₖ₊₁`.

### 3.4. Effect of `γ` on agent behaviour

| `γ` | Behaviour | Name |
|---|---|---|
| `γ = 0` | maximize only `Rₜ₊₁` | **short-sighted agent** |
| `γ < 1` | infinite sum is finite as long as `{Rₖ}` is bounded | well-defined |
| `γ → 1` | future rewards weighted almost as much as present | **far-sighted agent** |

### 3.5. Recursive nature of returns

Splitting off the first term:

```math
G_t = R_{t+1} + \gamma (R_{t+2} + \gamma R_{t+3} + \gamma^2 R_{t+4} + \cdots) = R_{t+1} + \gamma G_{t+1}
```

This **recursion** is the backbone of every Bellman equation later in the chapter.

> Although `Gₜ` is an infinite sum, it is finite when rewards are bounded and `γ < 1`. **Example**: constant reward `+1` forever ⇒
> ```math
> G_t = \sum_{k=0}^{\infty} \gamma^k = \frac{1}{1 - \gamma}
> ```

### 3.6. Pole-balancing as both episodic and continuing

Inverted pendulum: move a cart along a track to keep a hinged pole upright.
- As an **episodic** task: each balance attempt = one episode; reward `+1` per step without failure ⇒ return = number of steps before failure.
- As a **continuing** task: use discounting and let the trajectory go on indefinitely.

---

## 4. Policies

A **policy** is a mapping from states to a distribution over actions.

### 4.1. Deterministic policy

`π : 𝓢 → 𝓐`. Equivalent to a table mapping each state to a single action.

| State | Action |
|---|---|
| `s₀` | `a₁` |
| `s₁` | `a₀` |
| `s₂` | `a₀` |

In the gridworld, a deterministic policy is drawn as a single arrow in each cell.

### 4.2. Stochastic policy

`π(a | s) = P[Aₜ = a | Sₜ = s]`, with

```math
\sum_{a \in \mathcal{A}(s)} \pi(a \mid s) = 1, \qquad 0 \le \pi(a \mid s) \le 1
```

In a gridworld cell, a stochastic policy is drawn as arrows of different lengths / with percentages (e.g., `50% up, 50% right`).

> Deterministic = special case of stochastic with a single action getting probability `1`.

---

## 5. Value Functions

The **value** of a state (or state–action pair) is the expected return obtainable from it under a fixed policy `π`.

### 5.1. State-value function `vπ`

```math
v_\pi(s) \;\doteq\; \mathbb{E}_\pi[G_t \mid S_t = s] \;=\; \mathbb{E}_\pi\!\left[\sum_{k=0}^{\infty} \gamma^k R_{t+k+1} \;\Big|\; S_t = s\right], \qquad \forall s \in \mathcal{S}
```

- `vπ(s)` "predicts rewards into the future" — it averages returns over the (possibly infinite) tree of trajectories rooted at `s`.
- **The value of the terminal state, if any, is always `0`.**

### 5.2. Action-value function `qπ`

```math
q_\pi(s, a) \;\doteq\; \mathbb{E}_\pi[G_t \mid S_t = s, A_t = a] \;=\; \mathbb{E}_\pi\!\left[\sum_{k=0}^{\infty} \gamma^k R_{t+k+1} \;\Big|\; S_t = s, A_t = a\right]
```

- Conditioned on *both* the current state and the next action; thereafter the agent follows `π`.

| | `vπ(s)` | `qπ(s, a)` |
|---|---|---|
| Conditioned on | state | state **and** action |
| Result | scalar per state | scalar per (state, action) |
| Use | "How good is this state?" | "How good is this action here?" |

---

## 6. Bellman Equations

The defining feature of value functions: they satisfy **recursive** consistency relations between a state and its successors.

### 6.1. Bellman equation for `vπ`

Starting from the definition and using `Gₜ = Rₜ₊₁ + γ Gₜ₊₁`:

```math
\begin{aligned}
v_\pi(s) &\doteq \mathbb{E}_\pi[G_t \mid S_t = s] \\
&= \mathbb{E}_\pi[R_{t+1} + \gamma G_{t+1} \mid S_t = s] \\
&= \sum_{a} \pi(a \mid s) \sum_{s'} \sum_{r} p(s', r \mid s, a)\, \bigl[r + \gamma\, \mathbb{E}_\pi[G_{t+1} \mid S_{t+1} = s']\bigr] \\
&= \sum_{a} \pi(a \mid s) \sum_{s', r} p(s', r \mid s, a)\, \bigl[r + \gamma\, v_\pi(s')\bigr], \qquad \forall s \in \mathcal{S}
\end{aligned}
```

Three nested averages: over actions (under `π`), over next-state/reward pairs (under `p`), then a one-step lookahead `r + γ vπ(s')`.

### 6.2. Bellman equation for `qπ`

Same derivation, conditioned also on `Aₜ = a`:

```math
\begin{aligned}
q_\pi(s, a) &\doteq \mathbb{E}_\pi[G_t \mid S_t = s, A_t = a] \\
&= \sum_{s', r} p(s', r \mid s, a)\,\bigl[r + \gamma\, \mathbb{E}_\pi[G_{t+1} \mid S_{t+1} = s']\bigr] \\
&= \sum_{s', r} p(s', r \mid s, a)\left[ r + \gamma \sum_{a'} \pi(a' \mid s')\, q_\pi(s', a') \right]
\end{aligned}
```

It relates a state-action pair to its **possible successor state-action pairs** under `π`.

> Bellman equations are **linear** in the value function (for fixed `π`).

---

## 7. Worked Example: 2×2 Gridworld

Setup (from the slides):
- States `{A, B, C, D}` arranged as a 2×2 grid; agent can move N/S/E/W; hitting a wall bounces back to the same cell.
- Reward `+5` for every action that **leaves state B**; otherwise `0`.
- Discount `γ = 0.7`. Policy `π` = uniform random (`25%` each direction).

### 7.1. Setting up the system

The Bellman equation `vπ(s) = Σ_a π(a|s) Σ_{s', r} p(s', r | s, a)[r + γ vπ(s')]` instantiates to four linear equations:

```math
\begin{cases}
v_\pi(A) = \tfrac{1}{4}\bigl(5 + 0.7\, v_\pi(B)\bigr) + \tfrac{1}{4}\, 0.7\, v_\pi(C) + \tfrac{1}{2}\, 0.7\, v_\pi(A) \\[2pt]
v_\pi(B) = \tfrac{1}{2}\bigl(5 + 0.7\, v_\pi(B)\bigr) + \tfrac{1}{4}\, 0.7\, v_\pi(A) + \tfrac{1}{4}\, 0.7\, v_\pi(D) \\[2pt]
v_\pi(C) = \tfrac{1}{4}\, 0.7\, v_\pi(A) + \tfrac{1}{4}\, 0.7\, v_\pi(D) + \tfrac{1}{2}\, 0.7\, v_\pi(C) \\[2pt]
v_\pi(D) = \tfrac{1}{4}\bigl(5 + 0.7\, v_\pi(B)\bigr) + \tfrac{1}{4}\, 0.7\, v_\pi(C) + \tfrac{1}{2}\, 0.7\, v_\pi(D)
\end{cases}
```

> The `1/2` factors come from the two walls in `A`, `C`, `D` (and the `+5`-bouncing edge in `B`) which produce a self-loop with combined probability `0.5`.

### 7.2. Solution

Solving the 4×4 linear system (by hand or with a solver):

| State | `vπ(s)` |
|---|---|
| `A` | `4.2` |
| `B` | `6.1` |
| `C` | `2.2` |
| `D` | `4.2` |

### 7.3. Why this only works for small MDPs

| Problem size | States | Equations | Tractable? |
|---|---|---|---|
| 2×2 Gridworld | 4 | 4 linear | ✓ direct solve |
| Chess | ~`10⁴⁵` | ~`10⁴⁵` linear | ✗ infeasible |

Bellman equations **factor into** the solutions for large MDPs (next chapters: DP, MC, TD), but **direct linear-system inversion** is only feasible for small MDPs.

---

## 8. Optimal Policies and Optimal Value Functions

### 8.1. Policy ordering

A policy `π₁` is **better than or equal to** `π₂` iff

```math
\pi_1 \ge \pi_2 \quad \Longleftrightarrow \quad v_{\pi_1}(s) \ge v_{\pi_2}(s) \quad \forall s \in \mathcal{S}
```

This is a **partial order** on policies (two policies need not be comparable in general).

### 8.2. Existence theorem

> **Theorem.** For any finite MDP:
> 1. There exists an **optimal policy** `π*` such that `π* ≥ π` for every policy `π`.
> 2. All optimal policies share the same **optimal state-value function** `vπ*(s) = v*(s)`.
> 3. All optimal policies share the same **optimal action-value function** `qπ*(s, a) = q*(s, a)`.

### 8.3. Definitions of `v*` and `q*`

```math
v_*(s) \;\doteq\; \max_{\pi}\, v_\pi(s), \qquad \forall s \in \mathcal{S}
```

```math
q_*(s, a) \;\doteq\; \max_{\pi}\, q_\pi(s, a), \qquad \forall s \in \mathcal{S},\; a \in \mathcal{A}
```

### 8.4. Discount factor and optimal policy (example)

Two-action MDP from state `X`:
- `π₁(X) = A₁` → reward `+1` then back to `X`, alternating.
- `π₂(X) = A₂` → reward `0` then `+2`, alternating.

| `γ` | `vπ₁(X)` | `vπ₂(X)` | Winner |
|---|---|---|---|
| `0` | `1` | `0` | `π₁` |
| `0.9` | `1/(1 − 0.9²) ≈ 5.3` | `(0.9/(1 − 0.9²)) · 2 ≈ 9.5` | `π₂` |

> The optimal policy **depends on `γ`**. Myopic agents prefer immediate reward; far-sighted agents are willing to wait for a larger payoff.

### 8.5. Searching the policy space is intractable

- Number of deterministic policies is `|𝓐|^{|𝓢|}` — exponential in the number of states.
- **Brute-force search** (compute `vπ` for every `π`, pick the best) is infeasible for non-trivial MDPs.
- Better: characterize `v*` and `q*` directly via Bellman **optimality** equations, then derive `π*` from them.

---

## 9. Bellman Optimality Equations

The Bellman equation for `vπ` is linear because `π` averages actions. For the **optimal** value function, `π*` chooses the **best** action — the expectation over actions collapses to a `max`.

### 9.1. Bellman optimality equation for `v*`

Start from `vπ*(s) = Σ_a π*(a|s) Σ_{s', r} p(s', r | s, a)[r + γ v*(s')]`. Since `π*` puts all mass on the action maximizing the bracket:

```math
v_*(s) \;=\; \max_{a \in \mathcal{A}(s)} \sum_{s', r} p(s', r \mid s, a)\,\bigl[r + \gamma\, v_*(s')\bigr]
```

### 9.2. Bellman optimality equation for `q*`

Similarly, after taking `a`, the agent follows `π*` from `s'`, so the next-step term becomes `max_{a'} q*(s', a')`:

```math
q_*(s, a) \;=\; \sum_{s', r} p(s', r \mid s, a)\,\bigl[r + \gamma \max_{a'} q_*(s', a')\bigr]
```

### 9.3. Linear vs non-linear

| Equation | Form | Solvability |
|---|---|---|
| Bellman for `vπ` (fixed `π`) | **Linear** in `vπ` (a `Σ_a π(a\|s)` average over actions) | Closed-form for small MDPs (linear-system solve) |
| Bellman optimality for `v*` | **Non-linear** in `v*` (because of `max_a`) | **No closed-form** in general — needs iterative methods |

The iterative methods that solve the optimality equations are the subject of later chapters: **Value Iteration**, **Policy Iteration**, **Q-learning**, **Sarsa**.

---

## 10. Deriving the Optimal Policy from `v*` or `q*`

Once you know either optimal value function, the optimal policy is **greedy**:

### 10.1. From `v*` (requires the model `p`)

```math
\pi_*(s) \;=\; \arg\max_{a \in \mathcal{A}(s)} \sum_{s', r} p(s', r \mid s, a)\,\bigl[r + \gamma\, v_*(s')\bigr]
```

Need to look one step ahead through `p`.

### 10.2. From `q*` (no model needed)

```math
\pi_*(s) \;=\; \arg\max_{a \in \mathcal{A}(s)} q_*(s, a)
```

Pick the action with the highest action-value — **no model required**. This is why `q*` is the right object for **model-free** methods.

### 10.3. Solved gridworld (5×5 from Sutton & Barto)

- Special teleports: from state `A` → `+10` and jump to `A'`; from state `B` → `+5` and jump to `B'`. Off-edge actions: `−1` and stay. `γ = 0.9`.
- Solving Bellman optimality gives:

| `v*` row 1 | 22.0 | 24.4 | 22.0 | 19.4 | 17.5 |
|---|---|---|---|---|---|
| `v*` row 2 | 19.8 | 22.0 | 19.8 | 17.8 | 16.0 |
| `v*` row 3 | 17.8 | 19.8 | 17.8 | 16.0 | 14.4 |
| `v*` row 4 | 16.0 | 17.8 | 16.0 | 14.4 | 13.0 |
| `v*` row 5 | 14.4 | 16.0 | 14.4 | 13.0 | 11.7 |

The optimal policy `π*` is a set of arrows; **cells with multiple arrows indicate ties** — all those actions are simultaneously optimal.

---

## Key terms (glossary)

- **MDP** — `(𝓢, 𝓐, 𝓡, p, γ)` formalization of sequential decision making.
- **Dynamics function `p(s', r | s, a)`** — joint probability of next state and reward; sums to `1` over `(s', r)`.
- **Markov property** — `p` depends only on `(s, a)`, not on history.
- **Trajectory** — sequence `S₀, A₀, R₁, S₁, A₁, R₂, …`.
- **Episode / Terminal state** — episodic tasks end in a terminal state; its value is `0`.
- **Continuing task** — no terminal state; needs discounting.
- **Return `Gₜ`** — (discounted) sum of future rewards; random variable.
- **Discount factor `γ` ∈ [0, 1)** — short-sighted (`0`) vs far-sighted (`→1`).
- **Recursion** — `Gₜ = Rₜ₊₁ + γ Gₜ₊₁`.
- **Policy `π(a | s)`** — distribution over actions in each state; deterministic if degenerate.
- **State-value `vπ(s)`** — expected return from `s` under `π`.
- **Action-value `qπ(s, a)`** — expected return from `s` after taking `a`, then following `π`.
- **Bellman equation** — recursive linear relation for `vπ` / `qπ`.
- **Policy ordering** — `π₁ ≥ π₂` iff `vπ₁(s) ≥ vπ₂(s) ∀ s`.
- **Optimal policy `π*`** — exists, is `≥` every policy; not necessarily unique.
- **Optimal value functions `v*`, `q*`** — `max` over policies.
- **Bellman optimality equation** — same recursion but with `max_a` over actions; **non-linear**.
- **Greedy policy** — `π*(s) = argmax_a q*(s, a)`; with `v*` requires the model.

---

## Exam targets (likely written-exam questions)

1. **Define a finite MDP** (the 5-tuple) and write the dynamics function `p(s', r | s, a)` with its signature and the normalization condition.
2. **Define the return `Gₜ`** for episodic and continuing tasks; state when each definition is used and what discounting fixes.
3. **Derive the recursion** `Gₜ = Rₜ₊₁ + γ Gₜ₊₁` from the definition of `Gₜ`.
4. **State the Markov property** in terms of `p` and explain why "the present state is a sufficient statistic of the future".
5. **Write the Bellman equation** for `vπ` and for `qπ`, and **derive** one of them in three lines from the definition.
6. **Given a small gridworld** (e.g., the 2×2 with `γ = 0.7` and `+5` on leaving `B`), write the system of Bellman equations and solve for `vπ(A), vπ(B), vπ(C), vπ(D)`.
7. **Define the partial order on policies** and state the optimal-policy existence theorem.
8. **Write the Bellman optimality equations** for `v*` and `q*`. Explain why they are **non-linear** and how `π*` is recovered from each.
9. Given two policies `π₁, π₂` and two values of `γ`, **compute** `vπᵢ(s)` and decide which policy is optimal (show how the optimal choice depends on `γ`).

### Pitfalls

- **`vπ ≠ v*`** — never write `v(s)` without specifying the policy. `vπ` depends on `π`; `v*` is the max over all policies.
- The **Bellman equation** (for fixed `π`) is **linear**; the **Bellman optimality equation** is **non-linear** because of the `max`. Different solution methods.
- **Terminal-state value is always `0`** — don't include a non-zero `v(terminal)` in episodic Bellman sums.
- `p(s', r | s, a)` is a **joint** over `(s', r)`, not a product `p(s' | s, a) · r(s, a, s')`. The reward and next state are sampled together.
- **`γ < 1`** is strict for continuing tasks — `γ = 1` only makes sense for episodic tasks (or with extra care).
- **Optimal policies are not necessarily unique** — multiple actions can tie `argmax_a q*(s, a)`; the gridworld example shows cells with several optimal arrows.
- The optimal policy **depends on `γ`** — changing `γ` can flip which action is preferred (the `X / Y / Z` example).
- **Action set `𝓐(s)` can depend on the state** (recycling robot: `recharge` only available in `low`). Don't assume all actions are legal everywhere.
- **`q*` is model-free-friendly**: `π* = argmax_a q*(s, a)` needs no `p`. Recovering `π*` from `v*` does need `p`.
- **Brute force over deterministic policies is `|𝓐|^{|𝓢|}`** — exponential, not just large. The Bellman optimality machinery is what makes RL tractable.
