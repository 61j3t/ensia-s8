# Chapter 4 — Discrete-Time Fourier Transform (DTFT) and Discrete Fourier Transform (DFT)

---

## Bird's eye view

- **DTFT** maps an aperiodic, infinite-duration discrete-time sequence x[n] to a **continuous and 2π-periodic** spectrum X(e^{jω}); x[n] is recovered by the inverse DTFT integral over one period [-π, π].
- **DFT** is a sampled version of the DTFT: sampling X(e^{jω}) at N uniformly-spaced frequencies ω_k = 2πk/N yields N complex coefficients X[k] from an N-point finite sequence — both x[n] and X[k] live in {0, …, N-1}.
- **Key duality** of signal representations: discrete → periodic (time-domain discrete ↔ frequency-domain periodic), and aperiodic → continuous (time-domain aperiodic ↔ frequency-domain continuous).
- Nine **DTFT properties** (linearity, time shift, frequency shift, differentiation, conjugation, time reversal, convolution, multiplication, Parseval) each have parallel **DFT counterparts** — but with circular convolution replacing linear convolution in the DFT.
- The **FFT** (Cooley-Tukey, 1965) reduces DFT complexity from O(N²) multiplications to O(N/2 · log₂ N), making spectral analysis computationally practical for large N.
- **Frequency estimation** with DFT: locate the magnitude peak at index k* → ω̂₀ = 2πk*/N; accuracy improves dramatically by **zero-padding** (appending zeros to push N large before computing `fft`).

---

## Detailed notes

### 1. Part 1: Discrete-Time Fourier Transform (DTFT)

#### 1.1 Definition and derivation

DTFT is a frequency analysis tool for **aperiodic discrete-time signals**.

**Analysis equation (DTFT):**

```
X(e^{jω}) = Σ_{n=-∞}^{∞}  x[n] e^{-jωn}          (4.1)
```

**Derivation sketch:** Construct the continuous-time sampled signal with sampling interval T:

```
x_s(t) = Σ_{n=-∞}^{∞}  x[n] δ(t − nT)             (4.2)
```

Taking the Fourier transform and using the sifting property of δ(t):

```
X_s(jΩ) = ∫_{-∞}^{∞} x_s(t) e^{-jΩt} dt = Σ_{n=-∞}^{∞}  x[n] e^{-jΩnT}   (4.3)
```

Define **ω = ΩT** as the discrete-time frequency parameter. Writing X_s(jΩ) as X(e^{jω}) recovers eq. (4.1).

**Synthesis equation (Inverse DTFT):**

```
x[n] = (1/2π) ∫_{-π}^{π}  X(e^{jω}) e^{jωn} dω    (4.4)
```

**Proof of inverse:** Substituting (4.1) into (4.4):

```
(1/2π) ∫_{-π}^{π} [Σ_m x[m] e^{-jωm}] e^{jωn} dω
= (1/2π) Σ_m x[m] ∫_{-π}^{π} e^{jω(n-m)} dω
= (1/2π) Σ_m x[m] · [2 sin((n-m)π) / (n-m)]
= x[n]                                              (4.5)
```

(The integral evaluates to 2π when n = m and 0 otherwise — the discrete orthogonality condition.)

#### 1.2 Properties of X(e^{jω})

- X(e^{jω}) is **continuous in ω** and **2π-periodic**: X(e^{j(ω+2π)}) = X(e^{jω}).
- X(e^{jω}) is generally **complex-valued**. Represent using:

```
|X(e^{jω})| = sqrt( [Re{X(e^{jω})}]² + [Im{X(e^{jω})}]² )     (4.6)

∠X(e^{jω}) = arctan( Im{X(e^{jω})} / Re{X(e^{jω})} )           (4.7)
```

Both the magnitude spectrum and phase spectrum are continuous and 2π-periodic.

**Fig. 4.1 (described):** Left panel shows a discrete aperiodic time-domain sequence x[n] with stem plot. Right panel shows the resulting spectrum X(e^{jω}) which is continuous and repeats with period 2π, plotted from -3π to 2π. The DTFT arrow goes left→right (analysis) and the iDTFT arrow goes right→left (synthesis). Bottom labels: "discrete and aperiodic" ↔ "continuous and periodic."

#### 1.3 Convergence of DTFT

The DTFT converges if:

```
|X(e^{jω})| ≤ Σ_{n=-∞}^{∞} |x[n]| · |e^{-jωn}| = Σ_{n=-∞}^{∞} |x[n]| < ∞   (4.8)
```

This is the **absolute summability** condition.

For an LTI system with impulse response h[n], the following are equivalent:
- **S1.** The system is **stable**: Σ_{n=-∞}^{∞} |h[n]| < ∞
- **S2.** The DTFT of h[n], i.e. H(e^{jω}), **converges**

H(e^{jω}) is also called the **system frequency response**.

#### 1.4 Worked Examples

**Example 4.1 — DTFT of unit step u[n]:**

```
X(e^{jω}) = Σ_{n=0}^{∞} e^{-jωn}
Since Σ_{n=0}^{∞} |e^{-jωn}| = Σ_{n=0}^{∞} 1 = ∞,  X(e^{jω}) does NOT exist.
```

Equivalently: Σ |u[n]| = ∞ → stability condition fails → DTFT does not converge.

---

**Example 4.2 — DTFT of rectangular pulse x[n] = u[n] - u[n-N] (N=10):**

The sequence has x[n] = 1 for n = 0, 1, …, N-1 and 0 otherwise.

```
X(e^{jω}) = Σ_{n=0}^{N-1} e^{-jωn} = (1 - e^{-jωN}) / (1 - e^{-jω})
```

To obtain closed-form magnitude and phase, factor out e^{-jωN/2} and e^{-jω/2}:

```
X(e^{jω}) = e^{-jω(N-1)/2} · sin(ωN/2) / sin(ω/2)
```

Therefore:

```
|X(e^{jω})| = |sin(ωN/2) / sin(ω/2)|

∠X(e^{jω}) = -ω(N-1)/2 + ∠[sin(ωN/2)/sin(ω/2)]
```

Using the sinc function sinc(u) = sin(πu)/(πu), this can be rewritten as:

```
sin(ωN/2) / sin(ω/2) = N · sinc(ωN/(2π)) / sinc(ω/(2π))
```

**Fig. 4.2 (described):** Upper plot shows the magnitude response |X(e^{jω})| vs ω/π for N=10: peak value of 10 at ω=0, then decaying oscillations (sinc-like envelope) going to zero at multiples of 2π/N, with minor lobes between nulls. Lower plot shows the phase response ∠X(e^{jω}): a linear (sawtooth-wrapped) function with slope -ω(N-1)/2, showing the expected linear phase characteristic.

---

**Example 4.3 — Inverse DTFT of ideal rectangular spectrum:**

Given X(e^{jω}) = 1 for -w₀ < ω < w₀ and 0 otherwise (ideal lowpass, 0 < w₀ < π):

```
x[n] = (1/2π) ∫_{-w₀}^{w₀} e^{jωn} dω = sin(w₀n) / (πn) = (w₀/π) sinc(w₀n/π)
```

This is an **infinite-duration sinc sequence** — the ideal lowpass filter impulse response is non-causal and infinite.

---

**Example 4.4 — Inverse DTFT using time-shifting property:**

Given X(e^{jω}) = e^{-j2ω} / (1 + 0.7e^{-jω}).

Recognising that 1/(1+0.7e^{-jω}) is the DTFT of (-0.7)^n u[n] and e^{-j2ω} corresponds to a 2-sample delay:

```
x[n] = (-0.7)^{n-2} u[n-2]
```

---

#### 1.5 Frequency Analysis of LTI Systems

**Frequency response definition:**

```
H(e^{jω}) = Σ_{n=-∞}^{∞} h[n] e^{-jωn},    -π < ω ≤ π
```

**Eigenfunction property:** If x[n] = e^{jω₀n} (complex exponential), then:

```
y[n] = H(e^{jω₀}) e^{jω₀n}
```

The complex exponential e^{jω₀n} is an eigenfunction of any LTI system; H(e^{jω₀}) is the corresponding eigenvalue.

**Sinusoidal input:** If x[n] = A cos(ω₀n + φ), then by Euler's formula and linearity:

```
y[n] = A |H(e^{jω₀})| cos(ω₀n + φ + arg H(e^{jω₀}))
```

- **Magnitude response** |H(e^{jω})|: gain applied to frequency ω.
- **Phase response** arg H(e^{jω}): phase shift applied to frequency ω.
- The frequency response **completely characterizes** the LTI system in the frequency domain.

---

### 2. Properties of DTFT

DTFT properties follow from the z-transform (evaluated on the unit circle). ROC is not separately stated because the DTFT exists only when the ROC includes the unit circle.

| # | Property | Time Domain | Frequency Domain | Eq. |
|---|---|---|---|---|
| 1 | Linearity | ax₁[n] + bx₂[n] | aX₁(e^{jω}) + bX₂(e^{jω}) | (4.8) |
| 2 | Time Shifting | x[n − n₀] | e^{-jωn₀} X(e^{jω}) | (4.9) |
| 3 | Freq. Shifting (Modulation) | e^{jω₀n} x[n] | X(e^{j(ω-ω₀)}) | (4.10) |
| 4 | Differentiation | nx[n] | j dX(e^{jω})/dω | (4.11) |
| 5 | Conjugation | x*[n] | X*(e^{-jω}) | (4.13) |
| 6 | Time Reversal | x[-n] | X(e^{-jω}) | (4.14) |
| 7 | Convolution | x₁[n] ⊗ x₂[n] | X₁(e^{jω}) X₂(e^{jω}) | (4.15) |
| 8 | Multiplication | x₁[n] · x₂[n] | (1/2π) ∫ X₁(e^{jτ}) X₂(e^{j(ω-τ)}) dτ | (4.17) |
| 9 | Parseval's Relation | Σ|x[n]|² | (1/2π) ∫_{-π}^{π} |X(e^{jω})|² dω | (4.18) |

**Notes on selected properties:**

**Differentiation proof (property 4):**

Differentiating X(e^{jω}) = Σ x[n] e^{-jωn} with respect to ω:

```
dX(e^{jω})/dω = Σ x[n] (-jn) e^{-jωn}  →  nx[n] ↔ j dX(e^{jω})/dω
```

Equivalently, using the chain rule: -e^{jω} dX/de^{jω} = j dX/dω (4.12).

**Convolution (property 7) — LTI application:**

```
y[n] = x[n] ⊗ h[n]  ↔  Y(e^{jω}) = X(e^{jω}) H(e^{jω})     (4.16)
```

Convolution in time = multiplication in frequency. This is the fundamental theorem underlying all LTI filter design.

**Multiplication (property 8):** Multiplication in time = convolution in frequency (within one period, hence the 1/2π normalization and integration over [-π, π]).

**Parseval's relation (property 9) — proof:**

```
Σ_{n=-∞}^{∞} |x[n]|² = Σ x[n] x*[n]
= Σ x[n] [(1/2π) ∫ X(e^{jω}) e^{jωn} dω]*
= (1/2π) ∫ X*(e^{jω}) [Σ x[n] e^{-jωn}] dω
= (1/2π) ∫_{-π}^{π} |X(e^{jω})|² dω                         (4.19)
```

Parseval's relation states that total signal energy = total spectral energy (up to the 1/2π factor).

---

### 3. Part 2: Discrete Fourier Transform (DFT)

#### 3.1 Definition

The DFT is obtained by **sampling the DTFT** at N uniformly-spaced discrete frequencies ω_k = 2πk/N, k = 0, 1, …, N-1.

Let x[n], n = 0, 1, …, N-1, be an **N-point finite sequence**. The DFT pair is:

**DFT (Analysis):**

```
X[k] = Σ_{n=0}^{N-1} x[n] e^{-j2πkn/N},    0 ≤ k ≤ N-1     (4.20)
       (0 otherwise)
```

**Inverse DFT (iDFT / Synthesis):**

```
x[n] = (1/N) Σ_{k=0}^{N-1} X[k] e^{j2πkn/N},    0 ≤ n ≤ N-1  (4.21)
       (0 otherwise)
```

**Key structural observations:**
- X[k] is a **periodic sequence with period N** (since it samples the periodic DTFT).
- Both DFT and iDFT sums run from 0 to N-1 → finite computation.
- The **twiddle factor** W_N = e^{-j2π/N}, so X[k] = Σ x[n] W_N^{kn}.
- DFT relates N time-domain samples to N frequency-domain samples.

#### 3.2 DTFT–DFT Relationship

X[k] = X(e^{jω})|_{ω = 2πk/N}

That is, the DFT is a sampled (discrete) version of the continuous DTFT. As we append more zeros to x[n] (zero-padding) and increase N, the DFT provides a finer sampling of the DTFT curve.

**Zero-padding rule:** DFT will approach DTFT when we append infinite zeros at the end of x[n]. In practice, appending L zeros before computing the N+L point DFT produces a denser grid of N+L points on the same DTFT curve.

#### 3.3 Properties of the DFT

| # | Property | Time Domain | Frequency Domain |
|---|---|---|---|
| 1 | Linearity | ax₁[n] + bx₂[n] | aX₁[k] + bX₂[k] |
| 2 | Time-shifting | x[n − m] | e^{-j2πkm/N} X[k] |
| 3 | Freq-shifting (modulation) | e^{-j2πk₀n/N} x[n] | X[k − k₀] |
| 4 | Time reversal | x[-n] | X[-k] |
| 5 | Conjugation | x*[n] | X*[-k] |
| 6 | Time-convolution | x₁[n] ⊗ x₂[n] (circular) | X₁[k] X₂[k] |
| 7 | Freq-convolution | x₁[n] · x₂[n] | (1/N) X₁[k] ⊗ X₂[k] (circular) |
| 8 | Parseval's relation | E_x = Σ_{n=0}^{N-1} |x[n]|² | E_x = (1/N) Σ_{k=0}^{N-1} |X[k]|² |

**Critical distinction — Circular convolution:** In the DFT, the time-domain convolution property uses **circular (cyclic) convolution** ⊗ of period N, not linear convolution. This is because DFT treats signals as periodic with period N. Linear convolution can be obtained from circular convolution by zero-padding both sequences to length ≥ N₁ + N₂ - 1 before taking DFTs.

**Conjugate symmetry for real sequences:** For a real-valued x[n]:

```
Re{X[k]} = Re{X[N-k]}    and    Im{X[k]} = -Im{X[N-k]}
```

This means only about half the DFT coefficients carry independent information — the upper half is the conjugate-mirror image of the lower half.

#### 3.4 Worked Examples

**Example 4.5 — DFT of x[n] = {1,1,1,0,...,0} with N=3:**

Using W_N = e^{-j2π/N}:

```
X[k] = Σ_{n=0}^{2} x[n] W_3^{kn} = W_3^0 + W_3^k + W_3^{2k}
     = e^{-j2πk/3} [1 + 2cos(2πk/3)]
     = { 3, k=0;  0, k=1,2 }
```

With N=5 (zero-padding with x[3]=x[4]=0):

```
X[k] = W_5^0 + W_5^k + W_5^{2k}
     = e^{-j2πk/5} [1 + 2cos(2πk/5)],  k=0,1,2,3,4
```

The magnitude plot (Fig. 4.3) shows: stem plot at k=0,…,4 with magnitude 3 at k=0 and smaller values at k=1,2,3,4; phase plot shows the corresponding phase angles. Zero-padding changes the DFT length and thus the frequency grid spacing (2π/N), but does not change the underlying spectrum.

---

**Example 4.6 — Frequency estimation using DFT:**

Given x[n] = 2 cos(0.7πn + 1), n = 0, 1, …, 20 (N=21 samples).

Continuous-time analogy: e^{jΩ₀t} ↔ 2πδ(Ω − Ω₀), so frequency is located at the spectral peak.

For DFT index k, the corresponding discrete frequency is ω = 2πk/N. The DFT magnitude will have two peaks at k=7 (ω≈0.667π) and k=14 (ω≈-0.7π, aliased).

```
ω̂₀ = 2π·7/21 ≈ 0.6667π  (coarse estimate)
```

To improve accuracy, **zero-pad** to N=2001 (append 1980 zeros):

```
x = [A*cos(w*n+p)  zeros(1,1980)];
```

Peak found at k=702, N=2001:

```
ω̂₀ = 2π·702/2001 ≈ 0.7016π  (much closer to true 0.7π)
```

**Fig. 4.4 (N=21, no zero-padding):** Magnitude stem plot shows two prominent peaks at k=7 and k=14, with smaller oscillatory sidelobes elsewhere. Phase plot is scattered.

**Fig. 4.5 (N=2001, with zero-padding):** Magnitude is now nearly continuous-looking with two sharp peaks, closely approximating the DTFT. The two peaks are clearly resolved at ω≈0.7π and ω≈-0.7π (reflected to near k=1300).

---

**Example 4.7 — Inverse DFT:**

Given X[k] = {1,1,1,0,0} (N=5):

```
x[n] = (1/N) Σ_{k=0}^{N-1} X[k] W_N^{-kn}
     = (1/5)(W_5^0 + W_5^{-n} + W_5^{-2n})
     = (1/5) e^{j2πn/5} [1 + 2cos(2πn/5)],  n=0,1,…,4
```

**Fig. 4.6 (described):** Magnitude stem plot for x[n] at n=0,…,4: largest value at n=0 (≈0.6), smaller values at n=1 and n=4 (≈0.33), smallest at n=2 and n=3 (≈0.13). Phase plot shows corresponding angles.

This is the inverse of Example 4.5 — DFT and iDFT are exact duals for N-point sequences.

---

### 4. Fast Fourier Transform (FFT)

#### 4.1 Motivation

Direct DFT computation: each X[k] requires N complex multiplications and N-1 complex additions. For k = 0, …, N-1, the total cost is:

```
DFT:  N² complex multiplications,  N(N-1) complex additions
```

For large N this is prohibitively expensive. The FFT algorithm (Cooley and Tukey, 1965) dramatically reduces this.

```
FFT:  (N/2) log₂N  complex multiplications
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

The speedup factor at N=1024 is ~204×. MATLAB/OCTAVE commands: `fft` (FFT) and `ifft` (inverse FFT).

#### 4.3 FFT applicability

- FFT is most efficient when N is a power of 2 (Cooley-Tukey radix-2 algorithm).
- The `fft` command in MATLAB/OCTAVE handles arbitrary N but is fastest for powers of 2.
- Like the DFT/iDFT duality, there is an iFFT corresponding to iDFT.

---

## Key terms (glossary)

- **DTFT** — Discrete-Time Fourier Transform: maps a discrete aperiodic signal to a continuous periodic spectrum via X(e^{jω}) = Σ x[n] e^{-jωn}.
- **iDTFT** — Inverse DTFT: recovers x[n] from X(e^{jω}) by x[n] = (1/2π) ∫_{-π}^{π} X(e^{jω}) e^{jωn} dω.
- **DFT** — Discrete Fourier Transform: maps a length-N sequence to N frequency samples X[k] = Σ x[n] e^{-j2πkn/N}.
- **iDFT** — Inverse DFT: x[n] = (1/N) Σ X[k] e^{j2πkn/N}.
- **ω** — Discrete-time frequency parameter (dimensionless, radians/sample); periodic with period 2π.
- **Twiddle factor** W_N — e^{-j2π/N}; DFT kernel written as W_N^{kn}.
- **Frequency response** H(e^{jω}) — DTFT of impulse response h[n]; completely characterizes an LTI system.
- **Eigenfunction property** — complex exponential e^{jω₀n} passes through LTI system unchanged in frequency; output = H(e^{jω₀}) e^{jω₀n}.
- **Absolute summability** — Σ |x[n]| < ∞; sufficient condition for DTFT to converge (equivalent to BIBO stability for impulse response).
- **Circular convolution** — convolution modulo N, the natural operation in the DFT domain; differs from linear convolution unless zero-padded.
- **Zero-padding** — appending zeros to an N-point sequence before DFT to increase frequency resolution (sample the DTFT on a finer grid).
- **Parseval's relation** — energy equivalence: total time-domain energy = total spectral energy (with 1/2π or 1/N normalization for DTFT/DFT respectively).
- **FFT** — Fast Fourier Transform; Cooley-Tukey algorithm reducing DFT cost from O(N²) to O(N log₂N).
- **sinc function** — sinc(u) = sin(πu)/(πu); the DTFT magnitude of a rectangular pulse reduces to a sinc envelope.
- **Conjugate symmetry** — for real x[n]: X[k] = X*[N-k]; only N/2 + 1 independent DFT values.

---

## Exam targets

1. **Write both DTFT equations from memory** (analysis eq. 4.1, synthesis/inverse eq. 4.4) and state the domain of each (discrete aperiodic ↔ continuous periodic).

2. **Prove the inverse DTFT** (substitute analysis into synthesis, show integral gives Kronecker delta δ[n-m]) — this derivation appeared in the lecture (eq. 4.5).

3. **State and apply all 9 DTFT properties** from the table. Be able to compute DTFTs of shifted, scaled, or modulated signals without redoing the full sum.

4. **Derive the DTFT of a rectangular pulse** x[n] = u[n] - u[n-N]: geometric series → closed form → factor to get sinc-like expression (Ex. 4.2).

5. **Write both DFT equations** (eqs. 4.20, 4.21) and state the index ranges. Explain why X[k] is periodic with period N.

6. **Explain the DTFT–DFT relationship**: DFT = samples of DTFT at ω_k = 2πk/N. What changes when N changes (for the same underlying signal)? What does zero-padding achieve?

7. **Compare circular vs linear convolution** in the DFT context: explain why DFT convolution is circular, and how to obtain linear convolution results using zero-padding to length ≥ N₁+N₂-1.

8. **Reproduce the DFT properties table** (at least linearity, time-shift, circular convolution, Parseval). Note differences from DTFT properties (e.g. e^{-j2πkm/N} vs e^{-jωn₀}).

9. **State FFT complexity vs DFT complexity** (N² vs N/2 log₂N) and reproduce or explain the comparison table for key values of N. Know that FFT is an algorithm, not a new transform.

10. **Frequency estimation workflow** (Ex. 4.6): identify peak index k* in |X[k]|, convert to ω̂₀ = 2πk*/N, and understand why zero-padding improves the estimate.

11. **Parseval's relation for DTFT and DFT** — write both forms and prove the DTFT version (the proof uses the iDTFT to substitute for x*[n]).

---

## Pitfalls

- **DTFT output is continuous; DFT output is discrete.** Never write X(e^{jω}) with a bracket index [k] or treat the DTFT as producing N values.
- **The DTFT is 2π-periodic — not arbitrary period.** The principal interval is [-π, π] or [0, 2π]; both are valid for one period.
- **u[n] does NOT have a DTFT** in the classical (absolute summability) sense — Ex. 4.1 shows the sum diverges. This is a classic exam trap.
- **DFT circular convolution ≠ linear convolution.** If you compute X₁[k]·X₂[k] and take iDFT, you get circular (N-periodic) convolution. To get linear convolution, zero-pad to at least N₁+N₂-1 first.
- **Zero-padding does not add new information** about the signal; it only produces a finer sampling of the existing DTFT (interpolates in frequency domain). It does NOT improve spectral resolution of two close frequencies unless the signal itself is long enough.
- **The FFT requires N to be a power of 2 for peak efficiency** (radix-2 Cooley-Tukey). MATLAB's `fft` works for any N but is slower for non-powers of 2.
- **Conjugate symmetry applies only for real-valued sequences**: X[k] = X*[N-k]. Do not assume this for complex-valued x[n].
- **DFT index k maps to frequency ω = 2πk/N**, not ω = k. The DFT domain is indexed by integer k, the corresponding frequency is ω_k = 2πk/N.
- **Time-shift in DFT**: x[n-m] corresponds to e^{-j2πkm/N} X[k], not e^{-jωm} X(e^{jω}) — the exponent uses discrete index m and fixed N.
- **iDFT has +j in the exponent and a 1/N normalization**; iDTFT has +j in the exponent and 1/(2π) normalization. Do not confuse the two.
- **The Parseval DFT form** uses 1/N, while the DTFT form uses 1/(2π): E_x = (1/N) Σ|X[k]|² (DFT) vs E_x = (1/2π) ∫|X(e^{jω})|² dω (DTFT).
