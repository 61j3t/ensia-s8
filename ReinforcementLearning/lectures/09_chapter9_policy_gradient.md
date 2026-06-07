# Chapter 9 — Policy Gradient Methods

## Bird's eye view

- **Idea**: learn a **parameterized policy** `π(a|s,θ)` *directly* — actions selected **without consulting a value function**. A value function may still be used to **learn θ**, just not to pick actions.
- Updates approximate **gradient ascent** on a scalar **performance measure** `J(θ)`: `θₜ₊₁ = θₜ + α ∇J(θₜ)`. Any method in this schema = **policy gradient method**.
- If the method also learns a value function it is called **actor-critic** (actor = policy, critic = value function, usually state-value).
- **Two performance regimes**: episodic (value of start state) and continuing (average reward rate).
- **Core obstacle**: changing `θ` changes the stationary state distribution `μ(s)` — and `∇μ` is hard to estimate. The **policy gradient theorem** rescues us: the gradient can be written **without** `∇μ`.
- **Theorem** (episodic): `∇J(θ) ∝ Σₛ μ(s) Σₐ qπ(s,a) ∇π(a|s,θ)`.
- Sampling that expectation yields the **REINFORCE** update `θ ← θ + α Gₜ ∇lnπ(Aₜ|Sₜ,θ)` — Monte Carlo policy gradient. Add a state-dependent **baseline** `b(s)` (typically `v̂(s,w)`) to cut variance without bias.
- **Actor-critic (one-step)**: replace `Gₜ` by the bootstrapped TD error `δₜ = R + v̂(S',w) − v̂(S,w)` (or with `−R̄` for average-reward) — fully online, incremental, lower variance than REINFORCE.
- **Parameterizations**: **softmax in action preferences** for discrete actions; **Gaussian** `π(a|s,θ) = 𝒩(μ(s,θ), σ(s,θ)²)` for continuous actions — `μ` linear in features, `σ = exp(θσᵀ x(s))` to stay positive.
- **Why policy gradient?** Autonomous greedification (start exploratory, end deterministic), can represent **truly stochastic** optimal policies, sometimes simpler than the value function, and gives a natural way to handle **continuous** actions.

---

## 0. Outline of the chapter

1. Learning policies directly.
2. Policy approximation and its advantages.
3. The objective for learning policies.
4. The policy gradient theorem.
5. REINFORCE (and REINFORCE with baseline).
6. Actor-Critic methods.
7. Actor-Critic with Softmax policies.
8. Policy parameterization for continuous actions (Gaussian).

---

## 1. Learning policies directly

- Until now: learn `q̂(s,a,w)`, derive a policy (e.g. ε-greedy). Now: **learn the policy `π(a|s,θ)` itself**, with parameter vector `θ ∈ ℝᵈ'`.
- A value function **may still be used to learn θ**, but is **not required for action selection**.
- Notation: `π(a|s,θ) = Pr{Aₜ = a | Sₜ = s, θₜ = θ}` — probability of taking action `a` in state `s` under parameters `θ`.
- Learning rule (generic): pick a scalar **performance measure** `J(θ)` and follow its gradient.

```math
\boldsymbol{\theta}_{t+1} = \boldsymbol{\theta}_t + \alpha \, \widehat{\nabla J(\boldsymbol{\theta}_t)}
```

where `∇̂J(θₜ)` is a **stochastic estimate** whose expectation approximates `∇J(θₜ)`.

- All methods that follow this schema = **policy gradient methods**, whether or not they also learn a value function.
- Methods that learn **both** policy and (usually state-) value function = **actor-critic methods** (actor = policy, critic = value function).
- Two cases studied:
  - **Episodic**: `J(θ)` = value of the start state under `π`.
  - **Continuing**: `J(θ)` = average reward rate.

### 1.1. Example: a policy with no action-values

Mountain Car: the optimal policy is "push right if velocity ≥ 0, push left otherwise". A clean rule on `(position, velocity)` — easy to **represent as a policy**, but the corresponding action-value surface is messy. Some tasks have **simpler policies than values**.

### 1.2. Two networks side by side

| Architecture | Input | Output |
|---|---|---|
| **Value-based** | `s, a` → `W` | `q̂(s,a,w)` |
| **Policy-based** | `s, a` → `θ` | `π(a|s,θ)` |

---

## 2. Policy approximation and its advantages

### 2.1. Constraints on the parameterization

`π(a|s,θ)` can be parameterized any way we like, **as long as it is differentiable in `θ`** — i.e. `∇θ π(a|s,θ)` exists and is finite for all `s,a,θ`.

To keep exploration alive we further require the policy is **never deterministic**: `π(a|s,θ) ∈ [0,1]`. The two probability constraints:

```math
\pi(a \mid s, \boldsymbol{\theta}) \geq 0 \quad \text{for all } a \in \mathcal{A},\; s \in \mathcal{S}
```

```math
\sum_{a \in \mathcal{A}} \pi(a \mid s, \boldsymbol{\theta}) = 1 \quad \text{for all } s \in \mathcal{S}
```

### 2.2. Softmax in action preferences (discrete actions)

A simple choice that automatically satisfies both constraints:

```math
\pi(a \mid s, \boldsymbol{\theta}) \doteq \frac{e^{h(s,a,\boldsymbol{\theta})}}{\sum_{b \in \mathcal{A}} e^{h(s,b,\boldsymbol{\theta})}}
```

- `h(s,a,θ)` is the **action preference** for action `a` in state `s`.
- Preferences can be linear in features `h(s,a,θ) = θᵀ x(s,a)` or come from a deep neural network.

### 2.3. Advantages of learning a policy directly

| Advantage | What it means |
|---|---|
| **Autonomous adjustment** | Agent becomes more greedy autonomously — starts exploratory, gradually converges to (near-)deterministic if optimal. |
| **Handles complex envs.** | A **stochastic** policy can be optimal where any deterministic policy is suboptimal (aliased states, adversarial games). |
| **Simplicity** | Sometimes the policy is *simpler* to represent than the value function (see Mountain Car). |

---

## 3. The objective for learning policies

### 3.1. Forms of the return

The ultimate goal: collect as much reward as possible — `Rₜ, Rₜ₊₁, Rₜ₊₂, …`.

| Setting | Return |
|---|---|
| Episodic (undiscounted) | `Gₜ = Σ_{t=0}^{T} Rₜ` |
| Continuing (discounted) | `Gₜ = Σ_{t=0}^{∞} γᵗ Rₜ` |
| Continuing (average reward) | `Gₜ = Σ_{t=0}^{∞} (Rₜ − r(π))` |

### 3.2. The average reward objective (continuing case)

Average reward under a policy `π`:

```math
r(\pi) = \sum_s \mu(s) \sum_a \pi(a \mid s, \boldsymbol{\theta}) \sum_{s', r} p(s', r \mid s, a)\, r
```

- `μ(s)` = on-policy / **stationary** state distribution under `π`.
- Goal: find `θ` that **maximizes** `r(π)`.

### 3.3. Optimizing it

Take the gradient w.r.t. `θ` and ascend:

```math
\nabla r(\pi) = \nabla \sum_s \mu(s) \sum_a \pi(a \mid s, \boldsymbol{\theta}) \sum_{s', r} p(s', r \mid s, a)\, r
```

Methods that follow this idea = **policy gradient methods**.

### 3.4. Why is this hard?

Modifying `θ` modifies `μ`:

```math
\nabla_{\boldsymbol{\theta}}\, r(\pi) = \nabla_{\boldsymbol{\theta}} \sum_s \underbrace{\mu(s)}_{\text{depends on }\boldsymbol{\theta}} \sum_a \pi(a \mid s, \boldsymbol{\theta}) \sum_{s', r} p(s', r \mid s, a)\, r
```

Contrast with value function approximation, where in `VE(w) = Σₛ μ(s)[vπ(s) − v̂(s,w)]²` the weighting `μ(s)` is **independent of w** so the gradient passes cleanly through. Here `μ(s)` is **coupled** to the parameter being optimized — `∇μ` would require modeling long-term dynamics of the env. under `π`.

The product rule applied directly gives **two terms**, the first containing `∇μ(s)`:

```math
\nabla r(\pi) = \sum_s \nabla \mu(s) \sum_a \pi(a \mid s, \boldsymbol{\theta}) \sum_{s', r} p(s', r \mid s, a)\, r \; + \; \sum_s \mu(s) \nabla \sum_a \pi(a \mid s, \boldsymbol{\theta}) \sum_{s', r} p(s', r \mid s, a)\, r
```

`∇μ(s)` is not straightforward to estimate (depends on long-term agent–environment interaction). The **policy gradient theorem** is the theoretical fix.

---

## 4. The Policy Gradient Theorem

### 4.1. Statement (episodic / average-reward case)

```math
\nabla r(\pi) = \sum_s \mu(s) \sum_a q_\pi(s, a)\, \nabla \pi(a \mid s, \boldsymbol{\theta})
```

- Gradients are column vectors of partial derivatives w.r.t. components of `θ`.
- `μ` is the **on-policy distribution** under `π`.
- **No `∇μ` appears** — that is the whole point of the theorem.
- Proof: in the textbook (Sutton & Barto, 2nd ed.).
- Practical consequence: the gradient is **straightforward to estimate** from experience → we can build an incremental policy-gradient algorithm.

### 4.2. From the theorem to a sample estimate

Write the gradient as an expectation under `π`:

```math
\nabla J(\boldsymbol{\theta}) \propto \sum_s \mu(s) \sum_a q_\pi(s, a) \nabla \pi(a \mid s, \boldsymbol{\theta}) = \mathbb{E}_\pi\!\left[\sum_a q_\pi(S_t, a)\, \nabla \pi(a \mid S_t, \boldsymbol{\theta})\right]
```

A first generic stochastic gradient-ascent update (still summing over all `a`):

```math
\boldsymbol{\theta}_{t+1} \doteq \boldsymbol{\theta}_t + \alpha \sum_a \hat{q}(S_t, a, \mathbf{w})\, \nabla \pi(a \mid S_t, \boldsymbol{\theta})
```

with `q̂` some learned approximation of `qπ`.

### 4.3. Stochastic sample with a single action

Introduce `Aₜ` the same way `Sₜ` was introduced — multiply and divide by `π(a|Sₜ,θ)`:

```math
\sum_a q_\pi(S_t, a)\, \nabla \pi(a \mid S_t, \boldsymbol{\theta}) = \sum_a \pi(a \mid S_t, \boldsymbol{\theta})\, \frac{\nabla \pi(a \mid S_t, \boldsymbol{\theta})}{\pi(a \mid S_t, \boldsymbol{\theta})}\, q_\pi(S_t, a) = \mathbb{E}_\pi\!\left[\frac{\nabla \pi(A_t \mid S_t, \boldsymbol{\theta})}{\pi(A_t \mid S_t, \boldsymbol{\theta})}\, q_\pi(S_t, A_t)\right]
```

This bracketed quantity is **sampleable on each step** and its expectation equals the gradient. The sample-based update:

```math
\boldsymbol{\theta}_{t+1} \doteq \boldsymbol{\theta}_t + \alpha\, \frac{\nabla \pi(A_t \mid S_t, \boldsymbol{\theta})}{\pi(A_t \mid S_t, \boldsymbol{\theta})}\, q_\pi(S_t, A_t)
```

Using the **log-derivative trick** `∇π / π = ∇ ln π`:

```math
\boldsymbol{\theta}_{t+1} \doteq \boldsymbol{\theta}_t + \alpha\, \nabla \ln \pi(A_t \mid S_t, \boldsymbol{\theta})\, q_\pi(S_t, A_t)
```

The factor `∇ ln π(Aₜ|Sₜ,θ)` is the **eligibility vector**.

---

## 5. REINFORCE — Monte-Carlo policy gradient

### 5.1. Derivation

Replace `qπ(Sₜ,Aₜ)` by an **unbiased Monte-Carlo sample** of it, namely the realised return `Gₜ` (uses `E_π[Gₜ | Sₜ, Aₜ] = qπ(Sₜ, Aₜ)`):

```math
\nabla J(\boldsymbol{\theta}) = \mathbb{E}_\pi\!\left[G_t \, \frac{\nabla \pi(A_t \mid S_t, \boldsymbol{\theta})}{\pi(A_t \mid S_t, \boldsymbol{\theta})}\right]
```

### 5.2. REINFORCE update

```math
\boldsymbol{\theta}_{t+1} \doteq \boldsymbol{\theta}_t + \alpha\, G_t\, \frac{\nabla \pi(A_t \mid S_t, \boldsymbol{\theta})}{\pi(A_t \mid S_t, \boldsymbol{\theta})} = \boldsymbol{\theta}_t + \alpha\, G_t\, \nabla \ln \pi(A_t \mid S_t, \boldsymbol{\theta})
```

### 5.3. Pseudocode

```
REINFORCE: Monte-Carlo Policy-Gradient Control (episodic) for π*

Input: a differentiable policy parameterization π(a|s, θ)
Algorithm parameter: step size α > 0
Initialize policy parameter θ ∈ ℝᵈ' (e.g., to 0)

Loop forever (for each episode):
    Generate an episode S₀, A₀, R₁, …, S_{T-1}, A_{T-1}, R_T following π(·|·,θ)
    Loop for each step of the episode t = 0, 1, …, T-1:
        G ← Σ_{k=t+1}^{T} γ^{k-t-1} R_k                              (G_t)
        θ ← θ + α γ^t G ∇ ln π(A_t | S_t, θ)
```

### 5.4. REINFORCE with baseline

Subtract any function `b(s)` that does not depend on the action — the gradient is unchanged because the subtracted piece sums to zero:

```math
\sum_a b(s)\, \nabla \pi(a \mid s, \boldsymbol{\theta}) = b(s)\, \nabla \sum_a \pi(a \mid s, \boldsymbol{\theta}) = b(s)\, \nabla 1 = 0
```

Generalized theorem:

```math
\nabla r(\pi) \propto \sum_s \mu(s) \sum_a \big(q_\pi(s,a) - b(s)\big)\, \nabla \pi(a \mid s, \boldsymbol{\theta})
```

Resulting update:

```math
\boldsymbol{\theta}_{t+1} \doteq \boldsymbol{\theta}_t + \alpha\, (G_t - b(S_t))\, \frac{\nabla \pi(A_t \mid S_t, \boldsymbol{\theta})}{\pi(A_t \mid S_t, \boldsymbol{\theta})}
```

- Baseline can be **any function** (even a random variable) that does not vary with `a`.
- Strictly **generalizes** REINFORCE (recover REINFORCE when `b ≡ 0`).
- Typical choice: learned state-value `v̂(s,w)` — it **does not bias** the gradient but greatly **reduces variance**.

### 5.5. Pseudocode (with baseline)

```
REINFORCE with Baseline (episodic), for estimating πθ ≈ π*

Input: a differentiable policy parameterization π(a|s, θ)
Input: a differentiable state-value function parameterization v̂(s, w)
Algorithm parameters: step sizes α^θ > 0, α^w > 0
Initialize policy parameter θ ∈ ℝᵈ' and state-value weights w ∈ ℝᵈ (e.g., to 0)

Loop forever (for each episode):
    Generate an episode S₀, A₀, R₁, …, S_{T-1}, A_{T-1}, R_T following π(·|·,θ)
    Loop for each step of the episode t = 0, 1, …, T-1:
        G ← Σ_{k=t+1}^{T} γ^{k-t-1} R_k                              (G_t)
        δ ← G − v̂(S_t, w)
        w ← w + α^w δ ∇v̂(S_t, w)
        θ ← θ + α^θ γ^t δ ∇ ln π(A_t | S_t, θ)
```

---

## 6. Actor-Critic methods

### 6.1. Motivation

- Do we have to **choose** between learning policy parameters and learning a value function? **No.**
- Value-learning methods (e.g. TD) still have an important role inside policy gradient methods.
- The parameterized policy = **actor**. The value function = **critic** (it evaluates the actions chosen by the actor).
- Actor-critic methods were among the **earliest TD-based methods** in RL.

### 6.2. One-step actor-critic — bootstrapping the return

Replace the Monte-Carlo return `Gₜ` by a **one-step TD target**. For the **average-reward** continuing case:

```math
\boldsymbol{\theta}_{t+1} \doteq \boldsymbol{\theta}_t + \alpha\, \nabla \ln \pi(A_t \mid S_t, \boldsymbol{\theta})\, \big[R_{t+1} - \bar R + \hat v(S_{t+1}, \mathbf{w})\big]
```

- Fully **online**, **incremental** — no need to wait for episode end.
- Natural state-value learning method to pair with it: **semi-gradient TD**.

The two-network picture:

| Block | Inputs | Parameters | Output |
|---|---|---|---|
| **Actor** (policy gradient) | `s, a` | `θ` | `π(a|s,θ)` |
| **Critic** (average-reward semi-gradient TD(0)) | `s` | `w` | `v̂(s,w)` |

### 6.3. Adding the state-value baseline → TD error

Subtract the current-state value as a baseline:

```math
\boldsymbol{\theta}_{t+1} \doteq \boldsymbol{\theta}_t + \alpha\, \nabla \ln \pi(A_t \mid S_t, \boldsymbol{\theta})\, \big[R_{t+1} - \bar R + \hat v(S_{t+1}, \mathbf{w}) - \hat v(S_t, \mathbf{w})\big]
```

The bracketed expression is exactly the **TD error**:

```math
\delta_t = R_{t+1} - \bar R + \hat v(S_{t+1}, \mathbf{w}) - \hat v(S_t, \mathbf{w})
```

So the update is simply:

```math
\boldsymbol{\theta}_{t+1} \doteq \boldsymbol{\theta}_t + \alpha\, \nabla \ln \pi(A_t \mid S_t, \boldsymbol{\theta})\, \delta_t
```

(Exercise from the slides: show the two formulations — with and without `v̂(Sₜ,w)` — are equivalent in expectation.)

### 6.4. How actor and critic interact

- Environment sends `state` and `reward`.
- **Critic** updates `v̂` from the reward and the next state and produces the **TD error**.
- **Actor** uses that TD error as the scalar in its policy-gradient update, picks the next `action`.

### 6.5. Pseudocode (continuing average-reward actor-critic)

```
Actor-Critic (continuing), for estimating πθ ≈ π*

Input: a differentiable policy parameterization π(a|s, θ)
Input: a differentiable state-value function parameterization v̂(s, w)
Initialize R̄ ∈ ℝ to 0
Initialize state-value weights w ∈ ℝᵈ and policy parameter θ ∈ ℝᵈ' (e.g. to 0)
Algorithm parameters: α^w > 0, α^θ > 0, α^R̄ > 0
Initialize S ∈ 𝒮

Loop forever (for each time step):
    A ~ π(·|S, θ)
    Take action A, observe S', R
    δ ← R − R̄ + v̂(S', w) − v̂(S, w)
    R̄ ← R̄ + α^R̄ δ
    w ← w + α^w δ ∇v̂(S, w)
    θ ← θ + α^θ δ ∇ ln π(A | S, θ)
    S ← S'
```

---

## 7. Actor-Critic with Softmax policies (finite actions, continuous states)

### 7.1. Critic update (linear state-value)

```math
\mathbf{w} \leftarrow \mathbf{w} + \alpha_{\mathbf{w}} \, \delta \, \nabla \hat v(S, \mathbf{w}), \qquad \hat v(S, \mathbf{w}) \doteq \mathbf{w}^\top \mathbf{x}(s) \;\Rightarrow\; \nabla \hat v(S, \mathbf{w}) = \mathbf{x}(s)
```

### 7.2. Actor update (softmax in linear preferences)

Policy:

```math
\pi(a \mid s, \boldsymbol{\theta}) \doteq \frac{e^{h(s,a,\boldsymbol{\theta})}}{\sum_{b \in \mathcal{A}} e^{h(s,b,\boldsymbol{\theta})}}, \qquad h(s,a,\boldsymbol{\theta}) \doteq \boldsymbol{\theta}^\top \mathbf{x}_h(s,a)
```

Eligibility vector for softmax in action preferences:

```math
\nabla \ln \pi(A \mid S, \boldsymbol{\theta}) = \mathbf{x}_h(s, a) - \sum_b \pi(b \mid s, \boldsymbol{\theta})\, \mathbf{x}_h(s, b)
```

(observed feature minus mean feature under the policy)

Actor update:

```math
\boldsymbol{\theta} \leftarrow \boldsymbol{\theta} + \alpha_{\boldsymbol{\theta}} \, \delta \, \nabla \ln \pi(A \mid S, \boldsymbol{\theta})
```

### 7.3. Feature vector design — stacked state features

The critic only needs a state feature vector `x(s)`. The actor needs a **state-action feature vector** `xₕ(s,a)`. The standard trick: **stack copies of the state features**, one block per action, so that only the block corresponding to `a` is "active":

```
            ┌ x₀(s) ┐
            │ x₁(s) │   block for a₀
            │ x₂(s) │
            │ x₃(s) ┘
xₕ(s,a) =   ┌ x₀(s) ┐
            │ x₁(s) │   block for a₁
            │ x₂(s) │
            │ x₃(s) ┘
            ┌ x₀(s) ┐
            │ x₁(s) │   block for a₂
            │ x₂(s) │
            └ x₃(s) ┘
```

### 7.4. Demonstration — Pendulum swing-up

| Spec | Value |
|---|---|
| Goal | Balance the pendulum upright |
| State | `s = (β, β̇)` (angular position, angular velocity) |
| Actions | `a ∈ {−1, 0, +1}` (discrete) |
| Reward | `R ≐ −|β|`, with constraint `−2π ≤ β̇ ≤ 2π` |
| Task | **Continuing** — no episodes, no termination |
| Policy | **Softmax** (action space is discrete) |
| Value approx. | Linear `v̂(s,w) = wᵀ x(s)` |
| Action prefs | Linear `h(s,a,θ) = θᵀ xₕ(s,a)` |
| Features | **Tile coding** — 32 tilings of 8 × 8 |
| Step sizes | `α^θ < α^w` (critic learns faster than actor) |

Result after 100 runs: average reward climbs from ~−4 toward ~0 within ~10 000 training steps.

---

## 8. Policy parameterization for continuous actions

### 8.1. Discrete vs continuous action spaces

- **Discrete**: a few bars of probability mass at `{−1, 0, 1}`.
- **Continuous**: a probability *density* over a range, e.g. `a ∈ [−3, 3]` shaped like a Gaussian.

### 8.2. Gaussian density refresher

```math
p(x) \doteq \frac{1}{\sigma \sqrt{2\pi}} \exp\!\left(- \frac{(x - \mu)^2}{2 \sigma^2}\right)
```

- `μ`, `σ` = mean and standard deviation.
- `p(x)` is a **density**, not a probability — integrate over a range to get the probability that `x` falls in that range.

### 8.3. Gaussian policy

```math
\pi(a \mid s, \boldsymbol{\theta}) \doteq \frac{1}{\sigma(s, \boldsymbol{\theta}) \sqrt{2\pi}} \exp\!\left(-\frac{(a - \mu(s, \boldsymbol{\theta}))^2}{2 \, \sigma(s, \boldsymbol{\theta})^2}\right)
```

where `μ : 𝒮 × ℝᵈ' → ℝ`, `σ : 𝒮 × ℝᵈ' → ℝ⁺`, and `θ = [θμ, θσ]ᵀ`.

Parameterize the two outputs of the head:

```math
\mu(s, \boldsymbol{\theta}) \doteq \boldsymbol{\theta}_\mu^\top \mathbf{x}(s)
```

```math
\sigma(s, \boldsymbol{\theta}) \doteq \exp\!\big(\boldsymbol{\theta}_\sigma^\top \mathbf{x}(s)\big)
```

- `μ`: any linear function works (mean can be any real).
- `σ`: must always be **positive** → exponentiate a linear function.

### 8.4. Gradient of the log Gaussian policy

```math
\nabla \ln \pi(a \mid s, \boldsymbol{\theta}_\mu) = \frac{a - \mu(s, \boldsymbol{\theta})}{\sigma(s, \boldsymbol{\theta})^2} \, \mathbf{x}(s)
```

```math
\nabla \ln \pi(a \mid s, \boldsymbol{\theta}_\sigma) = \left(\frac{(a - \mu(s, \boldsymbol{\theta}))^2}{\sigma(s, \boldsymbol{\theta})^2} - 1\right) \mathbf{x}(s)
```

(Exercise from the slides: verify these expressions.)

### 8.5. Why continuous actions?

| Reason | Comment |
|---|---|
| Hard to discretize | E.g. a golf-robot's force depends continuously on distance & terrain. |
| **Generalize over actions** | A Gaussian smoothly assigns density to nearby actions — improving on one action automatically improves nearby ones. |
| Replace huge discrete sets | Even when actions are formally discrete but very large, treating them as continuous avoids per-action exploration. |

### 8.6. Pendulum swing-up with a Gaussian policy

Same task as 7.4 but `a ∈ [−3, 3]` continuous, policy = Gaussian over actions. Final-performance video shows the pendulum balancing upright.

---

## Key terms (glossary)

- **Policy parameter θ ∈ ℝᵈ'** — the vector being learnt; differentiable in `θ`.
- **`π(a|s,θ)`** — parameterized stochastic policy.
- **Performance measure `J(θ)`** — scalar to be maximized (start-state value or average reward).
- **Policy gradient method** — any update of the form `θ ← θ + α ∇̂J(θ)`.
- **Actor-critic** — policy gradient + learned value function (the critic).
- **On-policy distribution `μ(s)`** — stationary state distribution induced by `π`.
- **Average reward `r(π)`** — `Σₛ μ(s) Σₐ π(a|s,θ) Σ_{s',r} p(s',r|s,a) r`.
- **Policy gradient theorem** — `∇r(π) ∝ Σₛ μ(s) Σₐ qπ(s,a) ∇π(a|s,θ)`, with no `∇μ` term.
- **Eligibility vector** — `∇ ln π(Aₜ|Sₜ,θ) = ∇π / π`.
- **REINFORCE** — Monte-Carlo policy gradient: `θ ← θ + α γᵗ Gₜ ∇ ln π(Aₜ|Sₜ,θ)`.
- **Baseline `b(s)`** — state-only function subtracted from the return; unbiased, reduces variance.
- **TD error `δₜ`** — `Rₜ₊₁ − R̄ + v̂(Sₜ₊₁,w) − v̂(Sₜ,w)`; used as the actor's scalar in one-step actor-critic.
- **Softmax policy** — `π = e^{h(s,a,θ)} / Σ_b e^{h(s,b,θ)}` with action preferences `h`.
- **Action preference `h(s,a,θ)`** — un-normalized score; linear or NN.
- **Stacked state features** — `xₕ(s,a)` built by stacking copies of `x(s)`, one block per action.
- **Gaussian policy** — `𝒩(μ(s,θ), σ(s,θ)²)` for continuous actions; `σ = exp(θσᵀ x(s))`.
- **Continuing task** — no terminal state; use average reward, no `γ`, maintain `R̄`.

---

## Exam targets (likely written-exam questions)

1. **State the policy gradient theorem** for the episodic (or average-reward) case: `∇r(π) = Σₛ μ(s) Σₐ qπ(s,a) ∇π(a|s,θ)`. Explain *why it is important* (no `∇μ` term — distribution-shift problem avoided).
2. **Why is `∇μ(s)` problematic?** Explain that changing `θ` shifts the stationary state distribution, contrast with value-function approximation where `μ` is independent of `w`.
3. **Derive the REINFORCE update** from the policy gradient theorem: introduce `Aₜ` via `π/π`, replace `qπ(Sₜ,Aₜ)` with the sampled return `Gₜ`, apply the log-derivative trick. Give the final update `θ ← θ + α Gₜ ∇ ln π(Aₜ|Sₜ,θ)`.
4. **Write the REINFORCE-with-baseline update** and prove that subtracting any state-only baseline `b(s)` leaves the gradient unbiased (show `Σₐ b(s) ∇π(a|s,θ) = b(s) ∇1 = 0`). Explain the variance-reduction benefit.
5. **Compare** REINFORCE vs one-step actor-critic: MC vs bootstrapped target, offline (episode-end) vs fully online, high variance vs lower variance, unbiased vs biased.
6. **Write the one-step actor-critic update** (average-reward continuing case) using the TD error: `θ ← θ + α δ ∇ ln π(A|S,θ)`, `δ = R − R̄ + v̂(S',w) − v̂(S,w)`. Identify which term is the actor's "scalar" and which network is the critic.
7. **Softmax in action preferences**: write `π(a|s,θ)`, give the constraints on a policy parameterization, derive `∇ ln π(A|S,θ) = xₕ(s,a) − Σ_b π(b|s,θ) xₕ(s,b)` for linear preferences, explain stacked state features.
8. **Gaussian policy for continuous actions**: write `π(a|s,θ)`, justify `σ = exp(θσᵀ x(s))` (must stay positive), state both `∇ ln π` (w.r.t. `θμ` and `θσ`). Give two advantages of continuous actions over discrete ones.
9. **Advantages of learning policies directly** over value-based methods: autonomous greedification, can represent truly stochastic optima, sometimes simpler than the value function, natural for continuous actions.

### Pitfalls

- **Do not include `∇μ`** in the policy gradient — the whole point of the theorem is that it disappears. Writing the product-rule form on the exam without invoking the theorem loses marks.
- **`q̂(s,a,w)` is only needed if you don't have `Gₜ`**. REINFORCE uses the Monte-Carlo return; actor-critic uses the TD error. Don't mix the two.
- **The baseline must depend only on the state**, not on the action. A baseline that depends on `a` introduces bias.
- **`σ(s,θ)` must be positive** — that's why we exponentiate. Parameterizing `σ` directly as `θσᵀ x` is wrong.
- **Continuing case has no `γ`** — use `R̄` (running average) and average-reward TD error, not discounted returns.
- **Eligibility vector ≠ eligibility trace.** The eligibility vector is `∇ ln π(Aₜ|Sₜ,θ)` (instantaneous). Traces are a separate mechanism (Chapter on `λ`-returns).
- **A policy must stay stochastic during learning** (`π ∈ [0,1]`, never strictly 0/1) — otherwise exploration dies and the gradient signal goes to zero.
- **Softmax preferences are not action-values.** `h(s,a,θ)` is an unnormalized score; do **not** read it as `q(s,a)`.
- **Critic typically learns faster than actor** (`α^w > α^θ`) — otherwise the policy chases a wildly inaccurate critic.
