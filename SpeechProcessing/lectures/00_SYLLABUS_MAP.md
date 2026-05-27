# Speech Processing — Syllabus Skeleton (Phase 0)

> Course: Speech Processing / Digital Signal Processing — ENSIA (Dr Nafaa NACEREDDINE)
> Use this as a **mental scaffold** + quick-scan map. Each chapter has its own detailed notes file (`0X_chapterX_topic.md`).

## The arc of the course

This course builds a pipeline from **general DSP foundations → speech-specific analysis**:

- **Ch 1-6 = DSP foundations**: signals & systems, the three Fourier tools (FS, FT, DTFT, DFT), the z-transform, and the sampling bridge between analog and digital.
- **Ch 7-10 = speech processing**: time-frequency analysis (STFT), the source-filter speech model, and the two dominant feature/modeling techniques (MFCC, LPC).
- **Recurring thread**: the **source-filter model** (excitation × vocal-tract filter) underlies Ch 8, 9, and 10. The **all-pole filter** + **formants** idea ties LPC, cepstrum, and spectrograms together.

| Domain pair | Tool | Chapter |
|---|---|---|
| Continuous + periodic | Fourier Series | 2 |
| Continuous + aperiodic | Fourier Transform | 2 |
| Discrete + aperiodic | DTFT | 4 |
| Discrete + finite/periodic | DFT (via FFT) | 4 |
| Discrete + complex plane | z-Transform | 5 |
| Analog ↔ Discrete | Sampling/Reconstruction | 6 |
| Time-varying (speech) | STFT / Spectrogram | 7 |

---

## Ch 1 — Overview of Digital Signal Processing (15 pp)
- **Signal taxonomy**: continuous-time `x(t)` → discrete-time `x(nT)` → digital/quantized `x_Q(nT)` (discrete in time *and* amplitude).
- **ADC pipeline**: analog → sampler → quantizer → digital processor (worked 4-bit two's-complement quantization example).
- **System** = input→output mapping; a digital-signal system = a digital filter/processor.
- **DSP vs analog**: 6 advantages — flexibility, ease of development, reliability, cost, security, simplicity.
- **Application domains**: speech (LPC, synthesis, recognition, enhancement), audio (MP3), image/video (JPEG/MPEG), comms, biomedical, finance.
- *Conceptual chapter — no heavy math; rigor starts Ch 2.*

## Ch 2 — Review of Analog Signal Analysis (28 pp)
- **Fourier Series** (continuous-time *periodic*): `x(t) = Σ a_k e^{jkΩ₀t}`, `a_k = (1/T_p)∫ x(t)e^{-jkΩ₀t}dt`. Time continuous+periodic ↔ freq discrete+aperiodic.
- **Fourier Transform** (continuous-time *aperiodic*): `X(jΩ)=∫x(t)e^{-jΩt}dt`, inverse `x(t)=(1/2π)∫X(jΩ)e^{jΩt}dΩ`. Derived as FS limit T→∞.
- **Key pairs**: rect ↔ sinc; `e^{-at}u(t) ↔ 1/(a+jΩ)`; `δ(t) ↔ 1`; `e^{jΩ₀t} ↔ 2πδ(Ω−Ω₀)`; impulse train ↔ impulse train.
- **Dirac delta** + sifting property `∫f(t)δ(t−t₀)dt = f(t₀)`.
- **Analog LTI**: convolution `y(t)=x(t)*h(t)` ↔ multiplication `Y=X·H`.

## Ch 3 — Discrete-Time Signals and Systems (45 pp)
- **DT signals**: `x[n]=x(nT)`, unit impulse `δ[n]`, unit step `u[n]`, sifting/decomposition.
- **System properties**: memoryless, linear (superposition), time-invariant (shift), causal, BIBO-stable — with counterexamples.
- **LTI ⇔ impulse response h[n]**: causal iff `h[n]=0, n<0`; stable iff `Σ|h[n]| < ∞`.
- **Convolution sum**: `y[n]=Σ_m x[m]h[n−m]`; commutative/associative/distributive; finite-length output = M+N−1.
- **LCCDE** (difference equations): N-th order recursive form; MATLAB `filter(b,a,x)`.
- **Energy vs power** signals: `E=Σ|x[n]|²`, `P=lim (1/(2N+1))Σ|x[n]|²`.

## Ch 4 — DTFT & DFT (42 pp)
- **DTFT**: `X(e^{jω})=Σ x[n]e^{-jωn}` (discrete/aperiodic signal → continuous, 2π-periodic spectrum). Converges iff absolutely summable (= BIBO).
- **9 DTFT properties**: linearity, time/freq shift, differentiation, conjugation, reversal, convolution↔multiplication, Parseval.
- **DFT**: `X[k]=X(e^{jω})|_{ω=2πk/N}`, N-point pair, twiddle `W_N=e^{-j2π/N}`; **circular** convolution (vs linear for DTFT).
- **Frequency estimation**: peak at `k*` → `ω̂₀=2πk*/N`; **zero-padding** densifies the grid (no new info).
- **FFT**: Cooley-Tukey reduces `N²` → `(N/2)log₂N`; a fast DFT, not a new transform.

## Ch 5 — z-Transform (63 pp)
- **Definition**: `X(z)=Σ x[n]z^{-n}`; on unit circle `|z|=1` it reduces to the DTFT.
- **ROC** (8 properties): exterior disk (right-sided), interior disk (left-sided), annulus (two-sided), whole plane (finite). DTFT exists iff ROC ⊇ unit circle.
- **Poles & zeros**: rational `X(z)` factored form; poles excluded from ROC; pole-zero plot encodes causality + stability.
- **Common pairs** (same algebra, different ROC → different signal): `δ`, `a^n u[n]`, `−a^n u[−n−1]`, cos/sin.
- **Inverse z** (4 methods): inspection, partial fractions, power series/long division, contour integral.
- **System analysis**: `H(z)=Y(z)/X(z)`; causal ⇔ right-sided ROC; **causal+stable ⇔ all poles inside unit circle**.

## Ch 6 — Sampling and Reconstruction (28 pp)
- **Sampling**: `x[n]=x(nT)`; CD-converter model (impulse-train multiply + sequence extraction).
- **Sampled spectrum**: `X_s(jΩ)=(1/T)Σ_k X(j(Ω−kΩ_s))` — spectrum replicates every `Ω_s`, scaled `1/T`.
- **Nyquist–Shannon**: bandlimited (`X=0` for `|Ω|≥Ω_b`) recoverable iff **`Ω_s > 2Ω_b`**. Nyquist rate = `2Ω_b`.
- **Aliasing**: spectral copies overlap when `Ω_s<2Ω_b`; irreversible without prior bandlimiting.
- **Ideal reconstruction**: lowpass (gain T, cutoff `π/T`) → **sinc interpolation** `x_r(t)=Σ_k x[k] sinc((t−kT)/T)`.
- **Practical chain**: anti-aliasing LPF → ADC (quantization) → DSP → DAC. Ideal converters unrealizable (infinite/non-causal sinc).

## Ch 7 — STFT & Spectrogram (62 pp)
- **Motivation**: DFT/DTFT give "what but not when" — useless for non-stationary speech.
- **STFT**: `S[m,k]=Σ x[n]w[n−mH]e^{−j2πkn/N}` — a (freq bins × frames) complex matrix.
- **Windows**: Rectangular (narrow main lobe, −13 dB sidelobes), Bartlett, Hann, Hamming, Blackman — main-lobe-width vs sidelobe trade-off.
- **Uncertainty principle**: `Δt·Δf ≥ 1/(4π)` — can't resolve time *and* frequency perfectly.
- **Spectrogram**: `|S[m,k]|²` in dB. Long window → **narrowband** (resolves harmonics + F0); short window → **wideband** (resolves formants + pitch periods).
- **Speech**: narrowband ≈ 60 ms, wideband ≈ 6 ms; phoneme reading (vowels/fricatives/stops).

## Ch 8 — Speech Signal Analysis (65 + 13 pp; merged with source-filter supplement)
- **Source-filter model** (Z-domain cascade): `P_L(z) = R(z)·V(z)·G(z)·S(z)` — impulse-train/noise source + glottal pulse `G(z)` + voiced/unvoiced switch (gains `A_V`/`A_N`).
- **Vocal tract = all-pole filter** `V(z)=1/(1+Σ a_k z^{-k})`; poles `z_k=r_k e^{jω_k}` ↔ **formants**.
- **Radiation** `R(z)=1−0.99z^{-1}` (high-pass/differentiator), spectral-tilt correction.
- **Time-domain measures**: Short-Time Energy, Magnitude, **Zero-Crossing Rate**, Short-Time **Autocorrelation** (pitch via peak at `T_p`), AMDF.
- **Formants & phonemes**: F1/F2/F3 table for vowels; spectrogram signatures (vowels, nasals, fricatives, plosives w/ VOT, diphthongs, semivowels).

## Ch 9 — Cepstral Analysis & MFCC (28 pp)
- **Cepstrum**: `c[n]=F^{-1}{log|F{x[n]}|}` — "spectrum of a spectrum," lives in **quefrency** (seconds).
- **Homomorphic deconvolution**: `s[n]=e[n]*h[n]` → `c_s[n]=c_e[n]+c_h[n]` → **liftering** separates excitation (high quefrency) from vocal tract (low quefrency). Pitch: `F0 = Fs/q0`.
- **Mel scale**: `Mel(f)=2595·log10(1+f/700)` — linear <1 kHz, log above.
- **MFCC pipeline**: pre-emphasis (`1−0.97z^{-1}`) → frame (25 ms/10 ms) → Hamming → FFT → power → **Mel filterbank** → log → **DCT** → 12 coeffs.
- **Feature vector**: 39-dim = 12 static + energy + delta (×13) + delta-delta (×13).

## Ch 10 — Linear Predictive Coding (LPC) (27 pp)
- **All-pole model**: `H(z)=G/A(z)`; residual `e[n]` ≈ excitation (impulse-like voiced / white-noise unvoiced).
- **Linear prediction**: predict `s[n]` from `p` past samples; minimize squared error → **normal (Yule-Walker) equations** `Ra=r` (Toeplitz).
- **Levinson-Durbin**: O(p²) solver, order-by-order; yields **reflection coefficients** `k_i` with stability check `|k_i|<1`.
- **PARCOR / lattice**: reflection coeffs model acoustic-tube junction reflections.
- **LPC spectrum**: `|H(e^{jω})|` = smooth spectral envelope; pole angle → formant freq, pole radius → bandwidth. Order rule `p ≈ Fs_kHz + 2..4`.
- **Applications**: speech coding (CELP/vocoders), formant tracking, pitch, speaker ID. **Limit**: all-pole fails for nasals/fricatives (zeros).

---

## Cross-cutting themes (likely exam targets)
- **Four-domain Fourier map**: which transform for which signal type (periodicity × continuity) — see table above.
- **Convolution ↔ multiplication** duality, in every domain (analog FT, DTFT, z, DFT-circular).
- **Stability & causality**: `Σ|h[n]|<∞`; ROC ⊇ unit circle; poles inside unit circle.
- **Time–frequency trade-off**: uncertainty principle; window length choice (narrowband vs wideband).
- **Source-filter model** as the unifying speech framework (Ch 8/9/10): excitation × all-pole vocal tract; formants = poles.
- **Feature extraction** comparison: spectrogram vs cepstrum/MFCC vs LPC — what each captures and its limitations.

---

## Study plan from here
1. ✅ **Phase 0 — Syllabus map** (this file)
2. **Phase 1 — Per chapter**: read each `0X_*.md` (bird's eye for a quick pass; detailed notes for depth)
3. **Phase 2 — Practice**: work the formulas/derivations by hand (this is a written, math-heavy exam)
4. **Phase 3 — Cross-links**: revise the recurring themes above (Fourier map, stability, source-filter)
