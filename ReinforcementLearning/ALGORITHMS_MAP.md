# RL algorithms — classification map

> A clean Mermaid version of the hand-drawn taxonomy, covering **every algorithm in `lectures/`** organised on the classical three axes: **model-based vs model-free** · **value vs policy vs actor-critic** · **on-policy vs off-policy**.
> **Out of scope** (per request): Ch 2 bandits and Ch 10.3 MARL.

---

## How to read the map

- **Solid arrows** = taxonomy (is-a)
- **Dotted arrows** = algorithm lineage (X's successor fixes a problem of X)
- **Hexagons** = the question being asked at that fork
- **Stadium nodes** = concrete algorithms (with their chapter ref)
- **Dark-grey nodes** = deep-RL extensions (neural-net-parameterised version of a tabular ancestor)
- **Colour bands**:
  - 🟩 green = model-based / planning
  - 🟦 blue = model-free / learning
  - 🟧 orange = value-based
  - 🟪 pink = pure policy
  - 🟨 purple = actor-critic
  - ⬛ dark = deep RL

---

## The big picture

```mermaid
flowchart TB
    classDef root fill:#1a237e,color:#fff,stroke:#000,stroke-width:2px
    classDef q fill:#fff59d,color:#bf360c,stroke:#f9a825,stroke-width:2px
    classDef mb fill:#a5d6a7,color:#1b5e20,stroke:#2e7d32,stroke-width:2px
    classDef mf fill:#90caf9,color:#0d47a1,stroke:#1565c0,stroke-width:2px
    classDef val fill:#ffcc80,color:#bf360c,stroke:#e65100,stroke-width:2px
    classDef pol fill:#f48fb1,color:#880e4f,stroke:#ad1457,stroke-width:2px
    classDef ac fill:#b39ddb,color:#311b92,stroke:#512da8,stroke-width:2px
    classDef algo fill:#fafafa,color:#212121,stroke:#616161
    classDef deep fill:#263238,color:#eceff1,stroke:#000

    R((RL agent)):::root
    R --> Q1{{Environment dynamics known?}}:::q

    %% ── Model-based branch ─────────────────────────────
    Q1 -->|"yes"| MB[Model-based · Planning]:::mb
    MB --> DP[Dynamic Programming · Ch 4]:::mb
    DP --> IPE([Iterative Policy Eval]):::algo
    DP --> PI([Policy Iteration]):::algo
    DP --> VI([Value Iteration]):::algo
    DP --> ADP([Async DP]):::algo
    DP --> GPI([GPI · meta-pattern]):::algo
    MB --> DT[Decision-time planning · Ch 7]:::mb
    DT --> Dyna([Dyna-Q · Dyna-Q+]):::algo
    DT --> MCTS([MCTS]):::algo

    %% ── Model-free branch ──────────────────────────────
    Q1 -->|"no"| MF[Model-free · Learning]:::mf
    MF --> Q2{{What is learned?}}:::q
    Q2 -->|"value Q"| VB[Value-based]:::val
    Q2 -->|"policy π"| PB[Pure Actor]:::pol
    Q2 -->|"both"| AC[Actor-Critic]:::ac

    %% ── Value-based: on/off-policy ─────────────────────
    VB --> Q3{{Behavior = target?}}:::q
    Q3 -->|"yes · on-policy"| VBOn[On-policy]:::val
    Q3 -->|"no · off-policy"| VBOff[Off-policy]:::val
    VBOn --> MCon([MC ES, ε-soft · Ch 5]):::algo
    VBOn --> TD0([TD 0 prediction · Ch 6]):::algo
    VBOn --> SARSA([SARSA · Ch 6]):::algo
    VBOn --> ESARSA([Expected SARSA · Ch 6]):::algo
    VBOn --> NStep([n-step TD · Ch 6, 8]):::algo
    VBOff --> OffMC([Off-policy MC + IS · Ch 5]):::algo
    VBOff --> QL([Q-learning · Ch 6]):::algo
    QL --> DQN([DQN · Ch 10.1]):::deep

    %% ── Pure policy ────────────────────────────────────
    PB --> REIN([REINFORCE · Ch 9]):::algo
    PB --> REINBL([REINFORCE w/ baseline · Ch 9]):::algo

    %% ── Actor-Critic family ────────────────────────────
    AC --> AC1([One-step Actor-Critic · Ch 9]):::algo
    AC --> Q4{{Action space?}}:::q
    Q4 -->|"discrete · stochastic"| Disc[Stochastic AC family]:::ac
    Q4 -->|"continuous · deterministic"| Cont[Deterministic AC family]:::ac
    Disc --> A2C([A2C / A3C · Ch 10.1]):::deep
    Disc --> TRPO([TRPO · Ch 10.1]):::deep
    Disc --> PPO([PPO · Ch 10.1]):::deep
    Disc --> GRPO([GRPO · Ch 10.1]):::deep
    Cont --> DDPG([DDPG · Ch 10.1]):::deep
    Cont --> TD3([TD3 · Ch 10.1]):::deep
    Cont --> SAC([SAC · Ch 10.1]):::deep

    %% ── Evolution lineage (dotted) ─────────────────────
    REIN -.->|"+ baseline"| REINBL
    REINBL -.->|"+ bootstrap critic"| AC1
    AC1 -.->|"parallel · advantage"| A2C
    A2C -.->|"trust region · KL"| TRPO
    TRPO -.->|"clip ratio"| PPO
    PPO -.->|"group-relative"| GRPO
    DQN -.->|"continuous · actor μθ"| DDPG
    DDPG -.->|"twin Q · delayed"| TD3
    TD3 -.->|"+ max entropy"| SAC
```

---

## Lineage — what each dotted arrow means

### Stochastic policy chain
`REINFORCE → … → GRPO`

| Step | What it adds | Why |
|---|---|---|
| `+ baseline` | subtract `b(s)`, usually `v̂(s, w)` | cut variance without bias |
| `+ bootstrap critic` | replace `Gₜ` by TD error `δ` | online, even lower variance |
| `parallel · advantage` | A2C/A3C: many workers, `A = Q − V` | scale + decorrelation |
| `trust region · KL` | TRPO: largest step with `KL(πθ ‖ πθ_old) ≤ δ` | stop policy collapse |
| `clip ratio` | PPO: clip `r = πθ/πθ_old` to `[1−ε, 1+ε]` | TRPO without 2nd-order math |
| `group-relative` | GRPO: drop the value baseline | use group-normalised advantage |

### Deep value + deterministic chain
`Q-learning → … → SAC`

| Step | What it adds | Why |
|---|---|---|
| `+ deep net + target + replay` | DQN: NN `Qφ`, target net `φ⁻`, replay buffer | stabilise the deadly triad |
| `continuous · actor μθ` | DDPG: replace `maxₐ Q` by `Q(s, μθ(s))` | gradient flows through actor |
| `twin Q · delayed` | TD3: `min(Qφ₁, Qφ₂)` + slow actor update | curb overestimation |
| `+ max entropy` | SAC: add `α · H(π(·|s))` to objective | exploration + smoother optima |

---

## Confusables — Q-learning · DQN · DDPG comparison

The trio that gets mixed up in exams (and in the hand-drawn map's top-left table):

| | **Q-learning** (Ch 6) | **DQN** (Ch 10.1) | **DDPG** (Ch 10.1) |
|---|---|---|---|
| Action space | discrete | discrete | **continuous** |
| Function form | tabular `Q(s, a)` | NN `Qφ(s, a)` | NN `Qφ(s, a)` + actor `μθ(s)` |
| Policy | implicit (greedy on Q) | implicit (argmax NN) | explicit actor `μθ` + noise |
| Off-policy? | ✓ | ✓ | ✓ |
| Bootstrapping? | ✓ | ✓ | ✓ |
| Update target | `r + γ maxₐ' Q(s', a')` | same | `r + γ Qφ′(s', μθ′(s'))` |
| Stability tricks | none | target net + replay | target net + replay + **Polyak avg** |
| How action picked | tabular `argmax` | NN `argmax` | `μθ(s)` + exploration noise |

---

## Orthogonal axes — apply to *almost any* algorithm above

These are choices you make on top of the main taxonomy:

| Axis | "Plain" side | Extension side |
|---|---|---|
| **Function form** (Ch 8) | tabular `V(s)`, `Q(s,a)` | parameterised `v̂(s, w)`, `q̂(s, a, w)` — linear / NN |
| **Bootstrap depth** | one-step TD(0) | n-step TD · λ-return · MC (no bootstrap) |
| **Reward formulation** (Ch 8) | discounted `Σ γᵏ R` | average-reward `r(π)` + differential return |
| **Lifetime** | episodic | continuing |

Example: **SARSA** appears at three levels — tabular (Ch 6), linear semi-gradient (Ch 8 §6), and differential semi-gradient for average-reward continuing tasks (Ch 8 §9). Same update skeleton, different `w` / `R̄` accounting.

---

## Parallel paradigms in our folder (separate problem setup)

These don't fit the value/policy/AC trichotomy — they're solving a *different* problem.

### Imitation Learning (Ch 10.2) — no reward, only expert demos `Ξ`

```mermaid
flowchart LR
    classDef family fill:#ffe0b2,stroke:#e65100,color:#bf360c,font-weight:bold
    classDef algo fill:#fafafa,stroke:#616161,color:#212121

    D[Expert demos Ξ]:::family
    D --> Direct[Direct · learn π directly]:::family
    D --> Indirect[Indirect · recover R, then plan]:::family
    Direct --> BC([Behavior Cloning · supervised on s→a]):::algo
    Direct --> DAg([DAgger · query expert on visited states]):::algo
    Indirect --> IRL([Inverse RL · fit R from expert optimality]):::algo
    Indirect --> Appr([Apprenticeship · match feature expectations]):::algo
    BC -.->|"fix distribution shift"| DAg
    IRL -.->|"sidestep R ambiguity"| Appr
```

### Model-based deep RL (Ch 10.2) — World Models

```mermaid
flowchart LR
    classDef family fill:#a5d6a7,stroke:#2e7d32,color:#1b5e20,font-weight:bold
    classDef algo fill:#fafafa,stroke:#616161,color:#212121

    WM[World Models · Ha & Schmidhuber]:::family
    WM --> V([V · VAE encoder<br/>compress obs → z]):::algo
    WM --> M([M · MDN-RNN<br/>predict z' from z, a]):::algo
    WM --> C([C · tiny controller<br/>linear in z, h]):::algo
    M -.->|"train C in the dream"| C
```

### Hierarchical RL (Ch 10.3) — temporally-extended options

```mermaid
flowchart LR
    classDef family fill:#b39ddb,stroke:#5e35b1,color:#311b92,font-weight:bold
    classDef algo fill:#fafafa,stroke:#616161,color:#212121

    HRL[HRL · Ch 10.3]:::family
    HRL --> Opt[Options framework]:::family
    Opt --> Triple([Option = I, π, β<br/>initiation, policy, termination]):::algo
    Opt --> SMDP([Semi-MDP · option-duration τ]):::algo
    HRL --> HDQN([H-DQN · meta-controller + controller]):::algo
    HRL --> HQ([Hierarchical Q-learning]):::algo
```

---

## What's *not* in the diagram (and why)

| Topic | Where in our folder | Why excluded |
|---|---|---|
| Multi-armed bandits | Ch 2 (`02_chapter2_bandits.md`) | Stateless — different problem class |
| ε-greedy, UCB, gradient bandit | Ch 2 | Action-selection rules, not full RL algorithms |
| IQL, VDN, QMIX | Ch 10.3 (`12_chapter10_advanced_part3.md`) | MARL — out of scope per request |
| CTDE | Ch 10.3 | MARL paradigm |

Use this map as the **mental anchor** for revision; reach for the chapter `.md` files for the equations and pseudocode.
