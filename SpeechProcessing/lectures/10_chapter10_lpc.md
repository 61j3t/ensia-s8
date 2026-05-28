# Chapter 10 — Linear Predictive Coding (LPC)

## Bird's eye view

- **LPC** models speech via an **all-pole filter** $H(z) = G / (1 - \sum_{k=1}^{p} a_k z^{-k})$, where the $p$ coefficients $a_k$ represent the vocal tract and $G$ is the gain.
- The core idea: each speech sample $s[n]$ is **linearly predicted** from $p$ past samples; the prediction error $e[n] = s[n] - \hat{s}[n]$ is the residual (approximates the excitation).
- The optimal $a_k$ are found by **minimising total squared error**, leading to the **Yule-Walker (normal) equations** $Ra = r$ — a $p \times p$ Toeplitz system.
- The **Levinson-Durbin recursion** solves the Toeplitz system in $O(p^2)$ instead of $O(p^3)$, and simultaneously produces **reflection coefficients** $k_i$ (PARCOR) that guarantee filter stability when $|k_i| < 1$.
- The LPC spectrum $|H(e^{j\omega})|$ gives a **smooth spectral envelope**: harmonic structure is suppressed and **formant peaks become clearly visible** as poles of $H(z)$.
- LPC order $p \approx 2 \cdot (\text{number of formants expected}) \approx \text{sampling rate in kHz} + 2\text{–}4$ (rule of thumb: $p = 10\text{–}16$ for 8 kHz speech).
- Applications span speech coding (vocoders, CELP), formant tracking, pitch analysis, speaker recognition, and speech synthesis.

---

## Detailed notes

### 1. Introduction and Motivation

**Definition.** Linear Predictive Coding (LPC) is one of the most important techniques in speech processing for modeling and representing speech signals.

**Key insight — redundancy in speech.** Speech signals contain a large amount of sample-to-sample redundancy. A speech sample can be approximated as a weighted combination of previous samples. LPC exploits this redundancy to obtain:
- Compact, low-parameter representation of the vocal tract
- Efficient speech coding and compression
- Separation of the excitation source from the vocal tract filter
- Smooth spectral envelope that reveals formants

**Main applications:** speech coding/compression, formant estimation, pitch analysis, speech synthesis, speaker recognition, vocoders, mobile communications.

---

### 2. Speech Production Model (Source-Filter)

Speech is modelled in the z-domain as:

$$
S(z) = H(z) \cdot U(z)
$$

where:
- $U(z)$ = excitation signal (source)
- $H(z)$ = vocal tract transfer function (filter)
- $S(z)$ = output speech signal

**Diagram (source-filter block diagram):**
- For **voiced** speech: a periodic impulse train (pitch period $T_0$) drives the filter. The impulse train generator produces a sequence of pulses spaced $T_0$ samples apart, scaled by gain $G$, then passed through the time-varying digital filter $H(z)$ to produce $s[n]$.
- For **unvoiced** speech: a random noise generator (white noise) drives $H(z)$ instead.
- The vocal tract parameters (LPC coefficients) control $H(z)$ and change slowly over time (quasi-stationarity over ~20–30 ms frames).

**Interpretation.** The excitation source is shaped by the vocal tract resonances (formants) to produce the characteristic quality of each vowel/consonant. LPC analysis aims to estimate $H(z)$ from the observed $s[n]$.

---

### 3. All-Pole Vocal Tract Model

The vocal tract is modelled as an **all-pole (AR — auto-regressive) filter**:

$$
H(z) = \frac{G}{A(z)}
$$

where:

$$
A(z) = 1 - \sum_{k=1}^{p} a_k \cdot z^{-k}
$$

- $a_k$: LPC (vocal tract system) coefficients, $k = 1, \ldots, p$
- $p$: model order (prediction order)
- $G$: overall gain factor

The filter $H(z)$ has $p$ poles and no finite zeros (hence "all-pole"). Its denominator polynomial $A(z)$ is exactly the **prediction error (inverse) filter**.

**Why all-pole?** The acoustic tube model of the human vocal tract, modelled as a sequence of lossless cylindrical sections of varying cross-sectional area, leads naturally to an all-pole transfer function for vowels. Fricatives and nasals have zeros too, but the all-pole approximation works well in practice.

---

### 4. Linear Prediction Principle

**Prediction equation.** The current speech sample $s[n]$ is predicted from $p$ past samples:

$$
\hat{s}[n] = \sum_{k=1}^{p} a_k \cdot s[n - k]
$$

where:
- $\hat{s}[n]$: predicted value of $s[n]$
- $s[n-k]$: past speech samples ($k = 1, \ldots, p$)
- $a_k$: predictor coefficients (same as the LPC coefficients)
- $p$: prediction order

**Block diagram.** $s(n) \to [\text{Linear Predictor}] \to \hat{s}(n)$

The linear predictor is a FIR filter with coefficients $a_1, \ldots, a_p$ operating on past samples.

**Objective.** Determine the coefficients $a_k$ that minimise the prediction error energy over a frame of speech.

---

### 5. Prediction Error (Residual) Signal

**Definition.** The prediction error (residual) is:

$$
e[n] = s[n] - \hat{s}[n]
$$

Substituting:

$$
e[n] = s[n] - \sum_{k=1}^{p} a_k \cdot s[n - k]
$$

**Z-domain.** The error signal is the output of the **inverse filter** $A(z)$ applied to $S(z)$:

$$
E(z) = A(z) \cdot S(z) \implies E(z) \approx G \cdot U(z)
$$

So the inverse filter approximately recovers the (scaled) excitation.

**Characteristics of $e[n]$:**

| Speech type | Residual character |
|---|---|
| Voiced | Impulse-like peaks at pitch period boundaries; pitch information preserved |
| Unvoiced | Resembles white noise (random, flat spectrum) |

**Illustration.** For vowels "ah", "ee", "oh", "ay": the speech waveform $s[n]$ (left panels) shows quasi-periodic oscillations; the corresponding residual $e[n]$ (right panels) shows sharp impulses at each pitch period, confirming the source-filter separation.

**Physical meaning.** The inverse filter $A(z)$ cancels the vocal tract resonances, leaving mainly pitch pulses (voiced) or noise (unvoiced). This is the basis of **analysis-by-synthesis coders** such as CELP (Code-Excited Linear Prediction).

---

### 6. Inverse Filter Interpretation

Since $H(z) = G / A(z)$, applying $A(z)$ to the speech:

$$
E(z) = A(z) \cdot S(z) = A(z) \cdot H(z) \cdot U(z) = G \cdot U(z)
$$

The inverse filter $A(z)$:
- Removes vocal tract resonances (whitens the spectrum)
- Suppresses the spectral envelope
- Extracts the excitation signal

**Diagram of inverse filtering:** $s(n) \to \left[\sum_{k=1}^{p} a_k z^{-k}\right] \to \hat{s}(n) \to [-] \to e(n)$, where the subtraction is $s(n) - \hat{s}(n)$.

---

### 7. LPC Spectrum and Formant Estimation

**LPC spectrum.** Evaluate $H(z)$ on the unit circle:

$$
|H(e^{j\omega})| = \frac{|G|}{|A(e^{j\omega})|}
$$

This magnitude spectrum is the LPC spectral envelope.

**Characteristics of the LPC spectrum:**
- **Smooth** — no harmonic fine structure (harmonics are suppressed because they belong to the excitation, not the filter)
- Formants appear as **prominent peaks** corresponding to resonances of the vocal tract
- The harmonic structure visible in the Fourier spectrum is absent

**Key property.** The poles of $H(z)$ determine the formant frequencies. For a pole at $z = r \cdot e^{j\theta}$:
- **Pole angle** $\theta$ → formant frequency: $F = \theta \cdot F_s / (2\pi)$
- **Pole radius** $r$ → formant bandwidth: closer to the unit circle → narrower (sharper) formant

**Illustrated example (/ah/ phoneme, p=14, 8 kHz, 30 ms Hamming window):**
- The Fourier spectrum shows harmonic lines and a broad envelope with peaks near $F_1 \approx 700$ Hz, $F_2 \approx 1200$ Hz, $F_3 \approx 2600$ Hz.
- The LPC spectrum ($p=14$) traces a smooth curve through these peaks — the three formants are cleanly resolved.
- With $p=10$ the envelope is slightly oversmoothed; with $p=50$ it begins to track the harmonic structure, over-fitting the model.

**Order effect on spectrum:**
- Too low $p$: formants merge or are missed (underfit)
- Too high $p$: spectrum starts tracking harmonics (overfit), spurious poles appear
- Rule of thumb for 8 kHz speech: $p \approx 10\text{–}16$

---

### 8. Error Minimisation — Deriving the Normal Equations

**Total squared prediction error over a frame of $N$ samples:**

$$
E = \sum_{n=0}^{N-1} e^2[n]
$$

**Optimality condition.** Set the partial derivatives to zero:

$$
\frac{\partial E}{\partial a_i} = 0 \quad \text{for } i = 1, 2, \ldots, p
$$

Substituting $e[n] = s[n] - \sum_{k=1}^{p} a_k s[n-k]$:

$$
\sum_{n=0}^{N-1} e[n] \cdot s[n - i] = 0 \quad \text{for } i = 1, 2, \ldots, p
$$

**Orthogonality Principle.** The prediction error is orthogonal to all past speech samples used in the prediction. This is the fundamental optimality condition.

Expanding:

$$
\sum_{n=0}^{N-1} s[n] \cdot s[n-i] = \sum_{k=1}^{p} a_k \cdot \sum_{n=0}^{N-1} s[n-k] \cdot s[n-i]
$$

This gives $p$ equations in $p$ unknowns $a_1, \ldots, a_p$ — the **normal equations** (Yule-Walker equations in the autocorrelation case).

---

### 9. Autocorrelation Method

**Autocorrelation function.** For a finite frame of $N$ samples:

$$
R(i) = \sum_{n=0}^{N-1-i} s[n] \cdot s[n + i]
$$

where $R(i)$ is the autocorrelation at lag $i$, and $R(-i) = R(i)$ (symmetric).

**Properties:**
- Symmetric: $R(i) = R(-i)$
- Maximum at zero lag: $R(0) \geq R(i)$ for all $i$
- For voiced speech, $R(i)$ shows a periodicity at the pitch period

**Yule-Walker (normal) equations in autocorrelation form:**

$$
\sum_{k=1}^{p} a_k \cdot R(i - k) = R(i) \quad \text{for } i = 1, 2, \ldots, p
$$

These $p$ equations relate the LPC coefficients to the autocorrelation values of the signal.

**How $R(i)$ is obtained in practice.** The speech frame is windowed (e.g., Hamming window) before computing the autocorrelation, which ensures the autocorrelation matrix is positive definite and the Toeplitz structure is exact.

---

### 10. Matrix Form of the LPC Equations (Toeplitz System)

**Matrix formulation.** $Ra = r$

$$
R = \begin{bmatrix} R(0) & R(1) & \cdots & R(p-1) \\ R(1) & R(0) & \cdots & R(p-2) \\ \vdots & \vdots & \ddots & \vdots \\ R(p-1) & R(p-2) & \cdots & R(0) \end{bmatrix}
$$

$$
a = \begin{bmatrix} a_1 \\ a_2 \\ \vdots \\ a_p \end{bmatrix}, \qquad r = \begin{bmatrix} R(1) \\ R(2) \\ \vdots \\ R(p) \end{bmatrix}
$$

- $\mathbf{R}$: $p \times p$ autocorrelation matrix (symmetric, positive definite)
- $\mathbf{a}$: LPC coefficient vector (unknowns)
- $\mathbf{r}$: autocorrelation vector (right-hand side)

**Toeplitz structure.** $R$ is a Toeplitz matrix: $R(i,j) = R(|i-j|)$. Its entries are constant along each diagonal. Properties:
- Symmetric
- Positive definite (guarantees a unique solution)
- Constant diagonals

**Naive solution.** $a = R^{-1} r$ requires $O(p^3)$ operations (Gaussian elimination / LU decomposition) — prohibitively expensive for real-time processing.

**Efficient solution.** The Toeplitz structure enables the Levinson-Durbin recursion: $O(p^2)$ operations.

---

### 11. Levinson-Durbin Algorithm

This recursion exploits the Toeplitz structure to solve $Ra = r$ efficiently, building the order-$p$ solution from the order-$(p-1)$ solution.

**Main quantities computed at each step $i$:**
- $a_k^{(i)}$: LPC coefficient $k$ at iteration $i$ (order-$i$ model)
- $k_i$: reflection coefficient at step $i$ (also called PARCOR coefficient)
- $E_i$: prediction error energy at step $i$

**Initialisation (order 0):**

$$
E_0 = R(0), \qquad a_0^{(0)} = 1
$$

**Recursion at order $i$ (for $i = 1, 2, \ldots, p$):**

Step 1 — Compute reflection coefficient $k_i$:

$$
k_i = \frac{R(i) - \sum_{j=1}^{i-1} a_j^{(i-1)} \cdot R(i - j)}{E_{i-1}}
$$

The numerator is the residual autocorrelation not explained by the order-$(i-1)$ model. The denominator normalises by the previous error energy.

Step 2 — Update LPC coefficients:

$$
a_i^{(i)} = k_i
$$

$$
a_j^{(i)} = a_j^{(i-1)} - k_i \cdot a_{i-j}^{(i-1)} \quad \text{for } j = 1, 2, \ldots, i-1
$$

This "flips" the previous-order coefficients using the new reflection coefficient.

Step 3 — Update prediction error energy:

$$
E_i = (1 - k_i^2) \cdot E_{i-1}
$$

Since $(1 - k_i^2) \leq 1$, the error is non-increasing at each step. The **normalised error** at step $i$ is:

$$
V^{(i)} = E^{(i)} / R(0)
$$

**Final output.** After $i = p$ steps: $a_k = a_k^{(p)}$ for $k = 1, \ldots, p$, plus the set of reflection coefficients $k_1, \ldots, k_p$.

**Stability condition.**

$$
|k_i| < 1 \quad \text{for all } i = 1, \ldots, p
$$

If all reflection coefficients have magnitude strictly less than 1, then ALL poles of $H(z)$ lie strictly inside the unit circle, guaranteeing a stable synthesis filter. This is automatically monitored during the recursion.

**Complexity comparison.**
| Method | Complexity |
|---|---|
| Direct matrix inversion | $O(p^3)$ |
| Levinson-Durbin recursion | $O(p^2)$ |

**Advantages of Levinson-Durbin:**
- Exploits Toeplitz structure — avoids full matrix inversion
- Simultaneously produces reflection coefficients (useful for lattice filters, speech coding)
- Fast and memory efficient — suitable for real-time processing
- Provides gain estimation and residual analysis
- Amenable to hardware implementation

---

### 12. Reflection Coefficients (PARCOR) and the Vocal Tract

**Physical meaning.** Reflection coefficients model acoustic reflections at junctions between adjacent tube sections in the vocal tract.

The vocal tract is approximated as a series of $p$ lossless cylindrical tube sections of varying cross-sectional areas $A_1, A_2, \ldots, A_p$ (from glottis to lips). At each junction $k$ between section $k$ and section $k+1$, there is a partial reflection of the acoustic wave.

**Area function formula.** The reflection coefficient at junction $i$ is:

$$
r_i = \frac{A_{i+1} - A_i}{A_{i+1} + A_i}
$$

where $A_i$ is the cross-sectional area of tube section $i$.

**Relationship to Levinson-Durbin.** The reflection coefficients $r_i$ from the tube model are related to Levinson-Durbin's $k_i$ by:

$$
r_i = -k_i
$$

**Interpretation:**
- If $A_{i+1} = A_i$ (no area change) then $r_i = 0$ — no reflection at that junction
- Large area changes (e.g., at a constriction for a consonant) produce larger $|r_i|$
- The magnitude $|r_i|$ measures the energy reflected at each junction

**Lattice filter structure.** The reflection coefficients parameterise a **lattice filter** that is an equivalent realisation of the LPC analysis/synthesis filter. Each stage of the lattice corresponds to one tube section. The lattice form is numerically robust and directly uses $k_i$ values.

**Alternate name: PARCOR (PARtial CORrelation) coefficients.** The $k_i$ are also the partial correlation coefficients of the speech process — they measure the correlation between $s[n]$ and $s[n-i]$ after removing the effect of $s[n-1], \ldots, s[n-i+1]$.

---

### 13. Practical LPC Analysis Chain

A complete LPC analysis system for a speech frame proceeds through these stages:

| Stage | Purpose |
|---|---|
| **Pre-emphasis** | Boost high-frequency content ($H(z) = 1 - \alpha z^{-1}$, $\alpha \approx 0.97$) to compensate for spectral tilt in speech |
| **Framing** | Segment signal into short overlapping frames (~20–30 ms, ~50% overlap) within which speech is quasi-stationary |
| **Windowing** | Apply a Hamming window $w[n]$ to each frame to reduce edge effects and ensure Toeplitz structure of $R$ |
| **Autocorrelation computation** | Compute $R(0), R(1), \ldots, R(p)$ from the windowed frame |
| **Levinson-Durbin recursion** | Solve the Toeplitz system to obtain $a_1, \ldots, a_p$ and $k_1, \ldots, k_p$ |
| **LPC coefficient estimation** | Output $\{a_k\}$ for synthesis/coding, or $\{k_i\}$ for lattice implementation |

**Pre-emphasis detail.** The filter $H_{\text{pre}}(z) = 1 - \alpha z^{-1}$ (first-order FIR high-pass) increases the energy of high-frequency components before LPC analysis, making the spectral envelope more uniform and improving the conditioning of $R$.

**Order selection rule of thumb:** For speech sampled at $F_s$ kHz, choose $p \approx F_s + 2$ to $4$. For $F_s = 8$ kHz: $p \approx 10\text{–}12$. For $F_s = 16$ kHz: $p \approx 18\text{–}20$. More precisely, each formant requires approximately 2 poles, and the vocal tract has $\sim F_s/2000$ formants below $F_s/2$.

---

### 14. LPC Results and Interpretation

**RMS prediction error vs. order $p$ (empirical results):**

Experiment: 30 ms Hamming-windowed frames at 8 kHz, autocorrelation method.

- **Voiced speech:** normalised RMS error drops rapidly from ~1.0 at $p=0$ to ~0.1–0.2 by $p=8\text{–}10$, then plateaus. Voiced speech is highly linearly predictable (strong periodicity).
- **Unvoiced speech:** error drops more slowly and plateaus at a higher level (~0.4–0.5 by $p=10$). Unvoiced speech is less linearly predictable (noise-like, less redundancy).
- This difference in predictability validates the source-filter model: voiced excitation is more structured and thus more predictable.

**LPC spectrum vs. Fourier spectrum (for /ah/, p=14):**
- The short-time Fourier spectrum shows discrete harmonics at multiples of $F_0$ and a broad envelope.
- The LPC spectrum ($p=14$) traces a smooth envelope through the harmonic peaks — formants are clearly resolved.
- LP Analysis is a method of short-time spectral envelope estimation with removal of the excitation fine structure — a form of wideband spectrum analysis.
- With $p=10$: slightly oversmoothed, formants still visible.
- With $p=20$: good fit.
- With $p=50$: starts to track individual harmonics, defeating the purpose.

**Male vs. female speaker comparison (autocorrelation method):**
- Male: frame size ~320 samples (at 11 kHz), LPC order $p=14$. Prediction error shows clear pitch pulses at ~8–10 ms spacing.
- Female: frame size ~490 samples (at 14 kHz), LPC order $p=16$. Pitch period shorter (~5–6 ms), prediction error shows sharper pulses.
- The log magnitude spectrum of the error signal is approximately flat (white-noise-like), confirming that the LPC model has captured the spectral envelope.

---

### 15. Applications of LPC

**Speech coding.** LPC parameters ($p$ coefficients + gain + voicing + pitch) can represent a speech frame with far fewer bits than waveform coding. The basic **LPC vocoder** transmits only these parameters; the synthesizer reconstructs speech by driving $H(z)$ with an appropriate excitation.

**CELP (Code-Excited Linear Prediction).** A more advanced coder where the excitation is chosen from a codebook (not just a simple impulse train or noise). Used in GSM, CDMA, VoIP (G.729, AMR). This is the dominant speech coding paradigm since the 1980s.

**Vocoders.** The channel vocoder uses LPC to separate the spectral envelope from the source, enabling independent manipulation of pitch and timbre.

**Formant tracking.** Tracking the poles of $H(z)$ over time gives formant trajectories $F_1(t), F_2(t), F_3(t)$ — key features for vowel recognition and articulatory analysis.

**Pitch estimation.** The LPC residual $e[n]$ has a cleaner periodic structure than $s[n]$, making pitch detection easier (peaks of $|e[n]|$ correspond to glottal closure instants).

**Speaker recognition.** LPC coefficients (or derived features like LPC cepstrum / LPCC) characterise the vocal tract shape, which is speaker-specific.

**Speech synthesis.** Text-to-speech systems use LPC filter banks to synthesize natural-sounding speech from stored or computed LPC parameters.

**Mobile communications.** LPC is the backbone of low-bit-rate speech codecs used in mobile phones (GSM uses RPE-LTP at 13 kbit/s; CDMA uses QCELP/EVRC, all LPC-based).

---

### 16. Advantages and Limitations

**Advantages:**
- Compact representation: $p+1$ parameters (coefficients + gain) per frame instead of $N$ samples
- Efficient computation: $O(p^2)$ via Levinson-Durbin; suitable for real-time hardware
- Good spectral modeling: smoothly captures the spectral envelope
- Principled: grounded in the acoustic tube model and least-squares optimality
- Dual parameterisation: $\{a_k\}$ for filter implementation; $\{k_i\}$ for lattice/coding

**Limitations:**
- All-pole assumption: cannot represent zeros (anti-resonances), which occur in nasal sounds and fricatives. An all-zero or pole-zero model would be more accurate but more expensive.
- Sensitive to noise: the autocorrelation matrix becomes ill-conditioned in noisy environments, degrading the estimated poles.
- Less accurate for nasal sounds: nasals have prominent zeros (anti-resonances from the nasal cavity), which a pure all-pole model cannot capture.
- Frame-stationarity assumption: LPC is derived assuming stationarity within each frame; rapid spectral changes (e.g., stop consonants) are poorly captured.

---

## Key terms (glossary)

- **LPC (Linear Predictive Coding):** A method for modeling the spectral envelope of speech using an all-pole filter whose coefficients minimise the prediction error energy.
- **All-pole filter (AR model):** A filter with transfer function $H(z) = G / A(z)$ having only poles (no finite zeros).
- **Prediction order $p$:** The number of past samples used in the linear predictor; controls model resolution.
- **LPC coefficients $a_k$:** The weights in the linear prediction sum; equivalent to the negative of the denominator polynomial coefficients of $H(z)$.
- **Prediction error $e[n]$:** The residual $s[n] - \hat{s}[n]$; approximates the excitation signal.
- **Inverse filter $A(z)$:** The analysis filter that whitens speech and extracts the residual; $A(z) = 1 - \sum a_k z^{-k}$.
- **Normal equations (Yule-Walker equations):** The system $Ra = r$ obtained by setting $\partial E/\partial a_i = 0$; the optimality condition for LPC coefficients.
- **Autocorrelation method:** Method for computing $R(i) = \sum s[n]s[n+i]$ over a windowed frame, yielding a Toeplitz system.
- **Toeplitz matrix:** A matrix whose entries $R(i,j) = R(|i-j|)$ depend only on the lag, not the absolute position. Enables efficient algorithms.
- **Levinson-Durbin recursion:** An $O(p^2)$ algorithm that solves the Toeplitz LPC system order-by-order, yielding both LPC coefficients and reflection coefficients.
- **Reflection coefficients $k_i$ (PARCOR):** Intermediate quantities from Levinson-Durbin; physically correspond to acoustic reflections at vocal tract tube junctions; stability guaranteed iff $|k_i| < 1$ for all $i$.
- **Orthogonality principle:** The prediction error is orthogonal to all past speech samples used in the predictor — the fundamental LPC optimality condition.
- **LPC spectrum:** $|H(e^{j\omega})|$ — the smooth spectral envelope derived from the LPC filter; reveals formants.
- **Formant:** A resonance of the vocal tract visible as a peak in the spectrum; related to a complex pole pair of $H(z)$.
- **Pole angle / radius:** For a complex pole $z = r \cdot e^{j\theta}$: $\theta$ controls formant frequency, $r$ controls formant bandwidth.
- **Pre-emphasis:** A high-pass filter ($1 - \alpha z^{-1}$) applied before LPC analysis to flatten the spectral slope and improve conditioning.
- **Voiced / Unvoiced:** Voiced speech is produced by periodic glottal vibration (more predictable); unvoiced by turbulent noise (less predictable).
- **CELP (Code-Excited Linear Prediction):** An LPC-based speech coder where the residual excitation is quantised from a codebook; the dominant standard for low-bit-rate speech coding.
- **LPCC (LPC Cepstrum):** Cepstral coefficients derived from LPC coefficients; used in speech recognition as features that are more robust than raw $a_k$.
- **Vocal tract area function:** $A_1, \ldots, A_p$ — the cross-sectional areas of the equivalent tube sections; related to reflection coefficients by $r_i = (A_{i+1} - A_i)/(A_{i+1} + A_i)$.

---

## Exam targets

1. **State the LPC prediction equation** and define every symbol: $\hat{s}[n] = \sum_{k=1}^{p} a_k s[n-k]$.

2. **Derive the normal equations.** Start from $E = \sum e^2[n]$, apply $\partial E/\partial a_i = 0$, invoke the orthogonality principle, and arrive at $\sum_{k=1}^{p} a_k R(i-k) = R(i)$.

3. **Write the matrix form $Ra = r$.** Identify $R$ as a symmetric positive-definite Toeplitz matrix; state why direct inversion is $O(p^3)$ and why Levinson-Durbin is preferred ($O(p^2)$).

4. **Explain and apply the Levinson-Durbin recursion** step by step: initialisation $E_0 = R(0)$; reflection coefficient formula; coefficient update rule; error energy update $E_i = (1 - k_i^2)E_{i-1}$.

5. **State the stability condition:** $|k_i| < 1$ for all $i$ ensures all poles of $H(z)$ are strictly inside the unit circle.

6. **Explain the inverse filter.** Show $E(z) = A(z)S(z) \approx GU(z)$; interpret the residual for voiced vs. unvoiced speech.

7. **Describe the LPC spectrum and formant extraction.** $|H(e^{j\omega})|$ is smooth; poles give formant frequency (angle) and bandwidth ($1 - \text{radius}$).

8. **Explain the source-filter model and its connection to LPC.** $S(z) = H(z)U(z)$; $H(z)$ is estimated by LPC; $e[n]$ estimates the excitation.

9. **Compare LPC spectrum orders:** low $p$ → oversmoothed (missed formants); high $p$ → tracks harmonics (overfitted).

10. **Describe the full practical LPC analysis chain:** pre-emphasis → framing → windowing → autocorrelation → Levinson-Durbin → coefficients.

11. **Give the physical meaning of reflection coefficients** in terms of the acoustic tube model: $r_i = (A_{i+1} - A_i)/(A_{i+1} + A_i)$; relation to $k_i$: $r_i = -k_i$.

12. **Name and describe three applications of LPC** with enough detail to be convincing (CELP coding, formant tracking, pitch estimation).

---

## Pitfalls

- **$a_k$ are NOT the denominator of $H(z)$ directly.** $H(z) = G / (1 - \sum a_k z^{-k})$. The denominator is $A(z) = 1 - \sum a_k z^{-k}$, and the $a_k$ appear with a MINUS sign. Students often write $H(z) = G / (\sum a_k z^{-k})$ — wrong.

- **The inverse filter is $A(z)$, not $1/H(z)$ in the naive sense.** $E(z) = A(z)S(z)$. $A(z) = 1 - \sum a_k z^{-k}$ — it is the ANALYSIS filter; $H(z) = G/A(z)$ is the SYNTHESIS filter.

- **Levinson-Durbin produces $k_i$, not $r_i$.** The reflection coefficients from the tube model are $r_i = -k_i$. Confusing sign conventions is a common error.

- **Stability: $|k_i| < 1$ (strict), not $\leq$.** If any $|k_i| = 1$ the filter is marginally stable (poles ON the unit circle); $|k_i| > 1$ means unstable (poles outside). The condition must be strict for a stable all-pole filter.

- **Error energy is non-increasing, not necessarily strictly decreasing.** $E_i = (1 - k_i^2)E_{i-1}$; if $k_i = 0$ then $E_i = E_{i-1}$ (no improvement at that order).

- **Autocorrelation method requires windowing to guarantee Toeplitz structure.** Without a window (or with a rectangular window applied incorrectly), the autocorrelation matrix may not be Toeplitz.

- **LPC order $p$ is for the denominator, not total poles.** $H(z)$ has exactly $p$ poles — each complex-conjugate pole pair represents one formant. So $p=10$ gives 5 formant candidates (though not all will be vocal-tract formants).

- **Vocoders using LPC do NOT transmit the speech waveform.** They transmit parameters $\{a_k, G, \text{voiced/unvoiced, pitch period}\}$. The receiver synthesizes speech from these. This is a lossy process.

- **The LPC residual is not the same as the glottal flow.** It approximates the excitation, but includes simplifications from the all-pole model.

- **Pre-emphasis must be undone (de-emphasis) at the synthesis stage.** The analysis filter $1 - \alpha z^{-1}$ must be balanced by a synthesis de-emphasis filter $1/(1 - \alpha z^{-1})$.
