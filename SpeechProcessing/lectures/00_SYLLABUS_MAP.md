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
- **Signal taxonomy**: continuous-time $x(t)$ → discrete-time $x(nT)$ → digital/quantized $x_Q(nT)$ (discrete in time *and* amplitude).
- **ADC pipeline**: analog → sampler → quantizer → digital processor (worked 4-bit two's-complement quantization example).
- **System** = input→output mapping; a digital-signal system = a digital filter/processor.
- **DSP vs analog**: 6 advantages — flexibility, ease of development, reliability, cost, security, simplicity.
- **Application domains**: speech (LPC, synthesis, recognition, enhancement), audio (MP3), image/video (JPEG/MPEG), comms, biomedical, finance.
- *Conceptual chapter — no heavy math; rigor starts Ch 2.*

## Ch 2 — Review of Analog Signal Analysis (28 pp)
- **Fourier Series** (continuous-time *periodic*): $x(t) = \sum_k a_k e^{jk\Omega_0 t}$, $a_k = \frac{1}{T_p}\int x(t)e^{-jk\Omega_0 t}dt$. Time continuous+periodic ↔ freq discrete+aperiodic.
- **Fourier Transform** (continuous-time *aperiodic*): $X(j\Omega)=\int x(t)e^{-j\Omega t}dt$, inverse $x(t)=\frac{1}{2\pi}\int X(j\Omega)e^{j\Omega t}d\Omega$. Derived as FS limit $T\to\infty$.
- **Key pairs**: rect ↔ sinc; $e^{-at}u(t) \leftrightarrow \frac{1}{a+j\Omega}$; $\delta(t) \leftrightarrow 1$; $e^{j\Omega_0 t} \leftrightarrow 2\pi\delta(\Omega-\Omega_0)$; impulse train ↔ impulse train.
- **Dirac delta** + sifting property $\int f(t)\delta(t-t_0)dt = f(t_0)$.
- **Analog LTI**: convolution $y(t)=x(t)\asth(t)$ ↔ multiplication $Y=X\cdot H$.

## Ch 3 — Discrete-Time Signals and Systems (45 pp)
- **DT signals**: $x[n]=x(nT)$, unit impulse $\delta[n]$, unit step $u[n]$, sifting/decomposition.
- **System properties**: memoryless, linear (superposition), time-invariant (shift), causal, BIBO-stable — with counterexamples.
- **LTI ⇔ impulse response $h[n]$**: causal iff $h[n]=0,\; n<0$; stable iff $\sum|h[n]| < \infty$.
- **Convolution sum**: $y[n]=\sum_m x[m]h[n-m]$; commutative/associative/distributive; finite-length output = M+N−1.
- **LCCDE** (difference equations): N-th order recursive form; MATLAB `filter(b,a,x)`.
- **Energy vs power** signals: $E=\sum|x[n]|^2$, $P=\lim \frac{1}{2N+1}\sum|x[n]|^2$.

## Ch 4 — DTFT & DFT (42 pp)
- **DTFT**: $X(e^{j\omega})=\sum x[n]e^{-j\omega n}$ (discrete/aperiodic signal → continuous, $2\pi$-periodic spectrum). Converges iff absolutely summable (= BIBO).
- **9 DTFT properties**: linearity, time/freq shift, differentiation, conjugation, reversal, convolution↔multiplication, Parseval.
- **DFT**: $X[k]=X(e^{j\omega})|_{\omega=2\pi k/N}$, N-point pair, twiddle $W_N=e^{-j2\pi/N}$; **circular** convolution (vs linear for DTFT).
- **Frequency estimation**: peak at $k^{\ast}$ → $\hat{\omega}_0=2\pi k^{\ast}/N$; **zero-padding** densifies the grid (no new info).
- **FFT**: Cooley-Tukey reduces $N^2$ → $(N/2)\log_2 N$; a fast DFT, not a new transform.

## Ch 5 — z-Transform (63 pp)
- **Definition**: $X(z)=\sum x[n]z^{-n}$; on unit circle $|z|=1$ it reduces to the DTFT.
- **ROC** (8 properties): exterior disk (right-sided), interior disk (left-sided), annulus (two-sided), whole plane (finite). DTFT exists iff ROC ⊇ unit circle.
- **Poles & zeros**: rational $X(z)$ factored form; poles excluded from ROC; pole-zero plot encodes causality + stability.
- **Common pairs** (same algebra, different ROC → different signal): $\delta$, $a^n u[n]$, $-a^n u[-n-1]$, cos/sin.
- **Inverse z** (4 methods): inspection, partial fractions, power series/long division, contour integral.
- **System analysis**: $H(z)=Y(z)/X(z)$; causal ⇔ right-sided ROC; **causal+stable ⇔ all poles inside unit circle**.

## Ch 6 — Sampling and Reconstruction (28 pp)
- **Sampling**: $x[n]=x(nT)$; CD-converter model (impulse-train multiply + sequence extraction).
- **Sampled spectrum**: $X_s(j\Omega)=\frac{1}{T}\sum_k X(j(\Omega-k\Omega_s))$ — spectrum replicates every $\Omega_s$, scaled $1/T$.
- **Nyquist–Shannon**: bandlimited ($X=0$ for $|\Omega|\geq\Omega_b$) recoverable iff $\Omega_s > 2\Omega_b$. Nyquist rate = $2\Omega_b$.
- **Aliasing**: spectral copies overlap when $\Omega_s<2\Omega_b$; irreversible without prior bandlimiting.
- **Ideal reconstruction**: lowpass (gain $T$, cutoff $\pi/T$) → **sinc interpolation** $x_r(t)=\sum_k x[k]\,\mathrm{sinc}((t-kT)/T)$.
- **Practical chain**: anti-aliasing LPF → ADC (quantization) → DSP → DAC. Ideal converters unrealizable (infinite/non-causal sinc).

## Ch 7 — STFT & Spectrogram (62 pp)
- **Motivation**: DFT/DTFT give "what but not when" — useless for non-stationary speech.
- **STFT**: $S[m,k]=\sum x[n]w[n-mH]e^{-j2\pi kn/N}$ — a (freq bins × frames) complex matrix.
- **Windows**: Rectangular (narrow main lobe, −13 dB sidelobes), Bartlett, Hann, Hamming, Blackman — main-lobe-width vs sidelobe trade-off.
- **Uncertainty principle**: $\Delta t \cdot \Delta f \geq \frac{1}{4\pi}$ — can't resolve time *and* frequency perfectly.
- **Spectrogram**: $|S[m,k]|^2$ in dB. Long window → **narrowband** (resolves harmonics + F0); short window → **wideband** (resolves formants + pitch periods).
- **Speech**: narrowband ≈ 60 ms, wideband ≈ 6 ms; phoneme reading (vowels/fricatives/stops).

## Ch 8 — Speech Signal Analysis (65 + 13 pp; merged with source-filter supplement)
- **Source-filter model** (Z-domain cascade): $P_L(z) = R(z)\cdot V(z)\cdot G(z)\cdot S(z)$ — impulse-train/noise source + glottal pulse $G(z)$ + voiced/unvoiced switch (gains $A_V$/$A_N$).
- **Vocal tract = all-pole filter** $V(z)=\frac{1}{1+\sum a_k z^{-k}}$; poles $z_k=r_k e^{j\omega_k}$ ↔ **formants**.
- **Radiation** $R(z)=1-0.99z^{-1}$ (high-pass/differentiator), spectral-tilt correction.
- **Time-domain measures**: Short-Time Energy, Magnitude, **Zero-Crossing Rate**, Short-Time **Autocorrelation** (pitch via peak at $T_p$), AMDF.
- **Formants & phonemes**: F1/F2/F3 table for vowels; spectrogram signatures (vowels, nasals, fricatives, plosives w/ VOT, diphthongs, semivowels).

## Ch 9 — Cepstral Analysis & MFCC (28 pp)
- **Cepstrum**: $c[n]=\mathcal{F}^{-1}\{\log|\mathcal{F}\{x[n]\}|\}$ — "spectrum of a spectrum," lives in **quefrency** (seconds).
- **Homomorphic deconvolution**: $s[n]=e[n]\asth[n]$ → $c_s[n]=c_e[n]+c_h[n]$ → **liftering** separates excitation (high quefrency) from vocal tract (low quefrency). Pitch: $F_0 = F_s/q_0$.
- **Mel scale**: $\text{Mel}(f)=2595\cdot\log_{10}(1+f/700)$ — linear <1 kHz, log above.
- **MFCC pipeline**: pre-emphasis ($1-0.97z^{-1}$) → frame (25 ms/10 ms) → Hamming → FFT → power → **Mel filterbank** → log → **DCT** → 12 coeffs.
- **Feature vector**: 39-dim = 12 static + energy + delta (×13) + delta-delta (×13).

## Ch 10 — Linear Predictive Coding (LPC) (27 pp)
- **All-pole model**: $H(z)=G/A(z)$; residual $e[n]$ ≈ excitation (impulse-like voiced / white-noise unvoiced).
- **Linear prediction**: predict $s[n]$ from $p$ past samples; minimize squared error → **normal (Yule-Walker) equations** $\mathbf{R}\mathbf{a}=\mathbf{r}$ (Toeplitz).
- **Levinson-Durbin**: $O(p^2)$ solver, order-by-order; yields **reflection coefficients** $k_i$ with stability check $|k_i|<1$.
- **PARCOR / lattice**: reflection coeffs model acoustic-tube junction reflections.
- **LPC spectrum**: $|H(e^{j\omega})|$ = smooth spectral envelope; pole angle → formant freq, pole radius → bandwidth. Order rule $p \approx F_{s,\text{kHz}} + 2\ldots4$.
- **Applications**: speech coding (CELP/vocoders), formant tracking, pitch, speaker ID. **Limit**: all-pole fails for nasals/fricatives (zeros).

---

## Cross-cutting themes (likely exam targets)
- **Four-domain Fourier map**: which transform for which signal type (periodicity × continuity) — see table above.
- **Convolution ↔ multiplication** duality, in every domain (analog FT, DTFT, z, DFT-circular).
- **Stability & causality**: $\sum|h[n]|<\infty$; ROC ⊇ unit circle; poles inside unit circle.
- **Time–frequency trade-off**: uncertainty principle; window length choice (narrowband vs wideband).
- **Source-filter model** as the unifying speech framework (Ch 8/9/10): excitation × all-pole vocal tract; formants = poles.
- **Feature extraction** comparison: spectrogram vs cepstrum/MFCC vs LPC — what each captures and its limitations.

---

## Study plan from here
1. ✅ **Phase 0 — Syllabus map** (this file)
2. **Phase 1 — Per chapter**: read each `0X_*.md` (bird's eye for a quick pass; detailed notes for depth)
3. **Phase 2 — Practice**: work the formulas/derivations by hand (this is a written, math-heavy exam)
4. **Phase 3 — Cross-links**: revise the recurring themes above (Fourier map, stability, source-filter)
