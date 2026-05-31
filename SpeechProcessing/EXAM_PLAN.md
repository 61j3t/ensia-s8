# Speech Processing Exam Plan — Mon 2 June, 09:00

> **Today is Sat 31 May.** Scope: **Ch 5–10** (z-Transform onward).
> Time budget below assumes ~20 productive hours total. Tight but doable if you protect focus blocks.

---

## Time inventory

| Block | Window | Productive hours |
|---|---|---|
| **Sat 31 May (today)** | from now → bed | **6–8 h** |
| **Sun 1 June** | full day | **10–12 h** |
| **Mon 2 June** | wake → 08:30 | **2 h** review only |
| **Total** | | **18–22 h** |

The exam scope is 6 chapters × ~30-65 pp each = **~286 lecture pages / ~3,000 lines of notes**. Don't try to read it all at slide-by-slide depth. Plan: **bird's-eye → key derivations → worked problems**, not exhaustive prose reading.

---

## Prerequisites from Ch 1-4: what to check from before

You don't need to "study" Ch 1-4. You need targeted refreshes only on the things Ch 5-10 build on. Skip everything else.

### **Ch 3 — Discrete-Time Signals & Systems (REQUIRED, ~1 h)**
This is the foundation Ch 5, 8, 10 stand on. If you only have time for one prereq, do this one.
- **Unit impulse** $\delta[n]$ and **unit step** $u[n]$ — sifting property
- **LTI system** + **impulse response** $h[n]$ — what they mean
- **Convolution sum**: $y[n]=\sum_m x[m]h[n-m]$ — must understand visually + algebraically
- **BIBO stability**: $\sum |h[n]| < \infty$
- **Causality**: $h[n]=0$ for $n<0$
- Why these matter: z-transform of $h[n]$ is the **transfer function** (Ch 5). Speech is modeled as an LTI source-filter system (Ch 8). LPC fits an LTI all-pole filter (Ch 10).

### **Ch 4 — DTFT & DFT (REQUIRED, ~1 h)**
The DFT is the engine of MFCC (Ch 9) and STFT (Ch 7). The DTFT is the z-transform on the unit circle (Ch 5).
- **DTFT formula**: $X(e^{j\omega})=\sum_n x[n]e^{-j\omega n}$ — periodic in $\omega$ with period $2\pi$
- **DFT formula**: $X[k]=\sum_{n=0}^{N-1} x[n]e^{-j2\pi kn/N}$
- **Convolution ↔ multiplication** in frequency
- **The link**: DTFT exists iff signal is absolutely summable (= BIBO stable LTI). z-transform $X(z)|_{z=e^{j\omega}} = X(e^{j\omega})$.
- **Skip**: detailed proofs, all 9 DTFT properties (just know there's a convolution theorem).

### **Ch 2 — Analog Fourier (LIGHT, ~30 min)**
Only needed for **Ch 6 sampling theorem** derivation.
- **Fourier transform** of a continuous-time signal: $X(j\Omega)=\int x(t)e^{-j\Omega t}dt$
- **Dirac delta** $\delta(t)$ + sifting
- **Impulse train** ↔ impulse train in frequency (this is what gives spectrum replication when sampling)
- **Skip**: everything else (Fourier series, examples, properties).

### **Ch 1 — Intro DSP (SKIP)**
Pure conceptual, no math feeding into later chapters. Skip entirely.

**Prereq total: ~2.5 hours** (Ch 3: 1h, Ch 4: 1h, Ch 2: 30min)

---

## Per-chapter strategy (in scope)

Time budgets assume "first pass + worked exercises + active recall." Adjust based on your weak spots.

### **Ch 5 — z-Transform** (4–5 h) ★ HIGHEST PRIORITY
This is the heart of the exam. Big and math-dense. Likely the largest portion of the exam.

**Priority items**:
1. **Definition**: $X(z)=\sum_n x[n]z^{-n}$ and its link to DTFT (z on unit circle)
2. **ROC** — the 8 properties; how ROC determines whether signal is causal / anti-causal / two-sided
3. **Pole-zero plot** + **causal+stable ⇔ all poles inside unit circle**
4. **Common pairs** memorized: $\delta$, $a^n u[n]$, $-a^n u[-n-1]$, cos/sin
5. **Inverse z-transform** — focus on **partial fractions** (most exam-relevant). Power series and contour integral are less common.
6. **System analysis**: from difference equation → $H(z)=Y(z)/X(z)$ → poles/zeros → stability conclusion

**Watch on YouTube** (~30 min, optional):
- Search: **"inverse z transform partial fractions example"** (Iain Explains, Barry Van Veen)
- Search: **"z-transform ROC examples"** — concrete worked examples are key

### **Ch 6 — Sampling & Reconstruction** (2 h)
Conceptually clean; mostly Nyquist + aliasing intuition.

**Priority items**:
1. **Sampling**: $x[n] = x(nT)$, $\Omega_s = 2\pi/T$
2. **Spectrum replication**: $X_s(j\Omega) = \frac{1}{T}\sum_k X(j(\Omega - k\Omega_s))$ — draw a picture
3. **Nyquist–Shannon**: bandlimited recoverable iff $\Omega_s > 2\Omega_b$
4. **Aliasing**: what overlap looks like, why irreversible
5. **Sinc interpolation** for reconstruction (know the formula and that it's non-realizable)

**Watch** (~15 min): "Nyquist-Shannon sampling theorem visual" (3Blue1Brown-style)

### **Ch 7 — STFT & Spectrogram** (2.5 h)
Conceptually rich; bridges DFT (Ch 4) to speech (Ch 8).

**Priority items**:
1. **Motivation**: why DFT alone can't analyze non-stationary speech (no time localization)
2. **STFT formula**: $S[m,k]=\sum_n x[n]w[n-mH]e^{-j2\pi kn/N}$
3. **Window functions** comparison table — Rect, Hann, Hamming, Blackman — main-lobe width vs sidelobe trade-off
4. **Time-frequency trade-off / uncertainty principle**
5. **Narrowband vs wideband**: long window resolves harmonics + F0; short window resolves formants + pitch periods. Speech: NB ≈ 60ms, WB ≈ 6ms.
6. **How to read a spectrogram**: vowels (horizontal bands = formants), fricatives (broadband noise), stops (silence + burst)

**Watch** (~20 min): "STFT and spectrogram explained" + "Hamming window vs rectangular"

### **Ch 8 — Speech Signal Analysis** (3 h)
Largest chapter by content but mostly conceptual + a few key formulas.

**Priority items**:
1. **Source-filter model** (Z-domain cascade): $P_L(z) = R(z)V(z)G(z)S(z)$ — be able to draw the block diagram
2. **Vocal tract = all-pole filter** $V(z) = 1/(1+\sum a_k z^{-k})$ — its poles are the **formants** $F_1, F_2, F_3$
3. **Radiation** $R(z) = 1 - 0.99z^{-1}$ (high-pass, spectral-tilt correction)
4. **Time-domain measures**:
   - **Short-Time Energy**: pitch / voicing classifier
   - **Zero-Crossing Rate (ZCR)**: distinguishes voiced (low ZCR) from fricatives (high ZCR)
   - **Short-Time Autocorrelation**: pitch detection — pitch period = lag of first non-zero peak
   - **AMDF**: lighter alternative to autocorrelation
5. **Formants & phonemes**: the F1/F2/F3 table; know rough numbers for /a/, /i/, /u/
6. **Spectrogram reading**: vowels (formants), nasals (anti-resonances), fricatives, stops (silence + burst + VOT), diphthongs (moving formants)

**Watch** (~30 min): "source filter model speech production" + "formants F1 F2 explained"

### **Ch 9 — Cepstral Analysis & MFCC** (2 h)
Heavy math (homomorphic deconvolution), but the MFCC pipeline is a cleanly-memorizable sequence.

**Priority items**:
1. **Cepstrum definition**: $c[n] = \mathcal{F}^{-1}\{\log|\mathcal{F}\{x[n]\}|\}$
2. **Quefrency** domain (units: seconds, dual of frequency)
3. **Homomorphic deconvolution**: convolution → product in spectrum → sum in log-spectrum → separable by liftering
4. **Pitch from cepstrum**: $F_0 = F_s / q_0$ (where $q_0$ is the peak quefrency)
5. **Mel scale**: $\mathrm{Mel}(f) = 2595 \log_{10}(1 + f/700)$
6. **MFCC pipeline (MEMORIZE THIS SEQUENCE)**:
   pre-emphasis $(1 - 0.97 z^{-1})$ → frame (25ms / 10ms hop) → Hamming window → FFT → power spectrum → Mel filterbank → log → DCT → 12 coefficients
7. **Feature vector**: 39-dim = 12 static + energy + delta(×13) + delta-delta(×13)

**Watch** (~30 min): "MFCC step by step" (a clear pipeline video is invaluable; many on YouTube)

### **Ch 10 — Linear Predictive Coding (LPC)** (2.5 h)
Heavy math but a finite set of must-know items.

**Priority items**:
1. **All-pole model**: $H(z) = G/A(z) = G / (1 - \sum_{k=1}^p a_k z^{-k})$; residual $e[n]$ ≈ excitation
2. **Linear prediction**: predict $s[n]$ from $p$ past samples; minimize squared error
3. **Normal (Yule-Walker) equations**: $\mathbf{R}\mathbf{a} = \mathbf{r}$ — Toeplitz autocorrelation matrix
4. **Levinson-Durbin algorithm**: O(p²) recursive solver — at minimum know it exists, ideally trace one iteration
5. **Reflection coefficients** $k_i$ and **stability condition** $|k_i| < 1$ (PARCOR)
6. **LPC order rule of thumb**: $p \approx F_{s,\text{kHz}} + 2$ to $4$
7. **LPC spectrum interpretation**: pole angle → formant frequency; pole radius → formant bandwidth
8. **Limitations**: all-pole misses nasals/fricatives (which have zeros)

**Watch** (~30–45 min): "Linear predictive coding example" + "Levinson Durbin algorithm walkthrough"

---

## Day-by-day timeline

### **Saturday 31 May (today) — 6–8 h**

| Block | Time | Activity |
|---|---|---|
| 1 | 1 h | Ch 3 prereq refresh — convolution sum, BIBO, LTI |
| 2 | 1 h | Ch 4 prereq refresh — DTFT, DFT |
| **3** | **3 h** | **Ch 5 z-transform — part 1**: definition, ROC, common pairs, pole-zero, stability |
| 4 | 1.5 h | Ch 5 z-transform — part 2: inverse z (partial fractions), system analysis |
| 5 | 1 h | YouTube — z-transform worked problems (2–3 videos) |

**Stop / sleep by midnight at the latest.** Sleep matters more than 2 more hours of cramming.

### **Sunday 1 June — 10–12 h**

Morning (4 h):
| Block | Time | Activity |
|---|---|---|
| 1 | 30 min | Ch 2 light refresh — analog FT, Dirac, impulse train |
| 2 | 2 h | Ch 6 sampling & reconstruction (full chapter) |
| 3 | 1.5 h | Ch 7 STFT & spectrogram (skim full, deep-dive windows + NB/WB) |

Afternoon (4 h):
| Block | Time | Activity |
|---|---|---|
| 4 | 3 h | Ch 8 speech analysis — source-filter, formants, time-domain measures |
| 5 | 1 h | YouTube — source-filter model + spectrogram reading |

Evening (3–4 h):
| Block | Time | Activity |
|---|---|---|
| 6 | 2 h | Ch 9 cepstral + MFCC — memorize the pipeline cold |
| 7 | 2 h | Ch 10 LPC — normal equations, Levinson, LPC spectrum |

**Sleep by 23:00.** Important.

### **Monday 2 June (exam day) — 2 h review window**

Wake ~06:30. Light breakfast. **No new material**.

| Time | Activity |
|---|---|
| 06:45 – 07:30 | Flip through bird's-eye sections of each chapter (the summaries at the top of every notes file). |
| 07:30 – 08:00 | The cross-cutting table in `00_SYLLABUS_MAP.md` (Fourier-domain map + recurring themes). |
| 08:00 – 08:30 | Re-trace the **must-know derivations** on paper: (i) z-transform of $a^n u[n]$ → $1/(1 - az^{-1})$ with ROC $|z|>|a|$, (ii) MFCC pipeline list, (iii) source-filter cascade. |
| 08:30 – 08:50 | Travel + settle. |
| 09:00 | Exam. |

---

## YouTube — high-yield channels & searches

**Channels to trust for DSP**:
- **Iain Explains Signals, Systems, and Digital Comms** — short, clear, lots of worked z-transform / DTFT examples
- **Barry Van Veen** — classic DSP lecture series (longer, more rigorous)
- **3Blue1Brown** — Fourier transform intuition (one-time inspiration video)
- **Steve Brunton** — DSP playlist, very visual

**Targeted searches per chapter**:

| Ch | Search |
|---|---|
| 5 | `inverse z transform partial fractions example`, `z-transform ROC visual`, `pole zero stability` |
| 6 | `Nyquist Shannon sampling theorem proof visual`, `aliasing example sine` |
| 7 | `STFT spectrogram explained`, `window functions DSP Hamming Hann`, `narrowband vs wideband spectrogram` |
| 8 | `source filter model speech production`, `formants F1 F2 F3 explained`, `pitch detection autocorrelation` |
| 9 | `MFCC step by step explained`, `cepstrum pitch detection`, `Mel filterbank` |
| 10 | `Linear predictive coding speech`, `Levinson Durbin algorithm walkthrough`, `LPC spectrum formants` |

**Time budget for YouTube**: ~2 hours total across the exam scope. Don't go down rabbit holes — pick one solid video per chapter, watch at 1.5x.

---

## High-yield "know cold" cheatsheet

Memorize these so you can reproduce them under stress:

**z-Transform**:
- $X(z) = \sum_n x[n] z^{-n}$
- $a^n u[n] \xleftrightarrow{\mathcal{Z}} \frac{1}{1-az^{-1}}$ with ROC $|z| > |a|$
- $-a^n u[-n-1] \xleftrightarrow{\mathcal{Z}} \frac{1}{1-az^{-1}}$ with ROC $|z| < |a|$
- causal + stable $\Leftrightarrow$ all poles inside unit circle

**Sampling**:
- Nyquist rate = $2 \Omega_b$ (need $\Omega_s > 2\Omega_b$)
- Spectrum replication: $X_s(j\Omega) = \frac{1}{T}\sum_k X(j(\Omega - k\Omega_s))$

**STFT**:
- $S[m,k] = \sum_n x[n] w[n-mH] e^{-j2\pi kn/N}$
- Narrowband (long window, ≈60 ms) — resolves harmonics + F0
- Wideband (short window, ≈6 ms) — resolves formants + pitch periods

**Source-filter (Speech)**:
- $P_L(z) = R(z) V(z) G(z) S(z)$
- $V(z) = \frac{1}{1 + \sum a_k z^{-k}}$ (all-pole)
- $R(z) = 1 - 0.99 z^{-1}$
- Pitch period $T_p$ = lag of first autocorrelation peak

**MFCC pipeline (sequence!)**:
pre-emph $(1 - 0.97 z^{-1})$ → frame 25/10 ms → Hamming → FFT → power → Mel filterbank → log → DCT → 12 coeffs → +energy +Δ +ΔΔ = **39-dim**

**LPC**:
- $H(z) = G/A(z)$, all-pole
- Normal equations: $\mathbf{R}\mathbf{a} = \mathbf{r}$ (Toeplitz)
- Levinson-Durbin solves in $O(p^2)$ recursively
- Reflection coefficients $|k_i| < 1$ ⇒ stability
- Order: $p \approx F_{s,\text{kHz}} + 2..4$

---

## Anti-patterns to avoid

- **Don't read the PDFs cover-to-cover** — they're slide-by-slide, slow, low information density. Use the `.md` notes for prose-style coverage; PDFs only when you want to see the original diagrams or to verify a derivation.
- **Don't memorize ROC tables** — understand the *principle* (right-sided ↔ exterior, left-sided ↔ interior, two-sided ↔ annulus) and you can re-derive any specific case.
- **Don't go down rabbit holes on YouTube** — set a 20-min cap per video. If you can't get the concept in one watch, write a question down and move on.
- **Don't skip Ch 3-4 prereqs** — z-transform without convolution / DTFT will sink you.

---

## What to bring / setup

- Re-read each chapter's `## Pitfalls` section the night before — these are the trap questions.
- Pen + spare pen + calculator (if allowed).
- Eat breakfast Mon morning. Caffeine OK; don't over-do it.
- Get to the exam room 10 min early so you can settle.

**Good luck. You can do this.**
