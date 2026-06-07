# RL — Syllabus Skeleton (Phase 0)

> Course: Reinforcement Learning — ENSIA (Dr. Aissa Boulmerka; advanced chapters co-presented by Dr. Ferhi Nadir), 2025-2026.
> Textbook: Sutton & Barto, *Reinforcement Learning: An Introduction* (2nd ed., MIT Press 2018).
> Evaluation: Labs 5 % + Project 15 % + Midterm (90 min) 20 % + Final (90 min) 60 %.
> Use this as a **mental scaffold**, not a study substitute. Active recall does the real work.

---

## Ch 1 — Introduction to RL (53 pages)
- **RL =** programming agents by reward; trial-and-error; **delayed**, **sequential**, non-i.i.d. feedback; the agent's actions affect future data
- **Reward hypothesis**: any goal = maximizing expected cumulative scalar reward
- **Interaction loop**: agent emits `Aₜ`, env returns `Oₜ₊₁`, `Rₜ₊₁`. History `Hₜ`, agent state `Sₜᵃ`, env state `Sₜᵉ`. `Sₜ = f(Hₜ)`
- **Markov state**: `P[Sₜ₊₁|Sₜ] = P[Sₜ₊₁|S₁,…,Sₜ]` → past discarded once present known. Fully obs → **MDP**, partial → **POMDP**
- **3 building blocks**: Policy `π`, Value function `vπ`/`qπ`, Model `𝒫`/`𝓡` (agent has ≥1)
- **Taxonomies**: value-based / policy-based / actor-critic — and model-free / model-based
- **3 fundamental dichotomies**: Learning vs Planning, Exploration vs Exploitation, Prediction vs Control
- Maze + Gridworld worked examples for policy / value / model

## Ch 2 — Multi-armed Bandits (52 pages)
- **k-armed bandit** = simplest non-trivial RL: 1 state, `k` actions, stationary reward dist. Isolates **evaluative** feedback (vs supervised's **instructive**)
- True value `q*(a) = 𝔼[Rₜ | Aₜ = a]`; estimate `Qₜ(a)` via **sample average**
- **Incremental update**: `Qₙ₊₁ = Qₙ + (1/n)[Rₙ − Qₙ]` (= `NewEst ← OldEst + StepSize · [Target − OldEst]`)
- **Non-stationary**: replace `1/n` with constant `α` → **exponential recency-weighted average**
- **5 action-selection rules**:
  - **Greedy** — always exploit
  - **ε-greedy** — random `a` with prob `ε`
  - **Optimistic initial values** — high `Q₀` forces early exploration
  - **UCB**: `Aₜ = argmaxₐ [Qₜ(a) + c√(ln t / Nₜ(a))]`
  - **Gradient bandit**: softmax preferences `Hₜ(a)` updated by SGD with baseline
- **10-armed testbed**: 2000 problems, `q*(a) ~ 𝒩(0,1)`, `Rₜ ~ 𝒩(q*(a), 1)` — the canonical benchmark
- Parameter study table compares the four rules

## Ch 3 — Finite Markov Decision Processes (43 pages)
- **MDP** = (`𝒮`, `𝒜`, `𝓡`, `p`) with one-step dynamics `p(s', r | s, a)` — fully specifies the env
- **Markov property** is structural — the present state is a sufficient statistic for the future
- **Return**: `Gₜ = Rₜ₊₁ + γ Rₜ₊₂ + γ² Rₜ₊₃ + ⋯`. Recursion: `Gₜ = Rₜ₊₁ + γ Gₜ₊₁`
- **Episodic** (finite `T`, terminal state) vs **continuing** (`T = ∞`, needs `γ < 1`)
- **Discount factor** `γ ∈ [0, 1)`: `γ = 0` myopic, `γ → 1` far-sighted
- **Policy**: `π(a|s)` (stochastic) or `π(s)` (deterministic)
- **Bellman expectation equations** (linear in `v`):
  - `vπ(s) = Σₐ π(a|s) Σ_{s',r} p(s',r|s,a)[r + γ vπ(s')]`
  - `qπ(s,a) = Σ_{s',r} p(s',r|s,a)[r + γ Σ_{a'} π(a'|s') qπ(s',a')]`
- **Optimal policy** `π*` exists for any finite MDP; `v*(s) = maxπ vπ(s)`, `q*(s,a) = maxπ qπ(s,a)`
- **Bellman optimality equations** (non-linear, `max` replaces `Σπ`):
  - `v*(s) = maxₐ Σ_{s',r} p(s',r|s,a)[r + γ v*(s')]`
  - `q*(s,a) = Σ_{s',r} p(s',r|s,a)[r + γ maxₐ' q*(s',a')]`
- Recover `π*` greedily from `v*` (needs model) or directly from `q*` (no model needed)

## Ch 4 — Dynamic Programming (50 pages)
- **DP** = solve a known finite MDP by turning Bellman equations into iterative updates. **Planning**, not learning
- **Iterative Policy Evaluation** (prediction): repeated Bellman expectation backups; converges to `vπ`
- **Policy Improvement Theorem**: greedify w.r.t. `vπ` → `π' ≥ π`, strictly better unless already optimal
- **Policy Iteration** (control): alternate evaluation + improvement until stable. Converges in **finitely many** steps (finite MDPs have finitely many deterministic policies)
- **Value Iteration**: truncate evaluation to **one sweep** of `max`-backups: `vₖ₊₁(s) = maxₐ Σ p[r + γ vₖ(s')]`
- **Bootstrapping**: update one estimate from another (no real returns)
- **Synchronous** (full sweep per iter) vs **Asynchronous DP** (any order, possibly biased toward "important" states)
- **GPI (Generalized Policy Iteration)** = the master pattern — almost every later RL method is an instance
- Worked: 4×4 gridworld (evaluation), penalty-grid (full PI)

## Ch 5 — Monte Carlo Methods (32 pages)
- **MC** = learn from **sample episodes**; model-free; **no bootstrapping**. Needs **episodic** tasks
- **MC prediction**: average returns observed from each state. Two variants — **first-visit** (only the first occurrence per episode) and **every-visit** (every occurrence). Both converge to `vπ(s)`
- Prefer estimating **`qπ(s,a)`** rather than `vπ(s)` when no model — greedy action then needs no `𝒫`, `𝓡`
- **Exploration challenge**: never-visited `(s,a)` pairs never get evaluated. Two solutions:
  - **Exploring Starts (ES)**: each episode starts in a random `(S₀, A₀)` with all pairs > 0 probability
  - **ε-soft / ε-greedy** behaviour: `π(a|s) ≥ ε/|𝒜(s)|` for all `(s,a)`
- **On-policy vs Off-policy**:
  - **On-policy**: evaluate/improve the policy you act with (ε-greedy)
  - **Off-policy**: **target** `π` ≠ **behaviour** `b`. Coverage assumption `π(a|s) > 0 ⇒ b(a|s) > 0`
- **Importance sampling ratio**: `ρ_{t:T-1} = Π_{k=t}^{T-1} π(Aₖ|Sₖ)/b(Aₖ|Sₖ)` (env dynamics `p` cancel)
- **Ordinary IS** = unbiased, possibly infinite variance. **Weighted IS** = biased but variance-bounded
- Incremental weighted IS update for off-policy MC control; early-exit when `Aₜ ≠ π(Sₜ)`

## Ch 6 — Temporal-Difference Learning (36 pages)
- **TD = MC sampling + DP bootstrapping**. Model-free, online, incremental, handles continuing tasks and incomplete sequences
- **TD(0) update**: `V(Sₜ) ← V(Sₜ) + α[Rₜ₊₁ + γ V(Sₜ₊₁) − V(Sₜ)]`. Bracket = **TD error `δₜ`**
- Comparison table on **bootstrapping × sampling**:
  | Method | Bootstrap | Sample |
  |---|---|---|
  | DP | ✓ | ✗ |
  | MC | ✗ | ✓ |
  | TD | ✓ | ✓ |
- **3 TD-control algorithms** (differ only in the **target** of the Q-update):
  - **SARSA** (on-policy): `target = R + γ Q(S', A')`, `A' ~ π`
  - **Q-learning** (off-policy): `target = R + γ maxₐ' Q(S', a')`
  - **Expected SARSA**: `target = R + γ 𝔼π[Q(S', ·)]` — lower variance
- Sample-backup correspondence: TD(0) ↔ Iter. Policy Eval; SARSA ↔ Policy Iter; Q-learning ↔ Value Iter
- Worked: Windy Gridworld (SARSA), Cliff Walking (SARSA vs Q-learning paths)

## Ch 7 — Planning and Learning with Tabular Methods (47 pages)
- **Planning** = uses a model; **Learning** = uses real experience. Same value-function updates feed either source
- **Sample model** (returns one drawn transition) vs **distribution model** (full probabilities)
- **Random-sample one-step tabular Q-planning** = one-step Q-learning on simulated transitions
- **Dyna-Q** loop: (i) real act, (ii) direct RL update, (iii) update model, (iv) `n` planning updates on random prior `(s, a)` pairs
- Dyna maze: `n = 50` planning steps → effective policy in **2 episodes** vs many with `n = 0`
- **Dyna-Q+** for **changing models**: exploration bonus `κ√τ` (`τ` = time since `(s,a)` tried). Solves the **shortcut maze** Dyna-Q can't
- **Monte Carlo Tree Search (MCTS)** — decision-time planning by rollouts. 4 steps:
  1. **Selection** — descend via tree policy (e.g., UCT)
  2. **Expansion** — add a leaf
  3. **Simulation** — rollout policy to end
  4. **Backup** — propagate result upward
- Core engine behind **AlphaGo**
- All tabular RL classified along **width** (sample → distribution) × **depth** (one-step → full return) of update

## Ch 8 — Value Function Approximation (87 pages — covers both Sutton-Barto Ch 9 & 10)
- **VFA**: `v̂(s, w) ≈ vπ(s)` with `w ∈ ℝᵈ`, `d ≪ |𝒮|`. One weight change affects many states → **generalization** at the cost of **discrimination**
- **Prediction objective**: `VE(w) = Σₛ μ(s)[vπ(s) − v̂(s, w)]²`, weighted by on-policy state distribution `μ`
- **Gradient MC** (unbiased): `w ← w + α[Gₜ − v̂(Sₜ, w)] ∇v̂(Sₜ, w)`
- **Semi-gradient TD(0)**: `w ← w + α[Rₜ₊₁ + γ v̂(Sₜ₊₁, w) − v̂(Sₜ, w)] ∇v̂(Sₜ, w)` — `∇` of bootstrap target **dropped**
- **Linear methods**: `v̂(s, w) = wᵀ x(s)`, gradient is the feature vector `∇v̂ = x(s)`. Tabular = indicator features
- **TD fixed point**: `w_TD = A⁻¹ b`; bound `VE(w_TD) ≤ (1/(1−γ)) minₚ VE(w)`
- **Feature constructions** (linear): coarse coding, **tile coding** (the workhorse — used in Mountain Car), polynomial, Fourier, RBF, kernel
- **Nonlinear FA**: neural nets (universal approximation; depth helps composition)
- **On-policy control with approximation** (Sutton-Barto Ch 10):
  - **Episodic semi-gradient SARSA**: `w ← w + α[R + γ q̂(S', A', w) − q̂(S, A, w)] ∇q̂(S, A, w)`, then ε-greedy action
  - n-step semi-gradient SARSA, Expected SARSA, Q-learning as target variants
- **Average-reward setting** for continuing tasks (no `γ`):
  - `r(π) = lim_{h→∞} (1/h) Σ_{t=1}^h 𝔼[Rₜ]`
  - **Differential return**: `Gₜ = Σ_{k=0}^∞ (R_{t+k+1} − r(π))`
  - **Differential semi-gradient SARSA**: track `w` *and* `R̄ ≈ r(π)` with two step-sizes

## Ch 9 — Policy Gradient Methods (55 pages)
- Learn `π(a|s, θ)` **directly**; actions selected without consulting a value function (a critic may still learn `θ`)
- **Update**: `θₜ₊₁ = θₜ + α ∇J(θₜ)` for performance measure `J(θ)`
- **Obstacle**: changing `θ` shifts the state distribution `μ(s)`. **Policy Gradient Theorem** kills `∇μ`:
  ```math
  \nabla J(\theta) \propto \sum_s \mu(s) \sum_a q_\pi(s,a) \nabla \pi(a|s,\theta)
  ```
- **REINFORCE** (Monte Carlo PG): `θ ← θ + α Gₜ ∇ ln π(Aₜ|Sₜ, θ)` — unbiased, high variance
- **REINFORCE with baseline**: subtract `b(s)` (typically `v̂(s, w)`) — unbiased, lower variance
- **Actor-critic (one-step)**: replace `Gₜ` by **TD error** `δₜ = R + v̂(S', w) − v̂(S, w)`. Fully online, incremental, even lower variance
- **Parameterizations**:
  - **Softmax over preferences** (discrete actions): `π(a|s, θ) ∝ exp(h(s, a, θ))`
  - **Gaussian** (continuous): `π(a|s, θ) = 𝒩(μ(s, θ), σ(s, θ)²)`, with `σ = exp(θ_σᵀ x(s))`
- **Advantages**: stochastic optimal policies, smooth updates, continuous actions, autonomous greedification

## Ch 10 — Advanced RL Part 1 (179 pages — Deep Value-Based + Trust-Region/Clip Policy Gradient)
- **Q-learning convergence**: contraction operator `H` + Robbins-Monro step-size conditions
- **Vanilla DQN** (Mnih 2015): neural Q + **target network** (kills moving targets) + **experience replay** (de-correlates samples)
- **Value-based vs Policy-based**: PG learns `π` directly → can be stochastic, continuous, smooth
- **Policy-based family** (stochastic):
  - **REINFORCE** → **REINFORCE w/ baseline** (variance ↓)
  - **TRPO** (Schulman 2015): largest step with `KL(πθ ‖ πθₖ) ≤ δ`. Solved by Taylor expand + conjugate gradient (never form `H⁻¹`) + line search
  - **PPO**: first-order, just **clip the ratio** `r = πθ/πθₖ` to `[1−ε, 1+ε]`. Simpler, ≥ TRPO empirically
  - **GRPO** (DeepSeek 2024): group-relative advantage, drops the value baseline
- **Policy-based family** (deterministic): **DPG → DDPG → TD3**
  - **DDPG** = "deep Q-learning for continuous actions": learn `Qφ(s,a)` (off-policy, MSBE) + deterministic `μθ(s)` (gradient ascent through `Q`)
  - **Polyak averaging** for target nets: `φtarg ← ρ φtarg + (1−ρ) φ`
- **Hybrid**: A2C, A3C, SAC (mentioned)

## Ch 10 — Advanced RL Part 2 (58 pages — Model-Based, Imitation, MARL setup)
- **Model-based RL**: agent learns/uses model to **plan**. Models: deterministic / stochastic / sample
- **Dyna-Q** (recap): one real step + `n` simulated steps from learned model
- **World Models** (Ha & Schmidhuber): V = VAE encoder, M = MDN-RNN dynamics, C = tiny controller. Train **C in the dream**, transfer to real env
- **Imitation Learning** — no reward, only expert demos `Ξ = {ξ₁, …, ξ_D}`:
  - **Behavior Cloning** — supervised learning on `(state, action)`. **Compounding error** under distribution shift
  - **DAgger** — query expert on states the *learner* visits, aggregate. Fixes BC's drift
  - **Inverse RL** — fit a reward `R(x, u) = wᵀ φ(x, u)` that makes the expert optimal. **Reward ambiguity** (`w = 0`)
  - **Apprenticeship Learning** (Abbeel & Ng 2004) — sidestep `w*`, just match **feature expectations** `μ(π) ≈ μ(π*)`
- **MARL setup**:
  - 3 scenarios: cooperative / competitive / mixed
  - Formal: normal-form games → stochastic games → POSGs
  - **Nash equilibrium**: no agent gains by unilateral deviation
  - Central RL over `n` agents with `|A|` actions has `|A|ⁿ` joint actions → combinatorial explosion → need MARL
  - **CTDE** (Centralized Training, Decentralized Execution) — the workhorse compromise
  - 6 dimensions to memorize: Size, Knowledge, Observability, Rewards, Objective, Centralization/Communication

## Ch 10 — Advanced RL Part 3 (29 pages — QMIX & Hierarchical RL)
- **MARL cooperative case**: **Dec-POMDP** — local obs, single global reward. Challenges: credit assignment, coordination, non-stationarity
- **MARL value-decomposition progression**:
  - **IQL (1993)** — independent Q-learners; no factorization, no convergence guarantee
  - **VDN (2017)** — additive: `Q_tot = Σ Qₐ`. No monotonicity guarantee
  - **QMIX** — monotonic mixing via **hypernetworks**: `∂Q_tot/∂Qₐ ≥ 0` → **IGM (Individual-Global-Max)**: per-agent argmax = joint argmax
- **CTDE** in QMIX: global state mixes Q-values at training; agents execute on local Q-nets at runtime
- **Hierarchical RL (HRL)**: stack policies at multiple abstraction levels — high-level picks subgoals, low-level executes primitives. Mitigates long-horizon / sparse-reward exploration
- **Options framework**: option `o = (I, π, β)` — initiation set, internal policy, termination probability. Turns problem into **Semi-MDP**
- **H-DQN** navigation case study
- **Hierarchical Q-learning** with option-duration `τ` discounted target

---

## Cross-cutting themes (likely exam targets)

- **Bellman equations everywhere** — expectation (`vπ`/`qπ`, linear) vs optimality (`v*`/`q*`, non-linear with `max`). All later algorithms are sample-/bootstrap-/approximation-variants of these
- **Bootstrapping × Sampling × Approximation** triad — almost any algorithm is one (or several) of:
  | | Distribution model | Sample model | Real experience |
  |---|---|---|---|
  | **One-step bootstrap** | DP | Random-sample 1-step Q-planning | TD(0), SARSA, Q-learning |
  | **Full return** | Exhaustive search | MC tree rollouts | Monte Carlo |
- **Generalized Policy Iteration (GPI)**: alternate evaluation (`Q → qπ`) and improvement (`π → greedy(Q)`) — common to DP, MC, TD, AC
- **Value-based vs Policy-based vs Actor-Critic**: who picks the action, who learns from feedback
- **On-policy vs Off-policy**: same vs different policy for behaviour and target; off-policy → importance sampling (MC), `max` target (Q-learning), or replay buffers (DQN)
- **Tabular → Approximation**: weight vector `w` (or `θ`), semi-gradient updates, generalization-discrimination trade-off
- **Exploration mechanisms ladder**: ε-greedy → optimistic init → UCB → softmax preferences → entropy regularization → intrinsic curiosity / `κ√τ` bonus
- **Variance-reduction ladder for PG**: REINFORCE → baseline → TD error / advantage → GAE
- **Deadly triad**: function approximation + bootstrapping + off-policy — convergence not guaranteed

---

## Study plan from here

1. ✅ **Phase 0 — Syllabus map** (this file)
2. ⏳ **Phase 1 — Per chapter**: read each `.md`, do flashcards, Feynman-explain each algorithm aloud, attempt exam targets without notes
3. ⏳ **Phase 2 — Interleaved review**: mix questions across chapters — e.g. "derive the Q-learning update from the Bellman optimality equation", "explain GPI in DP, MC, TD, AC, DQN"
4. ⏳ **Phase 3 — Mock exam**: 90 min written; emphasise Bellman equations, MDP setup, algorithm pseudocodes, the MC ↔ TD ↔ DP table, policy-gradient theorem derivation, DQN/PPO core tricks
