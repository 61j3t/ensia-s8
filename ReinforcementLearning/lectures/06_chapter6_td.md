# Chapter 6 — Temporal-Difference Learning

## Bird's eye view

- **TD learning** is the central and novel idea of RL — a **hybrid** of Monte Carlo (MC) and Dynamic Programming (DP).
- Like **MC**: learns directly from **raw experience**, no model of the environment needed (model-free).
- Like **DP**: **bootstraps** — updates one estimate from another estimate, *without waiting* for the episode to end.
- **TD(0) update**: `V(Sₜ) ← V(Sₜ) + α[Rₜ₊₁ + γV(Sₜ₊₁) − V(Sₜ)]`. The bracketed quantity is the **TD error δₜ**, a recurring object across all of RL.
- **TD vs MC vs DP**: TD = sampling + bootstrapping; MC = sampling, no bootstrapping; DP = bootstrapping, no sampling (needs full model).
- Three TD **control** algorithms in this chapter — all update Q-values, differ only in the **target**:
  - **SARSA** (on-policy): target uses the *actual* next action `Aₜ₊₁ ∼ π`.
  - **Q-learning** (off-policy): target uses `maxₐ Q(Sₜ₊₁, a)` — independent of the behaviour policy.
  - **Expected SARSA**: target uses `𝔼π[Q(Sₜ₊₁, ·)]` — averages over the policy's action distribution; lower variance.
- TD methods are **online**, fully **incremental**, handle **continuing tasks** and **incomplete sequences** — MC cannot.
- Each TD algorithm corresponds to a **sample backup** of a Bellman equation: TD(0) ↔ Bellman for `vπ`, SARSA ↔ Bellman for `qπ`, Q-learning ↔ Bellman *optimality* for `q*`.

---

## 1. Introduction

- **Temporal-difference (TD) learning** is a *central and novel* idea in reinforcement learning.
- TD is the **combination of Monte Carlo ideas + Dynamic Programming ideas**.
- **Like MC**: TD methods learn **directly from raw experience** without a model of the environment's dynamics (no `𝒫`, no `𝓡` needed).
- **Like DP**: TD methods **update estimates based in part on other learned estimates**, *without waiting* for a final outcome — they **bootstrap**.
- The relationship between **TD, DP, MC** is a *recurring theme* in RL theory.

---

## 2. TD Prediction

### 2.1. Review — estimating values from returns

- In the **prediction problem**, the goal is to learn `vπ(s)`, the value function that estimates the return starting from `s`.
- A simple **every-visit Monte Carlo** method, suitable for **non-stationary** environments, defines:

```math
v_\pi(s) \doteq \mathbb{E}_\pi[G_t \mid S_t = s]
```

```math
V(S_t) \leftarrow V(S_t) + \alpha\big[G_t - V(S_t)\big]
```

- The MC target is `Gₜ` — the *actual* return. **Must wait until the episode ends** to know `Gₜ`.

### 2.2. Bootstrapping identity

Expand `Gₜ` recursively:

```math
G_t = R_{t+1} + \gamma R_{t+2} + \gamma^2 R_{t+3} + \cdots = R_{t+1} + \gamma G_{t+1}
```

Then:

```math
v_\pi(s) = \mathbb{E}_\pi[G_t \mid S_t = s] = \mathbb{E}_\pi[R_{t+1} + \gamma G_{t+1} \mid S_t = s] = \mathbb{E}_\pi[R_{t+1} + \gamma v_\pi(S_{t+1}) \mid S_t = s]
```

This identity is what makes **bootstrapping** legitimate: `vπ(s)` can be defined in terms of *itself one step later*.

### 2.3. The TD(0) update

At time `t+1`, TD forms a target immediately from the observed reward `Rₜ₊₁` and the current estimate `V(Sₜ₊₁)`:

```math
V(S_t) \leftarrow V(S_t) + \alpha\big[R_{t+1} + \gamma V(S_{t+1}) - V(S_t)\big]
```

| | **MC target** | **TD target** |
|---|---|---|
| Expression | `Gₜ` | `Rₜ₊₁ + γV(Sₜ₊₁)` |
| Known when? | At end of episode | At step `t+1` |
| Uses estimate? | No (pure sample) | Yes — bootstraps off `V(Sₜ₊₁)` |

- This is called **TD(0)**, or **one-step TD** — a special case of TD(λ) and n-step TD.
- **TD = sampling (MC) + bootstrapping (DP)**.

### 2.4. The TD error δₜ

The bracketed quantity in the TD(0) update is an **error** — the difference between the current estimate `V(Sₜ)` and the *better* estimate `Rₜ₊₁ + γV(Sₜ₊₁)`:

```math
\delta_t \doteq R_{t+1} + \gamma V(S_{t+1}) - V(S_t)
```

- The TD error **arises in many forms throughout RL** — keep an eye out.

**Exercise (Sutton & Barto)** — the MC error decomposes as a discounted sum of TD errors:

```math
G_t - V(S_t) = \sum_{k=t}^{T-1} \gamma^{k-t}\,\delta_k
```

(provided `V` does not change during the episode).

### 2.5. Tabular TD(0) algorithm

```
Tabular TD(0) for estimating vπ

Input: the policy π to be evaluated
Parameter: step size α ∈ (0, 1]
Initialize V(s) for all s ∈ 𝒮⁺ arbitrarily, except V(terminal) = 0

Loop for each episode:
    Initialize S
    Loop for each step of episode:
        A ← action given by π for S
        Take action A, observe R, S'
        V(S) ← V(S) + α[R + γV(S') − V(S)]
        S ← S'
    until S is terminal
```

---

## 3. Advantages of TD prediction

| Compared to | TD advantage |
|---|---|
| **DP** | No model of `𝒫`, `𝓡` needed — learns from experience. |
| **MC** | Online, fully incremental — updates after **every step**, not at episode end. Crucial for **very long episodes** and **continuing (non-episodic) tasks**. |
| **MC** | Empirically **converges faster** than constant-α MC on many tasks. |

> MC cannot be used on tasks where termination is not guaranteed for every policy (see Windy Gridworld below). TD can.

### 3.1. Random walk example

The 5-state random walk: states A–E, start at C, equal-probability transitions left/right, terminate at the extreme left (`r = 0`) or extreme right (`r = +1`), `γ = 1`.

True values: `vπ(A, B, C, D, E) = (1/6, 2/6, 3/6, 4/6, 5/6)` = `(0.167, 0.333, 0.5, 0.667, 0.833)`.

- After 0 episodes both methods are flat; after 100 episodes TD's estimates align closely with the true values (left subplot of S&B Fig 6.2).
- The RMS-error curve shows **TD beats constant-α MC** for every reasonable `α`. TD was **consistently better** on this task.

---

## 4. SARSA — On-policy TD Control

### 4.1. From state-value to action-value

- Control needs an **action-value function** `Q(s, a)`, not just `V(s)`, because we don't have a model and so cannot do greedy improvement from `V` alone.
- For **on-policy** control, estimate `qπ(s, a)` for the **current behaviour policy** `π`, for every `s, a`.
- An episode now alternates **states and state–action pairs**: `… Sₜ, Aₜ, Rₜ₊₁, Sₜ₊₁, Aₜ₊₁, Rₜ₊₂, Sₜ₊₂, …`.
- Formally identical to TD prediction — just a Markov chain over state–action pairs with a reward process.

### 4.2. SARSA update

The convergence theorems for TD(0) on state values extend directly to action values:

```math
Q(S_t, A_t) \leftarrow Q(S_t, A_t) + \alpha\big[R_{t+1} + \gamma Q(S_{t+1}, A_{t+1}) - Q(S_t, A_t)\big]
```

- Update is applied after every transition from a **non-terminal** `Sₜ`. If `Sₜ₊₁` is **terminal**, then `γQ(Sₜ₊₁, Aₜ₊₁) ≜ 0`.
- The rule uses the quintuple `(Sₜ, Aₜ, Rₜ₊₁, Sₜ₊₁, Aₜ₊₁)` — hence the name **SARSA**.

### 4.3. SARSA control via GPI

The standard **Generalized Policy Iteration** loop (Chapter 4): alternate **evaluation** (estimate `qπ`) and **improvement** (make `π` greedier w.r.t. `Q`).

- Continually estimate `qπ` for the behaviour policy `π`, and at the same time **change π toward greediness w.r.t. qπ** — typically `ε`-greedy.

```
SARSA (on-policy TD control) for estimating Q ≈ q*

Parameters: step size α ∈ (0, 1], small ε > 0
Initialize Q(s, a) for all s ∈ 𝒮⁺, a ∈ 𝒜(s), arbitrarily, except Q(terminal, ·) = 0

Loop for each episode:
    Initialize S
    Choose A from S using policy derived from Q (e.g., ε-greedy)
    Loop for each step of episode:
        Take action A, observe R, S'
        Choose A' from S' using policy derived from Q (e.g., ε-greedy)
        Q(S, A) ← Q(S, A) + α[R + γQ(S', A') − Q(S, A)]
        S ← S';  A ← A'
    until S is terminal
```

### 4.4. Windy Gridworld example

- Standard 4-action gridworld with **start** and **goal**, plus an **upward crosswind** in the middle columns. Wind strength (0, 1, or 2 cells) printed below each column.
- `ε`-greedy SARSA with `ε = 0.1`, `α = 0.5`, `Q(s, a) = 0` initially.
- Learning curve (`episodes` vs `time-steps`) has **increasing slope** — the agent reaches the goal more quickly over time.
- **MC cannot be used here**: an early bad policy could trap the agent and the **episode would never end**, so the MC return `Gₜ` is undefined.
- **Online** methods like SARSA learn *during* the episode that such policies are poor and **switch away**.

---

## 5. Q-Learning — Off-policy TD Control

### 5.1. Update rule

One of the **early breakthroughs** in RL (Watkins 1989):

```math
Q(S_t, A_t) \leftarrow Q(S_t, A_t) + \alpha\Big[R_{t+1} + \gamma \max_{a} Q(S_{t+1}, a) - Q(S_t, A_t)\Big]
```

- The learned `Q` **directly approximates `q*`** (the *optimal* action-value function), **independent of the policy being followed**.
- This dramatically simplifies the analysis and enabled the **early convergence proofs**.
- The behaviour policy still matters — it determines **which `(s, a)` pairs are visited and updated**. Correct convergence requires only that **all pairs continue to be updated**.
- Under standard step-size conditions, `Q → q*` **with probability 1**.

### 5.2. Algorithm

```
Q-learning (off-policy TD control) for estimating π ≈ π*

Parameters: step size α ∈ (0, 1], small ε > 0
Initialize Q(s, a) for all s ∈ 𝒮⁺, a ∈ 𝒜(s), arbitrarily, except Q(terminal, ·) = 0

Loop for each episode:
    Initialize S
    Loop for each step of episode:
        Choose A from S using policy derived from Q (e.g., ε-greedy)
        Take action A, observe R, S'
        Q(S, A) ← Q(S, A) + α[R + γ maxₐ Q(S', a) − Q(S, A)]
        S ← S'
    until S is terminal
```

Note the difference from SARSA — Q-learning does **not** sample `A'`; it uses the **max** over actions.

### 5.3. Cliff-Walking example (SARSA vs Q-learning)

- Undiscounted episodic gridworld. Actions = {up, down, right, left}. Reward = `−1` per step everywhere except the **Cliff region** below the optimal path, where stepping in gives `R = −100` and sends the agent back to the start.
- With `ε`-greedy (`ε = 0.1`):

| Method | Learns | Online performance |
|---|---|---|
| **Q-learning** | Optimal policy — walks **right along the edge of the cliff**. | Worse — `ε`-random exploration occasionally pushes it **off the cliff**. |
| **SARSA** | Longer **roundabout** path through the upper part of the grid (the *safer* path). | Better — total reward per episode is higher. |

> **Q-learning** learns the *optimal* values but its **online** performance is worse than SARSA, which learns the *safer* roundabout policy. If `ε` is **gradually reduced**, both methods converge **asymptotically to the optimal policy**.

### 5.4. Key contrast — SARSA vs Q-learning

| | **SARSA** | **Q-learning** |
|---|---|---|
| Class | **On-policy** | **Off-policy** |
| Learns | `qπ` for the policy actually followed | `q*` regardless of policy |
| Target | `R + γQ(S', A')` with `A' ∼ π` | `R + γ maxₐ Q(S', a)` |
| Accounts for exploration? | **Yes** — `A'` is sampled from the ε-greedy policy | **No** — assumes the *best* action is taken |
| Behaviour | Safer / "roundabout" near risky cliffs | Riskier short-term, optimal long-term |
| Stability | Smoother, more stable | Can be riskier in the short run |
| Convergence | Converges to `q*` as `ε → 0` | Converges to `q*` directly |

> As `ε → 0`, both methods mostly exploit and converge to the same optimal policy.

---

## 6. MC vs TD Control

**TD vs MC** in the control setting:

- **Lower variance** updates → more stable learning.
- **Online** — update after each step, not at episode end.
- **Handles incomplete sequences** — needed for ongoing / continuing tasks.

**The natural recipe** for TD control:

- Apply TD updates to `Q(S, A)` instead of waiting for full episode completion.
- Use an **ε-greedy** policy-improvement strategy to balance exploration and exploitation.
- Perform updates **at every time step**, enabling faster learning and adaptation.

---

## 7. Expected SARSA

### 7.1. Idea

**Expected SARSA** is just like Q-learning except that instead of the **max** over the next state's actions, it uses the **expected value** under the current policy — taking into account *how likely each action is* under `π`.

### 7.2. Update rule

```math
Q(S_t, A_t) \leftarrow Q(S_t, A_t) + \alpha\Big[R_{t+1} + \gamma\,\mathbb{E}_\pi[Q(S_{t+1}, A_{t+1}) \mid S_{t+1}] - Q(S_t, A_t)\Big]
```

Expanding the expectation:

```math
Q(S_t, A_t) \leftarrow Q(S_t, A_t) + \alpha\Big[R_{t+1} + \gamma\sum_{a} \pi(a \mid S_{t+1})\,Q(S_{t+1}, a) - Q(S_t, A_t)\Big]
```

- Given the next state `Sₜ₊₁`, this algorithm moves **deterministically in the same direction SARSA moves in expectation** — hence the name *Expected SARSA*.

### 7.3. Algorithm

```
Expected SARSA (off-policy TD control)

Parameters: step size α ∈ (0, 1], small ε > 0
Initialize Q(s, a) arbitrarily; Q(terminal, ·) = 0
Initialize policy π based on Q (e.g., ε-greedy)

For each episode:
    Initialize state s
    While s is not terminal:
        Choose action a using policy π (e.g., ε-greedy)
        Take action a, observe reward R and next state s'
        Q(s, a) ← Q(s, a) + α[R + γ Σₐ' π(a' | s') · Q(s', a') − Q(s, a)]
        s ← s'
Return Q and π
```

### 7.4. Cliff-Walking — Expected SARSA vs SARSA

Experiment: 100 episodes averaged over 50,000 independent runs.

- **Expected SARSA outperformed SARSA for almost all values of `α`**. It can use **larger `α` more effectively** because it **explicitly averages** over the randomness due to its own policy.
- The environment is **deterministic** — no other sources of randomness — so Expected SARSA's updates are **deterministic for a given `(s, a)`**. SARSA's updates **vary significantly depending on the next action** drawn.
- After 100,000 episodes: Expected SARSA's long-term behaviour is **unaffected by `α`** (deterministic updates), so `α` only controls how quickly estimates approach their target. **SARSA fails to converge for large `α`**; only as `α` decreases does its long-run performance approach Expected SARSA's.

---

## 8. Relationship between DP and TD

Every TD algorithm is the **sample backup** version of a DP **full backup**, applied to a Bellman equation:

| Bellman equation | Full backup (DP) | Sample backup (TD) |
|---|---|---|
| **Bellman expectation** for `vπ(s)` | Iterative Policy Evaluation | **TD Learning** (TD(0)) |
| **Bellman expectation** for `qπ(s, a)` | Q-Policy Iteration | **SARSA** |
| **Bellman optimality** for `q*(s, a)` | Q-Value Iteration | **Q-Learning** |

Update-rule comparison — let `x ⟵α y ≡ x ← x + α(y − x)`:

| | Full backup (DP) | Sample backup (TD) |
|---|---|---|
| `v` | `V(s) ← 𝔼[R + γV(S') \| s]` | `V(S) ⟵α R + γV(S')` |
| `qπ` | `Q(s, a) ← 𝔼[R + γQ(S', A') \| s, a]` | `Q(S, A) ⟵α R + γQ(S', A')` |
| `q*` | `Q(s, a) ← 𝔼[R + γ maxₐ' Q(S', a') \| s, a]` | `Q(S, A) ⟵α R + γ maxₐ' Q(S', a')` |

- **Full backup** = uses the **model** (`𝒫`, `𝓡`) to take a *full expectation* over next-state distributions.
- **Sample backup** = uses a **single sampled transition** instead. No model needed.

---

## 9. DP vs MC vs TD — the canonical comparison

| Property | **DP** | **MC** | **TD** |
|---|---|---|---|
| Needs a **model** (`𝒫`, `𝓡`)? | **Yes** | No | No |
| **Bootstraps** (uses other estimates)? | **Yes** | No | **Yes** |
| **Samples** (single trajectory)? | No (sweeps all states/transitions) | **Yes** | **Yes** |
| Updates **after each step**? | N/A (sweeps) | No — only at episode end | **Yes** |
| Works on **continuing** tasks? | Yes | **No** (no episode end) | **Yes** |
| Handles **non-terminating** policies? | Yes | **No** | **Yes** |
| Bias of value estimate | Unbiased (if model exact) | **Unbiased** (uses true `Gₜ`) | **Biased** (target uses estimate `V`) |
| Variance of update | Low | **High** (full return) | **Low** (one step + bootstrap) |
| Backup diagram | Full | Deep (whole episode) | Shallow (one step) |
| Convergence speed in practice | Fast per sweep, but sweeps are costly | Slower | **Often fastest** |

**Slogan**: *DP = bootstrap, no sample. MC = sample, no bootstrap. TD = both.*

---

## Key terms (glossary)

- **TD learning** — class of methods that learn from experience (no model) and bootstrap (no episode end).
- **Bootstrapping** — updating an estimate using another current estimate (DP, TD do; MC doesn't).
- **Sampling** — using a single experienced transition instead of an expectation (MC, TD do; DP doesn't).
- **TD(0) / one-step TD** — simplest TD method; target `Rₜ₊₁ + γV(Sₜ₊₁)`.
- **TD error δₜ** — `Rₜ₊₁ + γV(Sₜ₊₁) − V(Sₜ)`; recurs throughout RL.
- **TD target** — `Rₜ₊₁ + γV(Sₜ₊₁)` (prediction) or `Rₜ₊₁ + γQ(Sₜ₊₁, Aₜ₊₁)` (SARSA) or `Rₜ₊₁ + γ maxₐ Q(Sₜ₊₁, a)` (Q-learning).
- **SARSA** — on-policy TD control; named after the quintuple `(S, A, R, S', A')`.
- **Q-learning** — off-policy TD control; learns `q*` directly via the **max** in the target.
- **Expected SARSA** — same idea as Q-learning but uses `𝔼π[Q(S', A')]` instead of `maxₐ Q(S', a)`.
- **On-policy** — learn the value of the policy you are *currently following* (including its exploration).
- **Off-policy** — learn the value of a *different* (typically greedy / optimal) policy than the one collecting data.
- **Behaviour policy** — the policy actually taking actions (e.g., `ε`-greedy).
- **Quintuple / SARSA tuple** — `(Sₜ, Aₜ, Rₜ₊₁, Sₜ₊₁, Aₜ₊₁)`.
- **Full backup** — DP-style update over a full expectation (needs model).
- **Sample backup** — TD-style update from a single sampled transition.
- **GPI** — Generalized Policy Iteration: alternating evaluation ↔ improvement.

---

## Exam targets (likely written-exam questions)

1. **Define TD learning** and state precisely how it combines MC and DP. Explain *bootstrapping* and *sampling* with one example each.
2. **Write the TD(0) update** for state values, identify the **TD target** and **TD error** `δₜ`. Prove the identity `Gₜ − V(Sₜ) = Σ γᵏ⁻ᵗ δₖ` under fixed `V`.
3. **DP vs MC vs TD table** — fill in the comparison (model needed? bootstrap? sample? online? bias/variance?). Justify each cell.
4. **Tabular TD(0) algorithm** for `vπ` — write the pseudocode.
5. **SARSA** — derive the update, write the full ε-greedy control algorithm, explain why it is **on-policy**, and state when `γQ(Sₜ₊₁, Aₜ₊₁)` is set to 0.
6. **Q-learning** — write the update, write the algorithm, explain why it is **off-policy** and approximates `q*` directly.
7. **Cliff-Walking** — compare SARSA and Q-learning on this task: which finds the optimal path, which has better online performance, and *why*. State what happens as `ε → 0`.
8. **Expected SARSA** — derive the update rule from Q-learning's, explain *why* the expectation reduces variance, and discuss the deterministic-environment cliff-walking result (Expected SARSA outperforms SARSA across `α`).
9. **DP ↔ TD correspondence** — for each of `vπ`, `qπ`, `q*`, give the Bellman equation, the DP full-backup name, and the TD sample-backup name (Iterative Policy Evaluation ↔ TD; Q-Policy Iteration ↔ SARSA; Q-Value Iteration ↔ Q-learning).

### Pitfalls

- **TD is biased**. The target `Rₜ₊₁ + γV(Sₜ₊₁)` uses an *estimate* of `V`, so the update is biased; only MC is unbiased. The trade-off is **dramatically lower variance**, which is usually worth it.
- **SARSA ≠ Q-learning** — both look almost identical, but the target action differs: SARSA uses **the actual `A'` sampled from `π`**, Q-learning uses **`max` over all actions**. That single change is what makes one on-policy and the other off-policy.
- **MC fails when termination is not guaranteed** — Windy Gridworld and many continuing tasks. TD has no such issue.
- **Terminal states**: if `Sₜ₊₁` is terminal, **`γ V(Sₜ₊₁) ≡ 0`** and likewise `γ Q(Sₜ₊₁, Aₜ₊₁) ≡ 0`. Forgetting this corrupts terminal updates.
- **Q-learning's online performance can be *worse* than SARSA's** while still learning the optimal `q*` — because `ε`-greedy exploration can push it off the cliff. *Optimal value ≠ optimal online return.*
- **Expected SARSA needs the policy `π(·|s)`** — for ε-greedy this is trivial; for arbitrary policies you must be able to evaluate `π(a|s)`.
- **"On-policy" doesn't mean greedy** — SARSA with ε-greedy is on-policy; the behaviour policy and the target policy are the **same ε-greedy policy**, not the greedy one.
- **Step size α** matters: in SARSA, large `α` can prevent convergence (cliff-walking experiment); in Expected SARSA, large `α` is safe because updates are deterministic given `(s, a)`.
- **Bootstrapping amplifies bias under function approximation** — beware when leaving the tabular setting (foreshadows Chapters 8–9).
