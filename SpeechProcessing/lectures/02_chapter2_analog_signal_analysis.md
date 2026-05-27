# Chapter 2 — Review of Analog Signal Analysis

## Bird's eye view

- **Two tools for analyzing analog signals**: Fourier series (for continuous-time *periodic* signals) and Fourier transform (for continuous-time *aperiodic* signals); both convert between the time domain x(t) and the frequency domain X(jΩ).
- **Fourier series** decomposes a periodic signal into harmonically related complex exponentials at discrete frequencies ..., -Ω₀, 0, Ω₀, 2Ω₀, ...; the spectrum is **discrete and aperiodic**.
- **Fourier transform** decomposes an aperiodic signal into a continuum of frequencies; the spectrum X(jΩ) is **continuous and aperiodic** and is also called the spectrum.
- Key special signals: the **Dirac delta** δ(t) (unit area, sifting property) and the **unit step** u(t). The FT of δ(t) is 1 (flat spectrum). The FT of e^(jΩ₀t) is 2πδ(Ω − Ω₀).
- **Analog LTI systems** are fully characterized by the impulse response h(t); output = convolution y(t) = x(t) ⊗ h(t). Convolution in time ↔ multiplication in frequency: Y(jΩ) = X(jΩ)H(jΩ).
- **FT can be derived from FS** by letting the period T → ∞ (Ω₀ → 0), turning the discrete sum into an integral — the inverse FT formula.
- **Impulse train** (sum of shifted deltas, period T) has FT that is also an impulse train in frequency with spacing Ω₀ = 2π/T — a central result for sampling theory.

---

## Detailed notes

### 1. Fourier Series

#### 1.1 Periodicity and fundamental frequency

A continuous-time function x(t) is **periodic** if there exists T_p > 0 such that:

```
x(t) = x(t + T_p),    for all t ∈ (−∞, ∞)    (2.2)
```

The smallest such T_p is called the **fundamental period**. The **fundamental (angular) frequency** is:

```
Ω₀ = 2π / T_p    (rad/s)    (2.3)
```

In the frequency domain, Ω only takes discrete values: ..., −Ω₀, 0, Ω₀, 2Ω₀, ...

#### 1.2 Fourier series synthesis and analysis formulas

Every periodic function can be expanded (synthesized) as:

```
x(t) = Σ_{k=−∞}^{∞}  a_k · e^{jkΩ₀t},    t ∈ (−∞, ∞)    (2.4)
```

The **Fourier series coefficients** (analysis formula):

```
a_k = (1/T_p) ∫_{−T_p/2}^{T_p/2}  x(t) · e^{−jkΩ₀t} dt,    k = ...,−1, 0, 1, 2,...    (2.5)
```

The set {a_k} characterizes X(jΩ) — it is the frequency representation of x(t).

#### 1.3 Magnitude and phase of coefficients

Since a_k is generally complex:

```
|a_k| = sqrt( (Re{a_k})² + (Im{a_k})² )    (2.6)

angle(a_k) = arctan( Im{a_k} / Re{a_k} )    (2.7)
```

Fig. 2.1 (described): The left panel shows a continuous, periodic time-domain signal with period T_p. The right panel shows its frequency-domain representation as a discrete set of vertical lines (impulses) at multiples of Ω₀, with heights {a_k}. Summary: continuous + periodic in time ↔ discrete + aperiodic in frequency.

#### 1.4 Procedure for computing Fourier series coefficients

1. Determine the fundamental period T_p and fundamental frequency Ω₀ = 2π/T_p.
2. For each k, multiply x(t) by e^{−jkΩ₀t}, integrate over one complete period [−T_p/2, T_p/2], and divide by T_p.
3. Handle k = 0 and k ≠ 0 separately to avoid 0/0 indeterminate forms.

---

#### Example 2.1 — Cosine sum (inspection method)

Find the FS coefficients for x(t) = cos(10πt) + cos(20πt).

- Fundamental frequency: Ω₀ = 10π, so T_p = 2π/Ω₀ = 1/5.
- Using Euler's formula cos(u) = (e^{ju} + e^{−ju})/2:

```
x(t) = (1/2)e^{−j2Ω₀t} + (1/2)e^{−jΩ₀t} + (1/2)e^{jΩ₀t} + (1/2)e^{j2Ω₀t}
```

- By inspection (matching with 2.4): **a_{−2} = a_{−1} = a₁ = a₂ = 1/2**, all other a_k = 0.

---

#### Example 2.2 — Mixed sinusoids

Find the FS coefficients for x(t) = 1 + sin(Ω₀t) + 2cos(Ω₀t) + cos(3Ω₀t + π/4).

Using Euler formulas (sin(u) = (e^{ju} − e^{−ju})/(2j)):

```
x(t) = 1 + (1 + 1/(2j))e^{jΩ₀t} + (1 − 1/(2j))e^{−jΩ₀t}
         + (1/2)e^{jπ/4}e^{j3Ω₀t} + (1/2)e^{−jπ/4}e^{−j3Ω₀t}
```

Results:
```
a_{−3} = (√2/4)(1 − j),   k = −3
a_{−1} = 1 + j/2,          k = −1
a₀    = 1,                  k = 0
a₁    = 1 − j/2,           k = 1
a₃    = (√2/4)(1 + j),    k = 3
a_k   = 0,                  otherwise
```

Computing magnitude and phase for k = −3:
```
|a_{−3}| = sqrt( (√2/4)² + (−√2/4)² ) = 1/2

angle(a_{−3}) = arctan(−1) = −π/4
```

---

#### Example 2.3 — Rectangular pulse train (sinc-shaped spectrum)

x(t) is a periodic pulse train with fundamental period T and pulse width 2T₀ (where T > 2T₀). Over one period:
```
x(t) = 1,  −T₀ < t < T₀
x(t) = 0,  otherwise
```

Fundamental frequency: Ω₀ = 2π/T.

**For k = 0:**
```
a₀ = (1/T) ∫_{−T₀}^{T₀} 1 dt = 2T₀/T
```

**For k ≠ 0:**
```
a_k = (1/T) ∫_{−T₀}^{T₀} e^{−jkΩ₀t} dt
    = [−1/(jkΩ₀T)] · e^{jkΩ₀t} |_{−T₀}^{T₀}
    = sin(kΩ₀T₀) / (kπ)
    = sin(2πkT₀/T) / (kπ)
```

Note: The k=0 case is verified by L'Hopital's rule on the k≠0 formula:
```
lim_{k→0}  sin(2πkT₀/T) / (kπ)  =  2T₀/T
```
which matches a₀ — confirming consistency.

---

### 2. Fourier Transform

#### 2.1 Definition

The **Fourier transform** of a continuous-time, *aperiodic* signal x(t):

```
X(jΩ) = ∫_{−∞}^{∞}  x(t) · e^{−jΩt} dt    (2.8)
```

X(jΩ) is also called the **spectrum** of x(t). Unlike the FS, Ω here is continuous.

The **inverse Fourier transform**:

```
x(t) = (1/2π) ∫_{−∞}^{∞}  X(jΩ) · e^{jΩt} dΩ    (2.9)
```

Notation: x(t) ↔ X(jΩ)

Fig. 2.2 (described): Left panel — a smooth, non-periodic (finite-energy) time-domain signal. Right panel — its spectrum X(jΩ) is a smooth continuous curve (semi-circular shape shown). Both sides are continuous and aperiodic. This contrasts with the FS case.

#### 2.2 The Dirac delta function

The delta function δ(t) is defined by:

```
δ(t) = 0,    t ≠ 0                         (2.10)
∫_{−∞}^{∞} δ(t) dt = 1                     (2.11)
```

Key property — **sifting property**:

```
∫_{−∞}^{∞} f(t) · δ(t − t₀) dt = f(t₀)    (2.12)
```

where f(t) is any continuous-time signal.

Interpretation: δ(t) is zero everywhere except at t = 0, where it is not well-defined (it has an "infinite" value but finite area = 1). Represented graphically as an arrow at t = 0 with label 1 (Fig. 2.3).

#### 2.3 The unit step function

```
u(t) = 1,   t > 0
u(t) = 0,   t < 0    (2.13)
```

u(0) is not well defined (discontinuity at t = 0).

#### 2.4 Important Fourier transform pairs

**Pair 1 — Rectangular pulse → Sinc** (Example 2.4):

```
x(t) = 1,  −T₀ < t < T₀;   x(t) = 0 otherwise

X(jΩ) = 2sin(ΩT₀)/Ω  =  2T₀ · sinc(ΩT₀/π)
```

where sinc(u) = sin(πu)/(πu).

Fig. 2.4 (described): Left — rectangular pulse of height 1, width 2T₀. Right — sinc-shaped spectrum with peak value 2T₀ at Ω=0 and zero-crossings at ±π/T₀, ±2π/T₀, ...

**Pair 2 — Rectangular spectrum → Sinc in time** (Example 2.5, inverse FT):

```
X(jΩ) = 1,  −W₀ < Ω < W₀;   X(jΩ) = 0 otherwise

x(t) = sin(W₀t)/(πt)  =  (W₀/π) · sinc(W₀t/π)
```

Fig. 2.5 (described): Left — rectangular pulse in frequency domain with width 2W₀. Right — sinc function in time with peak W₀/π and zero-crossings at ±π/W₀, ±2π/W₀, ...

**Duality observation**: Comparing Examples 2.4 and 2.5 — a rectangular pulse in time gives a sinc in frequency, and a rectangular pulse in frequency gives a sinc in time. This is the **duality property** of the Fourier transform.

**Pair 3 — One-sided exponential** (Example 2.6):

```
x(t) = e^{−at} · u(t),   a > 0

X(jΩ) = 1/(a + jΩ)  =  (a − jΩ)/(a² + Ω²)

|X(jΩ)| = 1/sqrt(a² + Ω²)

angle(X(jΩ)) = −arctan(Ω/a)
```

Fig. 2.6 (described): Left magnitude plot — bell-shaped curve with peak 1/a at Ω=0, falling to √2/(2a) at Ω=±a. Right phase plot — starts at 0 for large negative Ω, crosses through −π/4 at Ω=a, and asymptotes to −π/2 for large positive Ω.

**Pair 4 — Delta function** (Example 2.7):

Using the sifting property with f(t) = e^{−jΩt} and t₀ = 0:

```
X(jΩ) = ∫_{−∞}^{∞} δ(t) e^{−jΩt} dt = e^{−jΩ·0} = 1

Therefore:  δ(t) ↔ 1    (flat spectrum at all frequencies)
```

**Pair 5 — Complex exponential** (derived from Pair 4 via inverse FT):

```
e^{jΩ₀t} ↔ 2π δ(Ω − Ω₀)    (2.16)
```

Derivation: If X(jΩ) = 2πδ(Ω − Ω₀), then by the inverse FT:
x(t) = (1/2π)∫ 2πδ(Ω − Ω₀)e^{jΩt}dΩ = e^{jΩ₀t}. ✓

#### 2.5 FT of a periodic signal (via FS coefficients)

Combining (2.4) and (2.16): the FT of a periodic signal expressed via its FS is:

```
Σ_{k=−∞}^{∞} a_k e^{jkΩ₀t}  ↔  Σ_{k=−∞}^{∞} 2π a_k δ(Ω − kΩ₀)    (2.17)
```

The spectrum of a periodic signal is a train of impulses at the harmonic frequencies kΩ₀, each weighted by 2π a_k.

#### 2.6 Impulse train and its FT (Example 2.8)

The **impulse train** (also called Dirac comb):

```
x(t) = Σ_{k=−∞}^{∞} δ(t − kT)
```

is periodic with period T. Using (2.5), the FS coefficients are:

```
a_k = (1/T) ∫_{−T/2}^{T/2} δ(t) e^{−jkΩ₀t} dt = 1/T
```

Using (2.17), its Fourier transform is:

```
X(jΩ) = (2π/T) Σ_{k=−∞}^{∞} δ(Ω − 2πk/T)  =  Ω₀ Σ_{k=−∞}^{∞} δ(Ω − kΩ₀)
```

Fig. 2.7 (described): Left — impulse train in time with equal unit-height impulses at ..., −2T, −T, 0, T, 2T, ... Right — impulse train in frequency with equal height Ω₀ impulses at ..., −2Ω₀, −Ω₀, 0, Ω₀, 2Ω₀, ... Key insight: the FT of an impulse train is another impulse train — the spacing in time (T) and the spacing in frequency (Ω₀ = 2π/T) are reciprocally related.

#### 2.7 Derivation of the Fourier transform from the Fourier series

The FT can be obtained from the FS by a limiting argument. Construct x̃(t) as the periodic extension of x(t) with period T (Fig. 2.8). As T → ∞ (i.e., Ω₀ → 0), x̃(t) → x(t).

The FS coefficients of x̃(t) are:

```
a_k = (1/T) X(jkΩ₀)    (2.20)
```

so the FS expansion of x̃(t) is:

```
x̃(t) = (1/2π) Σ_{k=−∞}^{∞} Ω₀ X(jkΩ₀) e^{jkΩ₀t}    (2.21)
```

As Ω₀ → 0, the sum becomes a Riemann integral:

```
x(t) = lim_{Ω₀→0} x̃(t) = (1/2π) ∫_{−∞}^{∞} X(jΩ) e^{jΩt} dΩ    (2.22)
```

This is exactly the inverse FT formula — confirming internal consistency.

---

### 3. Analog Linear Time-Invariant (LTI) Systems

#### 3.1 Properties defining an LTI system

**Linearity**: If (x₁(t), y₁(t)) and (x₂(t), y₂(t)) are input-output pairs, then:
```
a·x₁(t) + b·x₂(t)  →  a·y₁(t) + b·y₂(t)
```

**Time-invariance**: If x(t) → y(t), then:
```
x(t − t₀)  →  y(t − t₀)
```
(A time shift in the input causes the same time shift in the output — the system does not change over time.)

#### 3.2 Convolution (input-output relationship)

The output of an LTI system is given by the **convolution integral**:

```
y(t) = x(t) ⊗ h(t) = ∫_{−∞}^{∞} x(τ) h(t − τ) dτ    (2.23)
```

where:
- x(t) = input signal
- h(t) = **impulse response** (the system's output when the input is δ(t))
- y(t) = output signal

The impulse response h(t) **completely characterizes** the LTI system.

#### 3.3 Convolution-multiplication duality

**Convolution in the time domain corresponds to multiplication in the frequency domain:**

```
x(t) ⊗ h(t)  ↔  X(jΩ) · H(jΩ)    (2.24)
```

where H(jΩ) is the **transfer function** (= FT of the impulse response h(t)).

**Proof** (change of variable u = t − τ):

```
Y(jΩ) = FT{x(t) ⊗ h(t)}
       = ∫_{−∞}^{∞} ∫_{−∞}^{∞} x(τ)h(t−τ)e^{−jΩt} dτ dt
       = ∫_{−∞}^{∞} ∫_{−∞}^{∞} x(τ)h(u)e^{−jΩτ}e^{−jΩu} dτ du     [u = t−τ]
       = [∫_{−∞}^{∞} x(τ)e^{−jΩτ}dτ] · [∫_{−∞}^{∞} h(u)e^{−jΩu}du]
       = X(jΩ) · H(jΩ)
```

Therefore y(t) = IFT{ X(jΩ) H(jΩ) }.

---

## Key terms (glossary)

| Term | Definition |
|---|---|
| **Fundamental period T_p** | Smallest positive T_p such that x(t) = x(t + T_p) for all t. |
| **Fundamental frequency Ω₀** | Ω₀ = 2π/T_p (rad/s); the lowest frequency in the FS. |
| **Fourier series coefficients {a_k}** | Complex numbers characterizing the frequency content of a periodic signal at each harmonic kΩ₀. |
| **Harmonic** | A frequency component at an integer multiple k of Ω₀. |
| **Fourier transform X(jΩ)** | Continuous-frequency representation (spectrum) of an aperiodic signal. |
| **Spectrum** | Another name for X(jΩ); fully describes the signal in the frequency domain. |
| **Dirac delta δ(t)** | Generalized function: zero for t≠0, integrates to 1; defined via the sifting property. |
| **Sifting property** | ∫f(t)δ(t−t₀)dt = f(t₀); extracts the value of f at t₀. |
| **Unit step u(t)** | 1 for t>0, 0 for t<0; undefined at t=0. |
| **sinc function** | sinc(u) = sin(πu)/(πu); appears as the FT of a rectangular pulse. |
| **Duality property** | If x(t) ↔ X(jΩ), then X(jt) ↔ 2πx(−Ω); rect in time ↔ sinc in freq and vice versa. |
| **Impulse train** | Σ_{k} δ(t−kT); periodic signal whose FT is also an impulse train. |
| **Impulse response h(t)** | Output of an LTI system when the input is δ(t); fully characterizes the system. |
| **Transfer function H(jΩ)** | FT of the impulse response h(t); used for frequency-domain analysis of LTI systems. |
| **Convolution** | y(t) = ∫x(τ)h(t−τ)dτ; the time-domain operation for LTI output. |
| **Linearity** | Superposition holds: weighted sums of inputs map to weighted sums of outputs. |
| **Time-invariance** | System behaviour does not depend on when the input is applied. |

---

## Exam targets

1. **Write the FS synthesis and analysis formulas** (2.4 and 2.5) from memory, including correct limits of integration and the 1/T_p factor.

2. **Compute FS coefficients for a given periodic signal** — use Euler's formulas to convert sinusoids into complex exponentials, then read off the coefficients by inspection or integrate (2.5) directly. Always handle k=0 and k≠0 separately.

3. **Compute FS coefficients for a rectangular pulse train** — know the sinc-shaped result a_k = sin(kΩ₀T₀)/(kπ) = sin(2πkT₀/T)/(kπ) and the DC term a₀ = 2T₀/T. Verify using L'Hopital.

4. **Write the FT pair (2.8) and (2.9)** from memory. Know the difference from the FS: aperiodic ↔ continuous Ω; periodic ↔ discrete Ω.

5. **State and apply the sifting property** (2.12). Use it to compute the FT of δ(t) = 1 and the FT of e^{jΩ₀t} = 2πδ(Ω−Ω₀).

6. **Derive the FT of a periodic signal from its FS**: given FS coefficients {a_k}, the FT is a sum of weighted impulses at harmonics — equation (2.17).

7. **Derive the FT of the impulse train** (Example 2.8) — the FT is also an impulse train with spacing Ω₀ = 2π/T.

8. **State the convolution-multiplication property** (2.24) and reproduce its proof (change of variable u = t − τ, then separate the double integral into a product of two single integrals).

9. **Define linearity and time-invariance** with precise mathematical statements.

10. **Apply the complete LTI analysis pipeline**: given x(t) and h(t), compute Y(jΩ) = X(jΩ)H(jΩ), then take the inverse FT to get y(t).

---

## Pitfalls

- **Ω vs ω notation**: This course uses Ω for analog (continuous-time) angular frequency in rad/s. Do not confuse with ω which is sometimes used for the same thing in other texts.
- **Factor of 1/(2π) in the inverse FT**: The forward FT has no factor; the inverse FT divides by 2π (eq. 2.9). Getting this wrong is a common error.
- **Factor of 1/T_p in the FS analysis formula** (eq. 2.5): The synthesis formula (2.4) has no prefactor; the analysis formula divides by T_p. Also easy to swap.
- **k = 0 vs k ≠ 0 in FS computation**: The general formula for k≠0 gives 0/0 at k=0. Always compute a₀ separately as a simple average (integral of x(t)/T_p over one period), then verify by L'Hopital on the general formula.
- **Periodic → discrete spectrum; aperiodic → continuous spectrum**: Never apply the FS to an aperiodic signal or confuse which formula to use. Periodic/aperiodic in time ↔ discrete/continuous in frequency.
- **FT of a periodic signal is not a regular function**: It involves Dirac deltas (impulses) in the frequency domain (eq. 2.17) — do not expect a smooth X(jΩ).
- **Duality is not symmetry**: Duality says X(jt) ↔ 2πx(−Ω), not X(jt) ↔ x(Ω). The 2π factor and sign flip on Ω are often forgotten.
- **Impulse response vs transfer function**: h(t) is in the time domain; H(jΩ) = FT{h(t)} is in the frequency domain. They are a FT pair — do not confuse them.
- **Convolution is commutative**: x(t) ⊗ h(t) = h(t) ⊗ x(t). The impulse response can be written either way in the integral.
- **δ(t) vs δ(Ω)**: δ(t) ↔ 1 (in frequency). A constant (=1) in time ↔ 2πδ(Ω) in frequency. These are different — do not mix them up.
