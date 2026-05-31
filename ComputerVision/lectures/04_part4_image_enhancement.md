# Part 4 — Image Enhancement: Intensity Transformations

## Bird's eye view

- All techniques in this part operate in the **spatial domain**: the output image $g(x,y) = T[f(x,y)]$ is obtained by applying an operator T directly to the pixel values of f.
- When the neighborhood of (x,y) shrinks to a single pixel (1x1), T becomes a **point operation** (intensity transformation): $s = T(r)$, where r is the input intensity and s the output intensity. This is the focus of this part.
- Six families of point operations are covered: **negative**, **gain/bias**, **log**, **power-law (gamma)**, **contrast stretching / thresholding**, and **histogram-based** methods.
- The **image histogram** $h(r_k) = n_k$ is the empirical distribution of intensities; it is used to diagnose image problems (too dark, too bright, low contrast) and to drive automatic enhancement.
- **Histogram equalization** is the key automatic method: it applies the CDF of the input histogram as the transformation function, producing a (quasi-)uniform output histogram and maximizing use of the dynamic range.
- **Otsu thresholding** finds the optimal binary threshold automatically by minimizing the weighted within-class variance of the two pixel classes.

---

## Detailed notes

### 1. The spatial-domain framework

A spatial-domain process is written:

$$g(x, y) = T[ f(x, y) ]$$
- $f(x,y)$: input image (intensity at pixel (x,y))
- $g(x,y)$: output image
- T: operator defined over a **neighborhood** of (x,y)

The neighborhood is typically **rectangular**, **centered on (x,y)**, and much smaller than the image. The origin is at the top-left; x increases downward, y increases rightward.

**Point operations** are the special case where the neighborhood has size **1x1**. Then g depends only on f at (x,y), and T reduces to:
$$s = T(r)$$
where r = f(x,y) and s = g(x,y) are scalar intensity values. All transforms in this part are of this form.

---

### 2. Basic point operations

#### 2.1 Image negative

For an image with intensity range [0, L-1]:
$$s = L - 1 - r$$
- Maps black (0) to white (L-1) and vice versa; linearly inverts the curve.
- Produces a photographic negative.
- **Use case:** enhances white or gray details embedded in dark regions (e.g. mammogram negatives, where dark lesions become light and are easier to inspect).

#### 2.2 Gain and bias (linear scaling)
$$s = \alpha \cdot r + \beta \quad (\alpha > 0)$$
- $\alpha$ = gain: controls **contrast**. $\alpha > 1$ increases contrast (steeper slope); $\alpha < 1$ reduces it.
- $\beta$ = bias: controls **brightness**. Positive $\beta$ shifts the image brighter; negative $\beta$ darker.
- Example: $\alpha = 2$ doubles contrast (image becomes harsher, darker shadows and brighter highlights). $\beta = 100$ adds 100 to every pixel (overall brightening, clipped at L-1).
- Output must be clipped to [0, L-1] to avoid overflow.

---

### 3. Log transformations
$$s = c \cdot \log(1 + r)$$
Variables:
- r: input pixel intensity (normalized, e.g. in [0, 1] or [0, 255])
- s: output pixel intensity
- c: scaling constant, chosen so that the maximum output equals the maximum allowed value ($S_{\max}$):
$$c = \frac{S_{\max}}{\log(1 + R_{\max})}$$
The **+1** avoids log(0).

**Shape of the curve:** concave (steep rise for low r, then flattens). Plotted on axes 0-255 by 0-255, the curve climbs rapidly from (0,0) and approaches 255 asymptotically.

**Effect on pixel populations:**
- Small input values (dark pixels) → **large output changes** (dark pixels stretched apart across a wide range)
- Large input values (bright pixels) → **small output changes** (bright pixels compressed into a narrow range)

**Interpretation:**
- Dark regions become brighter; subtle low-intensity variations become visible.
- Bright regions change very little.
- Log transform is used to **reveal hidden details** in dark areas at the cost of losing detail in bright areas.
- Formally: *appropriate when we want to enhance the low pixel values at the expense of loss of information in the high pixel values.*

**Applications:**
- Visualizing the Fourier spectrum: the spectrum has a very bright center (DC component) and very dark edges. Log transform makes the weak frequency components visible alongside the dominant ones. Without it, the image appears almost entirely black except for a bright central spot; after log transform, the full ring structure becomes clearly visible.
- Caution: if important details lie in the high pixel values (e.g. a bright-toned image), log transform will wash them out (everything becomes uniformly bright, losing detail).

---

### 4. Power-law (gamma) transformations
$$s = c \cdot r^{\gamma} \quad (c > 0,\ \gamma > 0)$$
Both c and $\gamma$ are positive constants.

**Family of curves (the classic gamma fan diagram):**
- The diagram plots output s (vertical axis, 0 to L-1) against input r (horizontal, 0 to L-1).
- When **$\gamma < 1$** (e.g. 0.04, 0.10, 0.20, 0.40, 0.67): curves bow **upward** (concave shape, like log). Dark inputs are mapped to higher outputs → image brightened, dark detail revealed.
- When **$\gamma = 1$** and **c = 1**: identity transform (straight diagonal line, no change).
- When **$\gamma > 1$** (e.g. 1.5, 2.5, 5.0, 10.0, 25.0): curves bow **downward** (convex shape, opposite of log). Bright inputs remain bright; dark inputs are pushed darker → image darkened, contrast in bright regions enhanced.

**Gamma correction for display devices:**

*Problem:* Displays (historically CRT, but also modern monitors and projectors) are **not linear**. Their intensity-to-voltage response is a power law:
$$L \propto v^{\gamma_{\text{display}}} \quad \text{with } \gamma_{\text{display}} \approx 2.2$$
This means:
- Sending pixel value 128 does NOT produce half the brightness of 255.
- Dark values are compressed (appear too dark).
- Bright values are expanded (disproportionately bright).
- This is a **hardware non-linearity**.

*Consequence:* If you send a linear image to such a display, it appears **darker than intended**. The display applies a power curve with $\gamma \approx 2.2$, compressing dark tones.

*Solution — gamma correction:*
- Apply a gamma correction (pre-distortion) to the input signal before sending it to the display.
- The correction $\gamma = 1 / \gamma_{\text{display}}$. For $\gamma_{\text{display}} = 2.5$, correction $\gamma = 1/2.5 = 0.4$.
- This is called the **Image Gamma**.
- Algorithms like JPEG automatically apply this encoding gamma at capture/conversion time.
- The gamma-corrected image combined with the display's gamma produces an overall linear response:
$$\gamma_{\text{correction}}\ (0.4) + \gamma_{\text{display}}\ (2.2) \approx \text{linear final output} \approx 1.0$$
*Practical example:* A gradient from black to white:
- Original linear image fed to a $\gamma_{\text{display}} = 2.5$ monitor → darker on screen.
- Pre-process with gamma correction ($\gamma = 0.4$) → the corrected image looks brighter, but when displayed, the combination is correct and appears as intended.

**Contrast manipulation with gamma:**
- A washed-out (low-contrast, overly bright) aerial image corrected with $\gamma = 3$, 4, or 5 shows progressively darker, higher-contrast outputs, with more visible ground details.
- $\gamma > 1$ compresses bright values → useful for images that are overexposed or washed out.
- $\gamma < 1$ boosts dark values → useful for underexposed images.

---

### 5. Contrast stretching and piecewise-linear transformations

#### 5.1 Contrast stretching (piecewise-linear)

A family of transformations built from **piecewise-linear functions** controlled by two points:
$$\text{Control points: } (r_1, s_1) \text{ and } (r_2, s_2)$$
The transform T(r) is a piecewise-linear function connecting:
- $(0, 0) \to (r_1, s_1)$: low-input slope
- $(r_1, s_1) \to (r_2, s_2)$: mid-range slope (this segment is steep to stretch the middle range)
- $(r_2, s_2) \to (L-1, L-1)$: high-input slope

The shape resembles an S-curve (sigmoid). By setting $r_1 < r_2$ and $s_1 < s_2$ with the middle slope being steeper than the end slopes, the midtone range is expanded (stretched) and the shadow/highlight extremes are compressed.

**Effect:** A low-contrast, "flat" or grey-looking image gains distinct dark and bright regions. The image histogram, previously bunched in the middle, spreads across the full [0, L-1] range.

**Key insight:** The exact shape of the curve depends on the choice of $(r_1, s_1)$ and $(r_2, s_2)$. This is the main controllable parameter set.

#### 5.2 Thresholding

An **extreme case** of contrast stretching where the transform collapses to a binary step function:
$$s = \begin{cases} 0 & \text{if } r \leq t \\\\ 1 & \text{if } r > t \end{cases}$$
- Produces a binary (black-and-white) image.
- t is the threshold; if t is constant across the whole image it is **global thresholding**.
- If t depends on the spatial coordinates (x,y) it is **adaptive thresholding** (local threshold adjusted region by region).
- The transform curve is a step function jumping from 0 to 1 at threshold t.

---

### 6. Image histogram

#### 6.1 Definition

The image histogram is a discrete function:
$$h(r_k) = n_k$$
where:
- $r_k$ = the k-th intensity level, $k = 0, 1, \ldots, L-1$
- $n_k$ = number of pixels in the image with intensity $r_k$

The histogram can be **normalized** by dividing by the total number of pixels MN:
$$p_r(r_k) = \frac{n_k}{MN}$$
This gives the **probability** that a randomly chosen pixel has intensity $r_k$. It is an empirical estimate of the PDF of the image's intensity distribution.

**The image histogram is the empirical distribution of image intensities** — it discards all spatial information and treats I(x,y) as a random intensity emitter.

#### 6.2 Reading histograms

| Image type | Histogram shape |
|---|---|
| **Dark image** | Histogram mass concentrated toward the left (low intensities). Spikes cluster near 0. |
| **Light (bright) image** | Histogram mass concentrated toward the right (high intensities). Spikes cluster near L-1. |
| **Low-contrast image** | Histogram is narrow, bunched around the middle — a tall, narrow hump. |
| **High-contrast image** | Histogram is wide, spread across the full range — many levels used. |

A high-contrast image is generally preferred: it means the dynamic range is fully used and pixel values are well differentiated.

**Motivation for automation:** All earlier transforms (gain/bias, gamma, log) require manually tuning parameters. The histogram allows us to automatically determine good enhancement parameters.

---

### 7. Histogram equalization

#### 7.1 Goal

Stretch the histogram to fill the dynamic range [0, L-1] **and** make it as uniform (flat) as possible. A flat histogram means every intensity level is used equally often, maximizing the use of the dynamic range.

#### 7.2 The four-step algorithm (discrete)

**Step 1:** Compute histogram of input intensities.
$$n_k = \text{number of pixels with intensity } r_k$$
**Step 2:** Convert to probabilities (normalize).
$$p_r(r_k) = \frac{n_k}{M \cdot N}$$
$M \cdot N$ = total number of pixels. $p_r(r_k)$ is the probability that a randomly chosen pixel has intensity $r_k$.

**Step 3:** Compute the Cumulative Distribution Function (CDF).
$$T(r_k) = \sum_{j=0}^{k} p_r(r_j) \quad \text{[fraction of pixels with intensity} \leq r_k\text{]}$$
**Step 4:** Compute new intensity values.
$$s_k = (L - 1) \cdot T(r_k)$$
Which expands to:
$$s_k = (L - 1) \cdot \sum_{j=0}^{k} \frac{n_j}{MN} = \frac{L-1}{MN} \cdot \sum_{j=0}^{k} h(r_j)$$
**Application:** Every pixel whose input intensity is $r_k$ is replaced with $s_k$. Round to the nearest integer.

#### 7.3 Why does this work? (Mathematical justification)

Treat pixel intensities as continuous random variables r (input) and s (output). The image histogram of r approximates the PDF $p_r(r)$.

From probability theory, if $s = T(r)$:
$$p_s(s) = p_r(r) \cdot \left|\frac{dr}{ds}\right|$$
We want **$p_s(s) = \text{constant}$** (uniform output). Setting $p_s(s) = 1$ (normalized, s in [0,1]):
$$\frac{ds}{dr} = p_r(r)$$
$$s = \int_0^r p_r(w)\, dw$$

This is precisely the **CDF of r**. Therefore $s = \text{CDF}(r)$ is the unique transformation that maps any input distribution to a uniform output distribution.

In the continuous case:
$$T(r) = (L-1) \cdot \int_0^r p_r(w)\, dw$$
This yields $p_s(s) = \frac{1}{L-1}$, a uniform distribution over [0, L-1].

**Requirements on T:** T must be continuous, differentiable, and **strictly monotonic** (so the inverse exists and intensities are not re-ordered).

#### 7.4 Intuition

Histogram equalization works because it **maps pixel values according to their cumulative rank** in the image:
- Intensity regions where many pixels cluster (crowded) → stretched apart (large jump in s for a small range of r)
- Intensity regions with few pixels (sparse) → compressed together (small jump in s)
- Result: pixel density becomes uniform across the output range.

#### 7.5 Discrete approximation caveat

Since the discrete histogram is only an approximation of a continuous PDF, the resulting output histogram is **not perfectly flat** — it is a quasi-flat histogram. This is expected and still a good approximation.

#### 7.6 Visual results

Four input/output pairs are shown in the slides (seed image: a set of seeds/beans):

| Input image state | Input histogram | Output image state | Output histogram |
|---|---|---|---|
| Dark | Spikes clustered near 0 | Better contrast | Spread across full range |
| Light (bright) | Spikes clustered near L-1 | Better contrast | Spread across full range |
| Low-contrast | Narrow hump in middle | Higher contrast | Wide, spread distribution |
| High-contrast | Already wide | Similar, slightly adjusted | Remains wide |

After equalization, all four versions of the input produce similar-looking output images and similar-looking output histograms (spread, quasi-uniform).

---

### 8. Histogram for thresholding

#### 8.1 Using the histogram to choose a threshold

The histogram provides clues about the optimal threshold level.

- If an image is separable into two classes by thresholding, the histogram will show a **bimodal distribution**: two peaks separated by a valley of low probability.
- The threshold T1 is chosen in that valley (where the histogram dips close to zero between the two peaks).
- Thresholding is essentially a **clustering problem**: finding two clusters (dark pixels = class C1 and bright pixels = class C2) separated at some intensity level.

#### 8.2 Otsu thresholding

**Goal:** find the threshold T that **minimizes the weighted sum of within-class variances** of the two classes:
$$\arg\min_{T}\ \left(P_1 \cdot \sigma_{C_1}^2 + P_2 \cdot \sigma_{C_2}^2\right)$$

Definitions:
- $C_1$ = pixels with intensity $\leq T$ (dark class)
- $C_2$ = pixels with intensity $> T$ (bright class)
- $P_1$ = fraction of pixels in class $C_1$
- $P_2$ = fraction of pixels in class $C_2$
- $\sigma_{C_1}^2$ = variance of intensities in class $C_1$
- $\sigma_{C_2}^2$ = variance of intensities in class $C_2$

**Principle:** The best threshold makes each class as **compact** (internally homogeneous) as possible. A small within-class variance means pixels within each class are close in intensity — a good separation.

**Algorithm:** Evaluate the criterion for every possible threshold value $T = 0, 1, \ldots, L-2$ and pick the one that minimizes $P_1 \sigma_{C_1}^2 + P_2 \sigma_{C_2}^2$.

---

## Key terms (glossary)

- **Spatial domain** — the image plane of pixel coordinates (x,y). Spatial-domain processing operates directly on pixel values (contrast with the frequency domain).
- **Point operation / intensity transformation** — a transform $s = T(r)$ where the output depends only on the input intensity at the same pixel, not on any neighborhood.
- **r** — input pixel intensity; **s** — output pixel intensity after applying T.
- **Negative transform** — $s = L - 1 - r$; inverts the intensity scale.
- **Gain ($\alpha$)** — multiplier of r in $s = \alpha r + \beta$; controls contrast.
- **Bias ($\beta$)** — additive constant in $s = \alpha r + \beta$; controls brightness.
- **Log transform** — $s = c \cdot \log(1+r)$; compresses bright values, stretches dark values; reveals low-intensity details.
- **Dynamic range compression** — reducing the ratio between the largest and smallest pixel values; achieved by log transform.
- **Power-law (gamma) transform** — $s = c \cdot r^{\gamma}$; $\gamma < 1$ brightens, $\gamma > 1$ darkens.
- **Gamma correction** — pre-distorting pixel values with $\gamma = 1/\gamma_{\text{display}}$ so that a non-linear display produces a perceived-linear output.
- **Display gamma** — the exponent of the power-law response of a display device (typically $\approx 2.2$).
- **Image gamma** — the pre-correction gamma applied to the image before display (complement of display gamma).
- **Contrast stretching** — piecewise-linear transform controlled by two points $(r_1,s_1)$ and $(r_2,s_2)$ that expands the dynamic range of the midtone region.
- **Thresholding** — extreme contrast stretching; binary output: $s = 0$ if $r \leq t$, $s = 1$ if $r > t$.
- **Adaptive thresholding** — threshold t varies spatially with (x,y).
- **Histogram** $h(r_k) = n_k$ — number of pixels with intensity $r_k$; the empirical intensity distribution.
- **Normalized histogram** $p_r(r_k) = n_k / MN$ — estimates the probability of intensity $r_k$.
- **Histogram equalization** — transform $s = T(r) = (L-1) \cdot \text{CDF}(r)$ that produces a quasi-uniform output histogram.
- **CDF (Cumulative Distribution Function)** — $T(r_k) = \sum_{j=0}^{k} p_r(r_j)$; the fraction of pixels with intensity $\leq r_k$.
- **Density conservation law** — $p_r(r)\,dr = p_s(s)\,ds$; the number of pixels is conserved under any transformation.
- **Bimodal histogram** — histogram with two distinct peaks; indicates the image can be well separated by thresholding.
- **Otsu thresholding** — automatic threshold selection by minimizing weighted within-class variance $P_1 \sigma_{C_1}^2 + P_2 \sigma_{C_2}^2$.
- **Within-class variance** — variance of pixel intensities within a single class ($C_1$ or $C_2$).

---

## Exam targets

1. **Write and interpret the spatial-domain equation $g(x,y) = T[f(x,y)]$.** Explain what happens when the neighborhood shrinks to 1x1 (point operation $s = T(r)$).

2. **Give the formula for each basic transform and state its effect:**
   - Negative: $s = L - 1 - r$
   - Gain/bias: $s = \alpha r + \beta$ ($\alpha$ controls contrast, $\beta$ controls brightness)
   - Log: $s = c \cdot \log(1+r)$ where $c = S_{\max} / \log(1+R_{\max})$; why +1; why it stretches dark and compresses bright
   - Gamma: $s = c \cdot r^{\gamma}$; effect of $\gamma < 1$ vs $\gamma > 1$ vs $\gamma = 1$
   - Contrast stretching: piecewise-linear with control points $(r_1,s_1)$, $(r_2,s_2)$
   - Thresholding: binary step at t; global vs adaptive

3. **Explain gamma correction end-to-end:** why displays are non-linear ($L \propto v^{2.2}$); what happens without correction (image appears darker); how applying correction $\gamma = 1/\gamma_{\text{display}}$ pre-distorts the signal so the combined pipeline is linear.

4. **Define the image histogram.** Write $h(r_k) = n_k$. Write the normalized form. Describe what histogram shapes correspond to dark, bright, low-contrast, and high-contrast images.

5. **Derive or describe histogram equalization:**
   - The 4-step algorithm (histogram, normalize, CDF, scale by L-1).
   - The final formula: $s_k = \frac{L-1}{MN} \sum_{j=0}^{k} n_j$
   - Why CDF produces a uniform output (density conservation law argument: $p_r(r)\,dr = p_s(s)\,ds$, set $p_s = 1/L$, integrate to get $s = \text{CDF}(r)$).
   - Why the discrete result is quasi-flat (not perfectly flat).

6. **Explain Otsu thresholding:** criterion $\arg\min_T (P_1 \sigma_{C_1}^2 + P_2 \sigma_{C_2}^2)$; what $P_1$, $P_2$, $\sigma_{C_1}$, $\sigma_{C_2}$ mean; why minimizing within-class variance gives the best separation.

7. **Describe the histogram clue for thresholding:** bimodal histogram → threshold in the valley. Thresholding as a 2-cluster classification problem.

---

## Pitfalls

- **Log transform does NOT always improve an image.** If important details are in the high-intensity (bright) region, log transform will compress them and lose detail. Use it only when you want to enhance dark/low-intensity areas.
- **$\gamma < 1$ brightens; $\gamma > 1$ darkens.** Students frequently confuse the direction. Remember: $\gamma < 1$ curves bow upward (output > input for same r); $\gamma > 1$ curves bow downward.
- **Gamma correction $\gamma$ is $1 / \gamma_{\text{display}}$, not the display gamma itself.** If the display has $\gamma = 2.5$, the correction applied to the image is $\gamma = 0.4$ (not 2.5).
- **Histogram equalization gives a quasi-flat histogram, not a perfectly flat one.** Because we work with a discrete histogram (not a continuous PDF), the output is only approximately uniform. Do not claim perfect equalization.
- **The histogram contains no spatial information.** Two images with very different appearances can have identical histograms. The histogram only tells you the distribution of intensities, not where they are.
- **In Otsu's method, the criterion is minimizing within-class variance** (not maximizing it, and not the between-class variance directly — though the two are equivalent since total variance is fixed). Be precise about which variance is minimized.
- **Thresholding destroys all gray-level information.** Once binarized, you cannot recover intermediate gray levels. Only use thresholding when binary output is the goal.
- **The +1 in $\log(1+r)$ is mandatory to avoid $\log(0)$** (which is negative infinity). Forgetting it is an error.
- **Gain/bias output must be clipped to [0, L-1].** $s = \alpha r + \beta$ can exceed L-1 or go below 0; clipping (saturation) is required.
- **Contrast stretching control points must satisfy $r_1 < r_2$ and $s_1 < s_2$** for the transform to be monotonically increasing. Non-monotonic transforms are not valid intensity transforms (they would map different inputs to the same output, losing information).
