# Chapter 2 — Multi-armed Bandits

## Bird's eye view

- The **k-armed bandit** is the simplest non-trivial RL setting: one state, `k` actions, each action draws a reward from its own **stationary** distribution. No transitions — just repeated single-shot decisions.
- This isolates the **evaluative** side of RL (you only learn whether what you did was good, not what was correct) from sequential/state aspects covered later.
- Each action `a` has an unknown true value `q*(a) = E[Rₜ | Aₜ = a]`. The agent maintains an estimate `Qₜ(a)` and acts on it.
- The **sample-average** method estimates `q*(a)` by averaging the rewards observed when `a` was chosen. It converges as the count grows.
- The estimate can be maintained **incrementally** with constant memory: `Qₙ₊₁ = Qₙ + (1/n)[Rₙ − Qₙ]` — the canonical *NewEstimate ← OldEstimate + StepSize · [Target − OldEstimate]* form.
- For **non-stationary** problems, replace `1/n` with a constant step-size `α`, giving an **exponential recency-weighted average** that tracks change.
- The core dilemma: **exploration vs exploitation**. Pure greedy gets stuck on the first action that looked good; pure random never exploits what was learned.
- Four classic action-selection rules covered: **greedy**, **ε-greedy**, **optimistic initial values**, **UCB**, plus the policy-based **gradient bandit** (softmax preferences + stochastic gradient ascent with a baseline).
- The **10-armed testbed** is the empirical benchmark: 2000 random problems with `q*(a) ∼ 𝒩(0, 1)` and rewards `∼ 𝒩(q*(a), 1)`.

---

## 1. Introduction

- In RL the agent **generates its own training data by interacting with the world**.
- It must learn the consequences of its actions through **trial and error**, rather than being told the correct action (the *evaluative* feedback of RL vs. the *instructive* feedback of supervised learning).
- This chapter studies that evaluative aspect in a simplified **stateless** setting — the **bandit**.

---

## 2. The k-armed Bandit Problem

### 2.1. Setup

- A **decision-maker (agent)** repeatedly chooses among `k` options (the **actions / arms**).
- After choosing action `Aₜ`, the agent receives a numerical reward `Rₜ` drawn from a **stationary probability distribution** that depends only on the chosen action.
- **Goal**: maximize expected total reward over a horizon (e.g. 1000 time steps).
- The "slot-machine" metaphor: each arm has a hidden payout distribution; pull the right ones often enough.

### 2.2. Action-values

- The value of action `a` is the **expected reward** when `a` is taken:

```math
q_*(a) \doteq \mathbb{E}[R_t \mid A_t = a] = \sum_r p(r \mid a)\, r \qquad \forall a \in \{1, \dots, k\}
```

- The agent's goal can be written:

```math
\arg\max_a q_*(a)
```

- Worked example (slide 9):

| Action | Distribution support | `q*(a)` |
|---|---|---|
| 1 (pink) | `{-11, 9}` with `p = 0.5, 0.5` | `0.5·(-11) + 0.5·9 = -1` |
| 2 (yellow) | bell-shaped over `[-3, 5]` | `1` |
| 3 (blue) | uniform on `[1, 5]` | `3` |

### 2.3. Applications

Bandits are a workhorse in:
1. Online advertising (which banner to show)
2. Clinical trials (which treatment to assign)
3. Recommendation systems
4. Portfolio optimization in finance
5. Dynamic pricing in e-commerce
6. Traffic routing and navigation
7. Fraud detection

---

## 3. Estimating Action-Values

### 3.1. Why we need an estimate

- `q*(a)` is **unknown to the agent** — like a doctor who doesn't know each treatment's true cure rate.
- So we work with an estimate `Qₜ(a) ≈ q*(a)` and improve it as we collect data.

### 3.2. The Sample-Average method

The simplest estimator: average all rewards obtained from action `a` before step `t`.

```math
Q_t(a) \doteq \frac{\text{sum of rewards when } a \text{ taken prior to } t}{\text{number of times } a \text{ taken prior to } t} = \frac{\sum_{i=1}^{t-1} R_i \cdot \mathbb{1}[A_i = a]}{\sum_{i=1}^{t-1} \mathbb{1}[A_i = a]}
```

If `a` has never been taken, the denominator is 0 and `Qₜ(a)` is taken to be a default (e.g. 0).

By the **law of large numbers**, `Qₜ(a) → q*(a)` as the count of `a` goes to infinity.

### 3.3. Medical-trial walkthrough (slides 14-27)

- Three treatments P, Y, B. Reward = 1 if patient cured else 0. True values: `0.25, 0.75, 0.50`.
- Start with `Q₀(P) = Q₀(Y) = Q₀(B) = 0` and update with `Qₜ(a) = (Σ rewards) / (count)`.

| Step `t` | Action | Reward | `Q(P)` | `Q(Y)` | `Q(B)` |
|---|---|---|---|---|---|
| 0 | — | — | 0.0 | 0.0 | 0.0 |
| 1 | P | 1 | 1.0 | 0.0 | 0.0 |
| 2 | P | 0 | 0.5 | 0.0 | 0.0 |
| 3 | Y | 1 | 0.5 | 1.0 | 0.0 |
| 4 | B | 0 | 0.5 | 1.0 | 0.0 |
| 5 | Y | 0 | 0.5 | 0.5 | 0.0 |
| 6 | B | 1 | 0.5 | 0.5 | 0.5 |
| 7 | P | 0 | 0.33 | 0.5 | 0.5 |
| 8 | B | 1 | 0.33 | 0.5 | 0.66 |
| 9 | Y | 1 | 0.33 | 0.66 | 0.66 |
| 10 | B | 0 | 0.33 | 0.66 | 0.5 |
| 11 | P | 0 | 0.25 | 0.66 | 0.5 |
| 12 | Y | 1 | 0.25 | 0.75 | 0.5 |

After 12 steps the estimates have converged near the true values `(0.25, 0.75, 0.50)` — Y looks best.

---

## 4. Action Selection

### 4.1. Greedy action selection

- The simplest rule: pick the action with the **largest current estimate**.

```math
A_t \doteq \arg\max_a Q_t(a)
```

- Selecting the greedy action = **exploiting** current knowledge: extract the most reward right now.
- If several actions are tied, break ties arbitrarily (often randomly).

In the trial above, at `t = 12` the greedy action is **Y** (estimate `0.75`); P and B are non-greedy.

### 4.2. ε-greedy action selection

- Pure greedy never explores → it can lock onto a suboptimal arm if that arm happened to look good early.
- **ε-greedy** behaves greedily most of the time, but with small probability `ε` selects uniformly at random over **all** actions (including the greedy one).

```math
A_t = \begin{cases} \arg\max_a Q_t(a) & \text{with probability } 1 - \varepsilon \\ \text{random action} & \text{with probability } \varepsilon \end{cases}
```

- **Convergence guarantee** (in the limit, `t → ∞`):
  - every action is sampled an infinite number of times,
  - so every `Qₜ(a) → q*(a)`,
  - so the probability of picking the optimal action converges to **`> 1 − ε`** (near certainty).

#### Exercise (slide 31)

> With `k = 2` actions and `ε = 0.5`, what is the probability the *greedy* action is chosen?

`P(greedy) = (1 − ε) + ε · (1/k) = 0.5 + 0.5 · 0.5 = 0.75`. So `P(greedy) = 75 %`, `P(explore-the-other) = 25 %`.

---

## 5. Incremental Implementation

### 5.1. Why incremental

- Naïve sample-average: store every reward and re-sum → O(n) memory, O(n) work per step.
- We want **constant memory** and **constant per-step compute**.

### 5.2. The derivation

Let `Rᵢ` be the reward after the `i`-th selection of a fixed action, and `Qₙ` the estimate after `n − 1` selections:

```math
Q_n \doteq \frac{R_1 + R_2 + \cdots + R_{n-1}}{n - 1}
```

Then for the next step:

```math
\begin{aligned}
Q_{n+1} &= \frac{1}{n}\sum_{i=1}^{n} R_i \\
        &= \frac{1}{n}\!\left(R_n + \sum_{i=1}^{n-1} R_i\right) \\
        &= \frac{1}{n}\!\left(R_n + (n-1) Q_n\right) \\
        &= Q_n + \frac{1}{n}[R_n - Q_n]
\end{aligned}
```

- Only `Qₙ` and the count `n` need to be stored — constant memory, constant compute per step.

### 5.3. The general update form

```math
\text{NewEstimate} \leftarrow \text{OldEstimate} + \text{StepSize} \cdot \bigl[\text{Target} - \text{OldEstimate}\bigr]
```

- `[Target − OldEstimate]` is an **error** in the estimate; we move toward `Target`.
- For sample-average, `Target = Rₙ` and `StepSize = 1/n`.
- In general `StepSize` is a parameter `αₙ ∈ [0, 1]` (possibly depending on `n`).

### 5.4. A simple bandit algorithm (pseudo-code, slide 36)

```
Initialize, for a = 1 to k:
    Q(a) ← 0
    N(a) ← 0

Loop forever:
    A ← argmax_a Q(a)         with prob. 1 − ε     (ties broken randomly)
        random action          with prob. ε
    R ← bandit(A)
    N(A) ← N(A) + 1
    Q(A) ← Q(A) + (1/N(A)) · [R − Q(A)]
```

`bandit(A)` is the black-box environment that takes an action and returns its reward.

---

## 6. Non-stationary Bandit Problem

### 6.1. When `q*(a)` drifts

- Sample-average is right for **stationary** bandits (reward distributions fixed forever).
- In RL we often face **non-stationary** problems — distributions drift. Then it is sensible to **weight recent rewards more** than ancient ones.

### 6.2. Constant step-size update

Replace the `1/n` shrinking step with a **constant** `α ∈ (0, 1]`:

```math
Q_{n+1} = Q_n + \alpha [R_n - Q_n]
```

### 6.3. Exponential recency-weighted average

Unrolling the recursion:

```math
\begin{aligned}
Q_{n+1} &= Q_n + \alpha[R_n - Q_n] \\
        &= \alpha R_n + (1 - \alpha) Q_n \\
        &= \alpha R_n + (1 - \alpha)\alpha R_{n-1} + (1 - \alpha)^2 Q_{n-1} \\
        &\;\;\vdots \\
        &= (1 - \alpha)^n Q_1 + \sum_{i=1}^{n} \alpha (1 - \alpha)^{n-i} R_i
\end{aligned}
```

Reading the closed form:
- The **initial estimate `Q₁`** contributes `(1 − α)ⁿ` — **decays exponentially** with steps.
- The reward `Rᵢ` contributes `α(1 − α)ⁿ⁻ⁱ` — older rewards weighted **exponentially less**.
- The most recent rewards dominate the current estimate → the agent tracks changes.
- The weights sum to 1, so `Qₙ₊₁` is genuinely a (weighted) **average**.

### 6.4. Sample-average vs constant-step comparison

| Rule | StepSize | Use for | Initial-value bias |
|---|---|---|---|
| Sample average | `αₙ = 1/n` | **Stationary** problems | Disappears after each action is taken once |
| Constant α | `αₙ = α` (fixed) | **Non-stationary** problems | **Permanent** |

---

## 7. Exploration vs Exploitation Trade-off

### 7.1. The dilemma restated

- **Exploration** = try non-greedy actions to **improve the accuracy** of `Q(a)` and gain **long-term** benefit.
- **Exploitation** = pick the current greedy action to **maximize immediate reward**.
- Pure exploitation may lock in a sub-optimal arm; pure exploration leaves reward on the table.
- The question: **when** to explore, **when** to exploit?

### 7.2. ε-greedy revisited

- Already introduced (§4.2). Picks uniformly random with prob. `ε`, greedy otherwise. Indiscriminate exploration — all non-greedy actions are equally likely.

### 7.3. The 10-armed Testbed (the empirical benchmark)

- **2000** randomly generated `k`-armed bandit problems with `k = 10`.
- For each problem: `q*(a) ∼ 𝒩(0, 1)` (the true values).
- At each step the reward `Rₜ ∼ 𝒩(q*(Aₜ), 1)`.
- Performance is averaged over the 2000 runs.

Empirical findings for sample-average methods:

| ε | Average reward (long run) | % optimal action |
|---|---|---|
| 0 (greedy) | flattens early near ~1.0 | stuck around 30 – 40 % |
| 0.01 | rises slowly, eventually best | rises slowly past 50 % |
| 0.1 | rises fastest, very near optimum | ~80 % |

Intuition: with `ε = 0`, no exploration → estimates that look slightly better stick. With `ε = 0.1` the agent locates the truly best arm quickly; with `ε = 0.01` it locates it more accurately but more slowly.

### 7.4. Optimistic Initial Values

- All methods so far depend (a bit) on the initial estimates `Q₁(a)`.
- For sample-average (`αₙ = 1/n`), the **bias vanishes** once every action has been chosen once.
- For constant-α the **bias is permanent**.
- The downside: initial values are extra parameters to pick (even setting them all to zero is a choice).
- The upside: they encode **prior knowledge** about the reward scale — and, more cleverly, they can be used to **drive exploration**.

**Optimistic initialization trick**: pick `Q₁(a)` *unrealistically high* (e.g. `+5` when rewards are `𝒩(0, 1)`).
- Whichever action is tried first will produce a reward lower than `+5` → its estimate **falls**.
- Greedy action selection then prefers a still-untried action → all actions get tried early on.

Empirical comparison on the 10-armed testbed (slide 46):
- **Optimistic greedy** (`Q₁ = +5`, `ε = 0`): slow start, then climbs **above** ε-greedy in % optimal action.
- **Realistic ε-greedy** (`Q₁ = 0`, `ε = 0.1`): faster start, eventually plateaus lower.

### 7.5. Upper-Confidence-Bound (UCB) Action Selection

Motivation: ε-greedy forces non-greedy actions to be tried, but **indiscriminately** (uniformly). It would be better to favour the non-greedy actions that have the **most potential of actually being optimal** — i.e. those whose estimate is **uncertain**.

UCB picks:

```math
A_t \doteq \arg\max_a\!\left[Q_t(a) + c\,\sqrt{\frac{\ln t}{N_t(a)}}\right]
```

- `Nₜ(a)` = number of times `a` has been chosen prior to `t`.
- `c > 0` controls **the degree of exploration**.
- The square-root term is a confidence bonus that **shrinks** the more often `a` is tried (uncertainty drops) and **grows** with `ln t` (every other action is being tried, so untouched arms become more interesting over time).
- If `Nₜ(a) = 0`, action `a` is treated as having maximal exploration bonus and is selected.

Difficulties (the lecture flags these):
1. **Non-stationary problems** — confidence bonus doesn't expire, so old data wrongly looks certain.
2. **Large state spaces** — extending UCB beyond bandits to full RL with function approximation is non-trivial.

On the 10-armed testbed (slide 48), **UCB with `c = 2` outperforms ε-greedy with `ε = 0.1`** in average reward after an initial settling phase.

### 7.6. Gradient Bandit Algorithms

Instead of estimating values, learn a **numerical preference** `Hₜ(a)` for each action. Only the *relative* preferences matter (adding a constant to all changes nothing).

Convert preferences to probabilities with the **softmax distribution**:

```math
\Pr\{A_t = a\} \doteq \frac{e^{H_t(a)}}{\sum_{b=1}^{k} e^{H_t(b)}} \doteq \pi_t(a)
```

Learn the `H`'s by **stochastic gradient ascent** on expected reward:

```math
\begin{aligned}
H_{t+1}(A_t) &= H_t(A_t) + \alpha (R_t - \bar{R}_t)\bigl(1 - \pi_t(A_t)\bigr) \\
H_{t+1}(a)   &= H_t(a) - \alpha (R_t - \bar{R}_t)\,\pi_t(a) \qquad \forall a \neq A_t
\end{aligned}
```

- `α > 0` — step size.
- `R̄ₜ` — the **baseline**: average of all rewards up to but not including `t`. Subtracting it does not change the gradient in expectation, but cuts variance dramatically.

Empirical observation (slide 50): on a 10-armed testbed with the `q*(a)` shifted to be near `+4` (so all rewards are positive), the gradient bandit:
- **with baseline** achieves ~80 % optimal action,
- **without baseline** plateaus around 40 %.
- The baseline is what makes the method robust to shifts in the reward level.

---

## 8. Summary — comparing the methods

The lecture closes with a parameter study (slide 51): for each algorithm, sweep its main parameter and plot average reward over the first 1000 steps.

| Algorithm | Parameter swept | Best setting on 10-armed testbed |
|---|---|---|
| **ε-greedy** | `ε` | mid `ε` (~ 1/8 – 1/4) |
| **Gradient bandit** | step-size `α` | mid `α` (~ 1/4 – 1/2) |
| **UCB** | `c` | `c ≈ 1 – 2` |
| **Greedy with optimistic init** | initial `Q₀` | `Q₀ ≈ 1 – 2` |

At their **best parameter settings**, all four are within a narrow band on this benchmark, with **UCB slightly best**, followed by greedy-optimistic, then gradient bandit, then ε-greedy. Each curve is inverted-U: too-low and too-high values both hurt.

---

## Key terms (glossary)

- **k-armed bandit** — `k` actions, one stateless step, stationary reward distributions.
- **Action `Aₜ`**, **reward `Rₜ`** — chosen action and received reward at step `t`.
- **Action value `q*(a)`** — true expected reward of action `a`.
- **Estimate `Qₜ(a)`** — agent's current guess at `q*(a)`.
- **Sample-average** — mean of all rewards seen for `a` so far.
- **Greedy action** — `argmax_a Qₜ(a)`; pure exploitation.
- **ε-greedy** — greedy with prob. `1 − ε`, uniform random with prob. `ε`.
- **Step-size `α` (or `αₙ`)** — weight on the most recent error in the incremental rule.
- **Exponential recency-weighted average** — constant-`α` update; older rewards decay geometrically.
- **Stationary / non-stationary** — reward distributions fixed / drifting over time.
- **Optimistic initial values** — over-estimating `Q₁(a)` to *force* early exploration.
- **UCB** — adds `c · √(ln t / Nₜ(a))` exploration bonus.
- **Preference `Hₜ(a)`** — learnable score that only matters relative; mapped through softmax to a policy.
- **Baseline `R̄ₜ`** — running average reward used to reduce variance of gradient-bandit updates.
- **10-armed testbed** — 2000 random `k = 10` problems, `q*(a) ∼ 𝒩(0, 1)`, `Rₜ ∼ 𝒩(q*(Aₜ), 1)`; the chapter's empirical yardstick.

---

## Exam targets (likely written-exam questions)

1. **Define the k-armed bandit problem** and contrast its feedback (evaluative, single-state) with full RL (evaluative + sequential + stateful).
2. **Write the action-value `q*(a)`** as an expectation and as a sum over `r`; compute `q*(a)` from a given reward distribution (cf. slide 9).
3. **Derive the incremental update** `Qₙ₊₁ = Qₙ + (1/n)[Rₙ − Qₙ]` from the sample average, and identify the *NewEstimate ← OldEstimate + StepSize · [Target − OldEstimate]* form.
4. **Write the ε-greedy rule** and, for given `k` and `ε`, compute `P(greedy)` and `P(any specific non-greedy)` (cf. slide 31 exercise).
5. **Constant step-size update**: derive `Qₙ₊₁ = (1 − α)ⁿ Q₁ + Σ α(1 − α)ⁿ⁻ⁱ Rᵢ`; explain why this is appropriate for non-stationary problems and why it gives a **permanent** initial-value bias.
6. **Compare exploration strategies**: ε-greedy vs optimistic initial values vs UCB vs gradient bandit — one-line mechanism, one-line strength, one-line weakness for each.
7. **Write the UCB rule** with the confidence bonus; explain the meaning of `c`, of `Nₜ(a)`, and of the `ln t` factor.
8. **Write the gradient-bandit update**: softmax over preferences `Hₜ(a)`, then the two-case stochastic-gradient-ascent update with baseline `R̄ₜ`. State why the baseline does not change the expected update but reduces its variance.
9. **Walk through the medical-trial example** (or another worked sample-average example) for 5–10 steps, filling in `Qₜ(a)` after each reward.

### Pitfalls

- **`q*(a)` vs `Qₜ(a)`**: the former is the *true* unknown value, the latter is the agent's estimate. Never swap them in a formula.
- **Bandit ≠ MDP**. There is no state, no transitions, no `γ`. Don't drag concepts from later chapters in.
- **Sample-average is wrong for non-stationary problems** — old rewards drown out new evidence. Use constant `α` instead.
- **Pure greedy** is *not* the same as "the right answer once `Q` has converged" — convergence requires that every action is tried infinitely often, which pure greedy does not guarantee.
- **ε-greedy explores indiscriminately** — it doesn't favour uncertain actions; UCB does.
- **Optimistic initial values** only drive exploration **at the start**; they are a poor long-term exploration mechanism, especially in non-stationary settings.
- **UCB bonus uses `ln t`, not `t`** — confusing the two breaks the rate of exploration decay.
- **Gradient bandits learn preferences, not values** — the `Hₜ(a)` are *not* estimates of `q*(a)`. Only the *differences* between preferences matter.
- **Baseline ≠ value function** — `R̄ₜ` here is just the running mean reward (across actions and time), used as a control variate. It is not `Q` or `V`.
- **The 10-armed testbed averages 2000 runs** — single-run plots are noisy; never compare algorithms from one run.
