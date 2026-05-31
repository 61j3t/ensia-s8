# Chapter 8 — Speech Signal Analysis (Production, Spectral Modeling, Source–Filter Model)

> **Note:** These notes merge two source files:
> - **Main deck** (65 slides): `08_chapter8_speech_analysis.pdf` — covers speech production anatomy, phonemes, spectral modeling, source–filter LTI model, time-domain analysis (STE, ZCR, autocorrelation, AMDF), and frequency-domain analysis (spectrograms by phoneme class).
> - **Supplement** (13 slides): `08_chapter8_source_filter_model.pdf` — provides a focused mathematical treatment of the source–filter cascade (G(z), V(z), R(z)), poles, formants, and the LPC connection.

---

## Bird's eye view

- **Speech production** is modeled as a **source–filter system**: an excitation source (vocal folds or turbulence) drives a linear time-invariant **vocal tract filter**, whose output is shaped by a **radiation model** at the lips.
- **Three sound types**: voiced (periodic, vocal folds vibrating), unvoiced (turbulent noise, vocal folds open), plosive (silence + burst + aspiration/voicing).
- **Formants** (F1, F2, F3, …) are spectral peaks corresponding to resonances of the vocal tract; they are the primary perceptual cues that distinguish vowels and other phonemes.
- **Short-time analysis** is essential because speech is non-stationary — a frame of 10–30 ms is treated as locally stationary. Key time-domain measures: **Short-Time Energy (STE)**, **Zero-Crossing Rate (ZCR)**, **Short-Time Autocorrelation (STACF)**, and **AMDF**.
- **Pitch (F0)** is the fundamental frequency of voiced excitation; $F_0 = 1/T_p$ where $T_p$ is the pitch period. Autocorrelation peak-picking is the standard method for F0 estimation.
- The **complete Z-domain model** is $P_L(z) = R(z) \cdot V(z) \cdot G(z) \cdot S(z)$, a cascade of radiation, vocal tract, glottal pulse, and impulse train models.
- **LPC** (Linear Predictive Coding) estimates the all-pole vocal tract filter coefficients $a_k$, providing pole (formant) locations for coding and recognition.
- **Spectrograms** visualize how spectral content evolves over time; wideband windows reveal formant tracks, narrowband windows reveal individual harmonics.

---

## Detailed notes

### 1. Context and motivation

#### 1.1. The speech processing pipeline

Analog speech $x_c(t)$ is sampled (A/D) to produce the discrete signal $x[n]$, processed by an algorithm, then reconstructed (D/A) to $\hat{x}_c(t)$.

The goal of **speech signal analysis** is to extract a **representation** from $x[n]$ that is useful for downstream applications:

| Representation / Feature | Application |
|---|---|
| Waveform / spectral A(x, t) | Analysis / synthesis |
| Formants, pitch | Voice modification (pitch shift, time stretch) |
| Reflection coefficients | Speech coding (vocoders) |
| Voiced/unvoiced/silence | Enhancement, VAD |
| Sounds of language, speaker ID | Recognition, speaker verification |
| Emotions | Affective computing |

#### 1.2. Short-time processing paradigm

Because speech is **quasi-stationary** on short time scales but changes rapidly on longer scales:

```
x[n]  -->  short-time processing  -->  feature vector f[m]
```

- $x[n]$ = speech samples at, e.g., 8000 samples/sec
- $f[m]$ = {f1[m], f2[m], …, fL[m]}, extracted at ~100 frames/sec (one frame every 10 ms)
- L = size of feature vector (1 for pitch, 12 for autocorrelation, etc.)
- Frames must be **short** (properties approximately constant within) and **overlapping** (>50% overlap, to avoid losing signal content at boundaries)

---

### 2. Mechanisms of speech production

#### 2.1. Anatomy

The speech production system (illustrated as a sagittal cross-section of the human head):

| Component | Role in the model |
|---|---|
| **Lungs** | Power supply — provides the airstream (energy source) |
| **Vocal folds / Larynx** | Modulator — vibrate (voiced) or stay open (unvoiced) |
| **Pharynx** | Lower part of the vocal tract |
| **Oral cavity** | Variable-shape filter (tongue, jaw, lips) |
| **Nasal cavity** | Additional filter, coupled via velum |
| **Velum** | Switch — opens/closes nasal cavity |
| **Lips** | Radiation of sound into the air |

The signal path in the time domain: $e(t)$ [excitation, periodic puffs or noise] → Vocal Tract $v(t)$ [resonant cavity, modulator] → $s(t)$ [radiated speech wave].

Three source types visible in the waveform of a mixed utterance (e.g., "sh-o-p"):
- **"sh"** (unvoiced fricative): noisy, aperiodic
- **"o"** (voiced vowel): periodic
- **"p"** (plosive): impulsive burst

#### 2.2. Types of speech sounds — phonemes

**American English** has **48 phonemes** (ARPAbet system):
- 18 vowels / diphthongs
- 4 vowel-like consonants (semivowels)
- 21 standard consonants
- 4 syllabic sounds
- 1 glottal stop /Q/

**Phoneme taxonomy (voiced vs. unvoiced excitation):**

```
Phonemes
├── Vowels (front/mid/back): IY, IH, EH, AE, AH, AA, AO, ER, UH, UW, AX …   [vocal cords vibrating]
├── Diphthongs: AY, OY, AW, EY                                                [vocal cords vibrating]
├── Semivowels: W, L, R, Y                                                    [vocal cords vibrating]
└── Consonants
    ├── Nasals: M, N, NX                                                      [vocal cords vibrating]
    ├── Plosives: Voiced (B, D, G) / Unvoiced (P, T, K)
    ├── Fricatives: Voiced (V, DH, Z, ZH) / Unvoiced (F, TH, S, SH)
    ├── Affricates: J (CH voiced), CH (unvoiced)
    └── Whisper: HH (H)                                                       [noise-like excitation]
```

Example phonetic transcription: "My name is Larry" → /M/ /AY/ /N/ /EY/ /M/ /IH/ /Z/ /L/ /AE/ /R/ /IY/

---

### 3. Spectral modeling of speech — Source–Filter LTI Model

#### 3.1. Physical intuition

The speech production process is an **LTI system** that convolves the excitation with the vocal tract impulse response:

- **Source** $e(t)$: either a periodic pulse train (voiced) or broadband noise (unvoiced)
- **System (vocal tract)** $v(t)$: resonant cavity with multiple resonance frequencies (formants)
- **Output** $s(t) = e(t) \ast v(t)$ (convolution) → in frequency domain: $S(\Omega) = E(\Omega) \cdot V(\Omega)$

**Spectral picture of voiced speech:**
- $|E(\Omega)|$: a comb of harmonics spaced at $1/T_p$ (the pitch period), flat amplitude envelope
- $|V(\Omega)|$: a smooth envelope with resonance peaks at formant frequencies F1, F2, F3, …
- $|S(\Omega)| = |E(\Omega)| \cdot |V(\Omega)|$: harmonics modulated by the formant envelope

#### 3.2. LTI block diagram (detailed model)

**Full discrete-time source–filter block diagram:**

```
Pitch Period N_p
      |
[Impulse Train]  -->  [Glottal Pulse Model G(z)]  -->  x A_V  -->  \
   Generator                                                          [Voiced/Unvoiced Switch] --> e[n] --> [Vocal Tract V(z)] --> [Radiation R(z)] --> s[n]
   P(z), p[n]           G(z), g[n]                                  /
                                                                    /
[Random Noise Generator U(z), u[n]]  -->  x A_N  ----------------/
```

Each block and its parameters:

| Block | Z-domain | Role |
|---|---|---|
| Impulse train generator | $P(z)$ | $p[n] = \sum_k \delta[n - kN_p]$; models vocal fold openings |
| Glottal pulse model | $G(z)$ | Shapes impulses into realistic airflow pulses; introduces spectral tilt |
| Voiced gain | — | $u_v[n] = A_V \cdot u_G[n]$; controls voiced amplitude |
| Noise generator | $U(z)$ | $w[n] \sim \mathcal{N}(0, \sigma^2)$; white Gaussian noise for unvoiced |
| Noise gain | — | $u_n[n] = A_N \cdot w[n]$; controls unvoiced amplitude |
| V/UV switch | — | Selects voiced or unvoiced path |
| Vocal tract model | $V(z)$ | All-pole filter; shapes spectrum into formant peaks |
| Radiation model | $R(z)$ | Models lip radiation; high-pass effect |

#### 3.3. Impulse train generator

The excitation for **voiced sounds** is a periodic impulse train:

```math
s[n] = \sum_{k=-\infty}^{\infty} \delta[n - kT]
```
where $T$ = pitch period (in samples), and $k$ ranges over all integers.
- Generates periodic impulses
- Models vibration of vocal folds
- Period $T$ determines pitch: $F_0 = F_s / T$

#### 3.4. Glottal pulse model G(z)

The glottal pulse model converts ideal impulses into a realistic airflow waveform:

```math
U_G(z) = G(z) \cdot S(z)
```
- Shapes impulses into smooth, asymmetric pulses (closed phase, open phase, return phase) with period $P \approx$ 5–30 ms ($F_0 \approx$ 33–200 Hz for male, higher for female)
- **Introduces spectral tilt**: the glottal pulse spectrum falls off at approximately -12 dB/octave
- Models airflow through the vocal folds during the open phase

Glottal pulse time-domain shape (observed in a period of ~20 ms at 8 kHz):
- **Closed phase**: vocal folds fully shut, no airflow
- **Open phase**: folds open, airflow increases
- **Return phase**: folds snap shut again, creating the sharp negative transient

#### 3.5. Voiced / Unvoiced excitation

| Sound type | Excitation | Signal character |
|---|---|---|
| Voiced | Periodic impulse train through glottal filter | Periodic, harmonic structure |
| Unvoiced | Broadband random noise | Aperiodic, noise-like |

**Noise model:**

```math
u_n[n] \sim \mathcal{N}(0,\, \sigma^2)
```
White Gaussian noise with zero mean and variance $\sigma^2$. The voiced/unvoiced switch selects the appropriate excitation path.

#### 3.6. Vocal tract model V(z)

The vocal tract is modeled as an **all-pole (IIR) filter**:

```math
V(z) = \frac{1}{1 + \sum_{k=1}^{p} a_k \cdot z^{-k}}
```
- All-pole (purely recursive) — captures resonances (formants) efficiently
- The order $p$ is typically 8–14 for telephone speech (8 kHz sampling); higher for wideband
- The denominator coefficients $a_k$ are the **LPC coefficients** (see §8)
- Shape depends on the **articulators** (tongue position, jaw height, lip rounding, velum state)

#### 3.7. Radiation model R(z)

The radiation of sound at the lips acts as a **first-order high-pass filter**:

```math
R(z) = 1 - 0.99\, z^{-1} \qquad \text{(general form: } R(z) = 1 - \alpha z^{-1},\; \alpha \approx 0.99\text{)}
```
- Differentiates the acoustic volume velocity at the lips into sound pressure
- High-pass effect: boosts high frequencies by +6 dB/octave
- This partially compensates for the -12 dB/octave glottal tilt, resulting in a net -6 dB/octave spectral roll-off for voiced speech

#### 3.8. Overall system transfer function

The **complete speech production model** in Z-domain is a cascade:

```math
P_L(z) = R(z) \cdot V(z) \cdot G(z) \cdot S(z)
```
- Source–filter cascade
- Linear time-invariant approximation (valid over a short analysis frame)
- Parameters to estimate from the signal: pitch period $T$ (or $N_p$), voiced/unvoiced/silence decision, gains $A_V$ and $A_N$, glottal pulse shape, vocal tract polynomial coefficients $a_k$, radiation model parameter $\alpha$

**Analysis problem** (inverse): given $s[n]$, estimate all model parameters.

#### 3.9. Formants and poles of V(z)

**Formants** are the spectral peaks in the speech spectrum, corresponding to resonances of the vocal tract. In the all-pole model:
- Formants correspond to the **poles** of $V(z)$
- Poles close to the unit circle in the Z-plane produce sharp, prominent resonance peaks
- Each pole is a complex number:

```math
z_k = r_k \cdot e^{j\omega_k}
```
where $\omega_k$ = formant angular frequency (in rad/sample) and $r_k$ = bandwidth control ($r_k \to 1$ means narrower bandwidth / sharper formant).

**Physical interpretation:**
- Changing the vocal tract shape (articulators) moves the poles
- Different vowels = different pole locations
- /a/ (father): low F1, low F2 (~730 Hz, 1090 Hz)
- /i/ (beet): low F1, high F2 (~270 Hz, 2290 Hz) — tongue high and front
- /u/ (boot): low F1, low F2 (~300 Hz, 870 Hz) — tongue high and back

**Formant table for 10 vowels (approximate averages):**

| ARPAbet | IPA | Example | F1 (Hz) | F2 (Hz) | F3 (Hz) |
|---|---|---|---|---|---|
| IY | i | beet | 270 | 2290 | 3010 |
| IH | I | bit | 390 | 1990 | 2550 |
| EH | ε | bet | 530 | 1840 | 2480 |
| AE | æ | bat | 660 | 1720 | 2410 |
| AH | ʌ | but | 520 | 1190 | 2390 |
| AA | a | hot | 730 | 1090 | 2440 |
| AO | ɔ | bought | 570 | 840 | 2410 |
| ER | ɜ | bird | 490 | 1350 | 1690 |
| UH | ʊ | foot | 440 | 1020 | 2240 |
| UW | u | boot | 300 | 870 | 2240 |

Rules of thumb:
- **F1** increases as jaw opens (↑ jaw height → ↑ F1)
- **F2** decreases as tongue moves from front to back of mouth
- F1 and F2 together are sufficient to distinguish most vowels
- Each speaker is physically different, so formant values vary across speakers

**Formants in the spectrogram:**
- Wideband spectrogram (short window, ~5 ms): good time resolution, shows formant trajectories as horizontal bands
- Narrowband spectrogram (long window, ~25 ms): resolves individual harmonics, shows pitch structure

#### 3.10. Connection to LPC

**Linear Predictive Coding (LPC)** estimates the all-pole vocal tract filter coefficients $a_k$ by minimizing prediction error:

```math
V(z) = \frac{1}{1 + a_1 z^{-1} + a_2 z^{-2} + \cdots + a_p z^{-p}}
```
LPC provides:
- Pole locations (= formant frequencies and bandwidths)
- Efficient coded representation for **speech coding** (e.g., CELP vocoders in mobile telephony)
- Feature input for **speech recognition** (LPC cepstrum, MFCC)

---

### 4. Time-domain analysis

#### 4.1. Generic short-time processing formula

All short-time measures follow the general form:

```math
Q_{\hat{n}} = \left( \sum_{m=-\infty}^{\infty} T(x[m]) \cdot \tilde{w}[\hat{n}-m] \right)\bigg|_{n=\hat{n}}
```
where:
- $T(\cdot)$ is a linear or non-linear transformation of the signal
- $\tilde{w}[n]$ is a window function (usually finite length)
- $Q_{\hat{n}}$ is a local weighted average of $T(x[n])$ at time $n = \hat{n}$

The window $\tilde{w}$ slides along the signal; the output $Q_{\hat{n}}$ is sampled at a rate $F_s / R$ (much lower than $F_s$, e.g., 100/sec).

#### 4.2. Short-Time Energy (STE)

**Purpose:** differentiates voiced/unvoiced speech from silence (background noise).

**Definition:**

```math
E_{\hat{n}} = \sum_{m=-\infty}^{\infty} \bigl[x[m] \cdot \tilde{w}[\hat{n} - m]\bigr]^2
           = \sum_{m=-\infty}^{\infty} x^2[m] \cdot h[\hat{n} - m]
```
where $h[n] = \tilde{w}^2[n]$ (the squared window acts as a low-pass filter).

**Block diagram:**
```
x[n] ---> ( )^2 ---> x^2[n] ---> h[n] (lowpass) ---> E_{n-hat}   (at rate F_s/R)
```

**Properties:**
- Sensitive to large amplitude samples because of the squaring operation
- As window length $L$ increases, STE plots become smoother (averaging more samples)
- Provides the basis for **voiced/unvoiced** discrimination and for detecting **silence** (at medium-to-high SNR)
- Both rectangular window (RW) and Hamming window (HW) versions exist; HW gives smoother result with less leakage

#### 4.3. Short-Time Magnitude (STM)

An alternative to STE that is less sensitive to large outliers:

```math
M_{\hat{n}} = \sum_{m=-\infty}^{\infty} |x[m]| \cdot \tilde{w}[\hat{n} - m]
```
- Weighted sum of magnitudes rather than squared values
- Dynamic range of $M_{\hat{n}} \approx \sqrt{\text{dynamic range of } E_{\hat{n}}}$
- Level differences between voiced and unvoiced segments are smaller than with STE
- Avoids multiplications (uses only absolute values) — computationally lighter
- $E_n$ and $M_n$ can be sampled at ~100/sec for ~20 ms windows → efficient representation

**Comparison $E_n$ vs. $M_n$:** differences most noticeable in unvoiced regions, where STE shows spikes from large samples that STM suppresses.

#### 4.4. Zero-Crossing Rate (ZCR)

**Definition:** a zero crossing occurs when successive samples have different algebraic signs.

**Formal definition:**

```math
Z_{\hat{n}} = \frac{1}{2L_{\text{eff}}} \sum_{m=\hat{n}-L+1}^{\hat{n}} \bigl|\mathrm{sgn}(x[m]) - \mathrm{sgn}(x[m-1])\bigr| \cdot \tilde{w}[\hat{n} - m]
```
where:
- $\mathrm{sgn}(x[n]) = +1$ if $x[n] \geq 0$, $-1$ if $x[n] < 0$
- For a rectangular window: $\tilde{w}[n] = 1$ for $0 \leq n \leq L-1$, else $0$; $L_{\text{eff}} = L$

**Key property for a sinusoid at frequency $F_0$:**

```math
z_1 = \frac{2F_0}{F_s} \quad \text{crossings/sample} \qquad \text{(ZCR proportional to frequency)}
```
```math
z_M = M \cdot \frac{2F_0}{F_s} \quad \text{crossings per } M \text{ samples}
```
**Block diagram:**
```
x[n] ---> [sign( )] ---> first difference ---> |·| ---> lowpass w-tilde[n] ---> Z_{n-hat}
```
(Same structural form as $E_{\hat{n}}$ and $M_{\hat{n}}$.)

**Practical properties:**
- High ZCR → high-frequency content → unvoiced
- Low ZCR → low-frequency content → voiced
- Used with 15 ms windows, sampled at 100/sec
- Typical observed ranges: voiced speech ZCR ≈ 10–30 crossings per window; unvoiced ≈ 40–80

**Combined use of STE and ZCR:**
- Voiced: high energy, low ZCR
- Unvoiced: lower energy, high ZCR
- Silence: very low energy, high ZCR (background noise is broadband)

#### 4.5. Short-Time Autocorrelation (STACF)

**Definition:** the autocorrelation of a windowed segment at lag $k$:

```math
r_k = \sum_{i=0}^{N-k-1} s_i \cdot s_{i+k}
```
- $r_0$ = energy (the zero-lag value equals the signal energy in the window)
- $r_k$ is symmetric: $r_{-k} = r_k$

**Properties:**
- **Emphasises periodicity**: for a periodic signal with period $T_0$, $r_k$ has peaks at $k = T_0, 2T_0, 3T_0, \ldots$
- ACF is the basis for many spectrum analysis methods
- STACF is the basis for most **pitch detectors** (fundamental frequency estimators): find the first peak of $r_k$ beyond $k = 0$; the lag at the peak = pitch period $T_0$, so $F_0 = F_s / T_0$
- ACF is computationally expensive (inner loop for every data sample)
- STACF is often combined with ZCR to build a voiced/unvoiced detector

**Illustration of ACF for a voiced segment ($L=401$, $f_s=8$ kHz, $f_0=148$ Hz):**
- $T_p \approx 54$ samples
- Waveform shows clear periodicity
- ACF shows a prominent peak at lag ≈ 54 samples
- Spectrum (Fourier of ACF) shows $F_0$ at 148 Hz and harmonics

#### 4.6. Average Magnitude Difference Function (AMDF)

**Definition:**

```math
\gamma_{\hat{n}}[k] = \sum_{m=-\infty}^{\infty} \bigl|x[\hat{n}+m]\cdot\tilde{w}_1[m] - x[\hat{n}+m-k]\cdot\tilde{w}_2[m-k]\bigr|
```
where $\tilde{w}_1[m]$ and $\tilde{w}_2[m]$ are rectangular windows. If both windows have the same length, AMDF is structurally similar to STACF.

**Properties:**
- Used for $F_0$ estimation instead of autocorrelation
- Minimum of AMDF at lag $k = T_0$ (the pitch period)
- Operations are subtractions and absolute values → much simpler than multiplications
- Faster computation than autocorrelation

**AMDF plots:**
- Voiced (V): clearly periodic minima at multiples of $T_0$
- Unvoiced (U): no clear minima, essentially flat and noisy

---

### 5. Frequency-domain analysis — Spectrograms by phoneme class

The **spectrogram** is the standard time-frequency analysis tool for speech: it displays $|\text{STFT}(\omega, m)|^2$ as a 2D image (time on x-axis, frequency on y-axis, intensity as color/darkness).

- **Wideband spectrogram** (short window ≈ 5 ms, $T = 50$ samples at 10 kHz): good time resolution, smeared frequency resolution → shows formant tracks as broad horizontal bands, visible pitch pulses
- **Narrowband spectrogram** (long window ≈ 25 ms, $T = 250$ samples at 10 kHz): good frequency resolution → resolves individual harmonics (vertical striations), loses time resolution

#### 5.1. Voiced vowels

- Source $e(t)$: quasi-periodic, period $P$ (pitch period $P \approx$ 5–30 ms)
- Fundamental: $F_0 = 1/P$
- Spectrum $S(\Omega)$: harmonics at multiples of $F_0$, shaped by formant envelope
- Each vowel has a distinct formant configuration:
  - /i/ (eve): F1 low (~270 Hz), F2 high (~2290 Hz), F3 high (~3010 Hz); waveform shows low-frequency damped oscillation
  - /a/ (father): F1 high (~730 Hz), F2 moderate (~1090 Hz); waveform shows richer, more complex oscillation
  - /u/ (boot): F1 low (~300 Hz), F2 very low (~870 Hz)
- F1 and F2 alone are often sufficient to distinguish vowels

**Formant dynamics in articulation:**
- F1 increases as jaw opens, decreases as jaw closes
- F2 decreases as tongue moves from front to back of mouth

#### 5.2. Nasal consonants (M, N, NX)

- Periodic source (vocal folds vibrating)
- Air passes largely through the nasal cavity (velum open, oral closure)
- Low-energy, low resonance signal
- Spectrogram shows: very low energy bands, spectral "holes" (anti-resonances from the coupled nasal/oral cavities)
- Three articulation sites: bilabial /m/, alveolar /n/, velar /ng/ — identifiable by formant transitions of adjacent vowels
- /ng/ transition is less abrupt than /m/ and /n/

#### 5.3. Fricative consonants

**Unvoiced fricatives** (F, TH, S, SH):
- Source: turbulent airflow → random white noise
- Spectrum: no harmonics, no pitch structure; colored noise depending on obstruction location
- /s/: energy concentrated above 4 kHz
- /sh/: energy from about 2 kHz upward (lower cutoff than /s/)
- /f/: broadband but lower intensity than /s/ or /sh/

**Voiced fricatives** (V, DH, Z, ZH):
- Source: both turbulent noise AND periodic component (voice bar visible at low frequencies)
- Presence of harmonics in the spectrum distinguishes them from unvoiced counterparts
- The periodic component (voice bar) appears as a low-frequency band at the base of the spectrogram

#### 5.4. Plosive (stop) consonants

Structure of a plosive event (both voiced and unvoiced):

**Unvoiced plosive** (P, T, K):
- **Silence / Stop Gap**: complete oral closure builds pressure
- **Burst**: sudden release → sharp impulsive transient (wideband noise burst)
- **Aspiration**: turbulence from the constriction (like a brief fricative)
- **Voicing**: following vowel begins
- **VOT (Voice Onset Time)** = time between burst release and onset of voicing → longer for unvoiced plosives

**Voiced plosive** (B, D, G):
- **Voice bar**: low-frequency voicing during the closure period (vocal folds still vibrating)
- **Burst**: release of pressure + turbulence
- **Voicing**: vowel onset follows relatively quickly; VOT shorter than unvoiced

**Spectrograms of plosives:**
- Voiced /g-o/: wideband spectrogram shows voice bar during closure, then burst, then formant transitions
- Unvoiced /k-i/: clean silence gap, then burst with aspiration, then vowel formants
- /b-a/: spectrograms show low-frequency voice bar during /b/, then formant transitions as /a/ begins

#### 5.5. Diphthongs (AY, OY, AW, EY)

- **Voiced** — vocal folds vibrating throughout
- Characterized by continuously **moving formants**: F1 and F2 glide from one position to another as the vocal tract shape changes
- /OY/ (boy): F2 moves from ~mid to ~low over 400–500 ms
- /AY/ (buy): large F2 movement
- /AW/ (down): both F1 and F2 change
- /EY/ (bait): F2 drops significantly
- Narrowband spectrogram is clearest: formant tracks visible as smooth curved bands sweeping through frequency

#### 5.6. Semivowels / glides (W, L, R, Y)

- All voiced; characterized by **slow formant transitions** (slower than plosives, faster than steady vowels)
- Distinguishing formant properties:
  - /W/: F1 and F2 both very low; slow transition
  - /L/: F1 and F2 low; faster transition (similar to /W/ but quicker)
  - /Y/: F1 very low, F2 very high
  - /R/: F3 very low (distinctive; used in English rhotic sounds)
- /W/ and /L/ are very similar in spectrogram; distinguished mainly by transition rate and F3

---

### 6. Worked examples

#### Example 1: Pitch period from autocorrelation

Given speech sampled at $F_s = 8000$ Hz. The short-time autocorrelation of a voiced frame shows its first non-trivial peak at lag $k = 54$ samples.

- Pitch period in samples: $T_p = 54$
- Pitch period in seconds: $T_p = 54 / 8000 = 6.75$ ms
- Fundamental frequency: $F_0 = F_s / T_p = 8000 / 54 \approx 148$ Hz

This is within the normal male speaker range (80–200 Hz).

#### Example 2: Identifying voiced vs. unvoiced from ZCR

A frame of speech has ZCR = 60 crossings per 15 ms window and STE very low. This indicates **unvoiced or silence** (high ZCR from high-frequency noise or silence). A voiced frame would typically show ZCR ≈ 10–20 with high STE.

#### Example 3: Source–filter model computation

Given a voiced speech frame at $F_s = 10$ kHz, pitch period $T = 50$ samples, and a 10th-order vocal tract with poles at:
- $z_1 = 0.9 e^{j 2\pi \times 800/10000}$ → F1 ≈ 800 Hz
- $z_2 = 0.85 e^{j 2\pi \times 1800/10000}$ → F2 ≈ 1800 Hz
- $z_3 = 0.80 e^{j 2\pi \times 2600/10000}$ → F3 ≈ 2600 Hz
- remaining poles at higher frequencies

The complete spectrum $|P_L(e^{j\omega})|$ shows harmonic lines spaced at $F_0 = 200$ Hz, shaped by the three dominant formant peaks.

#### Example 4: Wideband vs. narrowband spectrogram trade-off

For speech at $F_s = 10$ kHz with pitch period in the range [5.5, 8] ms:
- Wideband window $T = 50/10000 = 5$ ms: resolves formant tracks but does NOT resolve individual harmonics (each pitch period is too short to fit in the window)
- Narrowband window $T = 250/10000 = 25$ ms: resolves individual harmonics (vertical striations) but smears formant transitions in time

---

### 7. Signal analysis system overview

From the signal $x[n]$, short-time analysis extracts an alternate representation $Q_{\hat{n}}$, which is then used for **parameter estimation** of the model:

```
x[n]  --[Short-Time Analysis]-->  Q_{n-hat}  --[Parameter Estimation]-->  Model parameters
```

Model parameters include:
- Pitch period $N_p$ (or $F_0$)
- Voiced / Unvoiced / Silence decision
- Gain $A_V$ or $A_N$
- Vocal tract polynomial coefficients $\{a_1, a_2, \ldots, a_p\}$
- Glottal pulse shape parameters
- Radiation model parameter $\alpha$

Applications downstream of parameter estimation: pitch shifting, time stretching, voice conversion (vocoder), speech coding, recognition.

---

## Key terms (glossary)

| Term | Definition |
|---|---|
| **Source–filter model** | Speech production as an LTI cascade: excitation source * vocal tract filter * radiation model |
| **Voiced sound** | Sound produced with vibrating vocal folds; periodic excitation |
| **Unvoiced sound** | Sound produced with vocal folds open; turbulent noise excitation |
| **Plosive (stop)** | Sound produced by complete oral closure followed by abrupt release; structured as silence–burst–aspiration–voicing |
| **Formant** | Resonance of the vocal tract; appears as a spectral peak; labelled F1, F2, F3 in ascending frequency order |
| **Fundamental frequency (F0)** | Lowest harmonic frequency of voiced speech; equals $1/T_p$; perceived as "pitch" |
| **Pitch period ($T_p$)** | Duration of one glottal cycle; $T_p = 1/F_0$ |
| **Glottal pulse model G(z)** | Filter that shapes impulse train into realistic airflow waveform; introduces -12 dB/octave spectral tilt |
| **Vocal tract model V(z)** | All-pole filter $1/(1 + \sum a_k z^{-k})$; resonances correspond to formants |
| **Radiation model R(z)** | First-order high-pass filter $R(z) = 1 - 0.99z^{-1}$; models lip radiation (+6 dB/octave boost) |
| **LPC (Linear Predictive Coding)** | Method to estimate all-pole vocal tract coefficients $a_k$; used in speech coding and recognition |
| **Phoneme** | Minimal unit of sound that distinguishes meaning; English has ~48 (ARPAbet) |
| **ARPAbet** | Machine-readable phonetic alphabet for American English (48 symbols) |
| **Short-Time Energy (STE)** | $E_{\hat{n}} = \sum x^2[m] h[\hat{n}-m]$; measures local signal power; used for voicing detection |
| **Short-Time Magnitude (STM)** | $M_{\hat{n}} = \sum |x[m]| \tilde{w}[\hat{n}-m]$; less sensitive to outliers than STE |
| **Zero-Crossing Rate (ZCR)** | Rate at which the speech waveform crosses zero; high for unvoiced, low for voiced |
| **Autocorrelation (STACF)** | $r_k = \sum s_i \cdot s_{i+k}$; emphasises periodicity; peak at lag $k = T_p$ used for pitch detection |
| **AMDF** | Average Magnitude Difference Function; minimum at lag = pitch period; computationally lighter than ACF |
| **Spectrogram** | Time-frequency representation $|\text{STFT}(\omega, m)|^2$; wideband shows formant tracks, narrowband shows harmonics |
| **VOT (Voice Onset Time)** | Time between plosive burst and onset of voicing; longer for unvoiced plosives |
| **Voice bar** | Low-frequency voicing visible in spectrogram during voiced plosive closure |
| **Diphthong** | Vowel with moving articulation; formants sweep continuously during production |
| **Semivowel / glide** | Voiced sound (W, L, R, Y) with slow formant transitions; between vowel and consonant |

---

## Exam targets

1. **Draw and label the complete source–filter block diagram** — impulse train generator, glottal pulse model $G(z)$, voiced/unvoiced switch with gains $A_V$ and $A_N$, random noise generator, vocal tract $V(z)$, radiation model $R(z)$. Label all Z-domain transfer functions.

2. **Write the four component Z-domain equations** and the overall equation:
   - Impulse train: $s[n] = \sum_k \delta[n - kT]$
   - Glottal: $U_G(z) = G(z) \cdot S(z)$
   - Vocal tract: $V(z) = 1 / (1 + \sum_{k=1}^{p} a_k z^{-k})$
   - Radiation: $R(z) = 1 - 0.99z^{-1}$
   - Overall: $P_L(z) = R(z) \cdot V(z) \cdot G(z) \cdot S(z)$

3. **Explain formants as poles** of $V(z)$: $z_k = r_k e^{j\omega_k}$; what do $r_k$ and $\omega_k$ control; why poles near the unit circle give sharp peaks.

4. **Derive the Short-Time Energy formula** from the generic short-time processing equation. Explain why it is sensitive to large amplitudes and what the window $h[n] = \tilde{w}^2[n]$ does.

5. **Write the ZCR formula** and derive that for a sinusoid at frequency $F_0$ sampled at $F_s$: $\text{ZCR} = 2F_0/F_s$ crossings/sample. State what ZCR tells you about voiced vs. unvoiced speech.

6. **Define the autocorrelation** $r_k = \sum s_i \cdot s_{i+k}$. State: (a) $r_0$ = energy; (b) why periodicity causes peaks at multiples of $T_p$; (c) how to estimate $F_0$ from the ACF peak location.

7. **Define the AMDF** $\gamma_{\hat{n}}[k]$. Compare to ACF: same structural purpose (pitch detection), but uses absolute values and subtraction instead of multiplication → computationally lighter; minimum at $k = T_p$.

8. **Identify phoneme classes from spectrograms**: given a spectrogram region, determine voiced/unvoiced, identify fricatives (broadband noise), plosives (silence gap + burst), nasals (low energy + spectral holes), diphthongs (moving formants), semivowels (slow formant transitions).

9. **State the formant–articulation relationships**: F1 depends on jaw height; F2 depends on tongue front–back position; F3 very low → /R/.

10. **Explain the role of LPC** in estimating the all-pole model: what the $a_k$ coefficients represent physically, and how they are used in speech coding and recognition.

11. **Explain the wideband vs. narrowband spectrogram trade-off**: short window → good time resolution, formant tracks; long window → good frequency resolution, individual harmonics visible.

---

## Pitfalls

- **The radiation model $R(z) = 1 - 0.99z^{-1}$ is a high-pass (differentiator), not a low-pass.** It boosts high frequencies. Confusing its effect with the glottal model (which rolls off) is a common error.

- **F0 is the fundamental frequency of the excitation, not of the vocal tract.** Formants F1, F2, F3 are resonances of the vocal tract. Do not confuse $f_0$ (pitch) with F1 (first formant).

- **$r_0$ in the autocorrelation equals the signal ENERGY, not 1.** Normalised ACF has $r_0 = 1$; unnormalised does not. The first peak beyond lag 0 gives the pitch period.

- **Short-Time Energy is sensitive to outlier samples** because of squaring; Short-Time Magnitude is less sensitive. Both are useful; STM has smaller dynamic range between voiced and unvoiced.

- **The voiced/unvoiced decision is NOT made on the full signal** — it is made frame-by-frame because the same speaker can have voiced and unvoiced regions within a single word.

- **High ZCR alone does not mean unvoiced.** Silence also has high ZCR (noise is broadband). Combine ZCR with STE: silence has both high ZCR and very low STE; unvoiced speech has moderate-to-high energy AND high ZCR.

- **Window length controls the trade-off for ALL short-time measures.** Shorter window → better time resolution but more variable estimates. Longer window → smoother estimates but slower to track signal changes. This is universal to STE, ZCR, STACF, and the spectrogram.

- **Nasal consonants have spectral holes (anti-resonances)**, not just resonances. The coupling of the nasal cavity introduces zeros in the transfer function. A purely all-pole model cannot capture this exactly.

- **The all-pole model $V(z)$ is exact only for non-nasal voiced sounds.** Nasals and fricatives require a more general pole-zero (ARMA) model, though all-pole LPC is used in practice as an approximation.

- **Formant values vary across speakers** because vocal tract length differs physiologically. Published tables are population averages.

- **Plosive VOT is longer for unvoiced stops than voiced stops.** Do not confuse the voice bar (low-frequency voicing during closure) with the burst (wideband transient at release).

- **Diphthongs are voiced throughout** — the formant movement is due to articulatory change, not a switch to a different excitation type.
