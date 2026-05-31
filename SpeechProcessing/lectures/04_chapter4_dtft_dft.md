# Chapter 4 — Discrete-Time Fourier Transform (DTFT) and Discrete Fourier Transform (DFT)

---

## Bird's eye view

- **DTFT** maps an aperiodic, infinite-duration discrete-time sequence $x[n]$ to a **continuous and $2\pi$-periodic** spectrum $X(e^{j\omega})$; $x[n]$ is recovered by the inverse DTFT integral over one period $[-\pi, \pi]$.
- **DFT** is a sampled version of the DTFT: sampling $X(e^{j\omega})$ at $N$ uniformly-spaced frequencies $\omega_k = 2\pi k/N$ yields $N$ complex coefficients $X[k]$ from an $N$-point finite sequence — both $x[n]$ and $X[k]$ live in $\{0, \ldots, N-1\}$.
- **Key duality** of signal representations: discrete → periodic (time-domain discrete ↔ frequency-domain periodic), and aperiodic → continuous (time-domain aperiodic ↔ frequency-domain continuous).
- Nine **DTFT properties** (linearity, time shift, frequency shift, differentiation, conjugation, time reversal, convolution, multiplication, Parseval) each have parallel **DFT counterparts** — but with circular convolution replacing linear convolution in the DFT.
- The **FFT** (Cooley-Tukey, 1965) reduces DFT complexity from $O(N^2)$ multiplications to $O(N/2 \cdot \log_2 N)$, making spectral analysis computationally practical for large $N$.
- **Frequency estimation** with DFT: locate the magnitude peak at index $k^{\ast}$ → $\hat{\omega}_0 = 2\pi k^{\ast}/N$; accuracy improves dramatically by **zero-padding** (appending zeros to push $N$ large before computing `fft`).

---

## Detailed notes

### 1. Part 1: Discrete-Time Fourier Transform (DTFT)

#### 1.1 Definition and derivation

DTFT is a frequency analysis tool for **aperiodic discrete-time signals**.

**Analysis equation (DTFT):**

```math
X(e^{j\omega}) = \sum_{n=-\infty}^{\infty} x[n]\, e^{-j\omega n}
```
*(Eq. 4.1)*

**Derivation sketch:** Construct the continuous-time sampled signal with sampling interval $T$:

```math
x_s(t) = \sum_{n=-\infty}^{\infty} x[n]\, \delta(t - nT)
```
*(Eq. 4.2)*

Taking the Fourier transform and using the sifting property of $\delta(t)$:

```math
X_s(j\Omega) = \int_{-\infty}^{\infty} x_s(t)\, e^{-j\Omega t}\, dt = \sum_{n=-\infty}^{\infty} x[n]\, e^{-j\Omega nT}
```
*(Eq. 4.3)*

Define **$\omega = \Omega T$** as the discrete-time frequency parameter. Writing $X_s(j\Omega)$ as $X(e^{j\omega})$ recovers eq. (4.1).

**Synthesis equation (Inverse DTFT):**

```math
x[n] = \frac{1}{2\pi} \int_{-\pi}^{\pi} X(e^{j\omega})\, e^{j\omega n}\, d\omega
```
*(Eq. 4.4)*

**Proof of inverse:** Substituting (4.1) into (4.4):

```math
\frac{1}{2\pi} \int_{-\pi}^{\pi} \left[\sum_m x[m]\, e^{-j\omega m}\right] e^{j\omega n}\, d\omega
= \frac{1}{2\pi} \sum_m x[m] \int_{-\pi}^{\pi} e^{j\omega(n-m)}\, d\omega
= \frac{1}{2\pi} \sum_m x[m] \cdot \frac{2\sin((n-m)\pi)}{n-m}
= x[n]
```
*(Eq. 4.5)*

(The integral evaluates to $2\pi$ when $n = m$ and $0$ otherwise — the discrete orthogonality condition.)

#### 1.2 Properties of $X(e^{j\omega})$

- $X(e^{j\omega})$ is **continuous in $\omega$** and **$2\pi$-periodic**: $X(e^{j(\omega+2\pi)}) = X(e^{j\omega})$.
- $X(e^{j\omega})$ is generally **complex-valued**. Represent using:

```math
|X(e^{j\omega})| = \sqrt{\left[\mathrm{Re}\{X(e^{j\omega})\}\right]^2 + \left[\mathrm{Im}\{X(e^{j\omega})\}\right]^2}
```
*(Eq. 4.6)*

```math
\angle X(e^{j\omega}) = \arctan\!\left(\frac{\mathrm{Im}\{X(e^{j\omega})\}}{\mathrm{Re}\{X(e^{j\omega})\}}\right)
```
*(Eq. 4.7)*

Both the magnitude spectrum and phase spectrum are continuous and $2\pi$-periodic.

**Fig. 4.1 (described):** Left panel shows a discrete aperiodic time-domain sequence $x[n]$ with stem plot. Right panel shows the resulting spectrum $X(e^{j\omega})$ which is continuous and repeats with period $2\pi$, plotted from $-3\pi$ to $2\pi$. The DTFT arrow goes left→right (analysis) and the iDTFT arrow goes right→left (synthesis). Bottom labels: "discrete and aperiodic" ↔ "continuous and periodic."

#### 1.3 Convergence of DTFT

The DTFT converges if:

```math
|X(e^{j\omega})| \leq \sum_{n=-\infty}^{\infty} |x[n]| \cdot |e^{-j\omega n}| = \sum_{n=-\infty}^{\infty} |x[n]| < \infty
```
*(Eq. 4.8)*

This is the **absolute summability** condition.

For an LTI system with impulse response $h[n]$, the following are equivalent:
- **S1.** The system is **stable**: $\sum_{n=-\infty}^{\infty} |h[n]| < \infty$
- **S2.** The DTFT of $h[n]$, i.e. $H(e^{j\omega})$, **converges**

$H(e^{j\omega})$ is also called the **system frequency response**.

#### 1.4 Worked Examples

**Example 4.1 — DTFT of unit step $u[n]$:**

```math
X(e^{j\omega}) = \sum_{n=0}^{\infty} e^{-j\omega n}
```
Since $\sum_{n=0}^{\infty} |e^{-j\omega n}| = \sum_{n=0}^{\infty} 1 = \infty$, $X(e^{j\omega})$ does NOT exist.

Equivalently: $\sum |u[n]| = \infty$ → stability condition fails → DTFT does not converge.

---

**Example 4.2 — DTFT of rectangular pulse $x[n] = u[n] - u[n-N]$ ($N=10$):**

The sequence has $x[n] = 1$ for $n = 0, 1, \ldots, N-1$ and $0$ otherwise.

```math
X(e^{j\omega}) = \sum_{n=0}^{N-1} e^{-j\omega n} = \frac{1 - e^{-j\omega N}}{1 - e^{-j\omega}}
```
To obtain closed-form magnitude and phase, factor out $e^{-j\omega N/2}$ and $e^{-j\omega/2}$:

```math
X(e^{j\omega}) = e^{-j\omega(N-1)/2} \cdot \frac{\sin(\omega N/2)}{\sin(\omega/2)}
```
Therefore:

```math
|X(e^{j\omega})| = \left|\frac{\sin(\omega N/2)}{\sin(\omega/2)}\right|
```
```math
\angle X(e^{j\omega}) = -\frac{\omega(N-1)}{2} + \angle\!\left[\frac{\sin(\omega N/2)}{\sin(\omega/2)}\right]
```
Using the sinc function $\mathrm{sinc}(u) = \sin(\pi u)/(\pi u)$, this can be rewritten as:

```math
\frac{\sin(\omega N/2)}{\sin(\omega/2)} = N \cdot \frac{\mathrm{sinc}(\omega N/(2\pi))}{\mathrm{sinc}(\omega/(2\pi))}
```
**Fig. 4.2 (described):** Upper plot shows the magnitude response $|X(e^{j\omega})|$ vs $\omega/\pi$ for $N=10$: peak value of 10 at $\omega=0$, then decaying oscillations (sinc-like envelope) going to zero at multiples of $2\pi/N$, with minor lobes between nulls. Lower plot shows the phase response $\angle X(e^{j\omega})$: a linear (sawtooth-wrapped) function with slope $-\omega(N-1)/2$, showing the expected linear phase characteristic.

---

**Example 4.3 — Inverse DTFT of ideal rectangular spectrum:**

Given $X(e^{j\omega}) = 1$ for $-\omega_0 < \omega < \omega_0$ and $0$ otherwise (ideal lowpass, $0 < \omega_0 < \pi$):

```math
x[n] = \frac{1}{2\pi} \int_{-\omega_0}^{\omega_0} e^{j\omega n}\, d\omega = \frac{\sin(\omega_0 n)}{\pi n} = \frac{\omega_0}{\pi}\mathrm{sinc}(\omega_0 n/\pi)
```
This is an **infinite-duration sinc sequence** — the ideal lowpass filter impulse response is non-causal and infinite.

---

**Example 4.4 — Inverse DTFT using time-shifting property:**

Given $X(e^{j\omega}) = e^{-j2\omega} / (1 + 0.7 e^{-j\omega})$.

Recognising that $1/(1+0.7 e^{-j\omega})$ is the DTFT of $(-0.7)^n u[n]$ and $e^{-j2\omega}$ corresponds to a 2-sample delay:

```math
x[n] = (-0.7)^{n-2}\, u[n-2]
```
---

#### 1.5 Frequency Analysis of LTI Systems

**Frequency response definition:**

```math
H(e^{j\omega}) = \sum_{n=-\infty}^{\infty} h[n]\, e^{-j\omega n}, \qquad -\pi < \omega \leq \pi
```
**Eigenfunction property:** If $x[n] = e^{j\omega_0 n}$ (complex exponential), then:

```math
y[n] = H(e^{j\omega_0})\, e^{j\omega_0 n}
```
The complex exponential $e^{j\omega_0 n}$ is an eigenfunction of any LTI system; $H(e^{j\omega_0})$ is the corresponding eigenvalue.

**Sinusoidal input:** If $x[n] = A\cos(\omega_0 n + \varphi)$, then by Euler's formula and linearity:

```math
y[n] = A\,|H(e^{j\omega_0})|\cos\!\left(\omega_0 n + \varphi + \arg H(e^{j\omega_0})\right)
```
- **Magnitude response** $|H(e^{j\omega})|$: gain applied to frequency $\omega$.
- **Phase response** $\arg H(e^{j\omega})$: phase shift applied to frequency $\omega$.
- The frequency response **completely characterizes** the LTI system in the frequency domain.

---

### 2. Properties of DTFT

DTFT properties follow from the z-transform (evaluated on the unit circle). ROC is not separately stated because the DTFT exists only when the ROC includes the unit circle.

| # | Property | Time Domain | Frequency Domain | Eq. |
|---|---|---|---|---|
| 1 | Linearity | $ax_1[n] + bx_2[n]$ | $aX_1(e^{j\omega}) + bX_2(e^{j\omega})$ | (4.8) |
| 2 | Time Shifting | $x[n - n_0]$ | $e^{-j\omega n_0} X(e^{j\omega})$ | (4.9) |
| 3 | Freq. Shifting (Modulation) | $e^{j\omega_0 n} x[n]$ | $X(e^{j(\omega-\omega_0)})$ | (4.10) |
| 4 | Differentiation | $nx[n]$ | $j\, dX(e^{j\omega})/d\omega$ | (4.11) |
| 5 | Conjugation | $x^{\ast}[n]$ | $X^{\ast}(e^{-j\omega})$ | (4.13) |
| 6 | Time Reversal | $x[-n]$ | $X(e^{-j\omega})$ | (4.14) |
| 7 | Convolution | $x_1[n] \ast x_2[n]$ | $X_1(e^{j\omega}) X_2(e^{j\omega})$ | (4.15) |
| 8 | Multiplication | $x_1[n] \cdot x_2[n]$ | $\frac{1}{2\pi} \int X_1(e^{j\tau}) X_2(e^{j(\omega-\tau)})\, d\tau$ | (4.17) |
| 9 | Parseval's Relation | $\sum\|x[n]\|^2$ | $\frac{1}{2\pi} \int_{-\pi}^{\pi} \|X(e^{j\omega})\|^2\, d\omega$ | (4.18) |

**Notes on selected properties:**

**Differentiation proof (property 4):**

Differentiating $X(e^{j\omega}) = \sum x[n]\, e^{-j\omega n}$ with respect to $\omega$:

```math
\frac{dX(e^{j\omega})}{d\omega} = \sum x[n]\, (-jn)\, e^{-j\omega n} \quad \Rightarrow \quad nx[n] \longleftrightarrow j\,\frac{dX(e^{j\omega})}{d\omega}
```
Equivalently, using the chain rule: $-e^{j\omega} dX/d(e^{j\omega}) = j\, dX/d\omega$ (4.12).

**Convolution (property 7) — LTI application:**

```math
y[n] = x[n] \ast h[n] \quad\longleftrightarrow\quad Y(e^{j\omega}) = X(e^{j\omega})\, H(e^{j\omega})
```
*(Eq. 4.16)*

Convolution in time = multiplication in frequency. This is the fundamental theorem underlying all LTI filter design.

**Multiplication (property 8):** Multiplication in time = convolution in frequency (within one period, hence the $1/2\pi$ normalization and integration over $[-\pi, \pi]$).

**Parseval's relation (property 9) — proof:**

```math
\sum_{n=-\infty}^{\infty} |x[n]|^2 = \sum x[n]\, x^{\ast}[n]
= \sum x[n] \left[\frac{1}{2\pi} \int X(e^{j\omega})\, e^{j\omega n}\, d\omega\right]^{\ast}
= \frac{1}{2\pi} \int X^{\ast}(e^{j\omega}) \left[\sum x[n]\, e^{-j\omega n}\right] d\omega
= \frac{1}{2\pi} \int_{-\pi}^{\pi} |X(e^{j\omega})|^2\, d\omega
```
*(Eq. 4.19)*

Parseval's relation states that total signal energy = total spectral energy (up to the $1/2\pi$ factor).

---

### 3. Part 2: Discrete Fourier Transform (DFT)

#### 3.1 Definition

The DFT is obtained by **sampling the DTFT** at $N$ uniformly-spaced discrete frequencies $\omega_k = 2\pi k/N$, $k = 0, 1, \ldots, N-1$.

Let $x[n]$, $n = 0, 1, \ldots, N-1$, be an **N-point finite sequence**. The DFT pair is:

**DFT (Analysis):**

```math
X[k] = \sum_{n=0}^{N-1} x[n]\, e^{-j2\pi kn/N}, \qquad 0 \leq k \leq N-1
```
*(Eq. 4.20)*

(0 otherwise)

**Inverse DFT (iDFT / Synthesis):**

```math
x[n] = \frac{1}{N} \sum_{k=0}^{N-1} X[k]\, e^{j2\pi kn/N}, \qquad 0 \leq n \leq N-1
```
*(Eq. 4.21)*

(0 otherwise)

**Key structural observations:**
- $X[k]$ is a **periodic sequence with period $N$** (since it samples the periodic DTFT).
- Both DFT and iDFT sums run from $0$ to $N-1$ → finite computation.
- The **twiddle factor** $W_N = e^{-j2\pi/N}$, so $X[k] = \sum x[n]\, W_N^{kn}$.
- DFT relates $N$ time-domain samples to $N$ frequency-domain samples.

#### 3.2 DTFT–DFT Relationship

```math
X[k] = X(e^{j\omega})\big|_{\omega = 2\pi k/N}
```
That is, the DFT is a sampled (discrete) version of the continuous DTFT. As we append more zeros to $x[n]$ (zero-padding) and increase $N$, the DFT provides a finer sampling of the DTFT curve.

**Zero-padding rule:** DFT will approach DTFT when we append infinite zeros at the end of $x[n]$. In practice, appending $L$ zeros before computing the $N+L$ point DFT produces a denser grid of $N+L$ points on the same DTFT curve.

#### 3.3 Properties of the DFT

| # | Property | Time Domain | Frequency Domain |
|---|---|---|---|
| 1 | Linearity | $ax_1[n] + bx_2[n]$ | $aX_1[k] + bX_2[k]$ |
| 2 | Time-shifting | $x[n - m]$ | $e^{-j2\pi km/N} X[k]$ |
| 3 | Freq-shifting (modulation) | $e^{-j2\pi k_0 n/N} x[n]$ | $X[k - k_0]$ |
| 4 | Time reversal | $x[-n]$ | $X[-k]$ |
| 5 | Conjugation | $x^{\ast}[n]$ | $X^{\ast}[-k]$ |
| 6 | Time-convolution | $x_1[n] \circledast x_2[n]$ (circular) | $X_1[k]\, X_2[k]$ |
| 7 | Freq-convolution | $x_1[n] \cdot x_2[n]$ | $\frac{1}{N} X_1[k] \circledast X_2[k]$ (circular) |
| 8 | Parseval's relation | $E_x = \sum_{n=0}^{N-1} \|x[n]\|^2$ | $E_x = \frac{1}{N} \sum_{k=0}^{N-1} \|X[k]\|^2$ |

**Critical distinction — Circular convolution:** In the DFT, the time-domain convolution property uses **circular (cyclic) convolution** of period $N$, not linear convolution. This is because DFT treats signals as periodic with period $N$. Linear convolution can be obtained from circular convolution by zero-padding both sequences to length $\geq N_1 + N_2 - 1$ before taking DFTs.

**Conjugate symmetry for real sequences:** For a real-valued $x[n]$:

```math
\mathrm{Re}\{X[k]\} = \mathrm{Re}\{X[N-k]\} \qquad \text{and} \qquad \mathrm{Im}\{X[k]\} = -\mathrm{Im}\{X[N-k]\}
```
This means only about half the DFT coefficients carry independent information — the upper half is the conjugate-mirror image of the lower half.

#### 3.4 Worked Examples

**Example 4.5 — DFT of $x[n] = \{1,1,1,0,\ldots,0\}$ with $N=3$:**

Using $W_N = e^{-j2\pi/N}$:

```math
X[k] = \sum_{n=0}^{2} x[n]\, W_3^{kn} = W_3^0 + W_3^k + W_3^{2k}
= e^{-j2\pi k/3}\left[1 + 2\cos(2\pi k/3)\right]
= \begin{cases} 3 & k=0 \\ 0 & k=1,2 \end{cases}
```
With $N=5$ (zero-padding with $x[3]=x[4]=0$):

```math
X[k] = W_5^0 + W_5^k + W_5^{2k}
= e^{-j2\pi k/5}\left[1 + 2\cos(2\pi k/5)\right], \quad k=0,1,2,3,4
```
The magnitude plot (Fig. 4.3) shows: stem plot at $k=0,\ldots,4$ with magnitude 3 at $k=0$ and smaller values at $k=1,2,3,4$; phase plot shows the corresponding phase angles. Zero-padding changes the DFT length and thus the frequency grid spacing ($2\pi/N$), but does not change the underlying spectrum.

---

**Example 4.6 — Frequency estimation using DFT:**

Given $x[n] = 2\cos(0.7\pi n + 1)$, $n = 0, 1, \ldots, 20$ ($N=21$ samples).

Continuous-time analogy: $e^{j\Omega_0 t} \leftrightarrow 2\pi\delta(\Omega - \Omega_0)$, so frequency is located at the spectral peak.

For DFT index $k$, the corresponding discrete frequency is $\omega = 2\pi k/N$. The DFT magnitude will have two peaks at $k=7$ ($\omega \approx 0.667\pi$) and $k=14$ ($\omega \approx -0.7\pi$, aliased).

```math
\hat{\omega}_0 = \frac{2\pi \cdot 7}{21} \approx 0.6667\pi \quad \text{(coarse estimate)}
```
To improve accuracy, **zero-pad** to $N=2001$ (append 1980 zeros):

```matlab
x = [A*cos(w*n+p)  zeros(1,1980)];
```

Peak found at $k=702$, $N=2001$:

```math
\hat{\omega}_0 = \frac{2\pi \cdot 702}{2001} \approx 0.7016\pi \quad \text{(much closer to true } 0.7\pi\text{)}
```
**Fig. 4.4 ($N=21$, no zero-padding):** Magnitude stem plot shows two prominent peaks at $k=7$ and $k=14$, with smaller oscillatory sidelobes elsewhere. Phase plot is scattered.

**Fig. 4.5 ($N=2001$, with zero-padding):** Magnitude is now nearly continuous-looking with two sharp peaks, closely approximating the DTFT. The two peaks are clearly resolved at $\omega \approx 0.7\pi$ and $\omega \approx -0.7\pi$ (reflected to near $k=1300$).

---

**Example 4.7 — Inverse DFT:**

Given $X[k] = \{1,1,1,0,0\}$ ($N=5$):

```math
x[n] = \frac{1}{N} \sum_{k=0}^{N-1} X[k]\, W_N^{-kn}
= \frac{1}{5}\!\left(W_5^0 + W_5^{-n} + W_5^{-2n}\right)
= \frac{1}{5}\, e^{j2\pi n/5}\left[1 + 2\cos(2\pi n/5)\right], \quad n=0,1,\ldots,4
```
**Fig. 4.6 (described):** Magnitude stem plot for $x[n]$ at $n=0,\ldots,4$: largest value at $n=0$ ($\approx 0.6$), smaller values at $n=1$ and $n=4$ ($\approx 0.33$), smallest at $n=2$ and $n=3$ ($\approx 0.13$). Phase plot shows corresponding angles.

This is the inverse of Example 4.5 — DFT and iDFT are exact duals for $N$-point sequences.

---

### 4. Fast Fourier Transform (FFT)

#### 4.1 Motivation

Direct DFT computation: each $X[k]$ requires $N$ complex multiplications and $N-1$ complex additions. For $k = 0, \ldots, N-1$, the total cost is:

```math
\text{DFT:} \quad N^2 \text{ complex multiplications,} \quad N(N-1) \text{ complex additions}
```
For large $N$ this is prohibitively expensive. The FFT algorithm (Cooley and Tukey, 1965) dramatically reduces this.

```math
\text{FFT:} \quad \frac{N}{2}\log_2 N \text{ complex multiplications}
```
**FFT is not a different transform — it is an efficient algorithm to compute the same DFT.**

#### 4.2 Complexity comparison table

| m | N = 2^m | DFT (N²) | FFT (N/2 log₂N) |
|---|---|---|---|
| 1 | 2 | 4 | 1 |
| 2 | 4 | 16 | 4 |
| 3 | 8 | 64 | 12 |
| 4 | 16 | 256 | 32 |
| 5 | 32 | 1,024 | 80 |
| 6 | 64 | 4,096 | 192 |
| 7 | 128 | 16,384 | 448 |
| 8 | 256 | 65,536 | 1,024 |
| 9 | 512 | 261,144 | 2,304 |
| 10 | 1,024 | 1,048,576 | 5,120 |

The speedup factor at $N=1024$ is ~204×. MATLAB/OCTAVE commands: `fft` (FFT) and `ifft` (inverse FFT).

#### 4.3 FFT applicability

- FFT is most efficient when $N$ is a power of 2 (Cooley-Tukey radix-2 algorithm).
- The `fft` command in MATLAB/OCTAVE handles arbitrary $N$ but is fastest for powers of 2.
- Like the DFT/iDFT duality, there is an iFFT corresponding to iDFT.

---

## Key terms (glossary)

- **DTFT** — Discrete-Time Fourier Transform: maps a discrete aperiodic signal to a continuous periodic spectrum via $X(e^{j\omega}) = \sum x[n]\, e^{-j\omega n}$.
- **iDTFT** — Inverse DTFT: recovers $x[n]$ from $X(e^{j\omega})$ by $x[n] = \frac{1}{2\pi} \int_{-\pi}^{\pi} X(e^{j\omega})\, e^{j\omega n}\, d\omega$.
- **DFT** — Discrete Fourier Transform: maps a length-$N$ sequence to $N$ frequency samples $X[k] = \sum x[n]\, e^{-j2\pi kn/N}$.
- **iDFT** — Inverse DFT: $x[n] = \frac{1}{N} \sum X[k]\, e^{j2\pi kn/N}$.
- **$\omega$** — Discrete-time frequency parameter (dimensionless, radians/sample); periodic with period $2\pi$.
- **Twiddle factor** $W_N$ — $e^{-j2\pi/N}$; DFT kernel written as $W_N^{kn}$.
- **Frequency response** $H(e^{j\omega})$ — DTFT of impulse response $h[n]$; completely characterizes an LTI system.
- **Eigenfunction property** — complex exponential $e^{j\omega_0 n}$ passes through LTI system unchanged in frequency; output $= H(e^{j\omega_0})\, e^{j\omega_0 n}$.
- **Absolute summability** — $\sum |x[n]| < \infty$; sufficient condition for DTFT to converge (equivalent to BIBO stability for impulse response).
- **Circular convolution** — convolution modulo $N$, the natural operation in the DFT domain; differs from linear convolution unless zero-padded.
- **Zero-padding** — appending zeros to an $N$-point sequence before DFT to increase frequency resolution (sample the DTFT on a finer grid).
- **Parseval's relation** — energy equivalence: total time-domain energy = total spectral energy (with $1/2\pi$ or $1/N$ normalization for DTFT/DFT respectively).
- **FFT** — Fast Fourier Transform; Cooley-Tukey algorithm reducing DFT cost from $O(N^2)$ to $O(N\log_2 N)$.
- **sinc function** — $\mathrm{sinc}(u) = \sin(\pi u)/(\pi u)$; the DTFT magnitude of a rectangular pulse reduces to a sinc envelope.
- **Conjugate symmetry** — for real $x[n]$: $X[k] = X^{\ast}[N-k]$; only $N/2 + 1$ independent DFT values.

---

## Exam targets

1. **Write both DTFT equations from memory** (analysis eq. 4.1, synthesis/inverse eq. 4.4) and state the domain of each (discrete aperiodic ↔ continuous periodic).

2. **Prove the inverse DTFT** (substitute analysis into synthesis, show integral gives Kronecker delta $\delta[n-m]$) — this derivation appeared in the lecture (eq. 4.5).

3. **State and apply all 9 DTFT properties** from the table. Be able to compute DTFTs of shifted, scaled, or modulated signals without redoing the full sum.

4. **Derive the DTFT of a rectangular pulse** $x[n] = u[n] - u[n-N]$: geometric series → closed form → factor to get sinc-like expression (Ex. 4.2).

5. **Write both DFT equations** (eqs. 4.20, 4.21) and state the index ranges. Explain why $X[k]$ is periodic with period $N$.

6. **Explain the DTFT–DFT relationship**: DFT = samples of DTFT at $\omega_k = 2\pi k/N$. What changes when $N$ changes (for the same underlying signal)? What does zero-padding achieve?

7. **Compare circular vs linear convolution** in the DFT context: explain why DFT convolution is circular, and how to obtain linear convolution results using zero-padding to length $\geq N_1+N_2-1$.

8. **Reproduce the DFT properties table** (at least linearity, time-shift, circular convolution, Parseval). Note differences from DTFT properties (e.g. $e^{-j2\pi km/N}$ vs $e^{-j\omega n_0}$).

9. **State FFT complexity vs DFT complexity** ($N^2$ vs $N/2\log_2 N$) and reproduce or explain the comparison table for key values of $N$. Know that FFT is an algorithm, not a new transform.

10. **Frequency estimation workflow** (Ex. 4.6): identify peak index $k^{\ast}$ in $|X[k]|$, convert to $\hat{\omega}_0 = 2\pi k^{\ast}/N$, and understand why zero-padding improves the estimate.

11. **Parseval's relation for DTFT and DFT** — write both forms and prove the DTFT version (the proof uses the iDTFT to substitute for $x^{\ast}[n]$).

---

## Pitfalls

- **DTFT output is continuous; DFT output is discrete.** Never write $X(e^{j\omega})$ with a bracket index $[k]$ or treat the DTFT as producing $N$ values.
- **The DTFT is $2\pi$-periodic — not arbitrary period.** The principal interval is $[-\pi, \pi]$ or $[0, 2\pi]$; both are valid for one period.
- **$u[n]$ does NOT have a DTFT** in the classical (absolute summability) sense — Ex. 4.1 shows the sum diverges. This is a classic exam trap.
- **DFT circular convolution ≠ linear convolution.** If you compute $X_1[k] \cdot X_2[k]$ and take iDFT, you get circular ($N$-periodic) convolution. To get linear convolution, zero-pad to at least $N_1+N_2-1$ first.
- **Zero-padding does not add new information** about the signal; it only produces a finer sampling of the existing DTFT (interpolates in frequency domain). It does NOT improve spectral resolution of two close frequencies unless the signal itself is long enough.
- **The FFT requires $N$ to be a power of 2 for peak efficiency** (radix-2 Cooley-Tukey). MATLAB's `fft` works for any $N$ but is slower for non-powers of 2.
- **Conjugate symmetry applies only for real-valued sequences**: $X[k] = X^{\ast}[N-k]$. Do not assume this for complex-valued $x[n]$.
- **DFT index $k$ maps to frequency $\omega = 2\pi k/N$**, not $\omega = k$. The DFT domain is indexed by integer $k$, the corresponding frequency is $\omega_k = 2\pi k/N$.
- **Time-shift in DFT**: $x[n-m]$ corresponds to $e^{-j2\pi km/N} X[k]$, not $e^{-j\omega m} X(e^{j\omega})$ — the exponent uses discrete index $m$ and fixed $N$.
- **iDFT has $+j$ in the exponent and a $1/N$ normalization**; iDTFT has $+j$ in the exponent and $1/(2\pi)$ normalization. Do not confuse the two.
- **The Parseval DFT form** uses $1/N$, while the DTFT form uses $1/(2\pi)$: $E_x = \frac{1}{N}\sum|X[k]|^2$ (DFT) vs $E_x = \frac{1}{2\pi}\int|X(e^{j\omega})|^2\, d\omega$ (DTFT).
