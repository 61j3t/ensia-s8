# Chapter 1 — Introduction to Reinforcement Learning

## Bird's eye view

- **RL** = programming agents by **reward** (no supervisor), trial-and-error, with **delayed**, **sequential**, non-i.i.d. feedback. The agent's actions change the data it later sees.
- Built on the **reward hypothesis**: any goal = maximizing expected **cumulative** scalar reward.
- The world is modelled as an **agent ↔ environment interaction loop**: at each step, agent picks `Aₜ`, environment returns `Oₜ₊₁` and `Rₜ₊₁`.
- **Markov state**: a state is Markov iff `P[Sₜ₊₁ | Sₜ] = P[Sₜ₊₁ | S₁,…,Sₜ]` — once you know the state, the past can be thrown away. Fully observable → **MDP**; partially observable → **POMDP**.
- An RL **agent** can contain three building blocks: **policy** (behaviour), **value function** (goodness), **model** (env. dynamics). It needs at least one.
- Two **agent taxonomies**: value-based / policy-based / actor-critic — and model-free / model-based.
- Two **fundamental problems**: Learning (env. unknown, must interact) vs Planning (env. known, just compute).
- Two **fundamental tensions**: Exploration (gather info) vs Exploitation (use info); Prediction (evaluate policy) vs Control (find best policy).

---

## 0. Course info

- **Instructor**: Dr. Aissa Boulmerka (ENSIA), 2025-2026. Advanced chapters (4, 5, 11) co-presented by Dr. Ferhi Nadir.
- **Textbook**: Sutton & Barto, *Reinforcement Learning: An Introduction*, 2nd ed., MIT Press 2018.
- **Prereqs**: linear algebra, probability, programming.
- **Evaluation**: Labs 5% + Project 15% + Midterm (90 min) 20% + Final (90 min) 60%.
- **Course content** (11 chapters): Intro → Bandits → Finite MDPs → DP → MC → TD → Planning+Learning (tabular) → On-policy Prediction with Approximation → On-policy Control with Approximation → Policy Gradient → Advanced Topics.

---

## 1. What is Reinforcement Learning?

- A way of **programming agents by reward and punishment** without specifying *how* the task should be done (Kaelbling, Littman, Moore 1996).
- The agent learns by **trial and error** — guided only by `+` reinforcement for good behaviour and `−` reinforcement for bad behaviour (e.g., learning to ride a bike: pain when you fall, reward when you don't).
- RL sits **alongside** supervised and unsupervised learning as one of three core ML paradigms.

### 1.1. What makes RL different from SL/UL?

| Feature | Supervised | Unsupervised | **RL** |
|---|---|---|---|
| Supervisor | Labels | None | **No supervisor, only a reward signal** |
| Feedback | Instantaneous | N/A | **Delayed** |
| Data | i.i.d. | i.i.d. | **Sequential, non-i.i.d.** — time matters |
| Effect of agent | None | None | **Agent's actions affect the data it later receives** |

Those four bullets are the four canonical "characteristics of RL".

---

## 2. Examples & Applications

| Domain | Example | Reference |
|---|---|---|
| Aerobatic flight | Helicopter stunt manoeuvers | Abbeel, Coates, Quigley, Ng — NIPS 2006 |
| Atari games | Human-level control on 49 Atari 2600 games | Mnih et al., Nature 2015 |
| Board games | Chess (Giraffe 2015), Go (AlphaGo Zero 2017) | Silver et al., Nature 2017 |
| Robotics | Pancake-flipping, injured robots learning to limp | Kormushev IROS 2010; Cully et al. Nature 2015 |
| Finance | Investment portfolio management | — |
| Supply chain | Amazon warehouse pick-and-place | — |
| Energy | DeepMind cut Google data-centre cooling by **40 %** | — |
| NLP | ChatGPT fine-tuning via **RLHF** | OpenAI |
| Recommender / ads | Online banner ads | — |

> **Rule of thumb**: any complex dynamical system that is hard to model analytically is a potential RL application.

### 2.1. Example rewards (concrete designs)

| Task | + reward | − reward |
|---|---|---|
| Helicopter stunt | follow desired trajectory | crash |
| Chess / Go | win the game | lose the game |
| Investment portfolio | money in bank | — |
| Power station | produce power | exceed safety threshold |
| Humanoid walking | forward motion | falling over |
| Atari games | score ↑ | score ↓ |

---

## 3. Formalizing the RL Problem

### 3.1. Reward `Rₜ` and Return `Gₜ`

- A **reward** `Rₜ` is a **scalar feedback signal** indicating how well the agent is doing at step `t`.
- The agent's job is to maximize the **cumulative reward** = the **return**:

```math
G_t = R_{t+1} + R_{t+2} + R_{t+3} + \cdots
```

- **Reward hypothesis**: *any* goal can be formalized as the outcome of maximizing the expected value of a cumulative scalar reward.

### 3.2. Sequential decision making

- Actions can have **long-term consequences**; reward can be **delayed**.
- It can be better to **sacrifice immediate reward** for greater long-term reward — investments mature, refuelling a helicopter prevents a crash hours later, learning a new skill is costly upfront.
- The agent must **balance short-term and long-term** rewards.

### 3.3. The interaction loop

At each step `t`:
- **Agent**: receives observation `Oₜ` and reward `Rₜ`, executes action `Aₜ`.
- **Environment**: receives `Aₜ`, emits next observation `Oₜ₊₁` and reward `Rₜ₊₁`.
- Time `t` is incremented at each env. step.

### 3.4. History, agent state, environment state

- **History** = full observable sequence:

```math
\mathcal{H}_t = O_0, A_0, R_1, O_1, \cdots, O_{t-1}, A_{t-1}, R_t, O_t
```

- **State** is *any function of history*: `Sₜ = f(Hₜ)`.
- **Environment state** `Sₜᵉ` = environment's internal representation. Usually **invisible** to the agent; even when visible may contain irrelevant info.
- **Agent state** `Sₜᵃ` = agent's internal representation, what the algorithm uses to pick the next action. Often **much smaller** than the env. state.

### 3.5. Information / Markov state

A state `Sₜ` is **Markov** iff:

```math
P[S_{t+1} \mid S_t] = P[S_{t+1} \mid S_1, \dots, S_t]
```

> "The future is independent of the past given the present" — `H_{1:t} → Sₜ → H_{t+1:∞}`.

Consequences:
- Once the Markov state is known, **the history may be thrown away** — the state is a **sufficient statistic** of the future.
- The environment state `Sₜᵉ` is Markov. The history `Hₜ` is trivially Markov.

### 3.6. Observability

| Setting | Definition | Formal model |
|---|---|---|
| **Fully observable** | Agent sees full env. state: `Oₜ = Sₜᵉ = Sₜᵃ` | **Markov Decision Process (MDP)** |
| **Partially observable** | Observations are not Markov (robot camera ≠ absolute location; trader sees only prices) | **POMDP** |

In a POMDP, the agent must build its own state. Three common constructions:
- **Complete history**: `Sₜᵃ = Hₜ` (lossless but huge).
- **Belief state**: `Sₜᵃ = (P[Sₜᵉ = s¹], …, P[Sₜᵉ = sⁿ])`.
- **Recurrent neural network**: `Sₜᵃ = σ(Sₜ₋₁ᵃ Wₛ + Qₜ W_o)`.

---

## 4. Components of an RL Agent

An RL agent has **one or more** of:

| Component | What it represents | Functional signature |
|---|---|---|
| **Policy** `π` | Agent's behaviour: map state → action | `π : 𝓢 → 𝓐` (or to a distribution over 𝓐) |
| **Value function** `v` / `q` | Prediction of future reward, ranks states/actions | `v : 𝓢 → ℝ` |
| **Model** `m`, `r` | Agent's representation of the environment dynamics | `m : 𝓢 → 𝓢`, `r : 𝓢 → ℝ` |
| **State update** | How agent state evolves given new observation | `u : 𝓢 × 𝓞 → 𝓢` |

All four are **functions** — can be tabular, linear, or neural networks (→ **deep RL**). Care: RL violates SL assumptions (i.i.d., stationarity).

### 4.1. Policy

- **Deterministic**: `a = π(s)`.
- **Stochastic**: `π(a | s) = P[A = a | S = s]`.

### 4.2. Value function

State-value under policy `π`:

```math
v_\pi(s) = \mathbb{E}[R_{t+1} + \gamma R_{t+2} + \gamma^2 R_{t+3} + \cdots \mid S_t = s, \pi]
```

Action-value (conditioned also on the action taken):

```math
q_\pi(s, a) = \mathbb{E}[R_{t+1} + \gamma R_{t+2} + \gamma^2 R_{t+3} + \cdots \mid S_t = s, A_t = a]
```

- **Discount factor** `γ ∈ [0, 1]` trades off immediate vs long-term rewards.
- The value **depends on the policy**.
- Rewards/values define the **utility** of states and actions (no supervised feedback).

### 4.3. Model

A model predicts what the environment does next:
- **Transition model** `𝒫(s, a, s') ≈ P[Sₜ₊₁ = s' | Sₜ = s, Aₜ = a]`.
- **Reward model** `𝓡(s, a) ≈ 𝔼[Rₜ₊₁ | Sₜ = s, Aₜ = a]`.

A model **does not by itself give a policy** — you still have to **plan** with it (e.g., tree search).

### 4.4. Maze illustration of all four

The lecture's grid maze (`-1` per step, actions = N/E/S/W, state = location):
- **Policy** = arrow in each cell pointing the way out.
- **Value** = negative numbers in each cell counting steps-to-goal under `π`.
- **Model** = grid layout + the `-1` per-step reward written in each cell.

---

## 5. Agent taxonomies

### 5.1. By representation

| Category | Has policy? | Has value? | Has model? |
|---|---|---|---|
| **Value-based** | implicit (greedy w.r.t. v) | ✓ | — |
| **Policy-based** | ✓ | — | — |
| **Actor-critic** | ✓ | ✓ | — |

### 5.2. By model

| Category | Policy &/or Value | Model |
|---|---|---|
| **Model-free** | ✓ | — |
| **Model-based** | optional | ✓ |

---

## 6. Subproblems of the RL Problem

### 6.1. Learning vs Planning

| | **Learning** | **Planning** |
|---|---|---|
| Environment | initially **unknown** | **known** (or already learnt) |
| Interaction | with **real** environment | with **the model** only (a.k.a. reasoning, search) |
| Goal | improve policy through experience | improve policy through computation |

*Atari illustration*: Learning = agent only sees pixels + score, doesn't know rules. Planning = agent has the emulator, can simulate "if I move right, what next?" → tree search.

### 6.2. Exploration vs Exploitation

- **Exploration**: gather more info about the environment.
- **Exploitation**: use known info to maximize reward.
- Trial-and-error needs **both** — losing too much reward while exploring is bad, but never exploring locks in a suboptimal policy.

| Setting | Exploit | Explore |
|---|---|---|
| Restaurant | go to favourite | try a new one |
| Online ad | show best-performing | show a different one |
| Oil drilling | drill at best known site | drill at a new site |
| Game play | play move you currently think is best | try a new strategy |

### 6.3. Prediction vs Control

| | **Prediction** | **Control** |
|---|---|---|
| Question | Evaluate the future for a given `π` | Optimize the future — find the best `π` |
| Output | `vπ(s)` | `π*(s) = argmaxπ vπ(s)` |

These are tightly coupled: if you can predict perfectly, finding `π*` reduces to a greedy step over `v*`.

> *Gridworld* example: Prediction = compute `vπ` for the uniform random policy. Control = compute `V*` and `π*` over all policies.

---

## Key terms (glossary)

- **Agent / Environment** — the two halves of the loop.
- **State (Sₜ)** — sufficient summary of history used to choose actions.
- **Markov property** — future ⫫ past | present.
- **MDP** — fully observable Markov decision process.
- **POMDP** — partially observable MDP.
- **Reward Rₜ** — scalar feedback at step `t`.
- **Return Gₜ** — cumulative (often discounted) reward from `t` onward.
- **Reward hypothesis** — goals = maximizing expected return.
- **Discount factor γ** — `[0, 1]` weighting of future rewards.
- **Policy π** — state-to-action mapping (deterministic or stochastic).
- **State-value vπ(s)**, **Action-value qπ(s, a)** — expected return from `s` (and `a`) under `π`.
- **Model** — agent's representation of `𝒫` and `𝓡`.
- **Learning vs Planning** — env. unknown (interact) vs known (compute).
- **Exploration vs Exploitation** — gather vs use information.
- **Prediction vs Control** — evaluate `π` vs find best `π`.

---

## Exam targets (likely written-exam questions)

1. **Define RL** and list the four characteristics that distinguish it from supervised/unsupervised learning.
2. **State the reward hypothesis** and give a critical discussion (can *every* goal really be reduced to a scalar?).
3. **Define the Markov property** formally, explain "sufficient statistic of the future", and contrast MDP vs POMDP with a concrete example.
4. **Distinguish History vs Environment state vs Agent state** and write the function relation between them.
5. Given a scenario (helicopter, chess, ads), **design the reward** with `+` and `−` components.
6. **Distinguish the three building blocks** (policy, value, model) — give formal signatures, and classify an agent (value-based / policy-based / actor-critic, model-free / model-based).
7. **Write the definitions** of `vπ(s)` and `qπ(s, a)` with discount factor `γ`.
8. **Contrast** Learning vs Planning, Exploration vs Exploitation, Prediction vs Control — each with a one-line example.
9. **Maze example**: be ready to draw the policy arrows, fill in the value cells (steps-to-goal), and write the per-cell reward (the model).

### Pitfalls

- **State ≠ Observation**. In POMDPs the observation is *not* Markov; the agent has to build its own state.
- **Reward is scalar**. If your "reward" is a vector, you haven't finished designing it.
- A **model gives you dynamics, not a policy** — planning is a separate step on top.
- **Value depends on a policy** — `vπ` and `v*` are different objects; never write `v(s)` without specifying which.
- **γ = 0** → myopic (only immediate reward); **γ = 1** → undiscounted (only valid for episodic tasks or careful infinite cases).
- Don't confuse **Prediction** (given `π`, find `vπ`) with **Control** (find the *best* `π`). The slogan: "Prediction is one policy, Control is over all policies."
- **Exploration is not random action for its own sake** — it's a *targeted* trade-off against exploitation; pure-random-forever is bad too.
