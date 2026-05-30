# Chapter 2 — Review of Analog Signal Analysis

## Bird's eye view

- **Two tools for analyzing analog signals**: Fourier series (for continuous-time *periodic* signals) and Fourier transform (for continuous-time *aperiodic* signals); both convert between the time domain $x(t)$ and the frequency domain $X(j\Omega)$.
- **Fourier series** decomposes a periodic signal into harmonically related complex exponentials at discrete frequencies ..., $-\Omega_0$, 0, $\Omega_0$, $2\Omega_0$, ...; the spectrum is **discrete and aperiodic**.
- **Fourier transform** decomposes an aperiodic signal into a continuum of frequencies; the spectrum $X(j\Omega)$ is **continuous and aperiodic** and is also called the spectrum.
- Key special signals: the **Dirac delta** $\delta(t)$ (unit area, sifting property) and the **unit step** $u(t)$. The FT of $\delta(t)$ is 1 (flat spectrum). The FT of $e^{j\Omega_0 t}$ is $2\pi\delta(\Omega - \Omega_0)$.
- **Analog LTI systems** are fully characterized by the impulse response $h(t)$; output = convolution $y(t) = x(t) \otimes h(t)$. Convolution in time $\leftrightarrow$ multiplication in frequency: $Y(j\Omega) = X(j\Omega)H(j\Omega)$.
- **FT can be derived from FS** by letting the period $T \to \infty$ ($\Omega_0 \to 0$), turning the discrete sum into an integral — the inverse FT formula.
- **Impulse train** (sum of shifted deltas, period $T$) has FT that is also an impulse train in frequency with spacing $\Omega_0 = 2\pi/T$ — a central result for sampling theory.

---

## Detailed notes

### 1. Fourier Series

#### 1.1 Periodicity and fundamental frequency

A continuous-time function $x(t)$ is **periodic** if there exists $T_p > 0$ such that:

$$
x(t) = x(t + T_p), \quad \text{for all } t \in (-\infty, \infty)
$$
*(Eq. 2.2)*

The smallest such $T_p$ is called the **fundamental period**. The **fundamental (angular) frequency** is:

$$
\Omega_0 = \frac{2\pi}{T_p} \quad (\text{rad/s})
$$
*(Eq. 2.3)*

In the frequency domain, $\Omega$ only takes discrete values: ..., $-\Omega_0$, 0, $\Omega_0$, $2\Omega_0$, ...

#### 1.2 Fourier series synthesis and analysis formulas

Every periodic function can be expanded (synthesized) as:

$$
x(t) = \sum_{k=-\infty}^{\infty} a_k \cdot e^{jk\Omega_0 t}, \quad t \in (-\infty, \infty)
$$
*(Eq. 2.4)*

The **Fourier series coefficients** (analysis formula):

$$
a_k = \frac{1}{T_p} \int_{-T_p/2}^{T_p/2} x(t) \cdot e^{-jk\Omega_0 t} \, dt, \quad k = \ldots,-1,0,1,2,\ldots
$$
*(Eq. 2.5)*

The set $\{a_k\}$ characterizes $X(j\Omega)$ — it is the frequency representation of $x(t)$.

#### 1.3 Magnitude and phase of coefficients

Since $a_k$ is generally complex:

$$
|a_k| = \sqrt{(\text{Re}\{a_k\})^2 + (\text{Im}\{a_k\})^2}
$$
*(Eq. 2.6)*

$$
\angle(a_k) = \arctan\!\left(\frac{\text{Im}\{a_k\}}{\text{Re}\{a_k\}}\right)
$$
*(Eq. 2.7)*

Fig. 2.1 (described): The left panel shows a continuous, periodic time-domain signal with period $T_p$. The right panel shows its frequency-domain representation as a discrete set of vertical lines (impulses) at multiples of $\Omega_0$, with heights $\{a_k\}$. Summary: continuous + periodic in time $\leftrightarrow$ discrete + aperiodic in frequency.

#### 1.4 Procedure for computing Fourier series coefficients

1. Determine the fundamental period $T_p$ and fundamental frequency $\Omega_0 = 2\pi/T_p$.
2. For each $k$, multiply $x(t)$ by $e^{-jk\Omega_0 t}$, integrate over one complete period $[-T_p/2, T_p/2]$, and divide by $T_p$.
3. Handle $k = 0$ and $k \neq 0$ separately to avoid $0/0$ indeterminate forms.

---

#### Example 2.1 — Cosine sum (inspection method)

Find the FS coefficients for $x(t) = \cos(10\pi t) + \cos(20\pi t)$.

- Fundamental frequency: $\Omega_0 = 10\pi$, so $T_p = 2\pi/\Omega_0 = 1/5$.
- Using Euler's formula $\cos(u) = (e^{ju} + e^{-ju})/2$:

$$
x(t) = \frac{1}{2}e^{-j2\Omega_0 t} + \frac{1}{2}e^{-j\Omega_0 t} + \frac{1}{2}e^{j\Omega_0 t} + \frac{1}{2}e^{j2\Omega_0 t}
$$

- By inspection (matching with 2.4): **$a_{-2} = a_{-1} = a_1 = a_2 = 1/2$**, all other $a_k = 0$.

---

#### Example 2.2 — Mixed sinusoids

Find the FS coefficients for $x(t) = 1 + \sin(\Omega_0 t) + 2\cos(\Omega_0 t) + \cos(3\Omega_0 t + \pi/4)$.

Using Euler formulas ($\sin(u) = (e^{ju} - e^{-ju})/(2j)$):

$$
x(t) = 1 + \left(1 + \frac{1}{2j}\right)e^{j\Omega_0 t} + \left(1 - \frac{1}{2j}\right)e^{-j\Omega_0 t}
       + \frac{1}{2}e^{j\pi/4}e^{j3\Omega_0 t} + \frac{1}{2}e^{-j\pi/4}e^{-j3\Omega_0 t}
$$

Results:

$$
a_{-3} = \frac{\sqrt{2}}{4}(1 - j), \quad k = -3
$$

$$
a_{-1} = 1 + \frac{j}{2}, \quad k = -1
$$

$$
a_0 = 1, \quad k = 0
$$

$$
a_1 = 1 - \frac{j}{2}, \quad k = 1
$$

$$
a_3 = \frac{\sqrt{2}}{4}(1 + j), \quad k = 3
$$

$$
a_k = 0, \quad \text{otherwise}
$$

Computing magnitude and phase for $k = -3$:

$$
|a_{-3}| = \sqrt{\left(\frac{\sqrt{2}}{4}\right)^2 + \left(-\frac{\sqrt{2}}{4}\right)^2} = \frac{1}{2}
$$

$$
\angle(a_{-3}) = \arctan(-1) = -\frac{\pi}{4}
$$

---

#### Example 2.3 — Rectangular pulse train (sinc-shaped spectrum)

$x(t)$ is a periodic pulse train with fundamental period $T$ and pulse width $2T_0$ (where $T > 2T_0$). Over one period:

$$
x(t) = \begin{cases} 1, & -T_0 < t < T_0 \\\\ 0, & \text{otherwise} \end{cases}
$$

Fundamental frequency: $\Omega_0 = 2\pi/T$.

**For $k = 0$:**

$$
a_0 = \frac{1}{T} \int_{-T_0}^{T_0} 1 \, dt = \frac{2T_0}{T}
$$

**For $k \neq 0$:**

$$
a_k = \frac{1}{T} \int_{-T_0}^{T_0} e^{-jk\Omega_0 t} \, dt
    = \frac{-1}{jk\Omega_0 T} \cdot e^{jk\Omega_0 t}\Big|_{-T_0}^{T_0}
    = \frac{\sin(k\Omega_0 T_0)}{k\pi}
    = \frac{\sin(2\pi k T_0/T)}{k\pi}
$$

Note: The $k=0$ case is verified by L'Hôpital's rule on the $k\neq 0$ formula:

$$
\lim_{k \to 0} \frac{\sin(2\pi k T_0/T)}{k\pi} = \frac{2T_0}{T}
$$

which matches $a_0$ — confirming consistency.

---

### 2. Fourier Transform

#### 2.1 Definition

The **Fourier transform** of a continuous-time, *aperiodic* signal $x(t)$:

$$
X(j\Omega) = \int_{-\infty}^{\infty} x(t) \cdot e^{-j\Omega t} \, dt
$$
*(Eq. 2.8)*

$X(j\Omega)$ is also called the **spectrum** of $x(t)$. Unlike the FS, $\Omega$ here is continuous.

The **inverse Fourier transform**:

$$
x(t) = \frac{1}{2\pi} \int_{-\infty}^{\infty} X(j\Omega) \cdot e^{j\Omega t} \, d\Omega
$$
*(Eq. 2.9)*

Notation: $x(t) \leftrightarrow X(j\Omega)$

Fig. 2.2 (described): Left panel — a smooth, non-periodic (finite-energy) time-domain signal. Right panel — its spectrum $X(j\Omega)$ is a smooth continuous curve (semi-circular shape shown). Both sides are continuous and aperiodic. This contrasts with the FS case.

#### 2.2 The Dirac delta function

The delta function $\delta(t)$ is defined by:

$$
\delta(t) = 0, \quad t \neq 0
$$
*(Eq. 2.10)*

$$
\int_{-\infty}^{\infty} \delta(t) \, dt = 1
$$
*(Eq. 2.11)*

Key property — **sifting property**:

$$
\int_{-\infty}^{\infty} f(t) \cdot \delta(t - t_0) \, dt = f(t_0)
$$
*(Eq. 2.12)*

where $f(t)$ is any continuous-time signal.

Interpretation: $\delta(t)$ is zero everywhere except at $t = 0$, where it is not well-defined (it has an "infinite" value but finite area = 1). Represented graphically as an arrow at $t = 0$ with label 1 (Fig. 2.3).

#### 2.3 The unit step function

$$
u(t) = \begin{cases} 1, & t > 0 \\\\ 0, & t < 0 \end{cases}
$$
*(Eq. 2.13)*

$u(0)$ is not well defined (discontinuity at $t = 0$).

#### 2.4 Important Fourier transform pairs

**Pair 1 — Rectangular pulse $\to$ Sinc** (Example 2.4):

$$
x(t) = \begin{cases} 1, & -T_0 < t < T_0 \\\\ 0, & \text{otherwise} \end{cases}
$$

$$
X(j\Omega) = \frac{2\sin(\Omega T_0)}{\Omega} = 2T_0 \cdot \operatorname{sinc}\!\left(\frac{\Omega T_0}{\pi}\right)
$$

where $\operatorname{sinc}(u) = \sin(\pi u)/(\pi u)$.

Fig. 2.4 (described): Left — rectangular pulse of height 1, width $2T_0$. Right — sinc-shaped spectrum with peak value $2T_0$ at $\Omega=0$ and zero-crossings at $\pm\pi/T_0$, $\pm 2\pi/T_0$, ...

**Pair 2 — Rectangular spectrum $\to$ Sinc in time** (Example 2.5, inverse FT):

$$
X(j\Omega) = \begin{cases} 1, & -W_0 < \Omega < W_0 \\\\ 0, & \text{otherwise} \end{cases}
$$

$$
x(t) = \frac{\sin(W_0 t)}{\pi t} = \frac{W_0}{\pi} \cdot \operatorname{sinc}\!\left(\frac{W_0 t}{\pi}\right)
$$

Fig. 2.5 (described): Left — rectangular pulse in frequency domain with width $2W_0$. Right — sinc function in time with peak $W_0/\pi$ and zero-crossings at $\pm\pi/W_0$, $\pm 2\pi/W_0$, ...

**Duality observation**: Comparing Examples 2.4 and 2.5 — a rectangular pulse in time gives a sinc in frequency, and a rectangular pulse in frequency gives a sinc in time. This is the **duality property** of the Fourier transform.

**Pair 3 — One-sided exponential** (Example 2.6):

$$
x(t) = e^{-at} \cdot u(t), \quad a > 0
$$

$$
X(j\Omega) = \frac{1}{a + j\Omega} = \frac{a - j\Omega}{a^2 + \Omega^2}
$$

$$
|X(j\Omega)| = \frac{1}{\sqrt{a^2 + \Omega^2}}
$$

$$
\angle(X(j\Omega)) = -\arctan(\Omega/a)
$$

Fig. 2.6 (described): Left magnitude plot — bell-shaped curve with peak $1/a$ at $\Omega=0$, falling to $\sqrt{2}/(2a)$ at $\Omega=\pm a$. Right phase plot — starts at 0 for large negative $\Omega$, crosses through $-\pi/4$ at $\Omega=a$, and asymptotes to $-\pi/2$ for large positive $\Omega$.

**Pair 4 — Delta function** (Example 2.7):

Using the sifting property with $f(t) = e^{-j\Omega t}$ and $t_0 = 0$:

$$
X(j\Omega) = \int_{-\infty}^{\infty} \delta(t) e^{-j\Omega t} \, dt = e^{-j\Omega \cdot 0} = 1
$$

Therefore: $\delta(t) \leftrightarrow 1$ (flat spectrum at all frequencies).

**Pair 5 — Complex exponential** (derived from Pair 4 via inverse FT):

$$
e^{j\Omega_0 t} \leftrightarrow 2\pi\,\delta(\Omega - \Omega_0)
$$
*(Eq. 2.16)*

Derivation: If $X(j\Omega) = 2\pi\delta(\Omega - \Omega_0)$, then by the inverse FT:

$$
x(t) = \frac{1}{2\pi}\int 2\pi\delta(\Omega - \Omega_0)e^{j\Omega t}\,d\Omega = e^{j\Omega_0 t}. \checkmark
$$

#### 2.5 FT of a periodic signal (via FS coefficients)

Combining (2.4) and (2.16): the FT of a periodic signal expressed via its FS is:

$$
\sum_{k=-\infty}^{\infty} a_k e^{jk\Omega_0 t} \;\leftrightarrow\; \sum_{k=-\infty}^{\infty} 2\pi a_k\,\delta(\Omega - k\Omega_0)
$$
*(Eq. 2.17)*

The spectrum of a periodic signal is a train of impulses at the harmonic frequencies $k\Omega_0$, each weighted by $2\pi a_k$.

#### 2.6 Impulse train and its FT (Example 2.8)

The **impulse train** (also called Dirac comb):

$$
x(t) = \sum_{k=-\infty}^{\infty} \delta(t - kT)
$$

is periodic with period $T$. Using (2.5), the FS coefficients are:

$$
a_k = \frac{1}{T} \int_{-T/2}^{T/2} \delta(t)\, e^{-jk\Omega_0 t} \, dt = \frac{1}{T}
$$

Using (2.17), its Fourier transform is:

$$
X(j\Omega) = \frac{2\pi}{T} \sum_{k=-\infty}^{\infty} \delta\!\left(\Omega - \frac{2\pi k}{T}\right) = \Omega_0 \sum_{k=-\infty}^{\infty} \delta(\Omega - k\Omega_0)
$$

Fig. 2.7 (described): Left — impulse train in time with equal unit-height impulses at ..., $-2T$, $-T$, 0, $T$, $2T$, ... Right — impulse train in frequency with equal height $\Omega_0$ impulses at ..., $-2\Omega_0$, $-\Omega_0$, 0, $\Omega_0$, $2\Omega_0$, ... Key insight: the FT of an impulse train is another impulse train — the spacing in time ($T$) and the spacing in frequency ($\Omega_0 = 2\pi/T$) are reciprocally related.

#### 2.7 Derivation of the Fourier transform from the Fourier series

The FT can be obtained from the FS by a limiting argument. Construct $\tilde{x}(t)$ as the periodic extension of $x(t)$ with period $T$ (Fig. 2.8). As $T \to \infty$ (i.e., $\Omega_0 \to 0$), $\tilde{x}(t) \to x(t)$.

The FS coefficients of $\tilde{x}(t)$ are:

$$
a_k = \frac{1}{T} X(jk\Omega_0)
$$
*(Eq. 2.20)*

so the FS expansion of $\tilde{x}(t)$ is:

$$
\tilde{x}(t) = \frac{1}{2\pi} \sum_{k=-\infty}^{\infty} \Omega_0\, X(jk\Omega_0)\, e^{jk\Omega_0 t}
$$
*(Eq. 2.21)*

As $\Omega_0 \to 0$, the sum becomes a Riemann integral:

$$
x(t) = \lim_{\Omega_0 \to 0} \tilde{x}(t) = \frac{1}{2\pi} \int_{-\infty}^{\infty} X(j\Omega)\, e^{j\Omega t} \, d\Omega
$$
*(Eq. 2.22)*

This is exactly the inverse FT formula — confirming internal consistency.

---

### 3. Analog Linear Time-Invariant (LTI) Systems

#### 3.1 Properties defining an LTI system

**Linearity**: If $(x_1(t), y_1(t))$ and $(x_2(t), y_2(t))$ are input-output pairs, then:

$$
a \cdot x_1(t) + b \cdot x_2(t) \;\to\; a \cdot y_1(t) + b \cdot y_2(t)
$$

**Time-invariance**: If $x(t) \to y(t)$, then:

$$
x(t - t_0) \;\to\; y(t - t_0)
$$

(A time shift in the input causes the same time shift in the output — the system does not change over time.)

#### 3.2 Convolution (input-output relationship)

The output of an LTI system is given by the **convolution integral**:

$$
y(t) = x(t) \otimes h(t) = \int_{-\infty}^{\infty} x(\tau)\, h(t - \tau) \, d\tau
$$
*(Eq. 2.23)*

where:
- $x(t)$ = input signal
- $h(t)$ = **impulse response** (the system's output when the input is $\delta(t)$)
- $y(t)$ = output signal

The impulse response $h(t)$ **completely characterizes** the LTI system.

#### 3.3 Convolution-multiplication duality

**Convolution in the time domain corresponds to multiplication in the frequency domain:**

$$
x(t) \otimes h(t) \;\leftrightarrow\; X(j\Omega) \cdot H(j\Omega)
$$
*(Eq. 2.24)*

where $H(j\Omega)$ is the **transfer function** (= FT of the impulse response $h(t)$).

**Proof** (change of variable $u = t - \tau$):

$$
Y(j\Omega) = \mathcal{F}\{x(t) \otimes h(t)\}
$$

$$
= \int_{-\infty}^{\infty} \int_{-\infty}^{\infty} x(\tau)\,h(t-\tau)\,e^{-j\Omega t} \, d\tau \, dt
$$

$$
= \int_{-\infty}^{\infty} \int_{-\infty}^{\infty} x(\tau)\,h(u)\,e^{-j\Omega\tau}\,e^{-j\Omega u} \, d\tau \, du \quad [u = t-\tau]
$$

$$
= \left[\int_{-\infty}^{\infty} x(\tau)\,e^{-j\Omega\tau}\,d\tau\right] \cdot \left[\int_{-\infty}^{\infty} h(u)\,e^{-j\Omega u}\,du\right]
$$

$$
= X(j\Omega) \cdot H(j\Omega)
$$

Therefore $y(t) = \mathcal{F}^{-1}\{ X(j\Omega)\, H(j\Omega) \}$.

---

## Key terms (glossary)

| Term | Definition |
|---|---|
| **Fundamental period $T_p$** | Smallest positive $T_p$ such that $x(t) = x(t + T_p)$ for all $t$. |
| **Fundamental frequency $\Omega_0$** | $\Omega_0 = 2\pi/T_p$ (rad/s); the lowest frequency in the FS. |
| **Fourier series coefficients $\{a_k\}$** | Complex numbers characterizing the frequency content of a periodic signal at each harmonic $k\Omega_0$. |
| **Harmonic** | A frequency component at an integer multiple $k$ of $\Omega_0$. |
| **Fourier transform $X(j\Omega)$** | Continuous-frequency representation (spectrum) of an aperiodic signal. |
| **Spectrum** | Another name for $X(j\Omega)$; fully describes the signal in the frequency domain. |
| **Dirac delta $\delta(t)$** | Generalized function: zero for $t\neq 0$, integrates to 1; defined via the sifting property. |
| **Sifting property** | $\int f(t)\,\delta(t-t_0)\,dt = f(t_0)$; extracts the value of $f$ at $t_0$. |
| **Unit step $u(t)$** | 1 for $t>0$, 0 for $t<0$; undefined at $t=0$. |
| **sinc function** | $\operatorname{sinc}(u) = \sin(\pi u)/(\pi u)$; appears as the FT of a rectangular pulse. |
| **Duality property** | If $x(t) \leftrightarrow X(j\Omega)$, then $X(jt) \leftrightarrow 2\pi x(-\Omega)$; rect in time $\leftrightarrow$ sinc in freq and vice versa. |
| **Impulse train** | $\sum_{k} \delta(t-kT)$; periodic signal whose FT is also an impulse train. |
| **Impulse response $h(t)$** | Output of an LTI system when the input is $\delta(t)$; fully characterizes the system. |
| **Transfer function $H(j\Omega)$** | FT of the impulse response $h(t)$; used for frequency-domain analysis of LTI systems. |
| **Convolution** | $y(t) = \int x(\tau)\,h(t-\tau)\,d\tau$; the time-domain operation for LTI output. |
| **Linearity** | Superposition holds: weighted sums of inputs map to weighted sums of outputs. |
| **Time-invariance** | System behaviour does not depend on when the input is applied. |

---

## Exam targets

1. **Write the FS synthesis and analysis formulas** (2.4 and 2.5) from memory, including correct limits of integration and the $1/T_p$ factor.

2. **Compute FS coefficients for a given periodic signal** — use Euler's formulas to convert sinusoids into complex exponentials, then read off the coefficients by inspection or integrate (2.5) directly. Always handle $k=0$ and $k\neq 0$ separately.

3. **Compute FS coefficients for a rectangular pulse train** — know the sinc-shaped result $a_k = \sin(k\Omega_0 T_0)/(k\pi) = \sin(2\pi k T_0/T)/(k\pi)$ and the DC term $a_0 = 2T_0/T$. Verify using L'Hôpital.

4. **Write the FT pair (2.8) and (2.9)** from memory. Know the difference from the FS: aperiodic $\leftrightarrow$ continuous $\Omega$; periodic $\leftrightarrow$ discrete $\Omega$.

5. **State and apply the sifting property** (2.12). Use it to compute the FT of $\delta(t) = 1$ and the FT of $e^{j\Omega_0 t} = 2\pi\delta(\Omega-\Omega_0)$.

6. **Derive the FT of a periodic signal from its FS**: given FS coefficients $\{a_k\}$, the FT is a sum of weighted impulses at harmonics — equation (2.17).

7. **Derive the FT of the impulse train** (Example 2.8) — the FT is also an impulse train with spacing $\Omega_0 = 2\pi/T$.

8. **State the convolution-multiplication property** (2.24) and reproduce its proof (change of variable $u = t - \tau$, then separate the double integral into a product of two single integrals).

9. **Define linearity and time-invariance** with precise mathematical statements.

10. **Apply the complete LTI analysis pipeline**: given $x(t)$ and $h(t)$, compute $Y(j\Omega) = X(j\Omega)H(j\Omega)$, then take the inverse FT to get $y(t)$.

---

## Pitfalls

- **$\Omega$ vs $\omega$ notation**: This course uses $\Omega$ for analog (continuous-time) angular frequency in rad/s. Do not confuse with $\omega$ which is sometimes used for the same thing in other texts.
- **Factor of $1/(2\pi)$ in the inverse FT**: The forward FT has no factor; the inverse FT divides by $2\pi$ (eq. 2.9). Getting this wrong is a common error.
- **Factor of $1/T_p$ in the FS analysis formula** (eq. 2.5): The synthesis formula (2.4) has no prefactor; the analysis formula divides by $T_p$. Also easy to swap.
- **$k = 0$ vs $k \neq 0$ in FS computation**: The general formula for $k\neq 0$ gives $0/0$ at $k=0$. Always compute $a_0$ separately as a simple average (integral of $x(t)/T_p$ over one period), then verify by L'Hôpital on the general formula.
- **Periodic $\to$ discrete spectrum; aperiodic $\to$ continuous spectrum**: Never apply the FS to an aperiodic signal or confuse which formula to use. Periodic/aperiodic in time $\leftrightarrow$ discrete/continuous in frequency.
- **FT of a periodic signal is not a regular function**: It involves Dirac deltas (impulses) in the frequency domain (eq. 2.17) — do not expect a smooth $X(j\Omega)$.
- **Duality is not symmetry**: Duality says $X(jt) \leftrightarrow 2\pi x(-\Omega)$, not $X(jt) \leftrightarrow x(\Omega)$. The $2\pi$ factor and sign flip on $\Omega$ are often forgotten.
- **Impulse response vs transfer function**: $h(t)$ is in the time domain; $H(j\Omega) = \mathcal{F}\{h(t)\}$ is in the frequency domain. They are a FT pair — do not confuse them.
- **Convolution is commutative**: $x(t) \otimes h(t) = h(t) \otimes x(t)$. The impulse response can be written either way in the integral.
- **$\delta(t)$ vs $\delta(\Omega)$**: $\delta(t) \leftrightarrow 1$ (in frequency). A constant (=1) in time $\leftrightarrow$ $2\pi\delta(\Omega)$ in frequency. These are different — do not mix them up.
