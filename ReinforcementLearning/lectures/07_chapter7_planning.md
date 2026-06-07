# Chapter 7 — Planning and Learning with Tabular Methods

## Bird's eye view

- **Planning** = produce/improve a policy by computing on a **model**. **Learning** = produce/improve a policy from **real experience**. Same value-function updates can be driven by either source.
- A **model** = anything the agent uses to predict the environment's response `(S, A) → (S', R)`. Two flavours: **sample model** (draws one transition) vs **distribution model** (full probabilities of all transitions).
- **Random-sample one-step tabular Q-planning** = run one-step Q-learning updates on **simulated** transitions drawn from a sample model. Converges to optimal policy under the same conditions as real Q-learning.
- **Dyna-Q** = the unifying architecture: at each step do (i) act in the real env, (ii) direct RL update, (iii) update the model, (iv) run `n` extra planning updates on random previously-seen `(s, a)` pairs sampled from the model.
- Planning **dramatically accelerates learning** in the maze example: with `n=50` planning steps an effective policy is built after just 2 episodes, vs many episodes with `n=0`.
- **Dyna-Q+** handles **changing / inaccurate models** by adding an **exploration bonus** `κ√τ` (where `τ` = time since `(s, a)` last tried), pushing the agent to revisit stale parts of the model — solves the **shortcut maze** that vanilla Dyna-Q cannot.
- **Monte Carlo Tree Search (MCTS)** = decision-time planning by repeated rollouts. Four-step loop: **Selection** (tree policy) → **Expansion** → **Simulation** (rollout policy) → **Backup**. Core engine behind AlphaGo.
- All tabular RL methods share three ideas: **approximate value function**, **approximate policy**, **Generalized Policy Iteration (GPI)**. They differ along two main axes: **width** of update and **depth (length)** of update.

---

## 0. Outline of the chapter

- Models and Planning
- Random Tabular Q-planning
- Dyna Algorithm
- When the Model is inaccurate
- Changing environments
- Monte Carlo Tree Search
- Space of reinforcement learning methods

---

## 1. Introduction: model-based vs model-free

- RL methods split by whether they **use a model of the environment** or not.
- **Model-based** methods (Dynamic Programming, Heuristic Search) require a model and rely on **planning**.
- **Model-free** methods (Monte Carlo, Temporal-Difference) rely on **learning** from experience.
- Chapter 7's goal: build a **unified view** of methods that combine **planning and learning** through the **Dyna architecture** and **Monte Carlo Tree Search**.

| | **Model-based** | **Model-free** |
|---|---|---|
| Needs a model? | Yes | No |
| Primary component | **Planning** | **Learning** |
| Examples | DP, heuristic search | MC, TD |

---

## 2. Models and Planning

### 2.1. What is a model?

- A **model** of the environment is anything the agent can use to **predict how the environment will respond to its actions**.
- Functional view: given `(state, action)` → predict `(next state, next reward)`.
- If the model is **stochastic**, there are several possible `(S', R)` pairs, each with some probability.

```
State  ─┐
        ├─→ [ Model ] ─→ Next state, Reward
Action ─┘
```

### 2.2. What is planning?

Planning uses the model to improve the policy **without interacting with the world**:

```
Model ──creates──> Simulated Experience ──updates──> Value function ──modifies──> Policy
                                                                                       ↑
                                                                                       └─── Planning! (Model directly improves Policy)
```

### 2.3. Two types of models

| | **Sample model** | **Distribution model** |
|---|---|---|
| Produces | **One** transition `(S', R)` sampled by underlying probabilities | **All** possible `(S', R)` and their **exact probabilities** |
| Example | Code that flips a coin once → H or T | Full probability tree of 5 coin flips, every leaf at `p = 1/32` |
| Memory | **Less** (more compact) | More |
| Computation per call | **Cheap** (random sample) | Heavier |
| Expected outcome | Approximate by **averaging many samples** | **Exact** — sum over all outcomes weighted by probability |
| Risk assessment | Hard | Easy (you see the full distribution) |

#### 2.3.1. Dice example (12 dice)

- Rolling 12 dice once = a sample model — cheap.
- Modelling the joint distribution of 12 dice = **over 2 billion outcomes** (`2 176 782 336`).
- The distribution model is far more work to build, but lets you compute the **exact expected outcome** or **quantify variability** directly.
- The doctor example: when **prescribing medicine** the distribution model is preferred because you need to consider all side effects and their probabilities (risk).

---

## 3. Random Tabular Q-planning

### 3.1. Idea

- Planning with a model **leverages the model to inform decisions without interacting with the world**.
- **Learning methods need only experience as input** — and that experience can be **real** or **simulated**. So a learning algorithm can be repurposed as a planning algorithm by feeding it simulated transitions.
- **Random-sample one-step tabular Q-planning** = take a one-step Q-learning update, but feed it samples drawn from the model.

### 3.2. Connection with Q-learning

| | **Q-learning** | **Q-planning** |
|---|---|---|
| Source of `(S, A, R, S')` | **Environment** (real experience) | **Model** (simulated experience) |
| Update | Q-learning update | Q-planning update (same formula) |

### 3.3. The algorithm (three steps)

**1. Sampling** — draw a state-action pair from the model:

```
S ~ 𝒮     ──┐
            ├─→ [ Model ] ─→ S', R
A ~ 𝒜     ──┘
```

**2. Q-planning update** (same form as Q-learning):

```math
Q(S, A) \leftarrow Q(S, A) + \alpha \left[R + \gamma \max_a Q(S', a) - Q(S, A)\right]
```

**3. Greedy policy improvement**:

```math
\pi(s) = \arg\max_a Q(s, a)
```

### 3.4. Convergence

- Random-sample one-step tabular Q-planning **converges to the optimal policy for the model** under the same conditions as one-step tabular Q-learning converges to the optimal policy for the real environment:
  - Every `(s, a)` selected an infinite number of times in Step 1.
  - `α` decreases appropriately over time.

### 3.5. Why planning helps

- **Planning uses only imagined experience** — no environment interaction during planning steps.
- Imagine: actions can only be taken at specific time points, but **learning updates run fast**. There's idle time between actions.
- **Fill the idle time with planning updates** that improve the value function further from the same data.

---

## 4. Dyna: Integrated Planning, Acting, and Learning

### 4.1. The Dyna architecture

Dyna **combines Q-learning and Q-planning** by feeding the policy/value functions from **two sources of experience**:

```
                Policy / value functions
                   ↑              ↑
        direct RL update    planning update
                   │              │
        environment experience    model experience
                   ↑              ↑       ↑
                   │              │       │ search control
                   │           model learning
                   │              │
              Environment ────────┘
```

- **Direct RL update**: Q-learning using real experience.
- **Model learning**: store observed transitions to build the model.
- **Search control**: choose `(s, a)` pairs from the model to plan with.
- **Planning update**: Q-planning update using sampled model experience.

### 4.2. Tabular Dyna-Q algorithm

```
Initialize Q(s, a) and Model(s, a) for all s ∈ 𝒮 and a ∈ 𝒜(s)

Loop forever:
  (a) S ← current (nonterminal) state
  (b) A ← ε-greedy(S, Q)
  (c) Take action A; observe resultant reward R and state S'
  (d) Q(S, A) ← Q(S, A) + α [ R + γ maxₐ Q(S', a) − Q(S, A) ]        # direct RL
  (e) Model(S, A) ← R, S'                                            # model learning (deterministic env)
  (f) Loop repeat n times:                                           # planning
        S  ← random previously observed state
        A  ← random action previously taken in S
        R, S' ← Model(S, A)
        Q(S, A) ← Q(S, A) + α [ R + γ maxₐ Q(S', a) − Q(S, A) ]
```

### 4.3. Anatomy of the algorithm

- **Acting**, **model-learning**, and **direct RL** require **little computation** — assumed to take only a fraction of the step time.
- **Planning** is computation-intensive — `n` simulated updates per real step. The extra wall-clock time goes here.
- `Model(s, a)` stores the predicted `(R, S')` for state-action pair `(s, a)`.
- Steps (d), (e), (f) implement direct RL, model learning, and planning respectively.
- **If (e) and (f) are removed → plain one-step tabular Q-learning.**

### 4.4. Example: Dyna Maze

- A 6 × 9 grid with obstacles, 47 states, 4 actions `{up, down, right, left}`. Movement is deterministic. Hitting an obstacle or edge → stay.
- Reward = **0** on all transitions, **+1** on entering the **goal G**. Episode ends at G, then restart at start state **S**.
- Discounted, episodic, `γ = 0.95`.

### 4.5. Effect of planning steps `n`

| `n` (planning steps) | Behaviour |
|---|---|
| `0` (direct RL only) | First episode > 800 steps; slow convergence |
| `5` | Far faster — converges to ~14 steps within ~5 episodes |
| `50` | Fastest — almost optimal after 2 episodes |

| | **Without planning (n=0)** | **With planning (n=50)** |
|---|---|---|
| End of episode 2 | Only **one step (the last)** of the policy is learned | An **extensive policy** is developed that reaches almost back to the start state |

The planning loop **propagates value information backward** through the state-space using imagined trajectories, so a single real-world transition triggers `n` extra updates that spread its effect.

---

## 5. When the Model is Inaccurate

### 5.1. How the model can be wrong

| Type | Cause | Picture |
|---|---|---|
| **Incomplete** | Limited experience → some transitions missing. `S2 → ??` | The model never saw what happens from `S2`. |
| **Incorrect** | Environment changes → stored transitions are now outdated. `S1→S2,+1` stored but the real transition is `S1→S3,+0`. | The model "remembers" an old world. |

### 5.2. Why this is hard — exploration vs exploitation for model accuracy

- Another version of the **exploration vs exploitation** conflict, but in a planning context:
  - **Exploration** = try actions that **improve the model**.
  - **Exploitation** = behave **optimally given the current model**.
- We want the agent to explore enough to detect changes, but not so much that performance is greatly degraded.
- **No perfect-and-practical solution** exists, but simple **heuristics** are often effective. Dyna-Q+ uses one such heuristic.

### 5.3. Dyna-Q+ — bonus reward for stale transitions

- **Track elapsed time** `τ` since each `(s, a)` was last tried in the real environment.
- **Larger `τ` ⇒ higher chance the model is wrong** for that pair.
- During **simulated** experience, augment the reward with a **bonus**:

```math
R^+ = r + \kappa \sqrt{\tau}
```

where:
- `r` = actual stored reward
- `τ` = time steps since the `(s, a)` transition was last tried
- `κ` = a small positive constant

The bonus encourages the agent to **test all accessible state transitions** and discover **new action sequences**, despite the cost of less greedy behaviour.

### 5.4. Dyna-Q vs Dyna-Q+ — the maze experiments

#### Blocking task

- Left environment (initial wall layout) used for first **1000 steps**, then a new (blocked) layout for the rest.
- Both Dyna-Q and Dyna-Q+ adapt, but Dyna-Q+ recovers faster because of the exploration bonus.

#### Shortcut task

- Left environment used for first **3000 steps**, then a **shortcut** is opened on the right side of the wall.
- **Dyna-Q+ quickly identifies and uses the shortcut** after the environment changes.
- **Dyna-Q fails to find the shortcut** within the given time — its model still says "wall here", and exploitation never re-tests that region.
- **Persistent, systematic exploration by Dyna-Q+** is crucial in this environment.

| | **Dyna-Q** | **Dyna-Q+** |
|---|---|---|
| Reward signal in planning | Stored `r` | `r + κ√τ` |
| Behaviour on stale model | Sticks to old "optimal" path | Re-tests long-untried transitions |
| Blocking task | Adapts | Adapts faster |
| Shortcut task | **Misses the shortcut** | **Finds and uses the shortcut** |

---

## 6. Monte Carlo Tree Search (MCTS)

### 6.1. What MCTS is

- A **decision-time planning** method using repeated **Monte Carlo simulations** (rollouts).
- Builds a **search tree** rooted at the current state, focusing simulations on **high-reward trajectories**.
- Uses a **tree policy** (e.g. UCB, ε-greedy) to balance **exploration vs exploitation** within the tree.
- Estimates action values by **averaging simulated returns** (the Monte Carlo principle).
- Effective in **games** (e.g. Go — AlphaGo) and general sequential decision problems where a model is available.

### 6.2. The four steps (repeated while time remains)

```
Selection ──→ Expansion ──→ Simulation ──→ Backup
   ▲                                          │
   └──────────────────────────────────────────┘
                (repeat while time remains)
```

#### 1. Selection

Starting at the **root node**, a **tree policy** based on the action values attached to the edges of the tree traverses the tree to select a **leaf node**.

#### 2. Expansion

On some iterations (depending on the application), the tree is **expanded from the selected leaf** by adding one or more child nodes reached via **unexplored actions**.

#### 3. Simulation

From the selected node (or one of its newly-added children), simulate a **complete episode** using the **rollout policy**.
The result is a **Monte Carlo trial**: actions chosen first by the **tree policy** inside the tree, then by the **rollout policy** beyond the tree.

#### 4. Backup

The **return generated by the simulated episode** is backed up the tree to **update (or initialize) the action values attached to the edges traversed by the tree policy**.
**No values are saved** for states/actions visited by the rollout policy beyond the tree.

### 6.3. Tree policy vs rollout policy

| | **Tree policy** | **Rollout policy** |
|---|---|---|
| Where it runs | Inside the existing tree | Outside the tree, until episode end |
| Goal | Balance exploration vs exploitation | Cheap, fast simulation |
| Examples | UCB, ε-greedy | Random, or a lightweight heuristic |
| Updates stored? | **Yes** — values backed up to tree edges | **No** — discarded after simulation |

### 6.4. AlphaGo case study

David Silver et al., *Mastering the game of Go with deep neural networks and tree search*, **Nature 529, 484–489 (2016)**.

| MCTS step in AlphaGo | What's used |
|---|---|
| **Selection** | Edge picked by `max [ Q + u(P) ]` — action value plus a bonus depending on the **prior probability** `P` stored on the edge |
| **Expansion** | New leaf processed by the **policy network** `p_σ`; output probabilities stored as priors `P` for each action |
| **Evaluation** | Leaf evaluated **two ways**: (1) **value network** `v_θ`, (2) fast **rollout policy** `p_π` to game end then a winner function `r` |
| **Backup** | Action values `Q` updated to track the **mean** of all evaluations `r(·)` and `v_θ(·)` in the subtree below that action |

This is the canonical example showing MCTS scales to **massive state spaces** (Go has more legal positions than atoms in the observable universe) when combined with learned function approximation.

---

## 7. Space of Reinforcement Learning Methods

### 7.1. Three things all tabular methods share

1. **Approximate value function** — they all seek to estimate value functions.
2. **Approximate policy** — they all operate by **backing up values along actual or possible state trajectories**.
3. **Generalized Policy Iteration (GPI)** — they all continually try to improve **each on the basis of the other** (policy ⇌ value).

### 7.2. The two main axes of variation

A slice through the space of RL methods highlights the **two most important dimensions**:

```
       (narrow update)       width of update       (wide update)
            ┌─────────────────────────────────────────────────┐
shallow   │ Temporal-difference learning ······ Dynamic programming
            │
            │
depth       │
(length)    │
of update   │
            │
            │
deep      │ Monte Carlo  ···························· Exhaustive search
            └─────────────────────────────────────────────────┘
```

| Method | Width | Depth | Position |
|---|---|---|---|
| **TD learning** | Sample (narrow) | Shallow (1 step) | Top-left |
| **Dynamic programming** | Full expectation (wide) | Shallow (1 step) | Top-right |
| **Monte Carlo** | Sample (narrow) | Full episode (deep) | Bottom-left |
| **Exhaustive search** | Full expectation (wide) | Full tree (deep) | Bottom-right |

### 7.3. Other dimensions identified through the course

- **Definition of return** — episodic vs continuing, discounted vs undiscounted.
- **Action values vs state values vs afterstate values.**
- **Action selection / exploration.**
- **Synchronous vs asynchronous.**
- **Real vs simulated** experience.
- **Location of updates.**
- **Timing of updates.**
- **Memory for updates.**

---

## Key terms (glossary)

- **Model** — any mechanism the agent can use to predict `(S', R)` given `(S, A)`.
- **Sample model** — produces one sampled `(S', R)`.
- **Distribution model** — produces all possible `(S', R)` with their probabilities.
- **Planning** — improving a policy using a model (no interaction with the real env).
- **Learning** — improving a policy from real experience.
- **Simulated experience** — `(S, A, R, S')` drawn from a model rather than from the world.
- **Direct RL** — learning updates from real experience.
- **Random-sample one-step tabular Q-planning** — Q-learning update applied to random model samples.
- **Search control** — how `(s, a)` pairs are chosen for planning updates.
- **Dyna-Q** — algorithm that interleaves acting, model learning, direct RL, and `n` planning updates per step.
- **Dyna-Q+** — Dyna-Q with an exploration bonus `κ√τ` for long-untried transitions.
- **Exploration bonus** — extra reward in simulated experience that encourages revisiting stale parts of the model.
- **MCTS (Monte Carlo Tree Search)** — decision-time planning by repeated rollouts: Selection → Expansion → Simulation → Backup.
- **Tree policy** — policy used inside the MCTS search tree (e.g. UCB).
- **Rollout policy** — fast policy used outside the tree to complete simulated episodes.
- **GPI (Generalized Policy Iteration)** — value and policy improving each other in a loop.
- **Width / Depth of update** — two main axes of the RL-method space.

---

## Exam targets (likely written-exam questions)

1. **Define a model** and distinguish a **sample model** from a **distribution model**. Give the dice/coin example and explain when each is preferable (memory vs exact expectation vs risk assessment).
2. **State and explain** random-sample one-step tabular Q-planning — write the three steps (sampling / Q-planning update / greedy improvement), give the update equation, and the convergence conditions.
3. **Distinguish planning from learning** — same update formula, different source of experience (model vs environment). Draw the Q-learning vs Q-planning diagrams.
4. **Write the Tabular Dyna-Q pseudocode** and label which step is direct RL, model learning, and planning. Explain what would remain if steps (e) and (f) were removed.
5. **Dyna Maze**: explain why `n = 50` planning steps yields a near-complete policy after 2 episodes, while `n = 0` learns only the last step per episode.
6. **Explain why a Dyna-Q model can be inaccurate** (incomplete vs incorrect), and how this relates to the **exploration vs exploitation conflict in planning context**.
7. **Dyna-Q+**: state the bonus reward formula `R⁺ = r + κ√τ`, explain each term, and describe the **shortcut maze** to show why Dyna-Q+ beats Dyna-Q in changing environments.
8. **MCTS**: describe the four steps (Selection, Expansion, Simulation, Backup), distinguish **tree policy** vs **rollout policy**, and explain what is and is not stored. Bonus: AlphaGo modifications (policy net, value net, prior `P`, `Q + u(P)`).
9. **Position TD, DP, Monte Carlo, and exhaustive search** in the **width-of-update vs depth-of-update** plane. State the three commonalities of all tabular RL methods.

### Pitfalls

- **Planning ≠ Learning** — the *formula* is the same; what differs is whether `(S, A, R, S')` comes from the **model** or the **environment**.
- **Sample model does not give the exact expected value** — you must average many samples. Distribution model can compute it directly.
- **Dyna-Q assumes a deterministic environment** in the simple pseudocode (`Model(S, A) ← R, S'`). For stochastic envs the model must store distributions or use sample-update bookkeeping.
- **Removing steps (e) and (f) collapses Dyna-Q back to one-step tabular Q-learning** — this is the key sanity check.
- **`n` planning steps trade computation for sample efficiency** — they don't reduce the amount of *real* experience needed if the model is wrong; they amplify the value of each real step **only if** the model is reliable.
- **Dyna-Q's exploitation locks in stale models** — once a path looks bad, vanilla Dyna-Q stops testing it (shortcut maze failure). Dyna-Q+ fixes this with `κ√τ`.
- **The bonus `κ√τ` is only added to simulated experience** in planning updates, not to actual rewards received from the environment.
- **MCTS backups only update the tree-policy edges** — values from the rollout-policy portion of the simulated episode are **not stored**. Don't confuse with full Monte Carlo control.
- **The tree policy and the rollout policy are different objects** — tree policy must balance exploration/exploitation (UCB); rollout policy should be cheap (often random or a fast heuristic).
- **MCTS is decision-time planning** — the tree is rebuilt at each new root (each real action taken). Earlier estimates may or may not be reused depending on the implementation.
- **"Width" and "depth" of update are orthogonal axes** — TD is narrow + shallow, DP is wide + shallow, MC is narrow + deep, exhaustive search is wide + deep. Don't confuse with on-/off-policy or bootstrap/no-bootstrap.
