# Chapter 6 — Sampling and Reconstruction of Analog Signals

## Bird's eye view

- **Sampling** converts a continuous-time signal $x(t)$ into a discrete-time sequence $x[n] = x(nT)$, where $T$ is the sampling period.
- The **CD converter** model multiplies $x(t)$ by an impulse train $i(t)$, producing a sampled signal $x_s(t)$; in the frequency domain, $X_s(j\Omega)$ is an infinite sum of shifted copies of $X(j\Omega)$ scaled by $1/T$.
- The **Nyquist–Shannon Sampling Theorem** (1928): a bandlimited signal with bandwidth $\Omega_b$ can be perfectly recovered from its samples if and only if the sampling frequency satisfies $\Omega_s > 2\Omega_b$. The minimum rate $2\Omega_b$ is the **Nyquist rate**; $\Omega_b$ itself is the **Nyquist frequency**.
- **Aliasing** occurs when $\Omega_s < 2\Omega_b$ — the spectral copies overlap and $X(j\Omega)$ cannot be recovered from $X_s(j\Omega)$. Multiple different analog signals map to the same discrete-time sequence.
- **Ideal reconstruction** (DC converter) applies a lowpass filter $H(j\Omega)$ with gain $T$ to $x_s(t)$; in the time domain this is sinc interpolation: $x_r(t) = \sum_{k} x[k]\,\mathrm{sinc}((t - kT)/T)$.
- **Practical DSP chain**: anti-aliasing lowpass filter → ADC (approximates CD converter, adds quantization error) → digital processor → DAC (approximates DC converter, cannot implement infinite sinc sum).

---

## Detailed notes

### 1. The Sampling Process

**Definition.** Sampling is the process of converting a continuous-time signal $x(t)$ into a discrete-time sequence $x[n]$ by extracting values at equally spaced time instants $t = nT$:

$$
x[n] = x(t)\big|_{t=nT} = x(nT), \qquad n = \ldots, -1, 0, 1, 2, \ldots
$$
*(Eq. 6.1)*

- $T$ is the **sampling period** (seconds); its reciprocal $F_s = 1/T$ is the **sampling frequency** (Hz).
- The angular sampling frequency is $\Omega_s = 2\pi/T$ (rad/s).

**Fig. 6.1** (conceptual): an analog signal $x(t)$ enters a switch that closes at instants $t = nT$; the output is the discrete sequence $x[n]$.

**Example 6.1.** Given $x(t) = \cos(200\pi t)$ and $T = 1/300$ s:

$$
x[n] = x(nT) = \cos\!\left(200\pi \cdot \frac{n}{300}\right) = \cos\!\left(\frac{2\pi n}{3}\right)
$$

The analog frequency is $200\pi$ rad/s; the discrete-time frequency is $2\pi/3$ rad/sample. Note the unit change: analog frequency has units rad/s, discrete-time frequency is dimensionless (rad/sample).

**Key question:** Can $x[n]$ uniquely represent $x(t)$, i.e., can we reconstruct $x(t)$ from $x[n]$?

Fig. 6.3 illustrates that three different analog signals $x_1(t)$, $x_2(t)$, $x_3(t)$ can all pass through the same sample points — so the answer is **not always yes**. It is yes only under specific conditions (see Section 3).

---

### 2. Frequency Domain Representation of the Sampled Signal

#### 2.1 Impulse-Train Sampling

Conceptually the CD converter works in two stages (Fig. 6.2):

1. **Multiply** $x(t)$ by the impulse train $i(t)$ to get $x_s(t)$.
2. **Convert** the impulse train $x_s(t)$ to the sequence $x[n]$.

The impulse train (Dirac comb):

$$
i(t) = \sum_{k=-\infty}^{\infty} \delta(t - kT)
$$
*(Eq. 6.2a)*

Multiplying $x(t)$ by $i(t)$ and using the sifting property of $\delta(t)$:

$$
x_s(t) = x(t) \cdot i(t) = x(t) \cdot \sum_{k} \delta(t - kT) = \sum_{k} x[k]\,\delta(t - kT)
$$
*(Eq. 6.2)*

**Fig. 6.2** (block diagram): $x(t)$ → multiplier ($\times i(t)$) → $x_s(t)$ → "impulse train to sequence" block → $x[n]$. Time-domain plots show $x(t)$ as a smooth curve, $x_s(t)$ as a train of weighted impulses at multiples of $T$, and $x[n]$ as a stem plot indexed by $n$.

#### 2.2 Spectrum of the Impulse Train

The Fourier transform of $i(t)$ is itself an impulse train in frequency:

$$
I(j\Omega) = \Omega_s \sum_{k=-\infty}^{\infty} \delta(\Omega - k\Omega_s), \qquad \Omega_s = \frac{2\pi}{T}
$$
*(Eq. 6.3)*

(This is a standard result from Chapter 2 of the course.)

#### 2.3 Spectrum of the Sampled Signal

Using the multiplication property of the Fourier transform (multiplication in time $\leftrightarrow$ convolution in frequency, scaled by $1/(2\pi)$):

$$
x_1(t)\cdot x_2(t) \;\leftrightarrow\; \frac{1}{2\pi}\,X_1(j\Omega) * X_2(j\Omega)
$$
*(Eq. 6.4)*

Applying (6.2)–(6.4) and the sifting property:

$$
X_s(j\Omega) = \frac{1}{T}\sum_{k=-\infty}^{\infty} X\!\left(j(\Omega - k\Omega_s)\right)
$$
*(Eq. 6.5)*

**Interpretation:** $X_s(j\Omega)$ is an **infinite sum of shifted copies** of $X(j\Omega)$, each scaled by $1/T$, centred at every integer multiple of $\Omega_s$.

---

### 3. The Nyquist–Shannon Sampling Theorem

#### 3.1 Conditions for Unique Reconstruction

For $x[n]$ to uniquely represent $x(t)$ we need two things:
1. $x(t)$ is **bandlimited**: $X(j\Omega) = 0$ for $|\Omega| \geq \Omega_b$ (bandwidth $\Omega_b$).
2. The sampling period $T$ is **sufficiently small** (i.e., $\Omega_s$ is large enough).

**Fig. 6.4** (no aliasing, $\Omega_s$ large enough): $X(j\Omega)$ is a triangle supported on $[-\Omega_b, \Omega_b]$. $I(j\Omega)$ is a train of impulses spaced $\Omega_s$ apart. $X_s(j\Omega)$ consists of non-overlapping triangular copies of $(1/T)\cdot X(j\Omega)$ centred at $0, \pm\Omega_s, \pm 2\Omega_s, \ldots$ The copies do not touch because $\Omega_s - \Omega_b > \Omega_b$, i.e., $\Omega_s > 2\Omega_b$. In this case $X(j\Omega)$ can be recovered by lowpass filtering $X_s(j\Omega)$.

#### 3.2 Formal Statement

**Nyquist Sampling Theorem (Nyquist, 1928):**

Let $x(t)$ be bandlimited with

$$
X(j\Omega) = 0, \qquad |\Omega| \geq \Omega_b
$$
*(Eq. 6.6)*

Then $x(t)$ is **uniquely determined** by its samples $x[n] = x(nT)$ if and only if:

$$
\Omega_s = \frac{2\pi}{T} > 2\Omega_b
$$
*(Eq. 6.7)*

Terminology:
- **Nyquist frequency**: $\Omega_b$ (the highest frequency in the signal)
- **Nyquist rate**: $2\Omega_b$ (the minimum allowable sampling frequency)
- $\Omega_s$ must strictly exceed the Nyquist rate to guarantee no aliasing

#### 3.3 Aliasing

**Fig. 6.5** (with aliasing, $\Omega_s < 2\Omega_b$): The triangular copies of $X(j\Omega)/T$ overlap in $X_s(j\Omega)$. The overlap region at frequencies near $\Omega_s - \Omega_b$ contains contributions from adjacent copies. It is impossible to separate out the original $X(j\Omega)$ — information is irreversibly lost. This distortion is called **aliasing**.

Aliasing means that a high-frequency component "masquerades" as a lower frequency. Different analog signals with the same samples are called **aliases** of each other.

#### 3.4 Practical Sampling Frequencies

| Application | Bandwidth $f_b = \Omega_b/(2\pi)$ | Sampling rate $f_s = \Omega_s/(2\pi)$ |
|---|---|---|
| Biomedical | < 500 Hz | 1 kHz |
| Telephone speech | < 4 kHz | 8 kHz |
| Music (CD) | < 20 kHz | 44.1 kHz |
| Ultrasonic | < 100 kHz | 250 kHz |
| Radar | < 100 MHz | 200 MHz |

**Table 6.1** — Note that $f_s$ is always strictly more than $2f_b$ in each application.

**Example 6.2.** Find the Nyquist frequency and Nyquist rate for:

$$
x(t) = 1 + \sin(2000\pi t) + \cos(4000\pi t)
$$

The component frequencies are $0$, $2000\pi$, and $4000\pi$ rad/s.
- Nyquist frequency = $4000\pi$ rad/s (highest frequency present)
- Nyquist rate = $8000\pi$ rad/s (= 4000 Hz in Hz terms)

So $f_s$ must exceed 4000 Hz to sample this signal without aliasing.

---

### 4. Reconstruction (DC Converter)

#### 4.1 Ideal Reconstruction via Lowpass Filter

To recover $x(t)$ from $x_s(t)$, we apply a lowpass filter $H(j\Omega)$ in the frequency domain (Fig. 6.6):

$$
H(j\Omega) = \begin{cases} T, & |\Omega| < \Omega_c \\ 0, & \text{otherwise} \end{cases}
$$
*(Eq. 6.8)*

where the cutoff $\Omega_c$ satisfies $\Omega_b < \Omega_c < \Omega_s - \Omega_b$ (any value in this gap works). For simplicity, set $\Omega_c$ at the midpoint:

$$
\Omega_c = \frac{\Omega_s}{2} = \frac{\pi}{T}
$$
*(Eq. 6.9)*

The gain $T$ ensures $X_r(j\Omega) = X(j\Omega)$ (since $X_s(j\Omega)$ has copies scaled by $1/T$). In the time domain: $x_r(t) = x_s(t) * h(t)$.

**Fig. 6.6**: Three panels — $X_s(j\Omega)$ (periodic triangular copies scaled $1/T$, with gap between $\Omega_b$ and $\Omega_s - \Omega_b$), $H(j\Omega)$ (rectangular window of height $T$ and width $2\Omega_c$), $X_r(j\Omega)$ (single triangle of height 1, $= X(j\Omega)$ recovered). The multiplication in frequency selects only the baseband copy.

#### 4.2 Impulse Response: the Sinc Function

Taking the inverse Fourier transform of $H(j\Omega)$:

$$
h(t) = \frac{1}{2\pi}\int_{-\pi/T}^{\pi/T} T\,e^{j\Omega t}\,d\Omega = \frac{T\sin(\pi t/T)}{\pi t} = \mathrm{sinc}(t/T)
$$
*(Eq. 6.10)*

where the sinc function is defined as:

$$
\mathrm{sinc}(u) = \frac{\sin(\pi u)}{\pi u}
$$

Key properties: $\mathrm{sinc}(0) = 1$ (by L'Hôpital's rule), $\mathrm{sinc}(n) = 0$ for all nonzero integers $n$.

**Fig. 6.7** (DC converter block diagram): $x[n]$ → "sequence to impulse train conversion" → $x_s(t)$ → $H(j\Omega)$ filter → $x_r(t)$.

#### 4.3 Sinc Interpolation Formula

Convolving $x_s(t)$ with $h(t)$:

$$
x_r(t) = x_s(t) * h(t)
        = \left(\sum_{k=-\infty}^{\infty} x[k]\,\delta(t - kT)\right) * h(t)
        = \sum_{k=-\infty}^{\infty} x[k]\,h(t - kT)
        = \sum_{k=-\infty}^{\infty} x[k]\,\mathrm{sinc}\!\left(\frac{t - kT}{T}\right)
$$
*(Eq. 6.11)*

This is the **sinc interpolation formula** — it reconstructs $x(t)$ at any real $t$ from the discrete samples $x[k]$.

#### 4.4 Verification at Sample Points

At $t = nT$:

$$
x_r(nT) = \sum_{k=-\infty}^{\infty} x[k]\,\mathrm{sinc}(n - k)
$$
*(Eq. 6.12)*

Since $\mathrm{sinc}(n-k) = 0$ when $n \neq k$ (integer argument, nonzero) and $\mathrm{sinc}(0) = 1$:

$$
x_r(nT) = x[n] = x(nT)
$$
*(Eq. 6.15)*

confirming that $x_r(t) = x(t)$ — perfect reconstruction at every sample point, and in fact everywhere for bandlimited signals.

---

### 5. Sub-Topic: Time Delay via Sinc Interpolation

**Example 6.3.** Given $x[n] = x(nT)$, generate a fractionally delayed version $y[n] = x(nT - \Delta)$ where $\Delta \neq mT$ (non-integer sample delay).

Applying (6.11) at $t = nT - \Delta$:

$$
y[n] = x(nT - \Delta) = \sum_{k=-\infty}^{\infty} x[k]\,\mathrm{sinc}\!\left(\frac{nT - kT - \Delta}{T}\right)
$$

With change of variable $l = n - k$:

$$
y[n] = \sum_{l=-\infty}^{\infty} x[n-l]\,\mathrm{sinc}\!\left(\frac{lT - \Delta}{T}\right)
$$
*(Eq. 6.11')*

**Special case:** When $\Delta = mT$ (integer multiple of $T$), this reduces to a simple shift: $y[n] = x[n-m]$. For non-integer delays, the full infinite sinc sum is required — not practical in real-time systems.

---

### 6. Sampling and Reconstruction in a Complete DSP System

#### 6.1 Ideal System (Conceptual)

**Fig. 6.8** (ideal digital processing chain):

```
x(t) → [CD converter] → x[n] → [Digital Signal Processor] → y[n] → [DC converter] → y(t)
```

- CD converter: produces $x[n]$ from $x(t)$ via impulse-train sampling
- DSP: operates entirely in the discrete-time domain to produce $y[n]$
- DC converter: reconstructs $y(t)$ from $y[n]$ via (6.16):

$$
y(t) = \sum_{k=-\infty}^{\infty} y[k]\,\mathrm{sinc}\!\left(\frac{t - kT}{T}\right)
$$
*(Eq. 6.16)*

#### 6.2 Practical System (Real-World)

**Fig. 6.9** (practical digital processing chain):

```
x(t) → [Anti-aliasing filter] → x_b(t) → [ADC] → x[n] → [DSP] → y[n] → [DAC] → y(t)
```

Four components and their ideal-to-practical substitutions:

| Ideal block | Practical block | Reason for substitution |
|---|---|---|
| CD converter | Anti-aliasing LPF + ADC | Cannot generate ideal $\delta(t)$; ADC introduces quantization error |
| DC converter | DAC | Cannot perform infinite sinc summation; cannot access future data |

Additional notes:
- Real signals may not be precisely bandlimited → the **anti-aliasing lowpass filter** (also called pre-filter) is mandatory before the ADC to bandlimit $x(t)$ to $f_b < f_s/2$.
- $x[n]$ and $y[n]$ in a real system are **quantized** (finite bit-depth) signals, not exact samples.
- The ideal sinc reconstruction (6.16) is non-causal (requires all past and future samples) and infinite — impossible in practice. DACs use practical approximations (zero-order hold, etc.).

---

### 7. Aliasing in Detail — Example 6.4

**Setup:** $x(t) = \cos(\Omega_0 t)$ sampled at $F_s = 1000$ Hz ($T = 1/1000$ s), yielding $x[n] = \cos(\pi n/4)$.

**Step 1: Find $\Omega_1$ (first/principal alias).**

Matching $x[n] = x(nT)$:

$$
\cos(\pi n/4) = \cos(\Omega_0\,n/1000)
$$

Comparing: $\Omega_0/1000 = \pi/4$, so:

$$
\Omega_1 = 250\pi \text{ rad/s} \quad (f_1 = 125 \text{ Hz})
$$

**Step 2: Find $\Omega_2$ (second alias).**

Using the periodicity of cosine: $\cos(\theta) = \cos(\theta + 2n\pi)$:

$$
\cos(\pi n/4) = \cos(\pi n/4 + 2n\pi) = \cos(9\pi n/4) = \cos(\Omega_2\,n/1000)
$$

So: $\Omega_2 = 2250\pi$ rad/s ($f_2 = 1125$ Hz).

**What does the DC converter produce?**

The DC converter (lowpass filter with cutoff at $\Omega_s/2 = 1000\pi$ rad/s = 500 Hz) only passes the **baseband alias** $\Omega_1 = 250\pi$ rad/s. It produces $x_r(t) = \cos(\Omega_1 t) = \cos(250\pi t)$, NOT $\cos(\Omega_2 t)$.

- $\cos(\Omega_2 t)$ has Nyquist frequency $2250\pi$ rad/s → Nyquist rate $= 4500\pi$ rad/s (> $\Omega_s = 2000\pi$) → $\cos(\Omega_2 t)$ is undersampled at 1000 Hz.
- $\cos(\Omega_1 t)$ has Nyquist frequency $250\pi$ rad/s → Nyquist rate $= 500\pi$ rad/s (< $\Omega_s = 2000\pi$) → $\cos(\Omega_1 t)$ is correctly sampled.

**Fig. 6.10:** Stem plot of $x[n] = \cos(\pi n/4)$ — shows 8 samples per period (period = 8 in discrete time).

**Fig. 6.11:** Two continuous-time sinusoids $\cos(\Omega_1 t)$ (solid red, slow oscillation) and $\cos(\Omega_2 t)$ (dashed blue, rapid oscillation) — they coincide at every sample instant $t = nT$, illustrating that both are valid "originals" for the same discrete-time sequence.

**Fig. 6.12:** Reconstructed $x_r(t)$ using 41 terms of the sinc sum ($k = -10$ to $30$), $T = 1/1000$ s — the output matches $\cos(\Omega_1 t)$, the lower-frequency alias.

**Example 6.5.** Synthesis of the musical note A (440 Hz) using MATLAB/OCTAVE at $f_s = 8000$ Hz:

```matlab
A = sin(2*pi*440*(0:1/8000:0.5));  % discrete-time sequence
sound(A, 8000);                     % DA conversion and playback
```

The sequence has 4001 samples covering 0.5 s. Other notes: B (493.88 Hz), C# (554.37 Hz), D (587.33 Hz), E (659.26 Hz), F# (739.99 Hz). A melody is produced by concatenating segments for each note.

---

## Key terms (glossary)

- **Sampling** — process of extracting $x(t)$ at discrete instants $t = nT$ to produce $x[n] = x(nT)$.
- **Sampling period $T$** — time interval between successive samples (seconds).
- **Sampling frequency $\Omega_s$** — angular frequency $\Omega_s = 2\pi/T$ (rad/s); in Hz: $F_s = 1/T$.
- **Impulse-train sampling** — modelling the sampling operation as multiplication of $x(t)$ by a Dirac comb $i(t) = \sum_k \delta(t - kT)$.
- **Bandlimited signal** — a signal whose Fourier transform is zero for $|\Omega| \geq \Omega_b$.
- **Bandwidth $\Omega_b$** — the highest frequency present in a bandlimited signal.
- **Nyquist frequency** — $\Omega_b$, the bandwidth of the signal (also called the Nyquist limit).
- **Nyquist rate** — $2\Omega_b$, the minimum sampling frequency required to avoid aliasing.
- **Aliasing** — spectral overlap of shifted copies of $X(j\Omega)/T$ when $\Omega_s < 2\Omega_b$; causes irreversible distortion and ambiguity between different analog signals.
- **CD converter** — continuous-to-discrete converter; conceptual model = impulse-train multiplier + sequence extractor.
- **DC converter** — discrete-to-continuous converter; conceptual model = impulse-train former + ideal lowpass filter $H(j\Omega)$.
- **Sinc interpolation** — ideal reconstruction formula $x_r(t) = \sum_k x[k]\,\mathrm{sinc}((t - kT)/T)$.
- **sinc(u)** — defined as $\sin(\pi u)/(\pi u)$; $\mathrm{sinc}(0) = 1$; $\mathrm{sinc}(\text{integer} \neq 0) = 0$.
- **Anti-aliasing filter** — lowpass filter applied before ADC to enforce bandlimitedness and prevent aliasing.
- **ADC (Analog-to-Digital Converter)** — practical approximation to the ideal CD converter; introduces quantization error.
- **DAC (Digital-to-Analog Converter)** — practical approximation to the ideal DC converter.
- **Quantization error** — error introduced by representing continuous-amplitude samples with finite-precision integers.

---

## Exam targets

1. **Write and explain equation (6.1):** $x[n] = x(nT)$. Given $x(t)$ and $T$, compute $x[n]$ explicitly (as in Example 6.1).

2. **Derive the spectrum of the sampled signal (6.5):** Start from the impulse-train definition, use the FT of $i(t)$ (equation 6.3), apply the multiplication-convolution duality (6.4), and arrive at $X_s(j\Omega) = (1/T)\sum_k X(j(\Omega - k\Omega_s))$. Be able to write all intermediate steps.

3. **State the Nyquist Sampling Theorem precisely:** Condition (6.6) on bandlimitedness + condition (6.7) $\Omega_s > 2\Omega_b$. Define Nyquist frequency vs Nyquist rate. Apply to a given signal (as in Example 6.2).

4. **Sketch spectra for both cases:** Draw $X(j\Omega)$ (bandlimited triangle), $I(j\Omega)$ (impulse train), and $X_s(j\Omega)$ — once for $\Omega_s > 2\Omega_b$ (non-overlapping, reconstruction possible) and once for $\Omega_s < 2\Omega_b$ (overlapping, aliasing). Clearly mark $\Omega_b$, $\Omega_s$, and $\Omega_s - \Omega_b$.

5. **Derive the sinc interpolation formula (6.11):** Start from $x_r(t) = x_s(t) * h(t)$, expand $x_s(t)$ as an impulse train, convolve with $h(t) = \mathrm{sinc}(t/T)$. Verify $x_r(nT) = x[n]$ using $\mathrm{sinc}(0) = 1$ and $\mathrm{sinc}(\text{integer} \neq 0) = 0$, quoting L'Hôpital for $\mathrm{sinc}(0)$.

6. **Write out the reconstruction filter $H(j\Omega)$ (6.8):** piecewise definition, explain gain $T$, and the constraint on cutoff frequency $\Omega_b < \Omega_c < \Omega_s - \Omega_b$. Know the default choice $\Omega_c = \Omega_s/2 = \pi/T$.

7. **Solve an aliasing example like Example 6.4:** Given $x[n] = \cos(\omega_0 n)$ and $F_s$, find the principal analog frequency $\Omega_1$ (compare coefficients); find a second alias $\Omega_2$ using the $2\pi$ periodicity of cosine; determine which one the DC converter will reconstruct and why (Nyquist rate comparison).

8. **Draw and explain the practical DSP chain (Fig. 6.9):** four blocks, purpose of each, why the ideal CD and DC converters cannot be realised in practice (quantization, non-causality, infinite sum).

9. **Compute Nyquist rate for multi-tone signals:** identify all component frequencies, take the highest, double it. Example: $x(t) = 1 + \sin(2000\pi t) + \cos(4000\pi t)$ → Nyquist rate = $8000\pi$ rad/s.

---

## Pitfalls

- **Nyquist frequency $\neq$ Nyquist rate.** Nyquist frequency = $\Omega_b$ (bandwidth). Nyquist rate = $2\Omega_b$. Students frequently swap these. The sampling frequency must exceed the **Nyquist rate**, not just the Nyquist frequency.
- **The condition is strict:** $\Omega_s$ **strictly greater than** $2\Omega_b$. Sampling exactly at the Nyquist rate ($\Omega_s = 2\Omega_b$) is a borderline case — the spectral copies touch but do not overlap; in theory it works for most signals but in practice it does not (the filter must be ideal brick-wall with zero transition band).
- **Aliasing is irreversible.** Once aliasing occurs (undersampling), no post-processing can recover the original $X(j\Omega)$. The anti-aliasing filter must come before the ADC.
- **$\mathrm{sinc}(u) = \sin(\pi u)/(\pi u)$, NOT $\sin(u)/u$.** The $\pi$ inside the sine and denominator is part of the definition used in this course (consistent with MATLAB's sinc).
- **The reconstruction filter gain must be $T$, not 1.** $X_s(j\Omega)$ has copies scaled by $1/T$; the filter compensates by multiplying by $T$ to restore unit amplitude.
- **Non-integer delays require the full infinite sinc sum.** A delay of $\Delta = mT$ (integer multiple) is trivial: $y[n] = x[n-m]$. A fractional delay requires an infinite weighted sum — not implementable as a simple shift.
- **The DC converter outputs the lowest-frequency alias.** When multiple analog frequencies correspond to the same $x[n]$, the ideal lowpass filter always selects the one in $[-\Omega_s/2, \Omega_s/2]$ — the principal (lowest) alias.
- **AD $\neq$ CD; DA $\neq$ DC.** Real ADCs introduce quantization error and operate with sample-and-hold circuits, not ideal Dirac deltas. Real DACs produce zero-order hold outputs, not infinite sinc sums.
- **Frequency units:** analog frequencies $\Omega$ are in rad/s; discrete-time frequencies $\omega$ are dimensionless (rad/sample). Converting: $\omega = \Omega T$. Do not mix the two.
