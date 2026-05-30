# Part 5 — Spatial Filtering

## Bird's eye view

- **Spatial filtering** is a neighborhood operator: the output value at each pixel is computed from a weighted sum of pixel values in a local neighborhood — the underlying mechanism is **convolution** (or correlation).
- A **linear filter** is fully characterized by its weight matrix (kernel/mask); different kernels give different operations (blur, sharpen, detect edges, etc.).
- A filter that is both **linear and shift-invariant** can always be implemented as a convolution.
- **Smoothing filters** (averaging, Gaussian) suppress noise by integration; **sharpening filters** (Laplacian, unsharp masking) enhance edges by differentiation.
- **Non-linear filters** (median, max, min, alpha-trimmed mean, bilateral) cannot be written as convolutions — they rank or conditionally weight pixels.
- **Correlation** slides the mask as-is; **convolution** first rotates the mask 180° — for symmetric kernels the two are identical.
- **Template matching** via normalized cross-correlation exploits the dot-product interpretation of filtering to locate patterns independent of brightness.

---

## Detailed notes

### 1. Neighborhood operators and spatial filtering

A **spatial filter** (neighborhood operator) computes the output pixel $g(x, y)$ from pixel values in a neighborhood N of $(x, y)$ in the input image $f$.

For a **linear spatial filter** with coefficient matrix W of half-size $a \times b$:

$$g(x, y) = \sum_{s=-a}^{a} \sum_{t=-b}^{b} w(s, t) \cdot f(x + s, y + t)$$

- The filter is defined by the kernel W (a $(2a+1) \times (2b+1)$ matrix of weights).
- The same kernel is slid over every position in the image (shift-invariant application).
- The output pixel sits at the center of the neighborhood.

**Expanded example for a 3×3 kernel** (indices s,t run from -1 to +1):

$$g(x, y) = w(-1,-1)\cdot f(x-1, y-1) + w(-1,0)\cdot f(x-1, y) + w(-1,1)\cdot f(x-1, y+1)$$

$$+ w(0,-1)\cdot f(x, y-1) + w(0,0)\cdot f(x, y) + w(0,1)\cdot f(x, y+1)$$

$$+ w(1,-1)\cdot f(x+1, y-1) + w(1,0)\cdot f(x+1, y) + w(1,1)\cdot f(x+1, y+1)$$

---

### 2. Properties of linear filters

A filter R is **linear** if it satisfies both:

| Property | Formula | Meaning |
|---|---|---|
| **Superposition (additivity)** | R(f + g) = R(f) + R(g) | Filtering the sum = sum of filtered results |
| **Scaling (homogeneity)** | R(kf) = k·R(f) | Scaling the image scales the output by the same factor |

A filter is **shift-invariant** if:

$$R\bigl(f(x - x_0)\bigr) = R(f)(x - x_0)$$

The response to a shifted input is just a shift of the response — same behavior everywhere.

**Key theorem:** A filter that is linear AND shift-invariant can be implemented as a **convolution**.

---

### 3. Smoothing (low-pass) filters

#### 3.1 Average filter

Replaces each pixel by the mean of all pixels in a $(2k+1) \times (2k+1)$ neighborhood:

$$R_{ij} = \frac{1}{(2k+1)^2} \sum_{u=i-k}^{i+k} \sum_{v=j-k}^{j+k} f_{uv}$$

The kernel is all-ones scaled by $1/n^2$. For $k=1$ (3×3):

$$\frac{1}{9} \begin{bmatrix} 1 & 1 & 1 \\\\ 1 & 1 & 1 \\\\ 1 & 1 & 1 \end{bmatrix}$$

**Visual effect (from slides):**
- 3×3: mild smoothing, barely visible blur.
- 5×5: clear smoothing, fine texture begins to disappear.
- 9×9: significant blur, small text remains readable.
- 15×15: strong blur, fine bars and small letters smear together.

**Drawback:** treats all neighbors equally → can produce ringing/blocking artifacts.

#### 3.2 Gaussian filter

A weighted average where closer pixels receive higher weight, following a 2D Gaussian bell shape:

**Continuous form:**

$$h(x, y) = \exp\!\left(-\frac{x^2 + y^2}{2\sigma^2}\right)$$

**Discrete kernel** of size $(2k+1) \times (2k+1)$: the $(i, j)$-th entry is:

$$H_{ij} = \frac{1}{2\pi\sigma^2} \exp\!\left(-\frac{(i-k-1)^2 + (j-k-1)^2}{2\sigma^2}\right)$$

The kernel is then normalized so its entries sum to 1 before use.

**Role of sigma:**

| sigma value | Effect |
|---|---|
| Small sigma | Only nearby pixels influence the average → little blur |
| Large sigma | Many neighbors influence the average → strong blur |
| Very large sigma | Image details disappear entirely |

$\sigma$ determines the **effective neighborhood size** for averaging.

**Why Gaussian instead of box average?**
- Box average treats all neighbors equally → causes artifacts (ringing).
- Gaussian gives progressively more weight to closer pixels → smoother, more natural results.

---

### 4. Sharpening filters

The goal of sharpening is to **highlight transitions in intensity** (edges). Sharpening is the inverse operation to smoothing:

| Smoothing | Sharpening |
|---|---|
| Integration (averaging pixels) | Differentiation (first or second order) |
| Removes details | Enhances details |

#### 4.1 Derivatives on discrete images

Since images are discrete, derivatives are approximated by **finite differences**:

**First-order derivative (1D):**

$$\frac{df}{dx} \approx f(x+1) - f(x)$$

**Second-order derivative (1D):**

$$\frac{d^2f}{dx^2} \approx f(x+1) + f(x-1) - 2f(x)$$

**Interpretation (from scan-line example in slides):**
- Along a flat region (constant intensity): first derivative = 0, second derivative = 0.
- Along a ramp: first derivative is nonzero and constant, second derivative = 0 in the interior.
- At a step edge: first derivative spikes once; second derivative fires at both boundaries (+ and -).

**Consequence for edge detection:**
- First derivative → thick edges (nonzero along the whole ramp).
- Second derivative → thin double-edge one pixel thick, separated by zeros. **Better for fine-detail enhancement, and easier to implement.**

#### 4.2 Unsharp masking (high-boost filtering)

A practical sharpening pipeline:

1. **Blur** the original image: $f_{\text{blur}} = \text{smooth}(f)$
2. **Subtract** the blurred image: $\text{mask} = f - f_{\text{blur}}$ (this is the unsharp mask — contains only high-frequency detail).
3. **Add** the mask back: $g = f + k \cdot \text{mask}$ ($k=1$ for unsharp masking; $k>1$ for high-boost filtering)

**Signal-domain intuition (from slides):** The blurred signal is a smoothed version of the original. The mask (original minus blurred) captures the sharp transitions. Adding it back to the original creates overshoot/undershoot at edges — perceived as sharpness.

#### 4.3 The Laplacian filter

The **Laplacian** is the simplest **isotropic** second-order derivative operator (responds equally in all directions — no directional preference for horizontal, vertical, or diagonal).

**Continuous definition:**

$$\nabla^2 f = \frac{\partial^2 f}{\partial x^2} + \frac{\partial^2 f}{\partial y^2}$$

**Discrete form (finite differences in both axes):**

$$\nabla^2 f(x,y) = f(x+1, y) + f(x-1, y) + f(x, y+1) + f(x, y-1) - 4f(x, y)$$

Since the Laplacian is a linear operator, it can be implemented as a convolution with the following kernels:

**4-neighbor Laplacian kernel (isotropic for 90° increments):**

$$\begin{bmatrix} 0 & 1 & 0 \\\\ 1 & -4 & 1 \\\\ 0 & 1 & 0 \end{bmatrix}$$

**8-neighbor Laplacian kernel (isotropic for 45° increments — includes diagonals):**

$$\begin{bmatrix} 1 & 1 & 1 \\\\ 1 & -8 & 1 \\\\ 1 & 1 & 1 \end{bmatrix}$$

The 8-neighbor version adds two diagonal difference terms to capture edges at 45°.

**Sharpening with the Laplacian:**

The Laplacian highlights intensity discontinuities and suppresses slow gradients. To sharpen while preserving background:

$$g(x, y) = f(x, y) + c \cdot \nabla^2 f(x, y)$$

where $c = +1$ if the Laplacian kernel has a negative center (as above), or $c = -1$ if the center is positive. This adds edge information back to the original.

**Worked example (moon image, from slides):** Original moon image → Laplacian response (grey background, white/dark rings at crater edges) → adding back gives sharpened image with clearly defined crater walls.

**Sharpening kernels** that combine original + Laplacian in a single convolution step:

For the 4-neighbor version, the combined kernel is:

$$\begin{bmatrix} 0 & -1 & 0 \\\\ -1 & 5 & -1 \\\\ 0 & -1 & 0 \end{bmatrix}$$

(center becomes $1 + 4 = 5$ because $g = f - \nabla^2 f$, adjusting sign convention)

---

### 5. Non-linear filters: Order-statistic filters

These filters are **non-linear** and **cannot be implemented as convolutions**. They work by:
1. Collecting all pixel values in the neighborhood.
2. Sorting (ordering) them.
3. Replacing the center pixel with a value determined by rank.

#### 5.1 Median filter

Replaces each pixel with the **median** of all intensity values in its neighborhood (including the center pixel itself).

- **Excellent at removing salt-and-pepper (impulse) noise** while preserving edges.
- Reason: isolated outlier values (very high or very low) are pushed to the extremes of the sorted list; the median is unaffected.

**Worked example:** 3×3 neighborhood values: {10, 20, 30, 25, 200, 15, 22, 18, 12}.
Sorted: {10, 12, 15, 18, 20, 22, 25, 30, 200}.
Median = 20. (The outlier 200 does not distort the result.)

#### 5.2 Max filter

Replaces each pixel with the **brightest** (maximum) intensity in the neighborhood. Useful for dilating bright regions; removes dark noise (pepper).

#### 5.3 Min filter

Replaces each pixel with the **darkest** (minimum) intensity in the neighborhood. Useful for eroding dark regions; removes bright noise (salt).

#### 5.4 Alpha-trimmed mean filter

A robust mean that discards outliers:
1. Eliminate the top $\alpha/2$ and bottom $\alpha/2$ values from the sorted neighborhood list.
2. Compute the average of the remaining $(n - \alpha)$ values.

- When $\alpha = 0$: reduces to the ordinary average filter.
- When $\alpha = n-1$ (all but one): approaches the median filter.
- Balances between average (good for Gaussian noise) and median (good for impulse noise).

**Visual result (from slides — circuit board example):** alpha-trimmed mean on a heavily noisy circuit board image effectively removes salt-and-pepper noise while being less aggressive than a pure median.

#### 5.5 Bilateral filter

A non-linear, **edge-preserving** smoothing filter. Unlike Gaussian smoothing (which blurs across edges), the bilateral filter considers both:
- **Spatial proximity** (nearby pixels matter more)
- **Intensity similarity** (pixels with similar intensity matter more)

**Formula:**

$$I'(x) = \frac{1}{W_p} \sum_{x_i \in N(x)} I(x_i) \cdot f_s(\|x_i - x\|) \cdot f_r(|I(x_i) - I(x)|)$$

where:

$$f_s(\|x_i - x\|) = \exp\!\left(-\frac{\|x_i - x\|^2}{2\sigma_s^2}\right) \quad \text{[spatial Gaussian]}$$

$$f_r(|I(x_i) - I(x)|) = \exp\!\left(-\frac{|I(x_i) - I(x)|^2}{2\sigma_r^2}\right) \quad \text{[range/intensity Gaussian]}$$

$$W_p = \sum_{x_i} f_s(\cdots) \cdot f_r(\cdots) \quad \text{[normalization]}$$

| Parameter | Effect |
|---|---|
| $\sigma_s$ | Spatial spread — how large the spatial neighborhood is |
| $\sigma_r$ | Intensity range — how similar intensities must be to contribute |

**Why bilateral filtering is slow:** The weights depend on both position and pixel intensity, so they must be **recomputed for every pixel and every neighbor**. This makes the filter non-linear and non-shift-invariant — it cannot be precomputed as a single fixed kernel.

**Visual result (from slides — portrait photo):** Pure Gaussian ($\sigma_s=2$) blurs both skin and hair edges. Bilateral ($\sigma_s=2$, $\sigma_r=10$) smooths skin while keeping hair edges sharp.

---

### 6. Correlation and convolution

#### 6.1 Definitions

Both operations slide a filter mask over an image and compute inner products at each location. The difference is whether the mask is flipped first.

**Correlation** (symbol: ⊗):

$$(w \otimes f)(x, y) = \sum_{s=-a}^{a} \sum_{t=-b}^{b} w(s, t) \cdot f(x + s, y + t)$$

The mask moves in the same direction as the shift — no rotation.

**Convolution** (symbol: *):

$$(w * f)(x, y) = \sum_{s=-a}^{a} \sum_{t=-b}^{b} w(s, t) \cdot f(x - s, y - t)$$

The mask is **rotated 180°** before the shift (equivalently, the image coordinates are negated in the sum).

#### 6.2 Key distinction

- Correlating a filter w with an impulse (image = all 0s with a single 1) yields a **180°-rotated copy of w** at the impulse location.
- Convolving a filter w with an impulse yields an **unrotated copy of w** at the impulse location.

This is the fundamental property: **convolution with a unit impulse reproduces the kernel at the impulse position.**

For **symmetric kernels** (e.g., Gaussian, average, Laplacian), correlation and convolution produce identical results.

#### 6.3 Border / padding handling for 2D

For a filter of size M×N applied to an image:
- Pad the image with **(M-1) rows at top and bottom** and **(N-1) columns at left and right**, filled with zeros.
- This ensures the filter can be centered on every input pixel.

**Full result size:** (image_rows + M - 1) × (image_cols + N - 1).
**Cropped (same-size) result:** take only the central portion of size equal to the original image.

#### 6.4 Worked 2D example (from slides)

Image f (5×5, padded to 9×9):
```
Origin f(x,y):
0 0 0 0 0
0 0 0 0 0
0 0 1 0 0   <-- impulse at (2,2)
0 0 0 0 0
0 0 0 0 0
```
Filter w (3×3):
```
1 2 3
4 5 6
7 8 9
```

**Correlation result (cropped to 5×5):**
```
0 0 0 0 0
0 9 8 7 0
0 6 5 4 0
0 3 2 1 0
0 0 0 0 0
```
The mask appears **rotated 180°** (9 8 7 / 6 5 4 / 3 2 1).

**Convolution result (cropped to 5×5):**
```
0 0 0 0 0
0 1 2 3 0
0 4 5 6 0
0 7 8 9 0
0 0 0 0 0
```
The mask appears **unrotated** — a direct copy of w at the impulse location.

---

### 7. Template matching via correlation

#### 7.1 SSD-based matching

To locate a template w in image f, compute the **Sum of Squared Differences** at every candidate position $(i, j)$:

$$E[i, j] = \sum_{s=-a}^{a} \sum_{t=-b}^{b} \bigl[w(s,t) - f(i+s, j+t)\bigr]^2$$

Expanding:

$$E[i,j] = \sum w(s,t)^2 + \sum f(i+s,j+t)^2 - 2 \sum w(s,t) \cdot f(i+s,j+t)$$

- The first term is constant (template energy, fixed).
- Minimizing $E[i,j]$ is equivalent to **maximizing** the cross-correlation term $\sum w(s,t)\cdot f(i+s,j+t) = (w \otimes f)(i,j)$.

So template matching reduces to **computing the correlation** and finding its peak.

#### 7.2 Problem: sensitivity to brightness

Raw correlation is a **dot product** between the vectorized template and the vectorized image patch. The dot product is largest when the two vectors are parallel (same direction). However, it can also be large simply because the image region is **bright** (high magnitude), not because it matches the template in shape.

**Example (from slides):** Template w looks like region A in image f. But raw correlation gives $R_{wf}(C) > R_{wf}(B) > R_{wf}(A)$, where C is brightest — the wrong answer.

#### 7.3 Normalized correlation (solution)

Divide by the magnitudes of both the template and the image patch:

$$N_{wf}[i,j] = \frac{\displaystyle\sum_{s,t} w(s,t)\cdot f(i+s, j+t)}{\sqrt{\displaystyle\sum_{s,t} w(s,t)^2} \cdot \sqrt{\displaystyle\sum_{s,t} f(i+s, j+t)^2}}$$

This is the **cosine similarity** between the template vector and the image-patch vector. It lies in $[-1, 1]$ and is **insensitive to brightness** (absolute intensity magnitude cancels out).

**Result:** $N_{wf}(A) > N_{wf}(B) > N_{wf}(C)$ — the best structural match (A) now scores highest.

**Visual example (from slides — playing card):** A king-of-spades card is correlated with a template of the king's face. The normalized correlation map is nearly black everywhere except a single bright peak exactly at the face location, marked by a highlighted bounding box.

#### 7.4 Filters as templates

This connection generalizes: **any filter kernel is implicitly a template**. When you slide a filter over an image, it responds most strongly where the local image patch **looks like the filter** (same orientation, same pattern of intensities). This is the mechanism behind oriented edge detectors, Gabor filters, and many feature detectors.

---

## Key terms (glossary)

| Term | Definition |
|---|---|
| **Spatial filter** | Neighborhood operator: output at each pixel = function of surrounding pixel values |
| **Linear filter** | Filter satisfying superposition and scaling; implemented as convolution |
| **Shift-invariant** | Same behavior regardless of position in the image |
| **Kernel / mask** | The weight matrix defining a filter |
| **Convolution** | Filter operation with 180° rotation of the mask; symbol * |
| **Correlation** | Filter operation without mask rotation; symbol ⊗ |
| **Average filter** | Box kernel — equal weights for all neighbors; smooths but causes artifacts |
| **Gaussian filter** | Weighted-average kernel with Gaussian weights; $\sigma$ controls blur amount |
| **sigma** | Standard deviation of a Gaussian; determines neighborhood size for averaging |
| **Smoothing** | Low-pass filtering; attenuates high-frequency noise; equivalent to integration |
| **Sharpening** | High-pass / derivative filtering; amplifies edges; equivalent to differentiation |
| **Unsharp masking** | Sharpening by subtracting blurred image from original and adding the difference back |
| **Laplacian** | Isotropic second-order derivative operator; $\nabla^2 f$; used for sharpening |
| **Isotropic** | Direction-invariant; responds equally in horizontal, vertical, and diagonal directions |
| **Finite difference** | Discrete approximation to a derivative using pixel differences |
| **Order-statistic filter** | Non-linear filter based on ranked pixel values in a neighborhood |
| **Median filter** | Replaces center pixel with median of neighborhood; robust to salt-and-pepper noise |
| **Max / Min filter** | Replaces center pixel with neighborhood maximum / minimum |
| **Alpha-trimmed mean** | Mean after discarding $\alpha/2$ extreme values; hybrid between mean and median |
| **Bilateral filter** | Non-linear edge-preserving smoother; weights by both spatial distance and intensity difference |
| **Template matching** | Locating a pattern in an image using correlation |
| **Normalized correlation** | Cosine similarity between template and image patch; removes brightness dependence |
| **Unit impulse response** | Convolution with a unit impulse reproduces the kernel at the impulse position |
| **Padding / border handling** | Extending the image (typically with zeros) so the filter can be applied at every pixel |

---

## Exam targets

1. **Write the linear filter equation** — the double sum $g(x,y) = \sum_s \sum_t w(s,t)\cdot f(x+s, y+t)$ — and expand it for a 3×3 kernel.

2. **State the three properties** (superposition, scaling, shift-invariance) and the key theorem: a linear shift-invariant filter can be implemented as a convolution.

3. **Draw and describe** the 3×3 average kernel and explain why increasing kernel size (3→5→9→15) progressively blurs the image.

4. **Write the discrete Gaussian kernel formula** $H_{ij}$ and explain the role of $\sigma$. Justify why Gaussian is preferred over box averaging.

5. **Describe unsharp masking** step by step (blur → subtract → add back) and explain what the mask contains.

6. **Derive the discrete Laplacian** from the 2D second-order finite difference formula, and show the two standard 3×3 kernels (4-neighbor and 8-neighbor). Write the sharpening equation $g = f + c\cdot\nabla^2 f$.

7. **Explain the difference between first and second derivative** for edge sharpening (thick vs. thin edges; why second-order is preferred).

8. **Distinguish correlation from convolution** — the 180° flip — and show with the impulse example what each operation produces.

9. **Write the 2D correlation and convolution formulas** and describe the zero-padding rule for a filter of size M×N.

10. **Explain the median filter**: why it removes salt-and-pepper noise better than averaging; why it is non-linear and cannot be a convolution.

11. **Describe the bilateral filter**: the two Gaussian weight functions ($f_s$ spatial, $f_r$ range), explain why $\sigma_s$ and $\sigma_r$ control different aspects, and explain why it is slow (weights recomputed per pixel).

12. **Template matching**: write $E[i,j]$ (SSD), show how minimizing SSD implies maximizing correlation, and write the normalized correlation formula $N_{wf}$. Explain why normalization is necessary.

---

## Pitfalls

- **Correlation vs convolution confusion:** The formula from the basic spatial filtering slides, $g(x,y) = \sum w(s,t)\cdot f(x+s, y+t)$, is the **correlation** formula, NOT convolution. Convolution uses $f(x-s, y-t)$. For symmetric kernels (Gaussian, Laplacian, average) it makes no difference; for asymmetric kernels it does.

- **Laplacian sign convention:** Whether you write $\nabla^2 f$ with center $-4$ (positive surroundings) or $+4$ (negative surroundings) changes the sign of $c$ in $g = f + c\cdot\nabla^2 f$. Always be consistent. The standard Berrani convention uses center = $-4$, so $c = +1$ for sharpening.

- **Unsharp masking is called "unsharp" but sharpens:** The name refers to the blurred (unsharp) mask that is subtracted, not to the result.

- **Second-order not first-order for sharpening:** First derivative gives thick edges (nonzero over the whole ramp); second derivative gives thin double-edges (zero over ramps, spikes at transitions) — this is why the Laplacian (second-order) is used for sharpening.

- **Median filter is non-linear:** Students often confuse it with a weighted average. It violates superposition (median of $(f+g) \neq \text{median}(f) + \text{median}(g)$), so it cannot be expressed as a convolution.

- **Bilateral filter is not shift-invariant:** Its weights depend on the actual pixel intensities at the current location, not just the offsets — so it violates shift-invariance and cannot be precomputed as a fixed kernel.

- **Normalized correlation range:** $N_{wf}$ lies in $[-1, 1]$. A value of 1 means perfect structural match (same pattern up to brightness scaling); 0 means no correlation; -1 means inverted pattern. Raw correlation has no bounded range.

- **Padding size:** For an M×N filter, you need M-1 rows of padding (top and bottom) and N-1 columns (left and right), not M/2. When using a $(2k+1)\times(2k+1)$ kernel you pad with $k$ rows/columns — be careful about off-by-one.

- **Gaussian kernel must be normalized:** The formula $H_{ij}$ gives unnormalized Gaussian weights. Before applying as a convolution mask, divide each entry by the sum of all entries, otherwise pixel intensities will be scaled.

- **Template matching via raw correlation fails for brightness variation:** This is a classic exam trap. Always use **normalized** correlation (cosine similarity) so that a bright version of the pattern does not score higher than a matched pattern of moderate intensity.
