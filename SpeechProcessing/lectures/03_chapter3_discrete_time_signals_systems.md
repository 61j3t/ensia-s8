# Chapter 3 — Discrete-Time Signals and Systems

## Bird's eye view

- A **discrete-time signal** $x[n]$ is a sequence of numbers indexed by integer $n$, obtained either by sampling a continuous-time signal at period $T$ (so $x[n] = x(nT)$) or generated synthetically.
- Two fundamental building-block sequences underpin everything: the **unit impulse** $\delta[n]$ (= 1 at $n = 0$, else 0) and the **unit step** $u[n]$ (= 1 for $n \geq 0$, else 0). Every signal can be written as a weighted sum of shifted impulses (sifting property).
- A **discrete-time system** is an operator $T$ that maps input $x[n]$ to output $y[n]$. Five key properties: memorylessness, linearity, time-invariance, causality, stability. A system that is both **linear and time-invariant (LTI)** is fully characterised by its **impulse response** $h[n]$.
- The input-output relationship of any LTI system is the **convolution sum**: $y[n] = x[n] \otimes h[n] = \sum_{m=-\infty}^{\infty} x[m]\, h[n-m]$. Convolution is commutative, associative, and linear (distributive over addition).
- An LTI system can equivalently be described by an **N-th order linear constant-coefficient difference equation (LCCDE)**, which also allows recursive computation of output samples.
- Signals are classified as **energy signals** (finite total energy, zero average power) or **power signals** (finite average power, infinite energy). A signal cannot belong to both classes simultaneously.

---

## Detailed notes

### 1. Discrete-Time Signals

#### 1.1 Definition and sampling

A continuous-time signal $x(t)$ is sampled at uniform intervals of length $T$ (the **sampling period**) to produce:

$$
x[n] = x(t)\big|_{t=nT} = x(nT), \qquad n = \ldots, -1, 0, 1, 2, \ldots \tag{3.1}
$$

$x[n]$ is a **sequence of numbers** $\ldots, x[-1], x[0], x[1], x[2], \ldots$ where $n$ is the (dimensionless) time index. The continuous variable $t$ is replaced by the integer $n$; information between samples is discarded.

Fig. 3.1 (conceptual): the smooth analog curve $x(t)$ is represented by vertical stems at $t = \ldots, -T, 0, T, 2T, \ldots$, each of height $x(nT)$. The stems are the discrete-time signal $x[n]$.

#### 1.2 Basic sequences

**Unit Sample (Unit Impulse)**

$$
\delta[n] = \begin{cases} 1, & n = 0 \\ 0, & n \neq 0 \end{cases} \tag{3.2}
$$

- Graphically: a single stem of height 1 at $n = 0$, zero elsewhere.
- Unlike the continuous-time Dirac delta $\delta(t)$, which is undefined at $t = 0$, $\delta[n]$ is perfectly well-defined for all $n$.
- **Sifting (decomposition) property** — any signal $x[n]$ can be written as a superposition of scaled, shifted impulses:

$$
x[n] = \sum_{k=-\infty}^{\infty} x[k]\, \delta[n - k] \tag{3.4}
$$

This identity is central: it says $x[n] = \ldots + x[-1]\delta[n+1] + x[0]\delta[n] + x[1]\delta[n-1] + \ldots$

**Unit Step**

$$
u[n] = \begin{cases} 1, & n \geq 0 \\ 0, & n < 0 \end{cases} \tag{3.3}
$$

- Graphically: stems of height 1 at $n = 0, 1, 2, \ldots$ and zero for $n < 0$.

**Relationships between $\delta[n]$ and $u[n]$**

$$
u[n] = \sum_{k=0}^{\infty} \delta[n - k] \tag{3.5}
$$

$$
\delta[n] = u[n] - u[n - 1] \tag{3.6}
$$

(3.5) expresses the step as an accumulation of impulses; (3.6) expresses the impulse as the first difference of the step.

#### 1.3 Deterministic signal generation

A **deterministic** discrete-time signal satisfies a known functional form:

$$
x[n] = f(\psi, n) \tag{3.7}
$$

where $\psi$ is the parameter vector and $n$ is the time index. Examples:

| Signal type | Form | Parameter vector $\psi$ |
|---|---|---|
| Time-shifted impulse | $\delta[n - n_0]$ | $[n_0]$ |
| Time-shifted step | $u[n - n_0]$ | $[n_0]$ |
| Real exponential | $\alpha^{n - n_0}$ | $[\alpha, n_0]$ |
| Sinusoid | $A\cos(\omega n + \theta)$ | $[A, \omega, \theta]$ |

**Example 3.1** — Generate $x[n] = A\cos(\omega n + \theta)$ with $A = 1$, $\omega = 0.3$, $\theta = 1$, $N = 21$ samples ($n = 0, \ldots, 20$):

```matlab
N = 21;   A = 1;   w = 0.3;   p = 1;
n = 0:N-1;
x = A .* cos(w .* n + p);
stem(n, x)          % stem plot (Fig. 3.2): discrete spikes
plot(n, x)          % line plot  (Fig. 3.3): interpolated appearance
```

Key MATLAB note: array indices start at 1, so if the loop runs `n = 1:N`, use `x(n) = A*cos(w*(n-1)+p)` to shift the time index to start at 0. Using `n = 0:N-1` with vectorised code avoids this issue.

Fig. 3.2 shows stem plot: 21 spikes oscillating between -1 and +1, first positive peak near $n = 0$ (value ~0.54), negative trough near $n = 7$–8.  
Fig. 3.3 shows plot: the same values connected, resembling a smooth cosine.

---

### 2. Discrete-Time Systems

#### 2.1 Definition

A discrete-time system is an **operator** $T$ that maps an input sequence $x[n]$ to an output sequence $y[n]$:

$$
y[n] = T\{x[n]\} \tag{3.8}
$$

#### 2.2 Memorylessness

$y[n]$ depends **only** on $x[n]$ at the same time index $n$ (not on past or future values of $x$).

- Every memoryless system is automatically **causal**.
- The converse is not true: a causal system may have memory (depend on past inputs).

#### 2.3 Linearity

$T$ is **linear** if it obeys **superposition**: given $y_1[n] = T\{x_1[n]\}$ and $y_2[n] = T\{x_2[n]\}$, then for any scalars $a$ and $b$:

$$
T\{a\,x_1[n] + b\,x_2[n]\} = a\,T\{x_1[n]\} + b\,T\{x_2[n]\} = a\,y_1[n] + b\,y_2[n] \tag{3.9}
$$

**Standard test procedure** (to check linearity):
1. Define $x_3[n] = a\,x_1[n] + b\,x_2[n]$.
2. Compute $y_3[n] = T\{x_3[n]\}$ directly from the system equation.
3. Also compute $a\,y_1[n] + b\,y_2[n]$.
4. If equal, the system is linear; otherwise nonlinear.

**Example 3.2** — $y[n] = \sum_{k=-\infty}^{n} x[k]$ (running accumulator):

- $y_3[n] = \sum_{k=-\infty}^{n} x_3[k] = \sum_{k=-\infty}^{n} (a\,x_1[k] + b\,x_2[k])$
  $= a\sum_{k=-\infty}^{n} x_1[k] + b\sum_{k=-\infty}^{n} x_2[k]$
  $= a\,y_1[n] + b\,y_2[n]$ => **linear**

**Example 3.3** — $y[n] = 3x^2[n] + 2x[n-3]$ (contains $x^2$ term):

- $y_3[n] = 3(a\,x_1[n] + b\,x_2[n])^2 + 2(a\,x_1[n-3] + b\,x_2[n-3])$
  $= 3(a^2 x_1^2 + b^2 x_2^2 + 2ab\,x_1 x_2) + \ldots$
  $\neq a(3x_1^2 + 2x_1[n-3]) + b(3x_2^2 + 2x_2[n-3])$ => **nonlinear**

The cross-term $2ab\,x_1[n]\,x_2[n]$ is the giveaway.

#### 2.4 Time-Invariance

$T$ is **time-invariant** if shifting the input by $n_0$ causes an identical shift in the output: if $y[n] = T\{x[n]\}$, then:

$$
y[n - n_0] = T\{x[n - n_0]\} \tag{3.10}
$$

**Standard test procedure**:
1. Let $x_1[n] = x[n - n_0]$; compute $y_1[n] = T\{x_1[n]\}$.
2. Also derive $y[n - n_0]$ by replacing $n$ with $n - n_0$ in the input-output relation.
3. If $y_1[n] = y[n - n_0]$, the system is time-invariant; otherwise time-variant.

**Example 3.4** — $y[n] = \sum_{k=-\infty}^{n} x[k]$:

- $y[n - n_0] = \sum_{k=-\infty}^{n-n_0} x[k]$
- $x_1[n] = x[n - n_0]$: $y_1[n] = \sum_{k=-\infty}^{n} x[k - n_0] = \sum_{l=-\infty}^{n-n_0} x[l]$ (substituting $l = k - n_0$)
  $= y[n - n_0]$ => **time-invariant**

**Example 3.5** — $y[n] = 3x[3n]$ (time-scaling = compression):

- $y[n - n_0] = 3x[3(n - n_0)] = 3x[3n - 3n_0]$
- $y_1[n] = 3x_1[3n] = 3x[3n - n_0]$
  Since $3x[3n - n_0] \neq 3x[3n - 3n_0]$ in general => **time-variant**

Key insight: any system that compresses or expands the time axis (e.g., $y[n] = x[cn]$ for integer $c \neq 1$) is time-variant.

#### 2.5 Causality

$y[n]$ at time $n$ depends on $x[k]$ only for $k \leq n$ (current and past inputs, never future inputs).

For LTI systems, the **equivalent causality condition** is:

$$
h[n] = 0, \qquad n < 0 \tag{3.11}
$$

**Quick test examples (from slides):**
- $y[n] = x[n] + x[n+1]$: depends on $x[n+1]$ (future) => **not causal**
- $y[n] = x[n] + x[n-2]$: depends on $x[n]$ and $x[n-2]$ (current and past) => **causal**

#### 2.6 Stability (BIBO)

A system is **stable** (Bounded-Input Bounded-Output, BIBO) if every bounded input produces a bounded output:

$$
|x[n]| < \infty \text{ for all } n \implies |y[n]| < \infty \text{ for all } n
$$

For LTI systems, the equivalent stability condition is absolute summability of the impulse response:

$$
\sum_{n=-\infty}^{\infty} |h[n]| < \infty \tag{3.12}
$$

**Quick test examples (from slides):**
- $y[n] = x[n] + x[n+1]$: both terms bounded if $x$ is bounded => **stable**
- $y[n] = 1/x[n]$: if $x[n] \to 0$, output $\to \infty$ => **not stable** (unbounded output from bounded input)

---

### 3. Convolution

#### 3.1 The convolution sum

For any LTI system, the input-output relationship is completely described by the **convolution sum**:

$$
y[n] = x[n] \otimes h[n] = \sum_{m=-\infty}^{\infty} x[m]\, h[n - m] \tag{3.13}
$$

$h[n]$ is the system's **impulse response** — its output when the input is $\delta[n]$.  
The formula involves only additions and multiplications, making it computationally tractable.

Equivalently (by commutativity, below):

$$
y[n] = \sum_{m=-\infty}^{\infty} h[m]\, x[n - m]
$$

#### 3.2 Properties of convolution

**Commutative:**

$$
x[n] \otimes h[n] = h[n] \otimes x[n] = \sum_{m=-\infty}^{\infty} x[m]\,h[n-m] = \sum_{m=-\infty}^{\infty} h[m]\,x[n-m] \tag{3.14}
$$

Implication (Fig. 3.4): two cascaded LTI systems with impulse responses $h_1[n]$ and $h_2[n]$ give the same overall output regardless of the order they are placed.

**Associative / Cascade:**

$$
y[n] = x[n] \otimes h_1[n] \otimes h_2[n] = x[n] \otimes h_2[n] \otimes h_1[n] = x[n] \otimes (h_1[n] \otimes h_2[n]) \tag{3.15}
$$

Two cascaded systems can be replaced by a single system whose impulse response is the convolution of the individual impulse responses.

**Linear (Distributive / Parallel):**

$$
y[n] = x[n] \otimes (h_1[n] + h_2[n]) = x[n] \otimes h_1[n] + x[n] \otimes h_2[n] \tag{3.16}
$$

Implication (Fig. 3.5): two parallel LTI systems can be collapsed into one system whose impulse response is the sum of their individual impulse responses.

#### 3.3 Finite-length output length rule

If $x[n]$ has length $M$ and $h[n]$ has length $N$, then the length of $y[n] = x[n] \otimes h[n]$ is $M + N - 1$.

#### 3.4 Worked examples

**Example 3.6** — $x[n] = u[n]$, $h[n] = \delta[n] + 0.5\,\delta[n-1]$:

Method 1 (substitute $x$ into convolution formula):

$$
y[n] = \sum_{m=-\infty}^{\infty} u[m]\,(\delta[n-m] + 0.5\,\delta[n-1-m]) = \sum_{m=0}^{\infty} (\delta[n-m] + 0.5\,\delta[n-1-m]) = u[n] + 0.5\,u[n-1]
$$

Method 2 (general formula then substitute $x[n] = u[n]$):

$$
y[n] = x[n] + 0.5\,x[n-1] \implies y[n] = u[n] + 0.5\,u[n-1]
$$

Stability check: $\sum_{n=-\infty}^{\infty} |h[n]| = |h[0]| + |h[1]| = 1 + 0.5 = 1.5 < \infty$ => **stable**  
Causality check: $h[n] = 0$ for $n < 0$ => **causal**

**Example 3.7** — $x[n] = a^n u[n]$, $h[n] = u[n] - u[n-10]$ (a finite-duration window of 10 ones):

Convolution splits into $y[n] = y_1[n] - y_2[n]$ where:

$$
y_1[n] = \sum_{m=0}^{\infty} a^m\, u[n-m] = \frac{1 - a^{n+1}}{1-a}\, u[n]
$$

$$
y_2[n] = \sum_{m=0}^{\infty} a^m\, u[n-10-m] = \frac{1 - a^{n-9}}{1-a}\, u[n-10]
$$

Combined result:

$$
y[n] = \begin{cases} 0, & n < 0 \\ \dfrac{1 - a^{n+1}}{1-a}, & 0 \leq n < 10 \\ a^{n-9}\,\dfrac{1 - a^{10}}{1-a}, & n \geq 10 \end{cases}
$$

$\sum |h[n]| = 10 < \infty$ => **stable**; $h[n] = 0$ for $n < 0$ => **causal**

**Example 3.8** — Finite-length convolution, $x[n] = \{1, 2, 5, 10\}$ for $n=0\ldots3$, $h[n] = \{1, 2, 3, 4\}$ for $n=0\ldots3$:

The sum reduces to 4 terms only:

$$
y[n] = x[0]\,h[n] + x[1]\,h[n-1] + x[2]\,h[n-2] + x[3]\,h[n-3]
$$

Result: $y[0]=1$, $y[1]=4$, $y[2]=12$, $y[3]=30$, $y[4]=43$, $y[5]=50$, $y[6]=40$, else 0.

Length $= 4 + 4 - 1 = 7$ samples ($n = 0, \ldots, 6$).

MATLAB: `y = conv(x, h)` gives the correct values. To find the correct starting time index, compute $y[0] = x[0]\,h[0] = 1 \cdot 1 = 1$ and locate this in the output vector.

---

### 4. Linear Constant-Coefficient Difference Equations (LCCDE)

#### 4.1 General form

For a LTI system, input $x[n]$ and output $y[n]$ are related by an **N-th order LCCDE**:

$$
\sum_{k=0}^{N} a_k\, y[n-k] = \sum_{k=0}^{M} b_k\, x[n-k] \tag{3.17}
$$

where $\{a_k\}$ are the feedback (output) coefficients and $\{b_k\}$ are the feedforward (input) coefficients. $N$ is the order of the system.

A system expressible in this form is guaranteed to be **both linear and time-invariant**.

**Solving for $y[n]$** (assuming $a_0 \neq 0$):

$$
y[n] = \frac{1}{a_0}\!\left(-\sum_{k=1}^{N} a_k\,y[n-k] + \sum_{k=0}^{M} b_k\,x[n-k]\right) \tag{3.18}
$$

**Solving for $x[n]$** (assuming $b_0 \neq 0$):

$$
x[n] = \frac{1}{b_0}\!\left(\sum_{k=0}^{N} a_k\,y[n-k] - \sum_{k=1}^{M} b_k\,x[n-k]\right) \tag{3.19}
$$

#### 4.2 Identifying LTI systems from LCCDE

**Example 3.9** — Which of the following correspond to LTI systems?

(a) $y[n] = 0.1\,y[n-1] + x[n] + x[n-1]$  
Rearranged: $y[n] - 0.1\,y[n-1] = x[n] + x[n-1]$  
This fits (3.17) with $N = M = 1$, $a_0 = 1$, $a_1 = -0.1$, $b_0 = b_1 = 1$. => **LTI**

(b) $y[n] = x[n+1] + x[n]$  
Shift by 1: $y[n-1] = x[n] + x[n-1]$  
This fits (3.17) with $N = M = 1$, $a_0 = 0$, $a_1 = 1$, $b_0 = b_1 = 1$. => **LTI**  
(Note: despite using $x[n+1]$ which looks non-causal, the shifted form reveals it is LTI.)

(c) $y[n] = 1/x[n]$  
Cannot be put in the linear form (3.17) because $x[n]$ and $y[n]$ are not linearly related. => **Not LTI**

If a system cannot be expressed in form (3.17), it can be: linear and time-variant; nonlinear and time-invariant; or nonlinear and time-variant.

#### 4.3 Finding impulse response from LCCDE

**Example 3.10** — $y[n] = x[n] - x[n-1]$:

Expand the convolution $y[n] = \sum_m h[m]\,x[n-m] = \ldots + h[-1]\,x[n+1] + h[0]\,x[n] + h[1]\,x[n-1] + \ldots$

Matching coefficients: $h[0] = 1$ (coefficient of $x[n]$), $h[1] = -1$ (coefficient of $x[n-1]$), all other $h[m] = 0$.

$$
h[n] = \delta[n] - \delta[n-1]
$$

#### 4.4 Computing output from LCCDE recursively

**Example 3.11** — $y[n] = 0.5\,y[n-1] + x[n] + x[n-1]$, $x[n] = u[n]$, $y[-1] = 0$:

For $n = 0$: $y[0] = 0.5 \cdot 0 + 1 + 0 = 1$  
For $n \geq 1$: $x[n] = x[n-1] = 1$, so $y[n] = 0.5\,y[n-1] + 2$

The output converges quickly toward 4 (visible in Fig. 3.6: stem plot shows $y$ rising from 1 at $n=0$ to approximately 4, settling at 4 for $n \geq$ ~10).

**MATLAB implementation — loop method:**
```matlab
N = 50;
y(1) = 1;                        % y[0] = 1
for n = 2:N+1
    y(n) = 0.5*y(n-1) + 2;      % x[n] = x[n-1] = 1 for n >= 1
end
n = 0:N;
stem(n, y)
```

**MATLAB implementation — filter command:**
Rewrite as $y[n] - 0.5\,y[n-1] = x[n] + x[n-1]$:
```matlab
x = ones(1, 51);       % x[n] = u[n] for n = 0..50
a = [1, -0.5];         % a_k coefficients: a_0=1, a_1=-0.5
b = [1,  1];           % b_k coefficients: b_0=1, b_1=1
y = filter(b, a, x);
stem(0:length(y)-1, y)
```

---

### 5. Energy and Power Signals

#### 5.1 Energy signal

An **energy signal** has finite total energy:

$$
0 < E < \infty
$$

For a continuous-time signal:

$$
E = \int_{-\infty}^{+\infty} |x(t)|^2\, dt
$$

For a discrete-time signal:

$$
E = \sum_{n=-\infty}^{\infty} |x[n]|^2
$$

- Energy signals are typically **finite-duration** or **decaying** (e.g., exponentially decaying signals).
- The **average power of an energy signal is 0** (finite energy divided over infinite time).

#### 5.2 Power signal

A **power signal** has finite average power:

$$
0 < P < \infty
$$

For a continuous-time signal:

$$
P = \lim_{T \to \infty} \frac{1}{T} \int_{-T/2}^{T/2} |x(t)|^2\, dt
$$

For a discrete-time signal:

$$
P = \lim_{N \to \infty} \frac{1}{2N+1} \sum_{n=-N}^{N} |x[n]|^2
$$

- Power signals are typically **periodic** or **infinite-duration** (e.g., a sinusoid that never ends).
- The **energy of a power signal is infinite**.

#### 5.3 Key rules

| Property | Energy signal | Power signal |
|---|---|---|
| Energy $E$ | $0 < E < \infty$ | $E = \infty$ |
| Average power $P$ | $P = 0$ | $0 < P < \infty$ |
| Typical examples | Finite pulse, decaying exponential | Sinusoid, unit step |
| Generally | Aperiodic signals | Periodic signals |

- A signal **cannot** be both energy and power simultaneously.
- A signal **may** be neither (e.g., $x[n] = n^2$ which grows without bound).

#### 5.4 Worked example (Example 3.12)

**Signal (a):** $x(t) = 5$ for $-2 \leq t \leq 2$, else 0 (single rectangular pulse):

Instantaneous power: $P_x(t) = 25$ for $-2 \leq t \leq 2$, else 0.

$$
E_x = \int_{-2}^{2} 25\, dt = 100 \quad \text{(finite)}
$$

$$
P_x = \lim_{T \to \infty} \frac{1}{T} \cdot 100 = 0
$$

=> **Energy signal**

**Signal (b):** $z(t)$ = periodic repetition of the above pulse with fundamental period 8:

$$
P_z = \frac{1}{8} \int_{-2}^{2} 25\, dt = \frac{100}{8} = 12.5 \quad \text{(finite)}
$$

$$
E_z = \int_{-\infty}^{+\infty} |z(t)|^2\, dt = \infty
$$

=> **Power signal**

---

## Key terms (glossary)

- **$x[n]$** — discrete-time signal; a sequence of real or complex numbers indexed by integer $n$.
- **Sampling period $T$** — time interval between consecutive samples; $x[n] = x(nT)$.
- **$\delta[n]$ (unit impulse / unit sample)** — equals 1 at $n = 0$, zero elsewhere. Building block for all DT signals via the sifting property.
- **$u[n]$ (unit step)** — equals 1 for $n \geq 0$, zero for $n < 0$.
- **Sifting property** — $x[n] = \sum_k x[k]\,\delta[n-k]$; every signal is a weighted superposition of shifted impulses.
- **System operator $T$** — mapping from input sequence $x[n]$ to output $y[n]$.
- **Memoryless system** — $y[n]$ depends only on $x[n]$ at the same instant $n$.
- **Linear system** — obeys superposition: $T\{a\,x_1 + b\,x_2\} = a\,T\{x_1\} + b\,T\{x_2\}$.
- **Time-invariant system** — shifting input by $n_0$ produces a shift of $n_0$ in output.
- **LTI system** — linear AND time-invariant; completely characterised by its impulse response $h[n]$.
- **Causal system** — $y[n]$ depends only on $x[k]$ for $k \leq n$; for LTI: $h[n] = 0$ for $n < 0$.
- **BIBO stable system** — bounded input $\Rightarrow$ bounded output; for LTI: $\sum|h[n]| < \infty$.
- **Impulse response $h[n]$** — output of an LTI system when input $= \delta[n]$.
- **Convolution sum** — $y[n] = x[n] \otimes h[n] = \sum_m x[m]\,h[n-m]$; the universal LTI input-output formula.
- **LCCDE (Linear Constant-Coefficient Difference Equation)** — recursive equation relating $y[n-k]$ and $x[n-k]$; the standard model for implementable LTI systems.
- **Energy signal** — $0 < E = \sum|x[n]|^2 < \infty$; average power $= 0$.
- **Power signal** — $0 < P = \lim \frac{1}{2N+1}\sum|x[n]|^2 < \infty$; energy $= \infty$.

---

## Exam targets

1. **Write the sifting property** (3.4) and use it to decompose a given signal into impulses. Also express $u[n]$ in terms of $\delta[n]$ via (3.5) and derive (3.6).

2. **Test linearity** of a given system using the standard three-input approach ($x_3 = a\,x_1 + b\,x_2$, check if $T\{x_3\} = a\,T\{x_1\} + b\,T\{x_2\}$). Know that any system with $x^2$, $|x|$, or $x \cdot y$-type terms is nonlinear.

3. **Test time-invariance**: substitute $x_1[n] = x[n - n_0]$, compute $y_1[n] = T\{x_1[n]\}$, compare with $y[n - n_0]$. Systems that compress/expand the time axis ($y[n] = x[cn]$, $c \neq 1$) are always time-variant.

4. **State the causality and stability conditions for LTI systems**: $h[n] = 0$ for $n < 0$ (causality); $\sum_{n=-\infty}^{\infty} |h[n]| < \infty$ (stability/BIBO).

5. **Apply the convolution sum** (3.13) to compute $y[n]$. For infinite-length sequences, use change-of-variable and geometric series. For finite-length sequences, reduce to a finite sum; remember output length $= M + N - 1$.

6. **State the three properties of convolution** (commutative, associative, linear) with formulas (3.14)–(3.16) and their block-diagram interpretations (cascade = convolution of impulse responses; parallel = sum of impulse responses).

7. **Identify, write, and solve an LCCDE**: recognise the general form (3.17); rearrange to find $y[n]$ recursively using (3.18); implement with a loop or MATLAB `filter(b,a,x)`.

8. **Find $h[n]$ from a given difference equation** by matching the convolution expansion to the equation (Example 3.10 method).

9. **Classify a signal as energy or power**: compute $E = \sum|x[n]|^2$ and $P = \lim \frac{1}{2N+1}\sum|x[n]|^2$; apply the mutual exclusion rules.

---

## Pitfalls

- **$\delta[n]$ vs $\delta(t)$**: $\delta[n]$ equals exactly 1 at $n = 0$. The continuous Dirac delta is undefined at 0. Do not confuse them — and do not write $\delta[n] = \infty$ at $n = 0$.

- **Sifting direction**: $x[n] = \sum_k x[k]\,\delta[n - k]$. The argument of $\delta$ is $(n - k)$, not $(k - n)$. Swapping signs shifts to the wrong time.

- **Time-invariance trap with time-compression**: $y[n] = x[3n]$ looks simple but is time-variant because the input is compressed, not merely shifted. Always run the formal test.

- **Causality does not imply memorylessness**: causal systems can still use past values. Memorylessness implies causality, but causality does NOT imply memorylessness.

- **BIBO stability for LTI is about $|h[n]|$, not $|h[n]|^2$**: the condition is absolute summability ($\sum |h[n]| < \infty$), not square-summability (which is the energy condition).

- **Convolution output length**: if $x[n]$ is $M$ points and $h[n]$ is $N$ points, the output is $M + N - 1$ points. MATLAB's `conv` starts its index at 1 by default; always verify the correct starting time index by computing $y[0]$ manually.

- **LCCDE and LTI**: fitting a system into form (3.17) is both necessary and sufficient for it to be LTI. If it cannot be fitted (e.g., $y[n] = 1/x[n]$), the system is not LTI — but you still need to separately determine whether it is linear, time-invariant, or neither.

- **Energy vs Power mutual exclusion**: a signal is either energy OR power OR neither. Writing "energy = 100, so it is a power signal" is wrong. Finite energy means zero average power, making it an energy signal.

- **`filter` command argument order**: in MATLAB/Octave, `filter(b, a, x)` uses `b` for the numerator ($x$-side, $\{b_k\}$ coefficients) and `a` for the denominator ($y$-side, $\{a_k\}$ coefficients). The vectors must match the ordering in (3.17), including the leading $a_0$ term.
