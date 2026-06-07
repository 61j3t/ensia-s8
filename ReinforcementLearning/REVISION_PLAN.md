# Reinforcement Learning — One-Day Revision Plan

> 12 chapters, ~721 PDF pages, ~5,915 lines of notes.
> Goal: walk through the whole syllabus once with **active recall** and **interleaving**, not passive re-reading.
> Budget: ~8 h of focused study + ~1.5 h of breaks/meals = a one-day block.

---

## Time inventory

| Block | Window | Activity | Hours |
|---|---|---|---|
| Morning Wave 1 | start → +3 h | Formalism: Intro + MDPs + DP | **3 h** |
| **Lunch + walk** | | | 1 h |
| Afternoon Wave 1 | +1 h → +3.5 h | Model-free trio: Bandits + MC + TD | **2.5 h** |
| **Short break** | | | 15 min |
| Afternoon Wave 2 | +15 min → +2 h | Planning + Approximation start | **2 h** |
| **Dinner + walk** | | | 1 h |
| Evening | +1 h → +3 h | Policy gradient + Advanced + interleaved recall | **3 h** |
| **Total study** | | | **~10.5 h** wall, **~8 h** focused |

Adjust the absolute clock to your own start time — the **relative ordering** is what matters.

---

## Triage — what gets time

### Tier 1 — HIGHEST PRIORITY (math-foundational, everything else builds on these)
- **Ch 3 MDPs** (50-60 min) — Bellman equations are the foundation of *everything* afterwards
- **Ch 6 TD** (45-55 min) — SARSA / Q-learning / Expected SARSA + the MC/TD/DP table
- **Ch 9 Policy Gradient** (50-60 min) — PG theorem + REINFORCE + actor-critic

### Tier 2 — HIGH (algorithm-dense chapters with examinable mechanics)
- **Ch 4 DP** (40-50 min) — Policy Iteration / Value Iteration / GPI
- **Ch 8 VFA** (60-75 min) — semi-gradient TD/SARSA, linear FA, tile coding, average reward
- **Ch 10 part 1** (60-75 min) — DQN, TRPO, PPO, DDPG

### Tier 3 — MEDIUM (focused coverage, single key idea per chapter)
- **Ch 2 Bandits** (30 min) — ε-greedy / UCB / gradient bandit, incremental update form
- **Ch 5 MC** (35-40 min) — first/every-visit, importance sampling ratio
- **Ch 7 Planning** (30-35 min) — Dyna-Q, MCTS 4 steps

### Tier 4 — LOW (conceptual skim)
- **Ch 1 Intro** (15-20 min) — vocabulary + agent taxonomy + 3 dichotomies
- **Ch 10 part 2** (25 min) — World Models / IL / MARL setup
- **Ch 10 part 3** (20-25 min) — QMIX + Options framework

---

## Block-by-block schedule

### Block A — RL Formalism (Morning Wave 1, ~3 h)

> **Theme**: *what is the problem*, and *what equations describe its optimum*.

| Duration | Activity |
|---|---|
| **20 min** | **Ch 1 — Intro** (Tier 4). Read [01_chapter1_intro.md](lectures/01_chapter1_intro.md) bird's-eye + glossary. Recall the 3 dichotomies (Learning/Planning, Explore/Exploit, Prediction/Control). |
| **55 min** | **Ch 3 — Finite MDPs** (Tier 1) ★. Read [03_chapter3_mdps.md](lectures/03_chapter3_mdps.md). Write the Bellman expectation and optimality equations for both `vπ` and `qπ` on paper from memory. Verify against notes. |
| **5 min recall** | Close notes: derive `Gₜ = Rₜ₊₁ + γ Gₜ₊₁` from the definition. Write `π* = argmaxₐ q*(s,a)` and explain why no model is needed. |
| **15 min break** | Stretch, water. |
| **45 min** | **Ch 4 — DP** (Tier 2) ★. Read [04_chapter4_dp.md](lectures/04_chapter4_dp.md). Re-implement iterative policy evaluation pseudocode by hand. Distinguish Policy Iteration vs Value Iteration in **one sentence each**. |
| **10 min recall** | Sketch GPI (two curves converging) and explain how DP, MC, TD, AC are all instances. |
| **30 min interleave drill** | Close all notes. Without peeking, answer: (a) the Bellman optimality eq for `v*`, (b) one full iteration of Policy Iteration on a 3-state MDP you invent, (c) why `v*` exists but `π*` may not be unique. |

### Lunch + walk (1 h)

Don't study during meals. The hippocampus does its filing while you walk.

---

### Block B — Model-Free Trio (Afternoon Wave 1, ~2.5 h)

> **Theme**: how does the agent learn `v` or `q` *without a model*?

| Duration | Activity |
|---|---|
| **30 min** | **Ch 2 — Bandits** (Tier 3). Read [02_chapter2_bandits.md](lectures/02_chapter2_bandits.md). Memorize the 5 action-selection rules (greedy / ε-greedy / optimistic init / UCB / gradient bandit) and the **incremental update**: `NewEst = OldEst + StepSize·[Target − OldEst]`. |
| **40 min** | **Ch 5 — Monte Carlo** (Tier 3). Read [05_chapter5_mc.md](lectures/05_chapter5_mc.md). Write down the importance-sampling ratio `ρ_{t:T-1}` and explain why dynamics `p` cancel. Contrast first-visit vs every-visit in two lines. |
| **5 min recall** | Cliff question: why do MC methods *need* either exploring starts or ε-soft policies? |
| **15 min break** | |
| **55 min** | **Ch 6 — Temporal Difference** (Tier 1) ★★. Read [06_chapter6_td.md](lectures/06_chapter6_td.md). Write **all three** TD-control updates (SARSA / Q-learning / Expected SARSA) from memory — they only differ in the *target*. |
| **15 min recall** | Reproduce the **DP/MC/TD bootstrapping × sampling table** on paper. Explain Cliff Walking: why does Q-learning find a riskier path than SARSA? |

---

### Block C — Planning + Approximation (Afternoon Wave 2, ~2 h)

> **Theme**: scale RL beyond tables.

| Duration | Activity |
|---|---|
| **35 min** | **Ch 7 — Planning & Learning** (Tier 3). Read [07_chapter7_planning.md](lectures/07_chapter7_planning.md). Sketch the **Dyna-Q** loop (4 inner steps). Recite the **4 MCTS steps** (Selection / Expansion / Simulation / Backup). |
| **5 min recall** | Why does Dyna-Q+ add the `κ√τ` bonus? Connect to exploration ladder from Ch 2. |
| **70 min** | **Ch 8 — Value Function Approximation** (Tier 2) ★★ — biggest single chapter. Read [08_chapter8_vfa.md](lectures/08_chapter8_vfa.md). Derive semi-gradient TD(0) update and identify *which* gradient is dropped (and why). Sketch tile coding. |
| **10 min recall** | Three-part question: (a) state the VE objective, (b) write the linear semi-gradient TD update with `x(s)`, (c) write differential semi-gradient SARSA with both step sizes. |

### Dinner + walk (1 h)

Same rule. No notes at the table.

---

### Block D — Policy Methods + Deep RL (Evening, ~3 h)

> **Theme**: parameterized policies and their deep-RL descendants.

| Duration | Activity |
|---|---|
| **55 min** | **Ch 9 — Policy Gradient** (Tier 1) ★. Read [09_chapter9_policy_gradient.md](lectures/09_chapter9_policy_gradient.md). Write the **Policy Gradient Theorem** statement. Write the REINFORCE update + REINFORCE-w/-baseline + one-step actor-critic with TD error δ. |
| **10 min recall** | Why does the baseline `b(s)` *not* introduce bias? (One-line proof.) |
| **15 min break** | |
| **70 min** | **Ch 10 part 1 — Advanced RL** (Tier 2) ★★. Read [10_chapter10_advanced_part1.md](lectures/10_chapter10_advanced_part1.md). Be able to **draw the DQN architecture** (target net + replay buffer). Compare **TRPO vs PPO vs GRPO** in one table. Explain DDPG as "deep Q-learning for continuous actions". |
| **25 min** | **Ch 10 part 2** (Tier 4). Skim [11_chapter10_advanced_part2.md](lectures/11_chapter10_advanced_part2.md). Memorize the 4 IL methods (BC / DAgger / IRL / Apprenticeship) and the 6 MARL dimensions. |
| **25 min** | **Ch 10 part 3** (Tier 4). Skim [12_chapter10_advanced_part3.md](lectures/12_chapter10_advanced_part3.md). Recall **IGM** (Individual-Global-Max) and the Options triple `(I, π, β)`. |
| **20 min final recall** | **Whole-syllabus quiz** (below). Whatever you miss tells you what to look at tomorrow. |

---

## Cross-chapter active-recall drills (do these between blocks)

Don't skip these. They are where the actual learning sticks.

### Drill 1 — *The Bellman ladder* (after Block A)
Without notes, write the **four** Bellman equations:
1. Expectation for `vπ` (linear)
2. Expectation for `qπ` (linear)
3. Optimality for `v*` (non-linear, has `max`)
4. Optimality for `q*` (non-linear, has `max`)

### Drill 2 — *MC × TD × DP* (after Block B)
Fill in this table from memory:
| Method | Bootstrap | Sample | Needs model | Episodic only |
|---|---|---|---|---|
| DP | | | | |
| MC | | | | |
| TD | | | | |

### Drill 3 — *The control-target-only* tabular ladder (after Block B)
Same `Q ← Q + α[target − Q]` skeleton — only the **target** changes:

| Algorithm | Target |
|---|---|
| **SARSA** | |
| **Q-learning** | |
| **Expected SARSA** | |
| **Sarsa(λ)** | (eligibility trace — out of scope here) |

### Drill 4 — *PG variance ladder* (after Block D, Ch 9)
Order these by **variance** (highest to lowest):
- REINFORCE
- REINFORCE with baseline
- One-step actor-critic (TD-error advantage)
- n-step actor-critic
- GAE actor-critic

### Drill 5 — *Whole-syllabus 10-question quiz* (end of day)
1. State the **reward hypothesis** in one sentence.
2. When is a state Markov? Why does it let you discard history?
3. Give the dynamics-form of the Bellman expectation equation for `vπ`.
4. Why does Policy Iteration **terminate** in finitely many steps on a finite MDP?
5. Importance-sampling ratio `ρ_{t:T-1}` — write it, explain why `p` cancels.
6. SARSA vs Q-learning target — which makes Q-learning *off-policy*?
7. Why is **semi-gradient TD** called "semi-"?
8. State the **Policy Gradient Theorem**.
9. Two tricks that turn `Q-learning` into `Vanilla DQN`?
10. PPO-Clip — what's the surrogate objective?

If you can answer all 10 from memory, you know the syllabus. If you miss 3+, that's where to put extra time tomorrow.

---

## High-yield cheatsheet (write this on a single page)

### The five core equations
```math
G_t = R_{t+1} + \gamma G_{t+1}                                              \quad\text{(return recursion)}
```
```math
v_\pi(s) = \sum_a \pi(a|s) \sum_{s',r} p(s',r|s,a)[r + \gamma v_\pi(s')]    \quad\text{(Bellman expectation)}
```
```math
v_*(s) = \max_a \sum_{s',r} p(s',r|s,a)[r + \gamma v_*(s')]                 \quad\text{(Bellman optimality)}
```
```math
Q(S,A) \leftarrow Q(S,A) + \alpha[R + \gamma \max_{a'} Q(S',a') - Q(S,A)]   \quad\text{(Q-learning)}
```
```math
\nabla J(\theta) \propto \sum_s \mu(s) \sum_a q_\pi(s,a) \nabla \pi(a|s,\theta)  \quad\text{(PG theorem)}
```

### Algorithm targets — memorize the *right column*
| Algorithm | Update target |
|---|---|
| TD(0) prediction | `R + γ V(S')` |
| SARSA | `R + γ Q(S', A')` |
| Q-learning | `R + γ maxₐ' Q(S', a')` |
| Expected SARSA | `R + γ 𝔼π[Q(S', ·)]` |
| Gradient MC | `Gₜ` |
| REINFORCE | `Gₜ ∇ln π(Aₜ|Sₜ,θ)` |
| Actor-critic | `δₜ ∇ln π(Aₜ|Sₜ,θ)` with `δₜ = R + γv̂(S',w) − v̂(S,w)` |

### Five key numbers
- **γ ∈ [0, 1)** — discount factor; `γ < 1` for continuing tasks
- **First-visit ↔ every-visit** — both converge to `vπ`
- **Replication / blocks** — N/A (that's the BDAV exam)
- **DQN — 2 tricks**: target net + replay buffer
- **PPO-Clip ε** — typically 0.2

---

## Pitfalls (the trap questions)

- **Reward ≠ Return**. `Rₜ` is one scalar; `Gₜ` is the (possibly discounted) sum.
- **`vπ` ≠ `v*`** — never write `v(s)` without saying *under which policy*.
- **DP requires the model**; MC and TD do not. Q-learning is model-free *and* off-policy.
- **Expected SARSA** target uses the *policy's* action distribution — that's why it's lower-variance than SARSA but still on-policy when `π = b`.
- **Off-policy MC**: ordinary IS is **unbiased but possibly infinite variance**; weighted IS is **biased but variance-bounded**.
- **Importance-sampling ratio cancels env dynamics `p`** — only policy ratios survive.
- **Semi-gradient TD** drops `∇` of the bootstrap target — the update is *not* a true gradient of any objective on `w`.
- **TD fixed point ≠ minimum of VE** — bound: `VE(w_TD) ≤ (1/(1−γ)) minw VE(w)`.
- **Policy gradient theorem kills `∇μ(s)`** — that's the whole point of the theorem.
- **REINFORCE baseline** must be a function of state only — *not* of the action — or it introduces bias.
- **DQN** is *not* TD-learning of `v` — it learns `Q`, off-policy, with replay; classic TD assumes sequential data.
- **TRPO/PPO**: the constraint / clip is on the **probability ratio**, not on `θ` directly.
- **DDPG** is deterministic — adds exploration noise externally (e.g., OU noise).
- **Q-learning convergence proof** requires the Robbins-Monro step-size conditions: `Σαₜ = ∞`, `Σαₜ² < ∞`.
- **MCTS != minimax** — MCTS samples; minimax exhausts (no exhaustive search at scale).
- **Dyna-Q+ bonus = `κ√τ`** — pushes you back toward stale `(s,a)`; without it Dyna-Q can't solve the *shortcut* maze.

---

## Tactical advice (from the user-memory revision rules)

1. **Don't re-read PDFs cover-to-cover** — the `.md` notes are denser. PDFs only for specific diagrams (Atari loop, gridworld, Cliff Walking) you find unclear.
2. **Active recall beats re-reading** — after each block, close all notes and try the drill. Get something *wrong* on purpose; that's where learning happens.
3. **Interleave** — Drills 1-4 above mix concepts from multiple chapters. They are the most efficient hour you'll spend.
4. **Hand-write the formulas** — typing doesn't lock in the visual layout the way pen on paper does. Especially the Bellman equations and the PG theorem.
5. **Use the [00_SYLLABUS_MAP.md](lectures/00_SYLLABUS_MAP.md) as a checkpoint** — at the end of each block, mark off which chapter you covered and what you missed.
6. **One pass + drills is enough for a *revision* (no exam pressure)**. If a chapter still feels shaky, schedule a focused 60-min repeat for tomorrow rather than dragging today out.
7. **Sleep > extra hours**. If you start nodding off at hour 7, stop. Long-term retention happens during sleep.

---

## After today

- Where you missed Drill 5 questions → re-read the corresponding chapter `.md` tomorrow (60 min focused).
- Pick one chapter to **Feynman-explain aloud** (record yourself) — usually the one you found *easiest*. The act of teaching exposes gaps you don't see when reading.
- Optional: implement one algorithm from scratch in NumPy (SARSA on CliffWalking, or REINFORCE on CartPole) — turns paper-knowledge into reflex.
