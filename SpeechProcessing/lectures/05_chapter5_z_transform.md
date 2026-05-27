# Chapter 5 — The z-Transform

## Bird's eye view

- The **z-transform** X(z) = sum_{n=-inf}^{inf} x[n] z^{-n} generalises the DTFT: evaluating X(z) on the unit circle |z| = 1 recovers the DTFT X(e^{jw}).
- The **Region of Convergence (ROC)** must always be stated alongside X(z); the same algebraic expression can correspond to different signals depending on the ROC.
- ROC shapes follow strict rules (8 properties): exterior disk for right-sided signals, interior disk for left-sided, annulus for two-sided, whole plane (minus z=0 and/or z=inf) for finite-duration.
- **Poles and zeros** fully characterise a rational X(z); poles are excluded from the ROC. Causality and stability of LTI systems are read directly from the ROC and pole locations.
- Four methods for the **inverse z-transform**: inspection (table lookup), partial fraction expansion (4 cases), power series expansion (long division), and the Cauchy contour integral.
- **Seven z-transform properties** (linearity, time shifting, modulation, differentiation, conjugation, time reversal, convolution) make it practical to compute transforms of complex signals from simple table entries.
- The **transfer function** H(z) = Y(z)/X(z) links a difference equation to the z-domain; causality/stability are determined by whether the ROC includes the unit circle and how poles relate to it.

---

## Detailed notes

### 1. Definition and relationship to the DTFT

**Definition (5.1)**

    X(z) = sum_{n=-inf}^{inf} x[n] z^{-n}

where z is a continuous complex variable. X(z) is in general complex-valued.

**Polar form of z.** Writing z = r e^{jw} (r = |z| > 0, w = angle of z):

    X(z)|_{z = r e^{jw}} = X(r e^{jw}) = sum_{n=-inf}^{inf} (x[n] r^{-n}) e^{-jwn}   (5.7)

This is the DTFT of the exponentially weighted sequence x[n] r^{-n}.

**Link to DTFT (5.8).** When r = 1, i.e. z = e^{jw} (the unit circle):

    X(z)|_{z = e^{jw}} = X(e^{jw}) = sum_{n=-inf}^{inf} x[n] e^{-jwn}

which is exactly the DTFT. Therefore: **the DTFT is the z-transform evaluated on the unit circle**, provided the unit circle lies within the ROC.

Fig 5.1 (description): The z-plane with real axis Re and imaginary axis Im. The unit circle (|z| = 1) is drawn; a point z = e^{jw} on the circle is at angle w from the positive real axis. The DTFT corresponds to traversing this circle.

**Why does X(e^{jw}) have period 2pi?** Because e^{-j(w+2kpi)n} = e^{-jwn} for any integer k. This periodicity is a fundamental property of the DTFT (5.5).

---

### 2. Region of Convergence (ROC)

**Definition.** The ROC is the set of z for which the sum converges absolutely:

    |X(z)| = |sum x[n] z^{-n}| <= sum |x[n] z^{-n}| < inf   (5.10)

The ROC must be specified; without it X(z) is ambiguous.

#### 2.1 Deriving the ROC shape via ratio test

Decompose X(z) = X_-(z) + X_+(z) where:
- X_+(z) = sum_{n=0}^{inf} x[n] z^{-n} (causal part) — converges when |z| > R_+
- X_-(z) = sum_{n=-inf}^{-1} x[n] z^{-n} (anti-causal part) — converges when |z| < R_-

Combining: **ROC for X(z) is R_+ < |z| < R_-** (an annulus).
- If R_+ < R_-, the ROC is a non-empty ring.
- If R_- < R_+, no ROC exists and X(z) does not exist.

Fig 5.2 (description): Three z-planes side by side. Left: X_+(z) with ROC = exterior of circle of radius R_+ (shaded outside). Centre: X_-(z) with ROC = interior of circle of radius R_- (shaded inside). Right: X(z) with ROC = annular ring R_+ < |z| < R_-.

#### 2.2 Poles and zeros

For a rational X(z) = P(z)/Q(z) in factored form:

    X(z) = b_0 (z - d_1)(z - d_2)...(z - d_M) / [a_0 (z - c_1)(z - c_2)...(z - c_N)]   (5.19)

- **Zeros**: values z = d_k where X(z) = 0 (roots of numerator)
- **Poles**: values z = c_k where X(z) = inf (roots of denominator)

Key counting rule (multiply z^{M+N} to (5.18)):
- If M > N: there are (M - N) additional poles at z = 0
- If M < N: there are (N - M) additional zeros at z = 0

**The ROC never contains a pole** (P3).

#### 2.3 Eight ROC properties

- **P1.** Four possible shapes: entire plane (minus z=0 and/or z=inf), ring, interior of circle, exterior of circle.
- **P2.** The DTFT exists if and only if the ROC includes the unit circle.
- **P3.** The ROC cannot contain any poles.
- **P4.** For finite-duration sequences: ROC = entire z-plane, possibly excluding z = 0 and/or z = inf.
- **P5.** Right-sided sequence: ROC of the form |z| > |p_max| (outside the largest pole).
- **P6.** Left-sided sequence: ROC of the form |z| < |p_min| (inside the smallest pole).
- **P7.** Two-sided sequence: ROC is an annulus |p_a| < |z| < |p_b| between two successive poles.
- **P8.** The ROC must be a connected region.

Fig 5.5 (description): Three pairs (signal / ROC) for infinite-duration sequences. Top pair: right-sided signal (nonzero for n >= some value) — ROC is exterior disk |z| > R_+. Middle pair: left-sided signal — ROC is interior disk |z| < R_-. Bottom pair: two-sided signal — ROC is annular ring R_+ < |z| < R_-.

Fig 5.7 (description): For X(z) with three poles at |a| < |b| < |c|, there are four possible connected ROCs: |z| < |a|; |a| < |z| < |b|; |b| < |z| < |c|; |z| > |c|. Each is a connected annular region (or disk/complement) that excludes all poles.

#### 2.4 ROC for finite-duration signals — summary table

| Type of finite-duration signal | ROC |
|---|---|
| Causal (nonzero only for n >= 0) | All z except z = 0 |
| Anti-causal (nonzero only for n <= 0) | All z except z = inf |
| Bi-directional (nonzero for both sides) | All z except z = 0 and z = inf |

Example (causal, page 20): x[n] = {1, 2, 5, 7, 0, 1} with n=0 at first element.
  X(z) = 1 + 2z^{-1} + 5z^{-2} + 7z^{-3} + z^{-5} — ROC = entire z-plane except z = 0.

---

### 3. Common z-transform pairs (Table 5.1)

| Sequence x[n] | Transform X(z) | ROC |
|---|---|---|
| delta[n] | 1 | All z |
| delta[n - m] | z^{-m} | |z| > 0 (m > 0); |z| < inf (m < 0) |
| a^n u[n] | 1 / (1 - a z^{-1}) | |z| > |a| |
| -a^n u[-n-1] | 1 / (1 - a z^{-1}) | |z| < |a| |
| n a^n u[n] | a z^{-1} / (1 - a z^{-1})^2 | |z| > |a| |
| -n a^n u[-n-1] | a z^{-1} / (1 - a z^{-1})^2 | |z| < |a| |
| a^n cos(bn) u[n] | (1 - a cos(b) z^{-1}) / (1 - 2a cos(b) z^{-1} + a^2 z^{-2}) | |z| > |a| |
| a^n sin(bn) u[n] | a sin(b) z^{-1} / (1 - 2a cos(b) z^{-1} + a^2 z^{-2}) | |z| > |a| |

**Critical observation:** a^n u[n] and -a^n u[-n-1] give the SAME algebraic X(z) = 1/(1-az^{-1}) but with DIFFERENT ROCs (|z|>|a| vs |z|<|a|). The ROC distinguishes which signal is meant.

---

### 4. Worked examples for forward z-transform

#### Example 5.1: x[n] = a^n u[n]

    X(z) = sum_{n=0}^{inf} (az^{-1})^n = 1/(1 - az^{-1}) = z/(z-a),   |z| > |a|

- Pole at z = a; zero at z = 0.
- DTFT exists when ROC includes unit circle, i.e. |a| < 1.
- Fig 5.3: When |a| < 1, the pole at a is inside the unit circle and the ROC |z| > |a| includes the unit circle (shaded exterior). When |a| > 1, the pole is outside the unit circle so the unit circle is not in the ROC — DTFT does not exist.

#### Example 5.2: x[n] = -a^n u[-n-1]

    X(z) = z/(z-a),   |z| < |a|

Same expression as Example 5.1 but ROC is the interior. DTFT exists when |a| > 1.

#### Example 5.3: x[n] = a^n u[n] + b^n u[-n-1], |a| < |b|

Using results of Ex 5.1 and 5.2:

    X(z) = 1/(1-az^{-1}) - 1/(1-bz^{-1}) = (a-b)z / [(z-a)(z-b)],   |a| < |z| < |b|

ROC is a ring (two-sided signal, aligns with P7).

#### Example 5.4: x[n] = delta[n+1]

    X(z) = z,   ROC = all z except z = 0 (anti-causal finite, aligns with table above)

#### Example 5.5: x[n] = a^n for 0 <= n <= N-1, 0 otherwise

    X(z) = sum_{n=0}^{N-1} (az^{-1})^n = (1 - (az^{-1})^N) / (1 - az^{-1}) = (1/z^{N-1}) * (z^N - a^N)/(z - a)

ROC = entire z-plane except z = 0 (finite causal duration).

---

### 5. Inverse z-Transform

**Formal definition (5.20)**:

    x[n] = (1/2pij) contour-integral_Gamma X[z] z^{n-1} dz

where Gamma is a counterclockwise closed circular contour in the ROC. In practice, four methods are used.

#### Method 1: Inspection

Match X(z) algebraically to a known table entry. Example 5.7:

Given X(z) = z/(2z-1) = 0.5 / (1 - 0.5 z^{-1}), |z| > 0.5.

Using a^n u[n] <-> 1/(1-az^{-1}) with a = 0.5:
    x[n] = 0.5 * (0.5)^n u[n] = (0.5)^{n+1} u[n]

#### Method 2: Partial Fraction Expansion

Express X(z) as a ratio of polynomials in z^{-1}:

    X(z) = sum_{k=0}^{M} b_k z^{-k} / sum_{k=0}^{N} a_k z^{-k}   (5.21)

To find poles and zeros: multiply numerator and denominator by z^{M+N} to get polynomials in z.

**Case 1: M < N, all poles of first order (distinct)**

    X(z) = sum_{k=1}^{N} A_k / (1 - c_k z^{-1})   (5.23)

Residue formula:

    A_k = (1 - c_k z^{-1}) X(z) |_{z = c_k}   (5.24)

Steps: (i) find poles c_k; (ii) compute A_k via (5.24); (iii) invert each term by inspection.

**Case 2: M >= N, all poles of first order**

    X(z) = sum_{l=0}^{M-N} B_l z^{-l} + sum_{k=1}^{N} A_k / (1 - c_k z^{-1})   (5.26)

B_l are found by long division of numerator by denominator (stop when remainder is lower degree than denominator). Then find A_k using (5.24).

**Case 3: M < N, one multiple-order pole at c_i of order s >= 2**

    X(z) = sum_{k=1, k!=i}^{N} A_k/(1-c_k z^{-1}) + sum_{m=1}^{s} C_m / (1-c_i z^{-1})^m   (5.27)

Coefficient formula:

    C_m = [1/((s-m)!(-c_i)^{s-m})] * d^{s-m}/dw^{s-m} [(1-c_i w)^s X(w^{-1})] |_{w = c_i^{-1}}   (5.28)

**Case 4: M >= N with multiple-order poles** — combine Cases 2 and 3 using (5.29).

**Full worked example (Example 5.8): H(z) = -(1+0.1z^{-1})/(1-2.05z^{-1}+z^{-2})**

Multiply by z^2: H(z) = -z(z+0.1)/(z^2 - 2.05z + 1). Poles: z = 0.8, z = 1.25. Zeros: z = 0, z = -0.1.

PFE (no ROC specified — investigate all 3 ROC cases):

A_1 = (1-0.8z^{-1}) H(z)|_{z=0.8} = 2; A_2 = -3.

H(z) = 2/(1-0.8z^{-1}) - 3/(1-1.25z^{-1})

- |z| > 1.25: h[n] = (2(0.8)^n - 3(1.25)^n) u[n] — causal, unstable (P5)
- 0.8 < |z| < 1.25: h[n] = 2(0.8)^n u[n] + 3(1.25)^n u[-n-1] — noncausal, stable (P7)
- |z| < 0.8: h[n] = (-2(0.8)^n + 3(1.25)^n) u[-n-1] — noncausal, unstable (P6)

The stable case is the annular ROC that includes the unit circle.

**Full worked example (Example 5.9): X(z) = (4-2z^{-1}+z^{-2})/(1-1.5z^{-1}+0.5z^{-2}), |z|>1**

M = N = 2 (Case 2). Poles: z = 0.5, z = 1. Long division gives B_0 = 2.

X(z) = 2 + A_1/(1-0.5z^{-1}) + A_2/(1-z^{-1})

A_1 = -4, A_2 = 6. With |z| > 1: x[n] = 2delta[n] - 4(0.5)^n u[n] + 6u[n].

**Worked example (Example 5.10): Case 3 — second-order pole**

X(z) = 4 / [(1+z^{-1})(1-z^{-1})^2], one first-order pole at z=-1, one second-order pole at z=1.

X(z) = A_1/(1+z^{-1}) + C_1/(1-z^{-1}) + C_2/(1-z^{-1})^2

A_1 = 1, C_1 = 1 (using 5.28), C_2 = 2.

#### Method 3: Power Series Expansion

Expand X(z) directly as a power series in z^{-1}:

    X(z) = ... + x[-1]z^1 + x[0] + x[1]z^{-1} + x[2]z^{-2} + ...   (5.30)

Read off x[n] as the coefficient of z^{-n}.

Useful series: 1/(1-lambda) = 1 + lambda + lambda^2 + ... for |lambda| < 1.

Example 5.13: X(z) = 1/(1-az^{-1}), |z| > |a| => expand with lambda = az^{-1}: x[n] = a^n u[n].

Example 5.14: X(z) = 1/(1-az^{-1}), |z| < |a| => rewrite as (-a^{-1}z)/(1-a^{-1}z): x[n] = -a^n u[-n-1].

Key point: The ROC determines which direction to expand. Different ROCs give different signals from the same X(z).

Example 5.12: X(z) = log(1+az^{-1}), |z|>|a|. Using log(1+lambda) = sum_{n=1}^{inf} (-1)^{n+1} lambda^n / n with lambda = az^{-1}:
x[n] = (-1)^{n+1} a^n / n * u[n-1].

#### Method 4: Cauchy Integral Theorem

Use the contour integral formula (5.20) directly with residue theorem. Rarely done by hand in exam context.

---

### 6. Properties of the z-Transform

#### Property 1: Linearity (5.31)

    a x_1[n] + b x_2[n]  <->  a X_1(z) + b X_2(z)

ROC includes R_{x1} ∩ R_{x2} (intersection of the individual ROCs). It may be larger if pole-zero cancellation occurs.

Example 5.15: y[n] = (0.2)^n u[n] + (-0.3)^n u[n] => Y(z) = 1/(1-0.2z^{-1}) + 1/(1+0.3z^{-1}), |z| > 0.3 (the larger of the two inner radii).

#### Property 2: Time Shifting (5.32)

    x[n - n_0]  <->  z^{-n_0} X(z)

ROC same as X(z) except possibly z = 0 or z = inf are added/removed.

Example 5.16: x[n] = a^{n-1} u[n-1]. Using time-shift with n_0=1 on a^n u[n] <-> 1/(1-az^{-1}):
X(z) = z^{-1} * 1/(1-az^{-1}) = z^{-1}/(1-az^{-1}), |z| > |a|.

#### Property 3: Modulation / Multiplication by Exponential (5.33)

    z_0^n x[n]  <->  X(z/z_0)

If ROC of x[n] is R_+ < |z| < R_-, then ROC of z_0^n x[n] is |z_0|R_+ < |z| < |z_0|R_-.

Application (Example 5.17): Derive a^n cos(bn) u[n] from u[n] <-> 1/(1-z^{-1}).
Write cos(bn) = (e^{jbn}+e^{-jbn})/2, apply modulation with z_0 = ae^{jb} and z_0 = ae^{-jb}, then add via linearity to obtain:

    a^n cos(bn) u[n]  <->  (1 - a cos(b) z^{-1}) / (1 - 2a cos(b) z^{-1} + a^2 z^{-2}),   |z| > |a|

(Agrees with Table 5.1.)

#### Property 4: Differentiation (5.34)

    n x[n]  <->  -z dX(z)/dz

ROC same as X(z), except possibly z = 0 or z = inf.

Example 5.18: n a^n u[n] <-> -z * d/dz [1/(1-az^{-1})] = az^{-1}/(1-az^{-1})^2, |z| > |a|. (Agrees with Table 5.1.)

#### Property 5: Conjugation (5.35)

    x*[n]  <->  X*(z*)

ROC identical to that of X(z).

#### Property 6: Time Reversal (5.36)

    x[-n]  <->  X(z^{-1})

If ROC of x[n] is R_+ < |z| < R_-, then ROC of x[-n] is 1/R_- < |z| < 1/R_+.

Example 5.19: x[n] = -na^{-n} u[-n]. Using na^n u[n] <-> az^{-1}/(1-az^{-1})^2 and time reversal:
X(z) = az/(1-az)^2 = a^{-1}z^{-1}/(1-a^{-1}z^{-1})^2, |z| < |a^{-1}|.

#### Property 7: Convolution (5.37)

    x_1[n] * x_2[n]  <->  X_1(z) X_2(z)

ROC includes R_{x1} ∩ R_{x2}.

**Proof sketch**: Y(z) = sum_n [sum_k x_1[k] x_2[n-k]] z^{-n} = sum_k x_1[k] [sum_n x_2[n-k] z^{-n}] = sum_k x_1[k] X_2(z) z^{-k} = X_1(z) X_2(z). (5.39)

---

### 7. Transfer Function of LTI Systems

#### 7.1 Definition

Starting from the general difference equation:

    sum_{k=0}^{N} a_k y[n-k] = sum_{k=0}^{M} b_k x[n-k]   (5.40)

Applying z-transform with linearity and time-shift properties:

    Y(z) sum_{k=0}^{N} a_k z^{-k} = X(z) sum_{k=0}^{M} b_k z^{-k}   (5.41)

The **transfer function** (system function):

    H(z) = Y(z)/X(z) = sum_{k=0}^{M} b_k z^{-k} / sum_{k=0}^{N} a_k z^{-k}   (5.42)

The impulse response h[n] is the inverse z-transform of H(z) with an appropriate ROC.

**Workflow**: y[n] = x[n] * h[n] in time domain <=> Y(z) = X(z) H(z) in z-domain.

#### 7.2 Causality and stability from poles and ROC

From Example 5.8 analysis (poles at 0.8 and 1.25):

| ROC | h[n] | Causal? | Stable? |
|---|---|---|---|
| |z| > 1.25 | (2(0.8)^n - 3(1.25)^n) u[n] | Yes | No (pole outside unit circle) |
| 0.8 < |z| < 1.25 | 2(0.8)^n u[n] + 3(1.25)^n u[-n-1] | No | Yes (unit circle in ROC) |
| |z| < 0.8 | (-2(0.8)^n + 3(1.25)^n) u[-n-1] | No | No |

**Rules**:
- **Causal** system: h[n] = 0 for n < 0 <=> ROC is exterior of circle beyond the outermost pole (right-sided, P5).
- **Stable** system: sum |h[n]| < inf <=> ROC includes the unit circle (P2/BIBO stability).
- **Causal and stable**: all poles must be inside the unit circle (|c_k| < 1 for all k) so that the ROC |z| > max|c_k| includes the unit circle.

#### 7.3 Worked examples with difference equations

**Example 5.20**: y[n] = 0.1y[n-1] + x[n] + x[n-1]

Applying z-transform: Y(z)(1 - 0.1z^{-1}) = X(z)(1 + z^{-1})
=> H(z) = (1 + z^{-1}) / (1 - 0.1z^{-1}), one pole at z = 0.1, one zero at z = -1.
Two ROC possibilities: |z| > 0.1 (causal) or |z| < 0.1 (anti-causal).

**Example 5.21**: Given H(z) = (1+z^{-1})(1-2z^{-1}) / [(1-0.5z^{-1})(1+2z^{-1})], find difference equation.
Cross-multiply: (1+1.5z^{-1}-z^{-2}) Y(z) = (1-z^{-1}-2z^{-2}) X(z)
=> y[n] + 1.5y[n-1] - y[n-2] = x[n] - x[n-1] - 2x[n-2].
Shows that H(z) and the difference equation are equivalent representations.

**Example 5.22**: y[n] = x[n] - x[n-1]
Applying z-transform: Y(z) = X(z)(1 - z^{-1}) => H(z) = 1 - z^{-1}, |z| > 0.
h[n] = delta[n] - delta[n-1] (finite-duration causal).

**Example 5.23**: x[n] = u[n], h[n] = delta[n] + 0.5 delta[n-1].
X(z) = 1/(1-z^{-1}), H(z) = 1 + 0.5z^{-1}.
Y(z) = H(z)X(z) = 1/(1-z^{-1}) + 0.5 z^{-1}/(1-z^{-1}), |z| > 1.
y[n] = u[n] + 0.5 u[n-1].

---

## Key terms (glossary)

- **z-transform** — bilateral power series X(z) = sum x[n] z^{-n}; maps a discrete-time sequence to a function of complex variable z.
- **Region of Convergence (ROC)** — set of z for which X(z) converges absolutely; must be stated with X(z) to uniquely specify x[n].
- **Unit circle** — set {z : |z| = 1}; z-transform on the unit circle equals the DTFT.
- **Pole** — value of z where X(z) = inf (root of denominator polynomial).
- **Zero** — value of z where X(z) = 0 (root of numerator polynomial).
- **Pole-zero plot** — diagram of the z-plane showing poles (x) and zeros (o) with the ROC shaded.
- **Right-sided sequence** — x[n] = 0 for n < N_1 (some finite N_1); ROC is exterior of a circle.
- **Left-sided sequence** — x[n] = 0 for n > N_2 (some finite N_2); ROC is interior of a circle.
- **Two-sided (bilateral) sequence** — nonzero on both sides; ROC is an annulus.
- **Causal system** — h[n] = 0 for n < 0; H(z) has ROC of the form |z| > r.
- **BIBO stable system** — ROC of H(z) includes the unit circle; equivalently, all poles inside unit circle for a causal system.
- **Transfer function H(z)** — ratio H(z) = Y(z)/X(z); z-transform equivalent of the impulse response h[n].
- **Partial fraction expansion** — decomposition of a rational X(z) into simpler fractions whose inverse z-transforms are known.
- **Power series expansion** — reading x[n] as coefficients of z^{-n} in the expanded series.
- **DTFT** — X(e^{jw}): z-transform evaluated on the unit circle (exists iff unit circle is in ROC).
- **Modulation property** — multiplying by z_0^n in time scales z by 1/z_0 in z-domain.

---

## Exam targets

1. **Compute the z-transform and state ROC** for standard sequences (a^n u[n], -a^n u[-n-1], finite sequences). Show all convergence steps using the ratio test.

2. **Determine ROC type from the signal type**: right-sided => exterior disk; left-sided => interior disk; finite => whole plane (minus z=0 and/or z=inf); two-sided => annulus.

3. **Apply the 8 ROC properties**: given poles at specified magnitudes, enumerate all possible ROCs and state which signal type (causal/left-sided/two-sided) each corresponds to (see Example 5.6, Fig 5.7).

4. **Inverse z-transform — all 4 methods**:
   - Inspection: rewrite in standard form, read from table.
   - Partial fractions Case 1 (M<N, distinct poles): compute A_k from (5.24), invert.
   - Partial fractions Case 2 (M>=N): do long division first to extract B_l, then compute A_k.
   - Partial fractions Case 3 (multiple poles): use (5.28) for C_m coefficients.
   - Power series: long divide or expand, read coefficients. Know how ROC determines expansion direction.

5. **Apply z-transform properties** to efficiently compute transforms: linearity (ROC intersection), time-shift (multiply by z^{-n0}), modulation/differentiation for deriving cosine/ramp transforms.

6. **Transfer function and difference equation**:
   - Apply z-transform to a difference equation to get H(z).
   - Given H(z), recover the difference equation by cross-multiplication and inverse transform.
   - Find h[n] by inverse z-transform of H(z) with the correct ROC.

7. **Causality and stability from H(z)**:
   - Causal and stable <=> all poles strictly inside unit circle.
   - Stable but not causal <=> unit circle is in ROC but ROC is not the exterior.
   - Given multiple ROC scenarios (as in Example 5.8), classify each as causal/noncausal and stable/unstable.

8. **DTFT existence**: X(e^{jw}) exists if and only if the ROC of the z-transform contains the unit circle. For a^n u[n]: need |a| < 1. For -a^n u[-n-1]: need |a| > 1.

---

## Pitfalls

- **Same X(z), different x[n]**: a^n u[n] and -a^n u[-n-1] both give z/(z-a) but with opposite ROCs. Omitting the ROC makes the answer wrong.

- **ROC cannot contain poles**: when determining ROC from a rational X(z), the ROC boundary is determined by pole magnitudes — poles are excluded.

- **Linearity ROC is the intersection**: when adding two z-transforms, the combined ROC is R_{x1} ∩ R_{x2}, which may be smaller than either individual ROC. If their intersection is empty, the sum's z-transform does not exist. (Example 5.15: |z|>0.2 intersected with |z|>0.3 gives |z|>0.3, not |z|>0.2.)

- **Time-shift can add/delete z=0 or z=inf**: shifting by n_0 > 0 introduces z^{-n_0} which causes a pole at z=0 (removes z=0 from ROC). Shifting by n_0 < 0 introduces z^{|n_0|} which causes a pole at z=inf.

- **Partial fractions require M < N for Case 1**: if M >= N, you must do long division first to extract the polynomial part B_l, then apply PFE to the remainder.

- **Power series ROC direction matters**: for |z| > |a|, expand in powers of az^{-1} (lambda = az^{-1}, |lambda|<1). For |z| < |a|, rewrite and expand in powers of a^{-1}z (|a^{-1}z|<1). Getting this wrong gives a divergent series.

- **Causal system does NOT automatically mean stable**: a causal system (ROC = exterior disk) is stable only if the exterior disk includes the unit circle, i.e. all poles must lie inside the unit circle. A pole at |z| = 1.25 with a causal ROC |z| > 1.25 does NOT include the unit circle — system is unstable.

- **Multiple ROC possibilities mean multiple valid signals**: when ROC is unspecified, you must consider all possible connected regions between consecutive pole magnitudes. Each gives a different h[n] with different causal/stable properties (as shown in Example 5.8 with |z|>1.25, 0.8<|z|<1.25, |z|<0.8).

- **Finite-duration sequences have no convergence issue** (only z=0 and/or z=inf are excluded). Do not apply infinite-sum convergence tests to them.

- **The unit step u[n] has a pole at z=1**: its ROC is |z|>1, so the unit circle is NOT in the ROC (it's the boundary). The DTFT of u[n] technically requires distribution theory.
