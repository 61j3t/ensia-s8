# Part 6 — Image Filtering in the Frequency Domain

## Bird's eye view

- The **frequency domain** describes how fast pixel intensities change across space: low frequencies correspond to smooth, slowly varying regions (sky, uniform areas); high frequencies correspond to edges, fine textures, and noise.
- Any image can be decomposed into a sum of sinusoids of different frequencies — the **Fourier transform** gives the amplitude and phase of each sinusoidal component.
- The **2-D Discrete Fourier Transform (DFT)** is the practical tool applied to MxN images; its fast implementation (FFT) makes frequency-domain filtering computationally efficient.
- The **Convolution Theorem** is the central reason to work in the frequency domain: spatial convolution becomes pointwise multiplication in the frequency domain — drastically cheaper for large kernels.
- Frequency-domain filtering follows a fixed 5-step pipeline: DFT the image, design a filter H(u,v), multiply, then apply the inverse DFT.
- Three families of **low-pass filters** (Ideal, Butterworth, Gaussian) trade off sharpness of cut-off against ringing artifacts; each has a complementary high-pass version.
- The **phase** of the DFT carries most of the structural/spatial information about an image; the amplitude mainly controls intensities. Discarding phase destroys recognisable content; discarding amplitude while preserving phase still yields a recognisable image outline.
- The Fourier spectrum encodes **orientation**: structures in the image map to spectral energy perpendicular to that orientation, and the DC coefficient F[0,0] equals the sum of all pixel values (mean brightness scaled by MN).

---

## Detailed notes

### 1. Motivation and intuition

**Why work in the frequency domain?**

| Reason | Explanation |
|---|---|
| Filtering is cheaper | Spatial convolution is O(MN * kernel_size^2). In frequency domain, filtering = pointwise multiplication after a single DFT. |
| Signal/noise separation | Noise concentrates at high frequencies; structures concentrate at low frequencies. |
| Convolution theorem | f(x) * h(x) in space <-> F(u) x H(u) in frequency. |
| Analysis | Can directly design and visualise filter shapes. |

**Intuitive frequency interpretation:**
- **Low frequency** = smooth regions: sky, large uniform areas, slowly varying intensity.
- **High frequency** = rapid changes: edges, fine textures, noise.
- **Smoothing** = removing high frequencies (low-pass filtering).
- **Sharpening** = amplifying high frequencies (high-pass filtering).

**Key idea:** Any image can be represented as a sum of simple sinusoidal wave patterns of different frequencies. The Fourier transform decomposes f into these components; the inverse reassembles them.

---

### 2. Sinusoids — building blocks

A 1-D sinusoid:

$$f(x) = A \sin(2\pi u x + \phi)$$

where:
- x: spatial position
- A: amplitude (peak value)
- T: period (width of one complete cycle)
- u: frequency = 1/T (cycles per unit length, in Hz for time signals)
- $\phi$: phase (shift in position, in radians)

A sinusoid is fully defined by: (1) amplitude, (2) frequency, (3) phase.

**Euler's formula (the bridge):**

$$e^{i\theta} = \cos(\theta) + i\sin(\theta)$$

This is verified by Taylor expansion: the even terms give cos, odd terms give sin. This allows complex exponentials to encode both cos and sin simultaneously, which is why the Fourier transform produces complex numbers.

**Consequence:** A real cosine decomposes into two complex exponentials:

$$\cos(2\pi k x) = \frac{1}{2}\left(e^{i2\pi kx} + e^{-i2\pi kx}\right)$$

So a single cosine of frequency k contributes spectral energy at both +k and -k. This is why DFT spectra are symmetric.

---

### 3. The 1-D Fourier Transform

**Forward transform (continuous):**

$$F(u) = \int_{-\infty}^{+\infty} f(x)\, e^{-i2\pi u x}\, dx$$

**Inverse transform (continuous):**

$$f(x) = \int_{-\infty}^{+\infty} F(u)\, e^{+i2\pi u x}\, du$$

**Key properties of F(u):**
- F(u) is complex-valued: $F(u) = \text{Re}\{F(u)\} + i\,\text{Im}\{F(u)\}$
- Each F(u) encodes the amplitude and phase of the sinusoid of frequency u.

**Extracting amplitude and phase:**

$$A(u) = \sqrt{\text{Re}\{F(u)\}^2 + \text{Im}\{F(u)\}^2}$$

$$\phi(u) = \text{atan2}\!\left(\text{Im}\{F(u)\},\, \text{Re}\{F(u)\}\right)$$

Note: atan2 (not atan) is used because it preserves the correct quadrant. atan(y/x) gives the same result for (1,1) and (-1,-1), but atan2(1,1) = 45 deg and atan2(-1,-1) = 225 deg, which is the correct distinction.

**Representative FT pairs (1-D):**

| Signal f(x) | Fourier Transform F(u) |
|---|---|
| $\cos(2\pi k x)$ | $(1/2)[\delta(u+k) + \delta(u-k)]$ — two spikes at +/-k in Re part |
| $\sin(2\pi k x)$ | $(1/2i)[\delta(u+k) - \delta(u-k)]$ — two spikes in Im part, opposite signs |
| Gaussian $e^{-ax^2}$ | Gaussian $\sqrt{\pi/a}\, e^{-\pi^2 u^2 / a}$ — Gaussian maps to Gaussian |
| Sum of two cosines at k1, k2 | Four spikes at +/-k1, +/-k2 |

The Gaussian self-similarity under FT is a crucial property: it means Gaussian filtering in space corresponds to Gaussian attenuation in the frequency domain, with no ringing.

---

### 4. The Convolution Theorem

**1-D convolution definition:**

$$g(x) = f(x) * h(x) = \int_{-\infty}^{+\infty} f(\tau)\, h(x - \tau)\, d\tau$$

**Convolution theorem (derivation sketch):**
Computing the FT of g(x) and expanding:

$$G(u) = \int g(x)\, e^{-i2\pi ux}\, dx$$

$$= \int\!\int f(\tau)\, h(x-\tau)\, e^{-i2\pi ux}\, d\tau\, dx$$

$$= \left[\int f(\tau)\, e^{-i2\pi u\tau}\, d\tau\right] \times \left[\int h(x-\tau)\, e^{-i2\pi u(x-\tau)}\, dx\right]$$

$$= F(u) \times H(u)$$

**Result — the duality table:**

| Spatial Domain | Frequency Domain |
|---|---|
| $g(x) = f(x) * h(x)$ (convolution) | $G(u) = F(u) \times H(u)$ (multiplication) |
| $g(x) = f(x) \times h(x)$ (multiplication) | $G(u) = F(u) * H(u)$ (convolution) |

**Practical workflow using the theorem:**

Steps: $\mathcal{F}[f] \to F(u)$, $\mathcal{F}[h] \to H(u)$, multiply pointwise $\to G(u)$, $\mathcal{F}^{-1} \to g(x)$

This is more efficient than direct spatial convolution for large kernels.

**Example: Gaussian smoothing via convolution theorem**
A noisy signal f(x) has a broad frequency spectrum |F(u)| with significant high-frequency content. A Gaussian kernel $h_\sigma(x)$ has a narrow Gaussian spectrum $|N_\sigma(u)|$ centered at 0. Multiplying $F(u) \times N_\sigma(u)$ suppresses the high-frequency noise components and leaves low-frequency structure. Taking the IFT gives the smoothed signal.

---

### 5. The 2-D Fourier Transform for Images

**Continuous 2-D forward transform:**

$$F(u,v) = \int_{-\infty}^{+\infty}\!\int_{-\infty}^{+\infty} f(x,y)\, e^{-i2\pi(ux + vy)}\, dx\, dy$$

**Continuous 2-D inverse transform:**

$$f(x,y) = \int_{-\infty}^{+\infty}\!\int_{-\infty}^{+\infty} F(u,v)\, e^{+i2\pi(ux + vy)}\, du\, dv$$

where u is the frequency along x and v is the frequency along y.

---

### 6. The 2-D Discrete Fourier Transform (DFT) — the practical formula

For an image f[m,n] of size M x N (m = 0...M-1 rows, n = 0...N-1 columns):

**Forward DFT:**

$$F[p,q] = \sum_{m=0}^{M-1} \sum_{n=0}^{N-1} f[m,n]\, e^{-i2\pi pm/M}\, e^{-i2\pi qn/N}$$

for $p = 0\ldots M-1$ and $q = 0\ldots N-1$

**Inverse DFT:**

$$f[m,n] = \frac{1}{MN} \sum_{p=0}^{M-1} \sum_{q=0}^{N-1} F[p,q]\, e^{+i2\pi pm/M}\, e^{+i2\pi qn/N}$$

for $m = 0\ldots M-1$ and $n = 0\ldots N-1$

**Why divide by MN in the inverse?**
The DFT sums contributions from all MN pixels, so coefficients are MN times larger than the original values. The IDFT divides by MN to correctly reconstruct the original image.

**Three normalization conventions (all mathematically equivalent):**

| Convention | Forward | Inverse |
|---|---|---|
| 1 (MATLAB/NumPy default) | F = DFT(f) | $f = \frac{1}{MN}$ IDFT(F) |
| 2 | $F = \frac{1}{MN}$ DFT(f) | f = IDFT(F) |
| 3 (symmetric) | $F = \frac{1}{\sqrt{MN}}$ DFT(f) | $f = \frac{1}{\sqrt{MN}}$ IDFT(F) |

They only differ in where the normalization factor is placed; the pair always stays consistent.

---

### 7. The Frequency Spectrum of an Image

**What F[p,q] means:**
- p = frequency along m (rows), q = frequency along n (columns).
- The magnitude spectrum is typically displayed as $\log(|F[p,q]|)$ to compress the large dynamic range.
- The spectrum is visualized with the DC component at the center (after fftshift).

**The DC coefficient:**

$$F[0,0] = \sum_{m=0}^{M-1} \sum_{n=0}^{N-1} f[m,n]$$

This is MN times the mean brightness of the image. It represents the average intensity (zero-frequency component).

**Spectrum orientation rule:**

| Image content | Spectrum pattern |
|---|---|
| Vertical stripes (vary along x) | Energy on horizontal frequency axis (q = 0) |
| Horizontal stripes (vary along y) | Energy on vertical frequency axis (p = 0) |
| Diagonal stripes | Energy on diagonal axis |
| Isotropic (e.g. circle) | Circularly symmetric spectrum, energy radiating outward from center |

Key rule: **orientation in image is perpendicular to orientation of energy in spectrum.**

**Example spectra:**
- A vertical sinusoidal grating $f(x,y) = \cos(2\pi u_0 x)$ gives exactly three bright points on the horizontal axis: the DC at (0,0), plus symmetric peaks at $(+u_0, 0)$ and $(-u_0, 0)$.
- A higher-frequency grating gives the same pattern but with the $\pm u_0$ peaks farther from center.
- A Rubik's cube (strong horizontal and vertical edges) gives an X-shaped spectrum with bright lines along both axes.
- A baboon face image (complex texture, no dominant direction) gives a roughly isotropic spectrum concentrated near the center.
- A pure noise image gives a nearly uniform flat spectrum — energy spread across all frequencies equally.

---

### 8. Frequency-Domain Filtering: The Step-by-Step Procedure

**5-step pipeline:**

1. Compute F[p,q] = DFT of input image f[m,n].
2. Design filter transfer function H[p,q] in the frequency domain.
3. Multiply: $G[p,q] = F[p,q] \times H[p,q]$ (pointwise).
4. Compute g[m,n] = IDFT of G[p,q].
5. Take the real part of g (small imaginary parts arise from numerical errors).

**What the filter does:**
- H(u,v) that passes low frequencies and blocks high = **low-pass filter** → blurs the image (smoothing).
- H(u,v) that blocks low frequencies and passes high = **high-pass filter** → enhances edges, reduces contrast.

The distance from the center of the spectrum to a point (u,v) is:

$$D(u,v) = \sqrt{u^2 + v^2}$$

All filter transfer functions H(u,v) depend on D(u,v) and a cutoff frequency $D_0$.

---

### 9. Low-Pass Filters

#### 9.1 Ideal Low-Pass Filter (ILPF)

$$H(u,v) = \begin{cases} 1 & \text{if } D(u,v) \leq D_0 \\\\ 0 & \text{if } D(u,v) > D_0 \end{cases}$$

Shape: a flat disk of radius $D_0$ centered at the origin in the frequency domain. Everything inside the disk passes; everything outside is blocked.

**$D_0$ is the cut-off frequency.** The point where H transitions from 1 to 0.

**Visual appearance of ILPF in spectrum:** a solid white circle (all values 1) on a black background (all values 0).

**Effect on image:**
- Larger $D_0$ (e.g. 60): mild blurring, fine detail removed, image still recognisable.
- Smaller $D_0$ (e.g. 30): strong blurring, significant loss of detail, severe ringing artifacts visible as concentric ripples around edges.

**Ringing artifact (Gibbs phenomenon):** The abrupt brick-wall cutoff in the frequency domain corresponds to a sinc function in the spatial domain. The sinc's oscillatory tails cause ringing (bright and dark fringes) around sharp edges in the filtered image. This is the ILPF's major problem.

#### 9.2 Butterworth Low-Pass Filter (BLPF)

$$H(u,v) = \frac{1}{1 + \left[\dfrac{D(u,v)}{D_0}\right]^{2n}}$$

where:
- n is the **order** of the filter (controls steepness of the roll-off).
- $D_0$ is the cutoff frequency (at $D(u,v) = D_0$, $H = 0.5$ regardless of n).

**Profile shape:** At n=1 the roll-off is very gradual (slow transition). As n increases, the profile becomes steeper, approaching the ideal brick-wall at $n \to \infty$. The filter in the frequency domain looks like a soft glowing blob (bright center fading toward edges), not a hard disk.

**Why BLPF is better than ILPF:** The gradual transition eliminates ringing artifacts because the corresponding spatial kernel decays smoothly without oscillatory tails.

**Example results:**
- n=2, $D_0$=60: smooth blurring, no ringing, readable text still visible.
- n=2, $D_0$=30: stronger blurring, no ringing, detail lost but no ripples.

#### 9.3 Gaussian Low-Pass Filter (GLPF)

$$H(u,v) = e^{-D(u,v)^2 / (2D_0^2)}$$

The filter profile is a 2-D Gaussian centered at the origin. It decays smoothly to zero with no abrupt cutoff.

**Key property:** The FT of a Gaussian is a Gaussian. Therefore, Gaussian filtering in the spatial domain and Gaussian filtering in the frequency domain are the same operation — there is no ringing whatsoever, and the filter kernel in the spatial domain is always non-negative.

**Visual appearance in spectrum:** A smooth circular bright blob centered at origin.

**Effect on image:** Progressive blurring as $\sigma$ increases, with perfectly smooth results and no artifacts.

**Comparison: ILPF vs BLPF vs GLPF at same cutoff**
- ILPF: sharpest effective cutoff, worst ringing.
- BLPF order n=2: no ringing, smoother result than ILPF.
- GLPF: no ringing, typically the smoothest and most natural result.

---

### 10. High-Pass Filters

High-pass = complement of low-pass: passes high frequencies, blocks low frequencies. Enhances edges and sharp detail, but reduces overall contrast (removes the DC component, so the mean brightness drops).

#### 10.1 Ideal High-Pass Filter (IHPF)

$$H(u,v) = \begin{cases} 0 & \text{if } D(u,v) \leq D_0 \\\\ 1 & \text{if } D(u,v) > D_0 \end{cases}$$

The exact complement of the ILPF: a black disk on a white background in the frequency domain.

**Effect on image:**
- $D_0$=60: edges and outlines visible, dark background, moderate ringing.
- $D_0$=30: stronger edge enhancement, more ringing, overall dark image (DC removed).

Ringing is again the major problem, for the same reason as in ILPF.

#### 10.2 Butterworth High-Pass Filter (BHPF)

$$H(u,v) = \frac{1}{1 + \left[\dfrac{D_0}{D(u,v)}\right]^{2n}}$$

Note the ratio is inverted compared to the BLPF: $D_0/D(u,v)$ instead of $D(u,v)/D_0$. At $D(u,v) = D_0$, $H = 0.5$. For $D(u,v) \gg D_0$, $H \to 1$ (high frequencies pass). For $D(u,v) \ll D_0$, $H \to 0$ (low frequencies blocked).

**Effect:** Smooth edge enhancement, no ringing artifacts, degree of sharpening controlled by n and $D_0$.

**Example results:**
- $D_0$=60: fine edge lines preserved, background dark, no ringing.
- $D_0$=30: broader range of edges visible, more detail in the result.

#### 10.3 Gaussian High-Pass Filter (GHPF)

$$H(u,v) = 1 - e^{-D(u,v)^2 / (2D_0^2)}$$

Complement of the GLPF: $H_\text{HPF} = 1 - H_\text{LPF}$. At $D=0$, $H=0$ (DC blocked). As D increases, $H$ approaches 1. No ringing.

---

### 11. The Role of Phase vs Amplitude

**Filtering so far has only modified the magnitude $|F(u,v)|$.** The phase carries separate and critical information.

**Interpretation:**
- **Amplitude** of each sinusoid determines intensity values in the image.
- **Phase** is the displacement of each sinusoid from its origin — it encodes where discernable objects and structures are located.

**Empirical demonstration (Curtis, Lim, Oppenheim 1984):**

| Experiment | Result |
|---|---|
| Apply DFT; set all phases to 0; preserve magnitude; IDFT | Reconstructed image looks like blurry, unrecognisable texture — location information is completely lost. |
| Apply DFT; preserve phase; use average magnitude from many images; IDFT | Reconstructed image is still recognisable — the subject's face structure remains. |
| Reconstruct using phase from one image + amplitude from a completely different image (e.g. white bar) | Reconstructed image strongly resembles the source of the phase, not the amplitude. |
| Reconstruct using phase only (set amplitude = constant 1) | Image shows outline/edge-like face structure — recognisable. |
| Reconstruct using amplitude only (set phase = 0) | Unrecognisable noisy pattern. |

**Conclusion: Phase carries more structural/semantic information than amplitude. Filtering operations that modify the phase can destroy the image in ways that amplitude modification does not.**

---

### 12. Worked Example: Gaussian Filtering Pipeline

**Spatial domain:**
Image $f(m,n)$ convolved with Gaussian kernel $h_\sigma(m,n)$ gives smoothed output $g(m,n)$.

**Frequency domain equivalent:**
1. DFT: $F(p,q) = \mathcal{F}[f(m,n)]$.
2. DFT of Gaussian: $N_\sigma(p,q)$ = Gaussian spectrum (also Gaussian in frequency).
3. Multiply: $G(p,q) = F(p,q) \times N_\sigma(p,q)$.
4. IDFT: $g(m,n) = \mathcal{F}^{-1}[G(p,q)]$.

**Key observation:** The Gaussian kernel has a very narrow FT profile when $\sigma$ is small. Multiplying by this profile suppresses all high-frequency components, leaving only the low-frequency bulk of $F(p,q)$. Larger $\sigma$ in spatial domain = broader Gaussian kernel = narrower FT profile = more aggressive high-frequency suppression = stronger blurring.

---

## Key terms (glossary)

| Term | Definition |
|---|---|
| **Frequency domain** | A representation of a signal in terms of the rates of change (frequencies) of its values, rather than the values themselves. |
| **Fourier Transform (FT)** | Operation mapping a spatial signal f(x) to its frequency representation F(u); each F(u) encodes amplitude and phase of the frequency-u sinusoid. |
| **Inverse Fourier Transform (IFT)** | Operation recovering f(x) from F(u). |
| **DFT** | Discrete Fourier Transform — the sampled, finite version of FT used on digital images. |
| **FFT** | Fast Fourier Transform — efficient $O(MN \log MN)$ algorithm to compute the DFT. Discovered early 1960s. |
| **Amplitude spectrum** | $|F(u,v)|$ — the magnitude of the complex DFT coefficients; typically displayed after log compression. |
| **Phase spectrum** | $\angle F(u,v)$ — the phase angle of each DFT coefficient; encodes spatial positions of structures. |
| **DC coefficient** | F[0,0] = sum of all pixel values; represents the mean brightness scaled by MN. |
| **Frequency (spatial)** | Rate at which intensity varies across the image. Low = smooth; High = edges/noise. |
| **Cut-off frequency $D_0$** | The frequency at which a filter transitions between passing and blocking; determines the spatial scale of the filtering effect. |
| **Transfer function H(u,v)** | The frequency-domain representation of a filter; applied by pointwise multiplication with F(u,v). |
| **Low-pass filter (LPF)** | Passes low frequencies, blocks high frequencies; produces smoothing/blurring. |
| **High-pass filter (HPF)** | Passes high frequencies, blocks low frequencies; produces edge enhancement, reduces contrast. |
| **ILPF / IHPF** | Ideal Low/High-Pass Filter — binary step function; causes ringing (Gibbs phenomenon). |
| **BLPF / BHPF** | Butterworth Low/High-Pass Filter — smooth roll-off parameterized by order n; no ringing. |
| **GLPF / GHPF** | Gaussian Low/High-Pass Filter — Gaussian profile; no ringing, maps to spatial Gaussian kernel. |
| **Ringing artifact** | Oscillatory fringes appearing around sharp edges after ILPF/IHPF filtering; caused by the sinc-shaped spatial kernel of the ideal filter. |
| **Convolution theorem** | $f * h$ in space $\leftrightarrow$ $F \times H$ in frequency; multiplication in frequency domain is equivalent to convolution in spatial domain. |
| **D(u,v)** | Euclidean distance from spectral origin: $D(u,v) = \sqrt{u^2 + v^2}$; all filter transfer functions depend on this. |
| **Butterworth order n** | Controls roll-off steepness; n=1 is very gradual; $n\to\infty$ approaches ideal brick-wall. |

---

## Exam targets

1. **Write the 2-D DFT forward and inverse formulas** and identify every symbol (f[m,n], F[p,q], M, N, p, q, the exponentials). State why the IDFT divides by MN.

2. **State and prove (or sketch the proof of) the Convolution Theorem.** Show the derivation $G(u) = F(u) H(u)$ starting from the definition of $g(x) = f(x)*h(x)$ and taking its FT.

3. **Draw and describe the 5-step frequency-domain filtering pipeline**: DFT -> design H(u,v) -> pointwise multiply -> IDFT -> take real part.

4. **Write H(u,v) for all six filters** (ILPF, BLPF, GLPF, IHPF, BHPF, GHPF) and sketch their radial profiles. Identify which cause ringing and why.

5. **Explain the ringing/Gibbs phenomenon**: the brick-wall ILPF has a sinc-shaped spatial kernel with oscillatory tails → ringing around edges. BLPF and GLPF avoid this with smooth roll-off.

6. **Interpret spectrum figures**: given a striped image, sketch where spectral energy appears. Given a spectrum, describe the likely image content. State the orientation perpendicularity rule.

7. **Explain what DC coefficient F[0,0] represents** and compute it symbolically.

8. **Compare ILPF/BLPF/GLPF** at same cutoff $D_0$: trade-off between sharpness of attenuation and ringing.

9. **Explain amplitude vs phase**: which carries intensity information, which carries structural/positional information, and what happens to an image when each is discarded (with reference to the Curtis-Lim-Oppenheim experiment).

10. **State the three DFT normalization conventions** and explain why they are all valid.

---

## Pitfalls

- **ILPF does NOT smooth cleanly** — it causes ringing. Do not confuse it with Gaussian smoothing, which is artifact-free.
- **The spectrum origin (DC) is at position [0,0] by default**, not at the center. The fftshift operation moves it to the center for display purposes. Always clarify which convention you are using.
- **The DFT of a real image is NOT real** — it is complex, and both magnitude and phase must be carried through the computation. Only take the real part of the IDFT result (imaginary parts are numerical artifacts).
- **Orientation in image is perpendicular to orientation in spectrum**, not parallel. A vertical grating has spectral energy on the horizontal axis, not the vertical axis.
- **Larger Butterworth order n approaches the ideal filter** — increasing n reduces the smooth transition zone, and at high n ringing reappears. n=2 is a common safe choice.
- **The Gaussian is self-similar under FT**: a narrow spatial Gaussian gives a broad frequency spectrum (mild filtering); a wide spatial Gaussian gives a narrow frequency spectrum (strong filtering). This is the uncertainty principle in disguise — you cannot have narrow support in both domains simultaneously.
- **HPF reduces mean image brightness** because it zeroes (or attenuates) the DC coefficient F[0,0]. Do not confuse HPF output (dark background with bright edges) with the original image.
- **Phase is more important than amplitude** for preserving recognisable structure. A common exam trap is assuming the amplitude spectrum determines image content — this is false.
- **The normalization factor (1/MN) belongs in the inverse transform** in the MATLAB/NumPy convention. If you put it in the forward transform, the frequency-domain multiplication still works but your coefficient magnitudes will differ.
- **The FFT algorithm computes the DFT exactly** — it is not an approximation. It is simply a computationally efficient factorization of the DFT matrix.
