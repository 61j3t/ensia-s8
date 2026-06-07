# Chapter 8 — Value Function Approximation

## Bird's eye view

- **Value Function Approximation (VFA)** = replace the tabular `v(s)` by a **parameterized functional form** `v̂(s, w) ≈ vπ(s)` with a weight vector `w ∈ ℝᵈ`, where typically `d ≪ |𝒮|`.
- Why: tabular RL fails when state spaces are huge or continuous. With `d ≪ |𝒮|`, **changing one weight changes the value of many states** — gives **generalization** but loses **discrimination**; the two must be traded off.
- The chapter has two halves. **Part I — On-policy Prediction with Approximation** (Sutton-Barto Ch. 9): how to estimate `vπ` with a parametric `v̂`. **Part II — On-policy Control with Approximation** (Ch. 10): extend to `q̂(s, a, w) ≈ q*` using ε-greedy GPI.
- **Prediction objective** = **Mean Squared Value Error** `VE(w) = Σₛ μ(s)[vπ(s) − v̂(s, w)]²`, weighted by an on-policy state distribution `μ`.
- **Stochastic Gradient Descent (SGD)** drives both Gradient MC and semi-gradient TD; in TD the gradient of the **bootstrap target** is dropped (the target depends on `w` too) → "semi-gradient".
- **Linear methods**: `v̂(s, w) = wᵀx(s)`; gradient is the feature vector itself `∇v̂ = x(s)`. Tabular and state-aggregation are special cases (indicator features).
- **Feature construction** for linear methods: **coarse coding**, **tile coding** (the practical workhorse — used in Mountain Car), polynomial / Fourier / RBF / kernel bases (mentioned).
- **TD fixed point** (linear semi-gradient TD): `w_TD = A⁻¹b`; bound `VE(w_TD) ≤ 1/(1−γ) · minₚ VE(w)`.
- **Nonlinear approximation**: artificial neural networks (universal approximation with one hidden layer; deep nets compose features more effectively).
- **Episodic semi-gradient SARSA** with ε-greedy = canonical on-policy control with approximation; expected-SARSA and Q-learning variants drop in via different targets.
- **Average reward** setting replaces `γ` for continuing tasks: `r(π) = limₕ→∞ (1/h)Σₜ E[Rₜ]`. Returns become **differential returns** `Gₜ = Σ(R_{t+k+1} − r(π))`, and differential semi-gradient SARSA tracks both `w` and an estimate `R̄ ≈ r(π)`.

---

## 0. Course context

- **Instructor**: Dr. Aissa Boulmerka, ENSIA, 2025-2026.
- **Course position**: Chapter 8 — bridges tabular methods (Ch. 1-7) and policy-gradient / advanced topics (Ch. 10-11). Largest core chapter (87 slides) because it spans Sutton-Barto Ch. 9 (on-policy prediction) **and** Ch. 10 (on-policy control).
- **Outline announced**:
  - **I. On-policy Prediction**: VFA, Generalization/Discrimination, VE objective, SGD & semi-gradient, Linear methods, Feature construction, Nonlinear methods.
  - **II. On-policy Control**: Episodic SARSA with approximation, Average reward setting.

---

# Part I — On-policy Prediction with Approximation

## 1. Value-Function Approximation

### 1.1. Parameterized value function

- The approximate value of state `s` given weight vector `w` is written:

```math
\hat v(s, \mathbf{w}) \approx v_\pi(s)
```

- `w ∈ ℝᵈ`; typically `d ≪ |𝒮|`.
- `v̂` may be **linear in features of the state** (Sec. 5) or **nonlinear** — e.g., a multi-layer neural network (Sec. 7).
- Changing **one weight** changes the value of **many states** → **generalization**.

### 1.2. Concrete linear example (gridworld)

On a 4×4 grid with state `s = (X, Y)`:

```math
\hat v(s, \mathbf{w}) \doteq w_1 X + w_2 Y
```

We store only **two** weights — yet the table has 16 cells.

| `w₁`, `w₂` | Effect (cell `(X,Y) = w₁X + w₂Y`) |
|---|---|
| `w₁=1, w₂=1` | corner `(1,1)=2`, opposite `(4,4)=8` — uniform slope |
| `w₁=2, w₂=1` | `X` doubles its weight; values stretch along X |
| `w₁=−1, w₂=1` | `X` inverts → diagonal pattern with negatives |
| `w₁=4, w₂=1` | strong X-axis tilt, `(4,4)=20` |

### 1.3. Limitation of plain linear approximation

If the true value pattern has a "central blob" (e.g. zeros on the boundary, 5s in the interior), then `wX + wY` **cannot represent it** — `X` and `Y` are not good features. **Solution**: better features (Sec. 6).

### 1.4. Tabular and state aggregation are linear methods

Choose features to be **indicator functions**:

```math
\mathbf{x}(s_i) = (0, \dots, 0, 1, 0, \dots, 0)^T \quad (1 \text{ at position } i)
```

Then `v̂(sᵢ, w) = ⟨w, x(sᵢ)⟩ = wᵢ`. Tabular ≡ linear FA with one feature per state. **State aggregation** ≡ one feature per group of states (group i → wᵢ).

### 1.5. Nonlinear approximation (preview)

A neural network takes state `s` as input, passes it through layers of weighted connections to produce `v̂(s, w)`. Discussed in Sec. 7.

---

## 2. Generalization and Discrimination

| Concept | Definition | Robot example |
|---|---|---|
| **Generalization** | An update to one state's value influences the values of *other* (similar) states | Locations from which it would take the same time to reach the nearest object should have similar value |
| **Discrimination** | Ability to make values of two states **different** | A state with an object 3 ft away behind a wall vs. 3 ft away with clear path → different values |

- Generalization can **speed learning**: you may not have to visit every state to learn its value.
- A method's quality is plotted on a 2-D axis: **Generalization** (vertical) vs **Discrimination** (horizontal):
  - **Tabular methods** = bottom-right (no generalization, full discrimination).
  - **Aggregate-all-states** = top-left (full generalization, no discrimination).
  - **Ideal** = top-right (high both).

### 2.1. Framing value estimation as supervised learning

- Supervised learning works on **offline datasets**; RL is **online** — the agent generates data by interaction. The FA technique must be usable online.
- TD methods add a complication: they **bootstrap** — the target depends on the *current* estimate, which changes as learning progresses. Targets are **moving** (unlike SL where labels are ground truth).

---

## 3. The Prediction Objective — `VE`

### 3.1. Setup

Imagine a stream of state / true-value pairs `(S₁, vπ(S₁)), (S₂, vπ(S₂)), …`. We want a parameterized `v̂(s, w)` whose output is close to `vπ(s)`. We need a **measure of "close"**.

### 3.2. Mean Squared Value Error

Choose a **state distribution** `μ(s) ≥ 0`, `Σ μ(s) = 1`, indicating how much we care about each state. Then:

```math
\overline{VE}(\mathbf{w}) \doteq \sum_{s \in \mathcal S} \mu(s) \bigl[v_\pi(s) - \hat v(s, \mathbf{w})\bigr]^2
```

- Under **on-policy training**, `μ` is typically the **on-policy distribution** — the fraction of time the agent spends in `s` under `π`.
- Goal: **minimize VE** over `w`.

---

## 4. Stochastic-Gradient and Semi-Gradient Methods

### 4.1. SGD

`w = (w₁,…,wd)ᵀ`, with `v̂(s, w)` differentiable in `w` for all `s`. The SGD update (one sample at a time) is:

```math
\mathbf{w}_{t+1} \doteq \mathbf{w}_t - \tfrac{1}{2}\alpha \nabla\bigl[v_\pi(S_t) - \hat v(S_t, \mathbf{w}_t)\bigr]^2
\\
= \mathbf{w}_t + \alpha \bigl[v_\pi(S_t) - \hat v(S_t, \mathbf{w}_t)\bigr] \nabla \hat v(S_t, \mathbf{w}_t)
```

- Step size `α > 0`.
- SGD **adjusts after each sample** in the direction that most reduces the error on that example.
- The expectation of the SGD direction equals the full-batch gradient of `VE` → it converges (under standard step-size conditions) to a **local minimum** of `VE`.

### 4.2. From batch to stochastic

Full gradient:

```math
\nabla \sum_{s \in \mathcal S} \mu(s)[v_\pi(s) - \hat v(s, w)]^2 \;\propto\; \sum_s \mu(s)[v_\pi(s) - \hat v(s,w)]\,\nabla \hat v(s, w)
```

Sampling state `Sₜ` from `μ` and dropping the expectation gives the SGD update above.

### 4.3. Gradient Monte Carlo

We don't have `vπ(Sₜ)`. Replace it by the **observed return** `Gₜ`:

```math
\mathbf{w}_{t+1} \doteq \mathbf{w}_t + \alpha \bigl[G_t - \hat v(S_t, \mathbf{w}_t)\bigr] \nabla \hat v(S_t, \mathbf{w}_t)
```

Because `Eπ[Gₜ | Sₜ] = vπ(Sₜ)`, the **expected** Gradient MC update equals the true MSE gradient — `Gₜ` is an **unbiased** sample of `vπ(Sₜ)`. The method converges to a local minimum of `VE`.

```
Gradient Monte Carlo Algorithm for Estimating v̂ ≈ vπ

Input: the policy π to be evaluated
Input: a differentiable function v̂ : 𝒮 × ℝᵈ → ℝ
Algorithm parameter: step size α > 0
Initialize value-function weights w ∈ ℝᵈ arbitrarily (e.g., w = 0)

Loop forever (for each episode):
    Generate an episode S₀, A₀, R₁, S₁, A₁, …, R_T, S_T using π
    Loop for each step of episode, t = 0, 1, …, T−1:
        w ← w + α[Gₜ − v̂(Sₜ, w)] ∇v̂(Sₜ, w)
```

### 4.4. Random-walk example (1000 states, state aggregation)

- States 1 … 1000 numbered left-to-right; episodes start in state 500.
- Each step jumps **uniformly** to one of the 100 neighbours on the left or 100 on the right. Near edges, missing-neighbour probability mass becomes the probability of **termination** on that side.
- Termination on the left: reward `−1`; on the right: reward `+1`; otherwise reward `0`.
- **State aggregation**: 10 groups of 100 contiguous states → one weight per group (10 features). For `s ∈ group j`, `x(s)` is the indicator at position `j` → `v̂(s, w) = wⱼ`.
- Gradient: only `wⱼ` updates: `wⱼ ← wⱼ + α [Gₜ − v̂(Sₜ, w)] · 1`.
- With `α = 2·10⁻⁵`, after many episodes the learned step-function `v̂` approximates the true smooth `vπ` (which is roughly linear from −1 to +1); the state distribution `μ` is peaked near the centre (state 500).

### 4.5. Semi-gradient TD(0)

Replace the true value `vπ(Sₜ)` by a **bootstrap target** `Uₜ = R_{t+1} + γ v̂(S_{t+1}, w)`:

```math
\mathbf{w}_{t+1} \doteq \mathbf{w}_t + \alpha \bigl[R_{t+1} + \gamma \hat v(S_{t+1}, \mathbf{w}_t) - \hat v(S_t, \mathbf{w}_t)\bigr] \nabla \hat v(S_t, \mathbf{w}_t)
```

**Why "semi"-gradient**: a true gradient of `(Uₜ − v̂)²` would include `∇Uₜ`. But `Uₜ` itself depends on `w` (because `v̂(S_{t+1}, w)` does), so:

```math
\nabla U_t = \gamma \nabla \hat v(S_{t+1}, \mathbf{w}) \neq 0
```

The update **ignores** this term — taking only "part of the gradient". Hence the name.

Consequence: convergence guarantees are weaker than for true SGD. But:
- **Converges reliably** in the linear case.
- **Faster** learning than Gradient MC (lower variance, can learn online during the episode).
- **Online / continual** — no need to wait for the end of an episode → handles **continuing problems**.

```
Semi-gradient TD(0) for Estimating v̂ ≈ vπ

Input: the policy π to be evaluated
Input: a differentiable v̂ : 𝒮⁺ × ℝᵈ → ℝ such that v̂(terminal, ·) = 0
Algorithm parameter: step size α > 0
Initialize weights w ∈ ℝᵈ arbitrarily (e.g., w = 0)

Loop for each episode:
    Initialize S
    Loop for each step of episode:
        Choose A ~ π(·|S)
        Take action A, observe R, S′
        w ← w + α[R + γ v̂(S′, w) − v̂(S, w)] ∇v̂(S, w)
        S ← S′
    until S is terminal
```

### 4.6. Gradient MC vs Semi-gradient TD — comparison

| Aspect | **Gradient Monte Carlo** | **Semi-gradient TD** |
|---|---|---|
| Target `Uₜ` | `Gₜ` (full return) | `R_{t+1} + γ v̂(S_{t+1}, w)` (bootstrap) |
| Estimate of gradient | **Unbiased** | **Biased** (target itself biased because `v̂` imperfect) |
| Convergence guarantee | To a **local minimum** of `VE` | Not guaranteed to reach a local minimum of `VE` (converges to TD fixed point, see Sec. 5.4) |
| Variance | Higher | Lower |
| Online? | No (needs full episode) | Yes |
| Speed in practice | Slower in early learning | **Faster** in early learning |

Experiment (1000-state random walk, state aggregation, 30 episodes, best `α`): TD `α = 0.22` learns faster than MC `α = 0.01`. **MC is better asymptotically; TD is better in practice** because we rarely run to asymptote.

---

## 5. Linear Methods

### 5.1. Linear approximation

For each state, a feature vector `x(s) = (x₁(s), …, xd(s))ᵀ`. Linear methods approximate:

```math
\hat v(s, \mathbf{w}) \doteq \mathbf{w}^T \mathbf{x}(s) \doteq \sum_{i=1}^d w_i x_i(s)
```

- Each `xᵢ : 𝒮 → ℝ` is a **feature** (basis function).
- Constructing `d`-dimensional features ≡ choosing `d` basis functions.
- **Linear in the weights**, not necessarily in the inputs.

### 5.2. SGD for linear methods

Gradient is the feature vector itself:

```math
\nabla \hat v(s, \mathbf{w}) = \mathbf{x}(s)
```

The general SGD update specialises to:

```math
\mathbf{w}_{t+1} \doteq \mathbf{w}_t + \alpha \bigl[U_t - \hat v(S_t, \mathbf{w}_t)\bigr] \mathbf{x}(S_t)
```

This is the most analytically tractable case — almost all known convergence results are for linear (or simpler) FA.

### 5.3. Linear semi-gradient TD(0)

```math
\mathbf{w}_{t+1} \doteq \mathbf{w}_t + \alpha \delta_t \mathbf{x}(S_t), \quad
\delta_t = R_{t+1} + \gamma \hat v(S_{t+1}, \mathbf{w}_t) - \hat v(S_t, \mathbf{w}_t)
```

`δₜ` is the TD error. **Converges** to a **TD fixed point** (not the global VE optimum).

Linear TD is a **strict generalisation** of tabular TD and TD with state aggregation: choose indicator features, and the update `wᵢ ← wᵢ + α δₜ` recovers the tabular rule.

### 5.4. Expected TD update and the TD fixed point

Expanding the linear semi-gradient TD update and taking expectations:

```math
\mathbb{E}[\Delta \mathbf{w}_t] = \alpha(\mathbf{b} - \mathbf{A}\mathbf{w}_t)
```

where:

```math
\mathbf{b} \doteq \mathbb{E}[R_{t+1} \mathbf{x}_t] \in \mathbb{R}^d, \qquad
\mathbf{A} \doteq \mathbb{E}[\mathbf{x}_t(\mathbf{x}_t - \gamma \mathbf{x}_{t+1})^T] \in \mathbb{R}^{d \times d}
```

At convergence `E[Δw_TD] = 0`, so:

```math
\mathbf{w}_{TD} = \mathbf{A}^{-1} \mathbf{b}
```

(assuming `A` invertible). `w_TD` is the **TD fixed point**.

- `w_TD` minimises `(b − Aw)ᵀ(b − Aw)` — extends the Bellman-equation connection to the FA setting.
- **VE bound at the TD fixed point**:

```math
\overline{VE}(\mathbf{w}_{TD}) \leq \tfrac{1}{1-\gamma} \min_{\mathbf{w}} \overline{VE}(\mathbf{w})
```

So linear semi-gradient TD reaches a VE no worse than `1/(1−γ)` times the minimum achievable. As `γ → 1`, this bound becomes loose — motivates the average-reward setting (Sec. 9).

---

## 6. Feature Construction for Linear Methods

The features chosen control **what can be represented** and **how the method generalises**. Bad features → linear FA fails (Sec. 1.3).

### 6.1. Coarse coding

- Each feature corresponds to a **circle** (or other receptive field) in state space.
- If state `s` is inside circle `i`, feature `xᵢ(s) = 1` ("present"); otherwise 0 ("absent").
- Such a 0/1 feature is a **binary feature**.
- A state is represented by the **set of overlapping circles** containing it → **coarse coding**.
- Two nearby states share many overlapping circles → strong generalisation between them.

**Receptive field shape controls generalization**:

| Layout | Generalization |
|---|---|
| Many small circles | **Narrow** generalization (only very near states affected) |
| Few large circles | **Broad** generalization (distant states affected) |
| Stretched ellipses (e.g. horizontal) | **Asymmetric** — broad along one axis, narrow along the other |

All three cases can have **the same total number of features** — what matters is the **size and shape of the receptive fields**.

### 6.2. Tile coding

- The receptive fields are **squares (tiles)** partitioning the state space — each partition is a **tiling**.
- Multiple **overlapping, offset** tilings cover the space → each state is in exactly one tile per tiling.
- Example: 2-D state space, 4 tilings → each point activates exactly **4 features** (the 4 overlapping tiles).
- Computationally cheap: features are **sparse** (only one per tiling is active), and indexing into a tile is constant-time.

**Why it works** (1000-state random walk, MC, 5000 episodes):
- Single tiling (state aggregation, one feature per group) → coarse final fit.
- 50 tilings, each width 200, offset by 4 states → **lower asymptotic** `√VE`.
- Step size adjusted by the number of tilings: `α = 0.0001` (single) vs `α = 0.0001/50` (50 tilings) so per-update magnitude matches.

### 6.3. Other feature families (mentioned)

- **Radial basis functions (RBFs)** — graded (continuous) values, like soft circles.
- **Polynomial features** — products of state components.
- **Fourier basis** — cosine features at various frequencies.
- **Kernel methods** / **least-squares TD (LSTD)** — direct closed-form computation of `w_TD = A⁻¹b` instead of incremental learning.

(The lecture slides treat coarse coding and tile coding in detail; the rest are listed as alternatives. Feature construction is, in general, the lever that determines the quality of linear FA.)

---

## 7. Nonlinear Function Approximation

### 7.1. Artificial Neural Networks

- A neural network takes the state vector `s` as input, passes it through **layers** of weighted connections, and outputs `v̂(s, w)`.
- All connections correspond to **real-valued weights** — these are the components of `w`.
- The network is **differentiable** → SGD / semi-gradient still works.

### 7.2. Deep Neural Networks

- A neural network with **a single hidden layer** can approximate any continuous function given sufficient width → **Universal Approximation Theorem**.
- In practice, **deep** networks (many layers) approximate complex functions more easily because depth allows **composition of features** — early layers learn modular components, later layers combine them into specialised ones.
- Example shown: LeNet-style CNN (input 32×32 → conv C1 → subsample S2 → conv C3 → subsample S4 → FC C5 → FC F6 → output 10). Depth significantly improves the agent's ability to **learn features**.

---

# Part II — On-policy Control with Approximation

## 8. Episodic SARSA with Function Approximation

### 8.1. Setup

- Now approximate the **action-value** function: `q̂(s, a, w) ≈ q*(s, a)` with `w ∈ ℝᵈ`.
- Follow the same **on-policy GPI** pattern: estimate `qπ` for the current policy, then improve `π` to be ε-greedy w.r.t. `q̂`.
- Action-value targets `Uₜ` can be any approximation of `qπ(Sₜ, Aₜ)`: full MC return `Gₜ`, n-step return, SARSA target, etc.

### 8.2. General action-value gradient update

```math
\mathbf{w}_{t+1} \doteq \mathbf{w}_t + \alpha \bigl[U_t - \hat q(S_t, A_t, \mathbf{w}_t)\bigr] \nabla \hat q(S_t, A_t, \mathbf{w}_t)
```

### 8.3. Episodic semi-gradient (one-step) SARSA

Set `Uₜ = R_{t+1} + γ q̂(S_{t+1}, A_{t+1}, w)`:

```math
\mathbf{w}_{t+1} \doteq \mathbf{w}_t + \alpha \bigl[R_{t+1} + \gamma \hat q(S_{t+1}, A_{t+1}, \mathbf{w}_t) - \hat q(S_t, A_t, \mathbf{w}_t)\bigr] \nabla \hat q(S_t, A_t, \mathbf{w}_t)
```

- For a **constant policy**, this method converges in the same way TD(0) does, with the same kind of error bound.
- **Policy improvement**: with a **small discrete** action set, compute `q̂(Sₜ, a, w)` for each `a` and pick `A*ₜ = argmaxₐ q̂(Sₜ, a, w)`. Then act ε-greedily.
- **Continuous / very large action sets**: open research, no clean resolution.

```
Episodic Semi-gradient Sarsa for Estimating q̂ ≈ q*

Input: a differentiable q̂ : 𝒮 × 𝒜 × ℝᵈ → ℝ
Algorithm parameters: step size α > 0, small ε > 0
Initialize weights w ∈ ℝᵈ arbitrarily (e.g., w = 0)

Loop for each episode:
    S, A ← initial state and action of episode (e.g., ε-greedy)
    Loop for each step of episode:
        Take action A, observe R, S′
        If S′ is terminal:
            w ← w + α[R − q̂(S, A, w)] ∇q̂(S, A, w)
            Go to next episode
        Choose A′ as a function of q̂(S′, ·, w)   (e.g., ε-greedy)
        w ← w + α[R + γ q̂(S′, A′, w) − q̂(S, A, w)] ∇q̂(S, A, w)
        S ← S′
        A ← A′
```

### 8.4. Example — Mountain Car

- Underpowered car at the bottom of a valley. Gravity > engine; full throttle alone cannot climb the right slope.
- **Solution policy**: drive **away** from the goal (up the left slope) to build inertia, then full-throttle right.
- **Reward**: `−1` per step until the goal at the top of the right mountain. Encourages short episodes.
- **Actions**: full throttle forward (+1), full throttle reverse (−1), zero throttle (0).
- **Dynamics**:

```math
x_{t+1} = \text{bound}\bigl[x_t + \dot x_{t+1}\bigr], \quad
\dot x_{t+1} = \text{bound}\bigl[\dot x_t + 0.001 A_t - 0.0025\cos(3x_t)\bigr]
```

  with bounds `−1.2 ≤ x_{t+1} ≤ 0.5` and `−0.07 ≤ ẋ_{t+1} ≤ 0.07`. Each episode starts at random `x ∈ [−0.6, −0.4)` with zero velocity.

- **Features**: grid-tile coding on `(x, ẋ)`, **8 tilings**, each covering `1/8` of the bounded distance in each dimension with **asymmetric offsets**. Combined linearly:

```math
\hat q(s, a, \mathbf{w}) \doteq \mathbf{w}^T \mathbf{x}(s, a) = \sum_{i=1}^d w_i x_i(s, a)
```

- **Result**: cost-to-go surface `−maxₐ q̂(s, a, w)` over `(position, velocity)`. Early (step 428): nearly flat. Later (ep. 12, 104, 1000, 9000): a clear valley develops near the goal as the agent learns the inertia trick.
- **Learning curves**: steps-per-episode (log scale) shows clear decrease over 500 episodes with `α ∈ {0.1/8, 0.2/8, 0.5/8}` (per-tile rate); larger `α` → faster initial learning. Averaged over 100 runs.

### 8.5. From SARSA to Expected SARSA and Q-learning

Same template, different target `Uₜ`.

| Method | Target `Uₜ` | Approximation update term |
|---|---|---|
| **SARSA** | `R + γ q̂(S′, A′, w)` | uses **sampled** next action `A′` |
| **Expected SARSA** | `R + γ Σ_{a′} π(a′\|S′) q̂(S′, a′, w)` | **expectation** under target policy |
| **Q-learning** | `R + γ maxₐ′ q̂(S′, a′, w)` | **greedy** target (off-policy) |

Expected SARSA with FA:

```math
\mathbf{w}_{t+1} \leftarrow \mathbf{w}_t + \alpha \Bigl[R_{t+1} + \gamma \sum_{a'} \pi(a'|S_{t+1}) \hat q(S_{t+1}, a', \mathbf{w}_t) - \hat q(S_t, A_t, \mathbf{w}_t)\Bigr] \nabla \hat q(S_t, A_t, \mathbf{w}_t)
```

Q-learning with FA:

```math
\mathbf{w}_{t+1} \leftarrow \mathbf{w}_t + \alpha \Bigl[R_{t+1} + \gamma \max_{a'} \hat q(S_{t+1}, a', \mathbf{w}_t) - \hat q(S_t, A_t, \mathbf{w}_t)\Bigr] \nabla \hat q(S_t, A_t, \mathbf{w}_t)
```

---

## 9. Average Reward — Continuing Tasks

### 9.1. Motivation: the "nearsighted MDP"

- Two rings of states intersecting at a hub state `S`. The only choice is at `S`: go **L** (left ring) or **R** (right ring).
- Rewards are 0 everywhere except: **+1 immediately after `S` going L**, and **+2 immediately before `S` returning from R**.
- Each ring is length 5.

**Discounted values at `S`** (from the slides):

```math
V_L(S) = \frac{1}{1 - \gamma^5}, \quad V_R(S) = \frac{2 \gamma^4}{1 - \gamma^5}
```

| `γ` | `V_L(S)` | `V_R(S)` | Better policy |
|---|---|---|---|
| 0.5 | ≈ 1 | ≈ 0.1 | **L** |
| 0.9 | ≈ 2.4 | ≈ 3.2 | **R** |

`V_R > V_L` iff `γ > 2⁻¹/⁴ ≈ 0.814`. If we **stretch each ring** to 100 states, the crossover moves to `γ > 2⁻¹/⁹⁹ ≈ 0.993`. So **with FA you have to push γ extremely close to 1** to make discounting prefer the truly better long-run policy. **Problematic**.

### 9.2. The average-reward objective

For continuing tasks (no terminal state), define the **average rate of reward** under `π`:

```math
r(\pi) \doteq \lim_{h \to \infty} \frac{1}{h} \sum_{t=1}^h \mathbb{E}[R_t \mid S_0, A_{0:t-1} \sim \pi]
\;=\; \sum_s \mu_\pi(s) \sum_a \pi(a|s) \sum_{s', r} p(s', r | s, a)\, r
```

- `μπ(s)` = stationary on-policy distribution.
- Quality of `π` = **average rate of reward** along trajectories; **no discounting**.
- In the nearsighted MDP: `r(πL) = 1/5 = 0.2`, `r(πR) = 2/5 = 0.4` → R is unambiguously better.

### 9.3. Differential returns

Replace `Gₜ = Σ γᵏ R_{t+k+1}` by the **differential return**:

```math
G_t \doteq (R_{t+1} - r(\pi)) + (R_{t+2} - r(\pi)) + (R_{t+3} - r(\pi)) + \cdots
```

— each reward is shifted by the average rate before being summed. This makes the sum **finite**.

**Numerical example** (nearsighted MDP, ring length 5):

- Under `πL` with `r(πL) = 0.2`, starting just after L decision:
  - Rewards: `1, 0, 0, 0, 0, 1, 0, …` → `Gₜ = (1−0.2) + (0−0.2) + (0−0.2) + (0−0.2) + (0−0.2) + … = 0.4` (sum over one cycle).
- Under `πR` with `r(πR) = 0.4`, starting just after R decision:
  - Rewards: `0, 0, 0, 0, 2, 0, 0, …` → `Gₜ = (0−0.4) + (0−0.4) + (0−0.4) + (0−0.4) + (2−0.4) + … = −0.8` (over one cycle).
- (The slides also show the **same trajectories evaluated under the *wrong* `r(π)`** — using `r(πL) = 0.4` on the L-trajectory yields `Gₜ = −1.8`, and using `r(πR) = 0.2` on the R-trajectory yields `Gₜ = 1.4` — illustrating that differential values are **relative to the policy's own average rate**.)

### 9.4. Differential value functions

Defined exactly as before, with the new return:

```math
v_\pi(s) \doteq \mathbb{E}[G_t \mid S_t = s], \qquad q_\pi(s, a) \doteq \mathbb{E}[G_t \mid S_t = s, A_t = a]
```

These are the **differential value functions**. Optima `v*`, `q*` defined similarly. They satisfy **differential Bellman equations**.

### 9.5. Differential Bellman equations

Remove all `γ`'s and subtract `r(π)` from each reward:

```math
v_\pi(s) = \sum_a \pi(a|s) \sum_{r, s'} p(s', r | s, a)\bigl[r - r(\pi) + v_\pi(s')\bigr]
```

```math
q_\pi(s, a) = \sum_{r, s'} p(s', r | s, a)\Bigl[r - r(\pi) + \sum_{a'} \pi(a'|s') q_\pi(s', a')\Bigr]
```

```math
v_*(s) = \max_a \sum_{r, s'} p(s', r | s, a)\bigl[r - \max_\pi r(\pi) + v_*(s')\bigr]
```

```math
q_*(s, a) = \sum_{r, s'} p(s', r | s, a)\bigl[r - \max_\pi r(\pi) + \max_{a'} q_*(s', a')\bigr]
```

### 9.6. Differential TD errors

Two-form TD error using an online estimate `R̄ₜ` of `r(π)`:

```math
\delta_t \doteq R_{t+1} - \overline{R}_t + \hat v(S_{t+1}, \mathbf{w}_t) - \hat v(S_t, \mathbf{w}_t)
```

```math
\delta_t \doteq R_{t+1} - \overline{R}_t + \hat q(S_{t+1}, A_{t+1}, \mathbf{w}_t) - \hat q(S_t, A_t, \mathbf{w}_t)
```

Most algorithms and theory **carry over unchanged** by substituting these for the discounted TD errors.

### 9.7. Differential semi-gradient SARSA

Standard semi-gradient SARSA update, but with the **differential** TD error and a **parallel update of `R̄`**:

```math
\mathbf{w}_{t+1} \doteq \mathbf{w}_t + \alpha \delta_t \nabla \hat q(S_t, A_t, \mathbf{w}_t)
```

```
Differential Semi-gradient SARSA for Estimating q̂ ≈ q*

Input: differentiable q̂ : 𝒮 × 𝒜 × ℝᵈ → ℝ
Algorithm parameters: step sizes α, β > 0
Initialize weights w ∈ ℝᵈ arbitrarily (e.g., w = 0)
Initialize average reward estimate R̄ ∈ ℝ arbitrarily (e.g., R̄ = 0)

Initialize state S, action A
Loop for each step:
    Take action A, observe R, S′
    Choose A′ as a function of q̂(S′, ·, w)   (e.g., ε-greedy)
    δ ← R − R̄ + q̂(S′, A′, w) − q̂(S, A, w)
    R̄ ← R̄ + β δ
    w ← w + α δ ∇q̂(S, A, w)
    S ← S′
    A ← A′
```

Two step-sizes: `α` for the weights, `β` for the running average reward.

---

## Key terms (glossary)

- **Value function approximation (VFA)** — replacing tabular `v` / `q` by a parameterized `v̂(s, w)` / `q̂(s, a, w)`.
- **Weight vector `w ∈ ℝᵈ`** — the parameters that are learned; usually `d ≪ |𝒮|`.
- **Generalization** — updating one state's value influences others.
- **Discrimination** — ability to assign different values to two states.
- **Mean Squared Value Error `VE`** — `Σₛ μ(s)[vπ(s) − v̂(s, w)]²`.
- **On-policy distribution `μ`** — fraction of time under `π` spent in each state.
- **SGD** — stochastic gradient descent on `VE` using one sample at a time.
- **Gradient Monte Carlo** — SGD with `Uₜ = Gₜ`; unbiased; converges to a local min of `VE`.
- **Semi-gradient TD** — SGD with bootstrap target `Uₜ = R + γ v̂(S′, w)`; gradient of the target is ignored.
- **TD error `δₜ`** — `R + γ v̂(S′, w) − v̂(S, w)`.
- **TD fixed point** — `w_TD = A⁻¹b`, the limit of linear semi-gradient TD; bounded VE.
- **Feature vector `x(s)`** — `d`-dim representation of state for linear FA.
- **Linear FA** — `v̂(s, w) = wᵀx(s)`; gradient is `x(s)`.
- **Coarse coding** — overlapping receptive fields → binary features; shape controls generalization.
- **Tile coding** — multiple offset partitions ("tilings") of state space; sparse binary features.
- **Receptive field** — region of state space where a feature equals 1.
- **Universal approximation** — a single-hidden-layer NN can approximate any continuous function.
- **Episodic semi-gradient SARSA** — on-policy control with `q̂`, ε-greedy improvement.
- **Expected SARSA / Q-learning with FA** — drop-in target replacements (expectation / max over `a′`).
- **Average reward `r(π)`** — long-run rate of reward under `π`; no discounting.
- **Differential return** — `Σ (R_{t+k+1} − r(π))`; finite for continuing tasks.
- **Differential value function** — expected differential return; satisfies differential Bellman equations.
- **Differential semi-gradient SARSA** — analogue of SARSA in the average-reward setting with two step-sizes `α, β`.

---

## Exam targets (likely written-exam questions)

1. **Define VFA**: write `v̂(s, w) ≈ vπ(s)`, give the indicator-feature linear form that recovers tabular RL, and explain why "changing one weight changes many states".
2. **Contrast generalization and discrimination** with the robot/sensor example; place tabular methods and aggregation on the 2-D axes.
3. **State the VE objective**: write the formula `VE = Σ μ(s)[vπ(s) − v̂(s, w)]²`; explain the role of `μ` (on-policy distribution).
4. **Derive the SGD update** from `½ ∇[vπ(s) − v̂(s,w)]²` and write the Gradient MC update with target `Gₜ`. State why it is unbiased.
5. **Define "semi-gradient"**: write the TD(0) update with `Uₜ = R + γ v̂(S′,w)`, show that `∇Uₜ = γ∇v̂(S′,w) ≠ 0` is **ignored**, and list the two advantages (online, low variance) and one disadvantage (no convergence guarantee at VE local min).
6. **Compare Gradient MC vs Semi-gradient TD** in a table (target, bias, convergence, variance, speed) — use the 1000-state random-walk result.
7. **For linear FA**: write `v̂(s,w) = wᵀx(s)`, give the gradient `∇v̂ = x(s)`, and the linear semi-gradient TD update `w ← w + αδ x(S)`. Then derive `w_TD = A⁻¹b` from `E[Δw] = α(b − Aw)`, and quote the bound `VE(w_TD) ≤ (1/(1−γ)) minₘ VE(w)`.
8. **Feature construction**: explain coarse coding (binary features from overlapping receptive fields) and tile coding (multiple offset partitions). Discuss how field shape (narrow / broad / asymmetric) controls generalization.
9. **Episodic semi-gradient SARSA**: write the algorithm in pseudocode, give the update for one-step SARSA with FA, and the variants for Expected SARSA and Q-learning (just change the next-action term). Walk through the Mountain Car setup (state = (x, ẋ), reward = −1/step, 8 tilings).
10. **Average reward**: motivate via the nearsighted MDP (why discounting breaks down with FA); define `r(π)`, the differential return, the differential TD error, and write the differential semi-gradient SARSA algorithm with the two step-sizes `α, β`.

### Pitfalls

- **Semi-gradient is not a true gradient** — it ignores `∇Uₜ`. It works in the linear case but has no general convergence guarantee at a `VE` minimum.
- The **on-policy distribution** `μ` matters: a state never visited has weight `μ(s) = 0` and contributes nothing to `VE` regardless of its error.
- **Bootstrapping + function approximation + off-policy** = the deadly combination — semi-gradient methods are *only safe on-policy* (mentioned implicitly here, treated in advanced topics).
- The **gradient of `v̂` in linear FA is simply `x(s)`** — don't mix it up with `v̂(s, w)` itself.
- For **tile coding**, scale the step size by the number of tilings (e.g. `α/8` for 8 tilings) — otherwise the per-update magnitude blows up.
- **Discounted setting can be misleading with FA**: the nearsighted MDP shows you may need `γ` astronomically close to 1 to prefer the truly best long-run policy. Use the **average-reward formulation** for continuing problems.
- **Differential return is relative to the policy's own `r(π)`** — evaluating one policy's trajectory under another policy's `r(π)` gives meaningless numbers.
- The **TD fixed point is not the VE minimum** — its `VE` is at most `1/(1−γ)` times the minimum, but the multiplier blows up as `γ → 1`.
- In control with continuous or very large action sets, the `argmaxₐ q̂(s, a, w)` step is **unsolved** in general — only "discrete and not too large" is handled cleanly.
- **MC's "unbiased gradient" doesn't make it faster** — variance dominates in early learning, where TD's biased-but-low-variance updates win.
