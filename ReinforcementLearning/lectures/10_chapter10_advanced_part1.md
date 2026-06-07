# Chapter 10 — Advanced RL Part 1 — Policy-based RL

> Guest lecture by **Dr. Nadir Farhi** (Univ. Gustave Eiffel / Cosys-Grettia), ENSIA, 27 April 2026. Title on slides: *"Reinforcement Learning and Optimal Control — Policy-based RL"*.

## Bird's eye view

- **RL = optimal control under uncertainty** of an MDP. Criterion: `maxπ 𝔼π Σ γᵏ Rₜ₊ₖ₊₁(sₖ, aₖ)`. Known dynamics → DP; unknown → MC sampling; RL = combines DP + MC.
- Review block: action-value Bellman, **Q-learning** (Watkins 1989, off-policy TD), full **convergence proof sketch** via the contraction `H` and the Robbins-Monro step-size conditions.
- Bridge: **Vanilla DQN (2015)** = NN + **target network** (moving-targets fix) + **experience replay** (correlated-data fix).
- **Value-based vs Policy-based** split: value methods choose actions *via* action values (no policy without `q`); policy methods learn `π(a|s, θ)` **directly** with gradient ascent on `J(θ)`.
- Why policy methods: (i) stochastic optimal policies, (ii) continuous action spaces, (iii) smooth policy updates → stronger convergence, (iv) discrete via softmax over preferences `h(s,a,θ)`.
- **Policy Gradient Theorem** (episodic): `∇J(θ) ∝ Σₛ μ(s) Σₐ qπ(s,a) ∇π(a|s,θ)`; on-policy state distribution `μ(s) = η(s) / Σ η`.
- **Family 1** (stochastic policies): **REINFORCE** (Williams 1992, MC PG) → **REINFORCE w/ baseline** `b(s)` (variance reduction) → **TRPO** (Schulman 2015, KL-constrained trust region, 2nd-order) → **PPO** (1st-order, KL-penalty or clip) → **GRPO** (DeepSeek 2024, group-relative advantage, no value baseline).
- **Family 2** (deterministic policies): **DPG → DDPG → TD3**. DDPG = "deep Q-learning for continuous actions": learns `Qφ(s,a)` (off-policy, Bellman MSBE loss) + deterministic policy `μθ(s)` (gradient ascent through `Q`).
- **Hybrid** (actor-critic): A2C → A3C → SAC (mentioned, not detailed).
- Two big REINFORCE problems TRPO/PPO fix: (1) past trajectories **not reused** (on-policy waste), (2) **unstable convergence** — large step size kills performance even when parameters are close.
- **TRPO core idea**: take *largest* step that still keeps `KL(πθ ‖ πθₖ) ≤ δ`. Solved via Taylor expand → `gᵀ(θ−θₖ) s.t. ½(θ−θₖ)ᵀH(θ−θₖ) ≤ δ` → analytic Lagrangian solution + backtracking line search; `H⁻¹g` via **conjugate gradient** (never form `H⁻¹`).
- **PPO-Clip core idea**: avoid `H`, just **clip the probability ratio** `r = πθ/πθₖ` so the surrogate stops rewarding excursions outside `[1−ε, 1+ε]`. Simpler, ≥ as good empirically.
- **DDPG core trick**: replace `maxₐ Q(s,a)` (intractable in continuous `a`) by `Q(s, μ(s))` and train `μ` by gradient ascent through the differentiable `Q`. **Polyak averaging** for target nets: `φtarg ← ρ φtarg + (1−ρ)φ`.

---

## 0. Course context

- Dr. **Nadir Farhi** — researcher at Université Gustave Eiffel (Paris). Habilitation in Math 2018 (Paris-Est), PhD Math 2008 (Paris 1 Sorbonne). Topics: RL + optimal control, applied to mobility, transportation networks, urban management.
- **Outline of the lecture**:
  1. Review & summary (DP, MC, Q-learning convergence, Vanilla DQN).
  2. Policy-based RL: **REINFORCE (VPG) → TRPO → PPO**, then **DDPG**.

---

## 1. RL context, optimization & review

### 1.1. The RL optimization problem

- Underlying object: a **Markov Decision Process** with states `s` and actions `a`.
- Optimization is **in time** ⇒ optimal control framework.
- **Under uncertainty** ⇒ stochasticity. The criterion is

```math
\max_{\pi}\; \mathbb{E}_\pi \sum_{k=0}^{\infty} \gamma^{k}\, R_{t+k+1}(s_k, a_k)
```

- Dynamics `p(s', r | s, a)` may be **known** (a priori) or not:

| Setting | Tool |
|---|---|
| Dynamics known | **Dynamic Programming** |
| Dynamics unknown | **Monte-Carlo sampling** (simulation) |
| Mixed | **Reinforcement Learning** = DP + MC ideas |

- `π` is the policy: a probability distribution on the action set `𝒜`.

### 1.2. Dynamic Programming — recap

Action-value function:

```math
q_\pi(s,a) \;\doteq\; \mathbb{E}_\pi[G_t \mid S_t=s, A_t=a] \;=\; \mathbb{E}_\pi\!\left[\sum_{k=0}^{\infty} \gamma^{k} R_{t+k+1} \,\bigg|\, S_t=s, A_t=a \right]
```

Bellman optimality equation:

```math
q_*(s,a) \;=\; \mathbb{E}\!\left[R_{t+1} + \gamma \max_{a'} q_*(S_{t+1},a') \,\bigg|\, S_t=s, A_t=a\right] \;=\; \sum_{s',r} p(s',r\mid s,a)\Big[r + \gamma \max_{a'} q_*(s',a')\Big]
```

Four DP building blocks:
1. Policy evaluation (prediction)
2. Policy improvement
3. Policy iteration + GPI
4. Value iteration

### 1.3. Monte-Carlo — recap

- Agent **interacts**, collects experience (no complete model needed).
- Estimates state-/action-value by **averaging returns**.
- Convergence: empirical mean → expected value.

---

## 2. Q-learning (Watkins, 1989) — off-policy TD control

Off-policy uses **two policies**:

| Policy | Role |
|---|---|
| **Target policy** | the policy being learned → `π*` |
| **Behavior policy** | exploratory, used to generate experience |

### 2.1. The algorithm

```
Algorithm: Q-learning (off-policy TD control) for π ≈ π*
Parameters: step size α ∈ (0,1], small ε > 0
Initialize Q(s,a) for all s ∈ S⁺, a ∈ A(s), arbitrarily, except Q(terminal,·) = 0
Loop for each episode:
    Initialize S
    Loop for each step:
        Choose A from S using a policy derived from Q (e.g. ε-greedy)
        Take action A, observe R, S'
        Q(S,A) ← Q(S,A) + α [ R + γ max_a Q(S',a) − Q(S,A) ]
        S ← S'
    until S is terminal
```

### 2.2. Convergence of the Q-factor (operator view)

Optimal action-value (max-over-strategies form):

```math
q(s,a) := \max_{\text{strat.}} \mathbb{E}\!\left\{ \sum_{k=0}^{\infty} \gamma^{k} g_{s_k}^{a_k} \,\bigg|\, S^0=s, A^0=a \right\}
```

Bellman equation:

```math
q^*(s,a) = \sum_{s'} p(s' \mid s,a)\Big( g_{ss'}^{a} + \gamma \max_{a'} q^*(s',a') \Big)
```

DP operator `F : q ↦ F(q)`:

```math
F(q)(s,a) = \sum_{s'} p(s' \mid s,a)\Big( g_{ss'}^{a} + \gamma \max_{a'} q(s', a') \Big)
```

- `F` is a **contraction** in `ℝⁿˣᵐ` (n = #states, m = #actions).
- ⇒ value iteration converges to `q*` (it is the unique fixed point of `F`).

### 2.3. Convergence of the Q-learning algorithm

Update:

```math
Q(S,A) \;\leftarrow\; Q(S,A) + \alpha \Big( R + \gamma \max_{a'} Q(S',a') - Q(S,A) \Big)
```

**Conditions for convergence**:
1. Pairs `(s,a)` must be generated **infinitely often**.
2. Successor `s'` must be **independently sampled** from the kernel.
3. The step size satisfies the **Robbins-Monro conditions**:

```math
\alpha^k > 0,\quad \forall k,\quad \sum_{k=0}^{+\infty} \alpha_k = \infty,\quad \sum_{k=0}^{+\infty} (\alpha_k)^2 < \infty
```

Example: `αₖ = c₁ / (k + c₂)`.

### 2.4. Sketch of the proof (4 slides)

**Step 1.** The optimal `Q*` is a fixed point of the contraction operator `H` defined by

```math
(Hq)(x,a) = \sum_{y \in \mathcal{X}} P_a(x,y)\big[ r(x,a,y) + \gamma \max_{b \in \mathcal{A}} q(y,b) \big]
```

`H` is a contraction in the sup-norm: `‖Hq₁ − Hq₂‖∞ ≤ γ ‖q₁ − q₂‖∞` (Eq. 1).

**Step 2.** Theorem 1 (main result): Given a finite MDP `(𝒳, 𝒜, P, r)`, the Q-learning update

```math
Q_{t+1}(x_t, a_t) = Q_t(x_t, a_t) + \alpha_t(x_t, a_t)\big[ r_t + \gamma \max_{b \in \mathcal{A}} Q_t(x_{t+1}, b) - Q_t(x_t, a_t) \big]
```

converges w.p. 1 to `Q*` provided `Σₜ αₜ(x,a) = ∞` and `Σₜ αₜ²(x,a) < ∞` for all `(x,a) ∈ 𝒳 × 𝒜`. Since `0 ≤ αₜ < 1`, this implies all `(x,a)` are visited infinitely often.

**Step 3.** Rewrite the update as a convex combination, subtract `Q*` to get `Δₜ = Qₜ − Q*`. Define

```math
F_t(x,a) = r(x, a, X(x,a)) + \gamma \max_{b \in \mathcal{A}} Q_t(y,b) - Q^*(x,a)
```

**Step 4.** Show

```math
\|\mathbb{E}[F_t(x,a)\mid \mathcal{F}_t]\|_\infty \leq \gamma \|Q_t - Q^*\|_\infty = \gamma \|\Delta_t\|_\infty
```
```math
\mathrm{var}[F_t(x)\mid \mathcal{F}_t] \leq C(1 + \|\Delta_t\|_W^2)
```

Intermediate result (Theorem 2): a process `Δₜ₊₁ = (1 − αₜ)Δₜ + αₜ Fₜ` with the same two hypotheses converges to 0 w.p. 1 if `0 ≤ αₜ ≤ 1`, the standard Robbins-Monro conditions hold, the conditional expectation is a `γ`-contraction (`γ < 1`), and the conditional variance is bounded as above. Apply Theorem 2 to finish.

---

## 3. Vanilla DQN (Mnih et al., 2015)

DQN extends Q-learning to neural networks. Three issues, three fixes:

| Problem with NN-based Q | Fix in DQN |
|---|---|
| Function approximator (no convergence proof of tabular Q-learning) | **Introduce a NN** Qθ |
| **Moving targets** (target uses same network being updated) | **Target network** `Q̂ = Q(·;θ⁻)`, copied every `C` steps |
| **Successively correlated data** (online TD targets are correlated) | **Experience replay buffer** `D`, sample mini-batch U(D) |

### 3.1. The Vanilla DQN algorithm

```
Vanilla DQN algorithm
Initialize step t = 0
Initialize the online Q-network Q with random weights θ_t
Initialize the target Q-network with weights θ_t⁻ = θ_t
Initialize the replay memory buffer D to capacity N with Nmin random transitions
for episode e = 1:E do
    Initialize sequence: observe initial state sₜ = φ(xₜ)
    while sₜ is not terminal do
        With probability ε select a random action aₜ ∈ A
        otherwise select aₜ = argmaxₐ Qθt(sₜ, a; θₜ)
        Execute aₜ in emulator x; observe reward rₜ₊₁ and next state sₜ₊₁ = φ(xₜ₊₁)
        Store transition (sₜ, aₜ, rₜ₊₁, sₜ₊₁) in D
        Sample random batch U(D) of M transitions (s, a, s', r')^(m) in D
        for m = 1:M do
            Set TD error  tₜ^(m) = r'^(m) + γ · maxₐ' Qθt⁻(s'^(m), a'; θₜ⁻) − Qθt(s^(m), a^(m); θₜ)
        end for
        Perform a gradient descent step on MSE loss Lt(θt) w.r.t. θt with Δt, α
        if t ≡ 0 (mod C) then
            Reset target Q-network Q̂ = online Q-network Q; θₜ⁻ = θₜ
        end if
        Decay ε with linear decay; increment step t = t+1
    end while
end for
```

---

## 4. Policy-based methods — overview

### 4.1. Value-based vs policy-based

- **Value-based**: learn `q(s,a)`, derive `π` (e.g. ε-greedy). The policy **doesn't exist without the action-value estimates**.
- **Policy-based / Policy Gradient**: learn a **parameterized policy** `π(a | s, θ) = P(Aₜ = a | Sₜ = s, θₜ = θ)` directly. **No value function needed** in principle.
- Optimize a performance measure `J(θ)` by **gradient ascent**:

```math
\theta_{t+1} = \theta_t + \alpha\, \widehat{\nabla} J(\theta_t)
```

`∇̂J(θₜ)` is a stochastic estimate of the true gradient of `J` w.r.t. `θ`.

### 4.2. The taxonomy used in this lecture

| Family | Members | Key trait |
|---|---|---|
| **Stochastic PG** | REINFORCE (= VPG) → TRPO → PPO → GRPO | learn `π(a|s, θ)` distribution |
| **Deterministic PG** | DPG → DDPG → TD3 | learn `μ(s)` |
| **Hybrid (actor-critic)** | A2C → A3C → SAC | actor + critic |

### 4.3. Policy approximation

- Keep policy stochastic for exploration: `π(a | s, θ) ∈ (0,1) ∀ s, a, θ`.
- Handles **continuous action spaces** (value methods can't).
- Discrete & not-too-large action space: parameterized preferences `h(s, a, θ)` + softmax:

```math
\pi(a \mid s, \theta) := \frac{e^{h(s,a,\theta)}}{\sum_b e^{h(s,b,\theta)}}
```

- `h(s,a,θ)` can be linear: `h(s,a,θ) := θᵀ x(s,a)` with feature vector `x(s,a)`; or a **deep ANN** (θ = all weights).

### 4.4. Advantages of policy approximation

1. Approach a **deterministic** policy in the limit (as preferences grow apart).
2. Enable selection of actions with **arbitrary probabilities** (true randomness needed in some games).
3. Action-value methods have **no natural way of finding stochastic optimal policies** — policy approximation does.
4. **Continuous policy parameterization** ⇒ action probabilities change smoothly with `θ`.
5. **Stronger convergence** than action-value methods (smooth dependence).

---

## 5. The Policy Gradient Theorem

### 5.1. Performance measure

Episodic case — value of the start state:

```math
J(\theta) := v_{\pi_\theta}(s_0)
```

### 5.2. Theorem (episodic case)

```math
\nabla J(\theta) \;\propto\; \sum_s \mu(s) \sum_a q_\pi(s,a)\, \nabla \pi(a \mid s, \theta)
```

- `∝` = proportional to.
- **Episodic** case: constant of proportionality = average length of an episode.
- **Continuing** case: constant of proportionality = 1 (true equality).
- `μ` is the **on-policy state distribution** under `π`.

### 5.3. Building `μ(s)`

- `h(s)` = probability that an episode begins in state `s`.
- `η(s)` = expected number of time steps spent in state `s` during one episode. It solves the system

```math
\eta(s) = h(s) + \sum_{\bar s} \eta(\bar s) \sum_a \pi(a \mid \bar s)\, p(s \mid \bar s, a), \quad \forall s \in \mathcal{S}
```

- Then `μ` is the normalized `η`:

```math
\mu(s) = \frac{\eta(s)}{\sum_{s'} \eta(s')},\quad \forall s \in \mathcal{S}
```

---

## 6. REINFORCE (Monte-Carlo Policy Gradient, Williams 1992)

### 6.1. Derivation — from sum to expectation

Starting from the PG theorem,

```math
\nabla J(\theta) \;\propto\; \sum_s \mu(s) \sum_a q_\pi(s,a)\, \nabla \pi(a \mid s, \theta)
```

If `π` is followed, states occur with proportions `μ(s)`, so

```math
\nabla J(\theta) \;\propto\; \mathbb{E}_\pi\!\left[ \sum_a q_\pi(S_t, a)\, \nabla \pi(a \mid S_t, \theta) \right]
```

**All-actions** algorithm (uses estimate `q̂`):

```math
\theta_{t+1} \doteq \theta_t + \alpha \sum_a \hat q(S_t, a, \mathbf{w})\, \nabla \pi(a \mid S_t, \theta)
```

Introduce `Aₜ` the same way `Sₜ` was introduced, replace the sum over `a` by an expectation under `π`, then sample:

```math
\nabla J(\theta) \;\propto\; \mathbb{E}_\pi\!\left[ \sum_a \pi(a \mid S_t, \theta)\, q_\pi(S_t, a)\, \frac{\nabla \pi(a \mid S_t, \theta)}{\pi(a \mid S_t, \theta)} \right]
\;=\; \mathbb{E}_\pi\!\left[ q_\pi(S_t, A_t)\, \frac{\nabla \pi(A_t \mid S_t, \theta)}{\pi(A_t \mid S_t, \theta)} \right]
\;=\; \mathbb{E}_\pi\!\left[ G_t\, \frac{\nabla \pi(A_t \mid S_t, \theta)}{\pi(A_t \mid S_t, \theta)} \right]
```

The last step uses `𝔼π[Gₜ | Sₜ, Aₜ] = qπ(Sₜ, Aₜ)`.

### 6.2. REINFORCE algorithm

```
REINFORCE: Monte-Carlo Policy-Gradient Control (episodic) for π*
Input: a differentiable policy parameterization π(a | s, θ)
Algorithm parameter: step size α > 0
Initialize policy parameter θ ∈ ℝᵈ' (e.g. to 0)
Loop forever (for each episode):
    Generate an episode S₀, A₀, R₁, ..., S_{T−1}, A_{T−1}, R_T  following π(·|·, θ)
    Loop for each step of the episode t = 0, 1, ..., T−1:
        G ← Σ_{k=t+1}^T γ^{k−t−1} R_k          (Gt)
        θ ← θ + α γ^t G ∇ ln π(Aₜ | Sₜ, θ)
```

Interpretation of the update: **proportional to the return** (push toward actions that gave high return) and **inversely proportional to `π(Aₜ|Sₜ,θ)`** (otherwise actions selected frequently would be at an unfair advantage). Equivalent to ascending `∇ ln π`.

### 6.3. Empirical behaviour

On the **short-corridor gridworld** (from Sutton-Barto), with three step sizes `α ∈ {2⁻¹², 2⁻¹³, 2⁻¹⁴}` averaged over 100 runs across 1000 episodes: returns climb toward the optimum `v*(s₀) ≈ −11.6`; bigger step sizes converge faster but more noisily, `2⁻¹²` plateaus around −40 (too aggressive), `2⁻¹³` ≈ optimum.

---

## 7. REINFORCE with baseline

### 7.1. The baseline

Generalize the PG theorem with an **arbitrary baseline** `b(s)`:

```math
\nabla J(\theta) \;\propto\; \sum_s \mu(s) \sum_a \big(q_\pi(s,a) - b(s)\big)\, \nabla \pi(a \mid s, \theta)
```

This is valid as long as `b` does not depend on `a`:

```math
\sum_a b(s)\, \nabla \pi(a \mid s, \theta) \;=\; b(s)\, \nabla \sum_a \pi(a \mid s, \theta) \;=\; b(s)\, \nabla 1 \;=\; 0
```

So the baseline leaves the expected update unchanged — but can drastically **reduce its variance**.

New update rule:

```math
\theta_{t+1} \doteq \theta_t + \alpha\, \big(G_t - b(S_t)\big)\, \frac{\nabla \pi(A_t \mid S_t, \theta_t)}{\pi(A_t \mid S_t, \theta_t)}
```

### 7.2. Natural choice: state-value baseline

One natural choice: `b(s) = v̂(Sₜ, w)`, an estimate of the state value. Learn `w` by another Monte-Carlo method.

### 7.3. Algorithm

```
REINFORCE with Baseline (episodic), for estimating π_θ ≈ π*
Input: a differentiable policy parameterization π(a | s, θ)
Input: a differentiable state-value function parameterization v̂(s, w)
Algorithm parameters: step sizes α^θ > 0, α^w > 0
Initialize policy parameter θ ∈ ℝᵈ' and state-value weights w ∈ ℝᵈ (e.g. to 0)
Loop forever (for each episode):
    Generate an episode S₀, A₀, R₁, ..., S_{T−1}, A_{T−1}, R_T  following π(·|·, θ)
    Loop for each step of the episode t = 0, 1, ..., T−1:
        G ← Σ_{k=t+1}^T γ^{k−t−1} R_k          (Gt)
        δ ← G − v̂(Sₜ, w)
        w ← w + α^w δ ∇v̂(Sₜ, w)
        θ ← θ + α^θ γ^t δ ∇ ln π(Aₜ | Sₜ, θ)
```

### 7.4. Empirical comparison

On the short-corridor gridworld: REINFORCE with baseline (with `αw = 2⁻⁶`, `αθ = 2⁻⁹`) **converges much faster** than plain REINFORCE (with `α = 2⁻¹³`) and reaches near-optimal performance in a few hundred episodes vs. ~1000.

### 7.5. OpenAI's "Vanilla Policy Gradient" (VPG) algorithm

```
Algorithm 1  Vanilla Policy Gradient Algorithm
1: Input: initial policy parameters θ₀, initial value function parameters φ₀
2: for k = 0, 1, 2, ... do
3:     Collect set of trajectories D_k = {τ_i} by running policy π_k = π(θ_k) in the environment
4:     Compute rewards-to-go R̂_t
5:     Compute advantage estimates Â_t (using any method of advantage estimation) based on the current value function V_{φ_k}
6:     Estimate policy gradient
          ĝ_k = (1/|D_k|) Σ_{τ ∈ D_k} Σ_{t=0}^T ∇_θ log π_θ(a_t|s_t)|_{θ_k} Â_t
7:     Compute policy update either using standard gradient ascent
          θ_{k+1} = θ_k + α_k ĝ_k
       or via another gradient ascent algorithm like Adam
8:     Fit value function by regression on mean-squared error
          φ_{k+1} = argmin_φ (1/(|D_k| T)) Σ Σ (V_φ(s_t) − R̂_t)²
       typically via some gradient descent algorithm
9: end for
```

---

## 8. TRPO — Trust Region Policy Optimization

### 8.1. Why TRPO? Two problems of REINFORCE

| Problem | Detail |
|---|---|
| **Past trajectories not reused** | Each `θ ← θ + α ∇J(θ)` step uses fresh on-policy data only. Wasteful. |
| **Unstable convergence** | Hard to find an `α` that works for the whole training; small parameter changes can mean *large* policy changes — and crashes. |

### 8.2. Abstract idea

- REINFORCE keeps new and old policies **close in parameter space**, but they can have **large differences in performance**.
- So large step sizes are **dangerous**.
- **TRPO** updates policies by taking the **largest step possible** to improve performance, *while satisfying a constraint on how close the new and old policies are*.
- Distance is measured by **KL-divergence** (Kullback-Leibler relative entropy).

### 8.3. Quick facts

| Property | Value |
|---|---|
| On-/off-policy | **On-policy** |
| Action space | Discrete **or** continuous |
| Uses Hessian | **Yes** (2nd derivatives + gradient) |
| Parallelizable | **Yes** |

### 8.4. Theoretical update

`πθ`: parameterized policy. The **theoretical TRPO update** is

```math
\theta_{k+1} = \arg\max_\theta \mathcal{L}(\theta_k, \theta) \quad \text{s.t.}\quad \bar D_{KL}(\theta \,\|\, \theta_k) \leq \delta
```

`𝓛(θₖ, θ)` is the **surrogate advantage** — how `πθ` performs relative to `πθₖ` using data from `πθₖ`:

```math
\mathcal{L}(\theta_k, \theta) = \mathbb{E}_{s, a \sim \pi_{\theta_k}}\!\left[ \frac{\pi_\theta(a \mid s)}{\pi_{\theta_k}(a \mid s)} A^{\pi_{\theta_k}}(s, a) \right]
```

`D̄KL(θ ‖ θₖ)` is the **average KL** across states visited by the old policy:

```math
D_{KL}(\theta \,\|\, \theta_k) = \mathbb{E}_{s \sim \pi_{\theta_k}}\!\big[ D_{KL}\!\big( \pi_\theta(\cdot \mid s) \,\|\, \pi_{\theta_k}(\cdot \mid s) \big) \big]
```

### 8.5. The MM (Minorize-Maximization) intuition

`η(πθ) = 𝔼{τ∼πθ}[Σ γᵗ rₜ]` is the true performance. TRPO and PPO use **MM**: at each iteration, find a surrogate `M(θ) = L(θ) − C·KL(θ)` (the **blue curve**) that

1. is a **lower bound** to `η(θ)`,
2. **approximates** `η` at the current policy `θᵢ`,
3. is **easy to optimize** (e.g. quadratic).

Then optimize `M`, move to its optimum `θᵢ₊₁`, repeat. The blue surrogate `M` is *always below* `η`, so each step of MM **guarantees** improvement (or stays).

### 8.6. KL divergence (refresher)

```math
KL(P \,\|\, Q) := \sum_x P(x) \log \frac{P(x)}{Q(x)} = \sum_x P(x)\log P(x) - \sum_x P(x) \log Q(x)
```

- `Σ P log P` = **entropy** `H(P)`.
- `Σ P log Q` = **cross-entropy** between `P` and `Q`.

### 8.7. Solving the trust-region problem

Taylor expand the objective and constraint around `θold`:

```math
\mathcal{L}_{\theta_{old}}(\theta) \approx \underbrace{\mathcal{L}_{\theta_{old}}(\theta_{old})}_{0} + \nabla_\theta \mathcal{L}_{\theta_{old}}(\theta)\big|_{\theta_{old}} (\theta - \theta_{old}) + \cdots
```
```math
\bar D_{KL}(\theta \,\|\, \theta_{old}) \approx \underbrace{\bar D_{KL}(\theta_k \,\|\, \theta_{old})}_{0} + \underbrace{\nabla_\theta \bar D_{KL}(\theta \,\|\, \theta_{old})\big|_{\theta_{old}}}_{0}(\theta - \theta_{old}) + \tfrac{1}{2}(\theta - \theta_{old})^T \nabla_\theta^2 \bar D_{KL}(\theta \,\|\, \theta_{old})\big|_{\theta_{old}} (\theta - \theta_{old}) + \cdots
```

Reduces the trust-region program to a **QCQP**:

```math
\theta_{k+1} = \arg\max_\theta g^T (\theta - \theta_k) \quad \text{s.t.}\quad \tfrac{1}{2}(\theta - \theta_k)^T H (\theta - \theta_k) \leq \delta
```

with

```math
g \doteq \nabla_\theta \mathcal{L}_{\theta_{old}}(\theta)\big|_{\theta_{old}}, \qquad H \doteq \nabla_\theta^2 \bar D_{KL}(\theta \,\|\, \theta_{old})\big|_{\theta_{old}}
```

### 8.8. Analytic solution + backtracking

Via Lagrangian duality:

```math
\theta_{k+1} = \theta_k + \sqrt{\frac{2\delta}{g^T H^{-1} g}}\; H^{-1} g
```

Add a **backtracking line search** to guarantee the original (not approximated) KL constraint holds:

```math
\theta_{k+1} = \theta_k + \alpha^j \sqrt{\frac{2\delta}{g^T H^{-1} g}}\; H^{-1} g
```

- `α^j` = backtracking coefficient.
- `j` = smallest non-negative integer such that `πθₖ₊₁` satisfies the KL constraint **and** produces a positive surrogate advantage.

### 8.9. Avoiding `H⁻¹` — conjugate gradient

Computing and storing `H⁻¹` is expensive. TRPO uses the **conjugate gradient** algorithm to solve `Hx = g` for `x = H⁻¹g`. Set up a symbolic Hessian-vector product:

```math
Hx = \nabla_\theta\!\left( \big(\nabla_\theta \bar D_{KL}(\theta \,\|\, \theta_k)\big)^T x \right)
```

No need to ever form `H` explicitly.

### 8.10. Full TRPO algorithm

```
Algorithm 1  Trust Region Policy Optimization
1: Input: initial policy parameters θ₀, initial value function parameters φ₀
2: Hyperparameters: KL-divergence limit δ, backtracking coefficient α, max number of backtracking steps K
3: for k = 0, 1, 2, ... do
4:     Collect set of trajectories D_k = {τ_i} by running policy π_k = π(θ_k) in the environment
5:     Compute rewards-to-go R̂_t
6:     Compute advantage estimates Â_t (using any method of advantage estimation) based on the current value function V_{φ_k}
7:     Estimate policy gradient
          ĝ_k = (1/|D_k|) Σ Σ ∇_θ log π_θ(a_t|s_t)|_{θ_k} Â_t
8:     Use the conjugate gradient algorithm to compute
          x̂_k ≈ Ĥ_k^{-1} ĝ_k
       where Ĥ_k is the Hessian of the sample average KL-divergence
9:     Update the policy by backtracking line search with
          θ_{k+1} = θ_k + α^j √(2δ / (x̂_k^T Ĥ_k x̂_k)) x̂_k
       where j ∈ {0, 1, 2, ..., K} is the smallest value which improves the sample loss
       and satisfies the sample KL-divergence constraint
10:    Fit value function by regression on mean-squared error
          φ_{k+1} = argmin_φ (1/(|D_k| T)) Σ Σ (V_φ(s_t) − R̂_t)²
       typically via some gradient descent algorithm
11: end for
```

---

## 9. PPO — Proximal Policy Optimization

### 9.1. Motivation

- **Same question as TRPO**: how can we take the biggest possible improvement step on a policy, using the data we currently have, without stepping so far that we accidentally cause performance collapse?
- TRPO solves it with a **complex 2nd-order** method (Hessian, conjugate gradient, line search).
- **PPO** = a family of **1st-order** methods that use other tricks. Significantly simpler to implement. **Empirically perform at least as well as TRPO**.

### 9.2. Two variants

| Variant | Mechanism |
|---|---|
| **PPO-Penalty** | Approximately solve a KL-constrained update like TRPO, but penalize KL in the objective instead of a hard constraint. Automatically adjusts the penalty coefficient over training. |
| **PPO-Clip** | No KL term at all. No constraint. Uses **specialized clipping** in the objective to remove incentives for the new policy to get far from the old. |

### 9.3. PPO-Penalty algorithm

```
Algorithm 4  PPO with Adaptive KL Penalty
Input: initial policy parameters θ₀, initial KL penalty β₀, target KL-divergence δ
for k = 0, 1, 2, ... do
    Collect set of partial trajectories D_k on policy π_k = π(θ_k)
    Estimate advantages Â_t^{π_k} using any advantage estimation algorithm
    Compute policy update
        θ_{k+1} = argmax_θ  𝓛_{θ_k}(θ) − β_k D̄_KL(θ ‖ θ_k)
    by taking K steps of minibatch SGD (via Adam)
    if D̄_KL(θ_{k+1} ‖ θ_k) ≥ 1.5 δ then
        β_{k+1} = 2 β_k
    else if D̄_KL(θ_{k+1} ‖ θ_k) ≤ δ / 1.5 then
        β_{k+1} = β_k / 2
    end if
end for
```

### 9.4. PPO-Clip objective

PPO-Clip takes multiple steps of SGD to maximize the objective:

```math
\theta_{k+1} = \arg\max_\theta \mathbb{E}_{s, a \sim \pi_{\theta_k}}\!\big[ L(s, a, \theta_k, \theta) \big]
```

where

```math
L(s, a, \theta_k, \theta) = \min\!\left( \frac{\pi_\theta(a \mid s)}{\pi_{\theta_k}(a \mid s)} A^{\pi_{\theta_k}}(s, a),\; \mathrm{clip}\!\left(\frac{\pi_\theta(a \mid s)}{\pi_{\theta_k}(a \mid s)}, 1-\epsilon, 1+\epsilon\right) A^{\pi_{\theta_k}}(s, a) \right)
```

### 9.5. Simplified (OpenAI) version

```math
L(s, a, \theta_k, \theta) = \min\!\left( \frac{\pi_\theta(a \mid s)}{\pi_{\theta_k}(a \mid s)} A^{\pi_{\theta_k}}(s, a),\; g(\epsilon, A^{\pi_{\theta_k}}(s, a)) \right)
```

```math
g(\epsilon, A) = \begin{cases}(1 + \epsilon) A & A \geq 0 \\ (1 - \epsilon) A & A < 0\end{cases}
```

**Intuition**:
- If `A ≥ 0` (good action): cap the upside at `(1+ε)A` — no reward for pushing ratio above `1+ε`.
- If `A < 0` (bad action): cap the downside at `(1−ε)A` — no incentive to push ratio below `1−ε`.

### 9.6. PPO-Clip algorithm

```
Algorithm 1  PPO-Clip
1: Input: initial policy parameters θ₀, initial value function parameters φ₀
2: for k = 0, 1, 2, ... do
3:     Collect set of trajectories D_k = {τ_i} by running policy π_k = π(θ_k) in the environment
4:     Compute rewards-to-go R̂_t
5:     Compute advantage estimates Â_t (using any method of advantage estimation) based on the current value function V_{φ_k}
6:     Update the policy by maximizing the PPO-Clip objective:
          θ_{k+1} = argmax_θ (1/|D_k| T) Σ Σ min( π_θ(a_t|s_t)/π_{θ_k}(a_t|s_t) A^{π_k}(s_t, a_t),  g(ε, A^{π_k}(s_t, a_t)) )
       typically via stochastic gradient ascent with Adam
7:     Fit value function by regression on mean-squared error
          φ_{k+1} = argmin_φ (1/|D_k| T) Σ Σ (V_φ(s_t) − R̂_t)²
       typically via some gradient descent algorithm
8: end for
```

---

## 10. GRPO — Group Relative Policy Optimization

- **GRPO** = the RL algorithm used to train **DeepSeek**. Reference: Shao et al., *DeepSeekMath: Pushing the Limits of Mathematical Reasoning in Open Language Models* (2024). arXiv 2402.03300.
- Helps a model **learn better by comparing different actions** (no separate value baseline).
- Plot in slides: DeepSeekMath-7B jumps from ~30 % to ~52 % MATH Top@1 accuracy, leapfrogging LLaMA1-65B, WizardMath-70B, Qwen-72B, etc., to land at GPT-4-API performance.

### 10.1. GRPO principle

For current policy `πθ` and a given state `s`:

1. **Generate** a group of `N` sampled actions `a₁, a₂, …, a_N` using `πθ`.
2. **Evaluate** each `aᵢ` with a reward function `R(aᵢ)` (immediate or discounted).
3. **Calculate the advantage** of each `aᵢ` w.r.t. the **other actions in the group**:

```math
A(a_i) := R(a_i) - \bar R,\quad \text{where } \bar R := \frac{1}{N}\sum_{i=1}^{N} R(a_i)
```

4. **Policy update**: update `θ` to
   - **increase** the likelihood of actions with **positive** advantage,
   - **decrease** the likelihood of actions with **negative** advantage.

5. **KL-divergence constraint**: ensure the updated policy does **not deviate too much** from the old policy (stability — analogous to TRPO/PPO).

### 10.2. GRPO for LLM training

| Step | What happens |
|---|---|
| **Group sampling** | For a given prompt, the LLM generates multiple responses. |
| **Reward scoring** | A reward model evaluates each response. |
| **Advantage calculation** | Responses compared to the group's average reward to determine which are better/worse. |
| **Policy update** | LLM policy adjusted to favor high-reward responses while avoiding drastic changes via a **KL-divergence constraint**. |
| **Iterative training** | Repeats — gradually improves the LLM's ability to generate high-quality, aligned text. |

---

## 11. DDPG — Deep Deterministic Policy Gradient

### 11.1. Quick facts

| Property | Value |
|---|---|
| Policy type | **Deterministic** `μ(s)` |
| On-/off-policy | **Off-policy** |
| Action space | **Continuous only** |
| Analogy | "**Deep Q-learning for continuous action spaces**" |
| Parallelization | **Does not support** parallelization |

**Abstract**: DDPG learns a Q-function `Qφ(s,a)` and a policy `μθ(s)`. Uses off-policy data + the Bellman equation to learn the Q-function, then uses the Q-function to learn the policy.

### 11.2. The argmax problem in continuous spaces

We need `a*(s) = argmaxₐ Q*(s, a)`.

| Action space | `argmaxₐ Q(s,a)` |
|---|---|
| **Discrete** | enumeration through the `Q` table |
| **Continuous** | ❓ — full optimization at every action selection is **prohibitively expensive** |

**DDPG's trick**:
1. `Q*(s, a)` is presumed **differentiable** w.r.t. `a`.
2. Use a **gradient-based learning rule** for a policy `μ(s)`.
3. Approximate `maxₐ Q(s, a) ≈ Q(s, μ(s))`.

### 11.3. Q-Learning side of DDPG (the "critic")

Minimize the **MSBE** (mean-squared Bellman error) loss via stochastic gradient descent:

```math
L(\phi, \mathcal{D}) = \mathbb{E}_{(s, a, r, s', d) \sim \mathcal{D}}\!\left[ \Big( Q_\phi(s, a) - \big(r + \gamma (1 - d)\, Q_{\phi_{targ}}(s', \mu_{\theta_{targ}}(s'))\big) \Big)^2 \right]
```

where `μθtarg` is the **target policy** and `d` indicates a terminal transition.

### 11.4. Policy Learning side of DDPG (the "actor")

- Learn a **deterministic** policy `μθ(s)` maximizing `Qφ(s, a)`.
- Perform **gradient ascent** w.r.t. policy parameters only:

```math
\max_\theta \mathbb{E}_{s \sim \mathcal{D}}\!\big[ Q_\phi(s, \mu_\theta(s)) \big]
```

The Q-function parameters are **treated as constants** here.

**Exploration tricks**:
- Add **noise** to actions at training time (deterministic policy → no built-in exploration).
- **Uncorrelated, mean-zero Gaussian noise** works perfectly well in practice.
- **Reduce the noise scale** over the course of training.

### 11.5. Polyak averaging for target networks

| Algorithm | Target network update |
|---|---|
| **DQN** | Target net **copied every fixed number of steps** (hard update) |
| **DDPG** | Target net updated **once per main-network update** (soft update): |

```math
\phi_{targ} \leftarrow \rho\, \phi_{targ} + (1 - \rho)\, \phi
```

with `0 ≤ ρ ≤ 1`, usually **close to 1** (slow, smooth tracking).

### 11.6. Full DDPG algorithm

```
Algorithm 1  Deep Deterministic Policy Gradient
1: Input: initial policy parameters θ, Q-function parameters φ, empty replay buffer D
2: Set target parameters equal to main parameters: θ_targ ← θ, φ_targ ← φ
3: repeat
4:     Observe state s and select action a = clip(μ_θ(s) + ε, a_low, a_high), where ε ∼ N
5:     Execute a in the environment
6:     Observe next state s', reward r, and done signal d to indicate whether s' is terminal
7:     Store (s, a, r, s', d) in replay buffer D
8:     If s' is terminal, reset environment state
9:     if it's time to update then
10:        for however many updates do
11:            Randomly sample a batch of transitions B = {(s, a, r, s', d)} from D
12:            Compute targets
                  y(r, s', d) = r + γ(1 − d) Q_{φ_targ}(s', μ_{θ_targ}(s'))
13:            Update Q-function by one step of gradient descent using
                  ∇_φ (1/|B|) Σ (Q_φ(s, a) − y(r, s', d))²
14:            Update policy by one step of gradient ascent using
                  ∇_θ (1/|B|) Σ Q_φ(s, μ_θ(s))
15:            Update target networks with
                  φ_targ ← ρ φ_targ + (1 − ρ) φ
                  θ_targ ← ρ θ_targ + (1 − ρ) θ
16:        end for
17:    end if
18: until convergence
```

---

## Key terms (glossary)

- **Policy gradient theorem** — gives `∇J(θ) ∝ Σ μ(s) Σ q(s,a) ∇π(a|s,θ)`, the foundation of every algorithm in this lecture.
- **REINFORCE** — Monte-Carlo policy gradient (Williams 1992); update `θ ← θ + α γᵗ G ∇ ln π(Aₜ|Sₜ,θ)`.
- **Baseline `b(s)`** — any state-dependent function subtracted from `Gₜ`; preserves expectation, reduces variance.
- **VPG (Vanilla Policy Gradient)** — OpenAI's name for REINFORCE-with-baseline + advantage estimation.
- **TRPO** — Trust Region Policy Optimization (Schulman 2015); maximize `𝓛` under `KL(πθ ‖ πθₖ) ≤ δ`; 2nd-order via Hessian + conjugate gradient + line search.
- **Surrogate advantage `𝓛(θₖ,θ)`** — importance-sampled expected advantage `𝔼[πθ/πθₖ · Aπθₖ]`.
- **KL divergence** — `KL(P‖Q) = Σ P log(P/Q) = H(P, Q) − H(P)`.
- **MM algorithm (Minorize-Maximization)** — at each iter, optimize a lower-bound surrogate `M(θ) = L(θ) − C·KL(θ)`; guarantees monotone improvement.
- **Conjugate gradient (CG)** — iterative method to solve `Hx = g` without forming `H⁻¹`.
- **PPO** — Proximal Policy Optimization; first-order successor of TRPO. Two flavours: Penalty (adaptive β·KL) and Clip (cap probability ratio).
- **Probability ratio `r(θ) = πθ/πθₖ`** — central object in PPO-Clip; clipped to `[1−ε, 1+ε]`.
- **GRPO** — Group Relative Policy Optimization (DeepSeek 2024); replaces a value baseline by the group mean reward `R̄`.
- **DPG / DDPG / TD3** — deterministic policy gradient family; DDPG = continuous-action deep Q-learning.
- **Target network** — slowly tracking copy of online weights, stabilizes bootstrap targets.
- **Polyak averaging** — soft update `φtarg ← ρ φtarg + (1−ρ)φ`.
- **Experience replay** — buffer of past transitions, decorrelates training data and reuses experience.
- **MSBE loss** — Mean-Squared Bellman Error: `(Qφ(s,a) − [r + γ(1−d) Qφtarg(s', μθtarg(s'))])²`.
- **Robbins-Monro conditions** — `Σ αₖ = ∞, Σ αₖ² < ∞`; convergence prerequisite for Q-learning.
- **Stochastic vs deterministic policy** — `π(a|s,θ)` distribution vs `μ(s)` single action.

---

## Exam targets (likely written-exam questions)

1. **Q-learning convergence**: state the three sufficient conditions for tabular Q-learning convergence and sketch the contraction-based proof (operator `H`, sup-norm, Robbins-Monro on `αₖ`).
2. **Vanilla DQN**: name the two problems with naive NN Q-learning and explain how (a) the **target network** and (b) the **experience replay buffer** solve them. Write the DQN TD target.
3. **Value-based vs policy-based**: list four advantages of policy-based methods. Write the softmax policy with preferences `h(s,a,θ) = θᵀ x(s,a)`.
4. **Policy Gradient Theorem**: write the formula for the episodic case and define the on-policy state distribution `μ(s)` via `h(s)` and `η(s)`.
5. **REINFORCE**: derive the update rule from the PG theorem (the four-line derivation that ends in `𝔼π[Gₜ ∇ln π(Aₜ|Sₜ,θ)]`) and write the full algorithm. Explain *why* the update is "proportional to return and inversely proportional to action probability".
6. **REINFORCE with baseline**: prove `Σₐ b(s) ∇π(a|s,θ) = 0`. Explain why a baseline can reduce variance without biasing the gradient. Name a natural choice.
7. **TRPO**: write the constrained optimization problem (with KL constraint), Taylor-expand to obtain the QCQP `max gᵀ(θ−θₖ) s.t. ½(θ−θₖ)ᵀH(θ−θₖ) ≤ δ`, give the analytic update `θₖ + √(2δ/(gᵀH⁻¹g)) H⁻¹g`, and explain why TRPO uses **conjugate gradient** (avoid forming `H⁻¹`).
8. **PPO-Clip**: write the clipped surrogate `L(s,a,θₖ,θ) = min(r·A, clip(r, 1−ε, 1+ε)·A)`, give the OpenAI piecewise simplification with `g(ε, A)`, and explain what happens when `A > 0` vs `A < 0`.
9. **PPO-Penalty vs PPO-Clip**: contrast the two variants and how PPO-Penalty adaptively updates `β` (the `×2` / `÷2` rule with the `1.5δ` and `δ/1.5` thresholds).
10. **GRPO**: define the group-relative advantage `A(aᵢ) = R(aᵢ) − R̄`, list the five GRPO steps for LLM training, and explain why it is **value-baseline-free**.
11. **DDPG**: explain why standard Q-learning fails in continuous action spaces and how DDPG replaces `maxₐ Q(s,a)` by `Q(s, μθ(s))`. Write the MSBE critic loss and the policy gradient `∇θ Q(s, μθ(s))`. Compare hard-update (DQN) vs Polyak (DDPG) targets.
12. **MM intuition**: draw the two curves (`η(θ)` in red, surrogate `M(θ) = L(θ) − C·KL` in blue), explain why MM guarantees non-decreasing performance and how TRPO/PPO use this idea.

### Pitfalls

- **PG theorem is *proportional*, not equal** — the constant of proportionality is 1 only in the continuing case; in the episodic case it's the average episode length.
- **Baselines must not depend on `a`** — if `b(s,a)` you bias the gradient. State-only is fine; advantage `A(s,a) = q(s,a) − v(s)` works because `v(s)` doesn't depend on `a`.
- **REINFORCE is high-variance** — the `1/π(Aₜ|Sₜ,θ)` factor explodes when the chosen action is rare; baselines and PPO/TRPO exist exactly to tame this.
- **TRPO needs the constraint on the *new* policy's KL** to be enforced after the line search — the Taylor approximation is only valid near `θₖ`, which is why backtracking is mandatory, not optional.
- **Don't form `H⁻¹` in TRPO** — always Hessian-vector products + conjugate gradient.
- **PPO ratio is `πθ / πθₖ`, NOT `πθ / πθold`** — `πθₖ` is the policy that *collected* the data, fixed for the whole update; `θ` is what you're optimizing. Misreading this is a classic exam trap.
- **PPO-Clip does *not* guarantee a KL bound** — it only removes the incentive to walk too far. Use multiple epochs of SGD but stop if KL blows up.
- **GRPO has no critic** — its baseline is the **group-mean reward**, not a learned `V(s)`. Don't write a `V`-update.
- **DDPG is for continuous actions only** — never apply it to discrete spaces; use DQN/Rainbow instead.
- **DDPG policy is *deterministic*** — you *must* add exploration noise (Gaussian or OU) at training time, and you *should* anneal it.
- **Polyak `ρ` is close to 1** — `ρ = 0.995` is typical. Don't confuse with the DQN "every `C` steps hard copy".
- **MM is monotone, but only on the *surrogate*** — `η` improves only because `M` is a *lower bound*. If you forget the lower-bound property (e.g., drop the `−C·KL`), monotonicity is gone.
- **Action vs state baselines** — `q(s,a) − v(s)` (advantage) is fine; `q(s,a) − q(s,b)` (action baseline) is not in general — it can change the expectation.
