# Part 8 — Edge Detection

## Bird's eye view

- An **edge** is a local intensity discontinuity in an image; detecting edges reduces data while preserving structural information about object boundaries.
- **First-derivative operators** (Roberts, Prewitt, Sobel) approximate the image gradient; edge strength is the gradient magnitude and edge orientation is the gradient direction.
- The **Laplacian** is a second-derivative operator; edges correspond to **zero-crossings** of the Laplacian. The **Laplacian-of-Gaussian (LoG / Marr-Hildreth)** combines Gaussian smoothing with the Laplacian to reduce noise sensitivity.
- The **Canny detector** (1986) is the gold standard: it derives from three formal optimality criteria and implements them through a four-stage pipeline (smooth → gradient → non-maximum suppression → hysteresis thresholding).
- **Non-maximum suppression** thins gradient ridges to single-pixel-wide edges; **hysteresis thresholding** uses two thresholds to link weak edges to strong ones, eliminating isolated noise responses.
- **Evaluation** of edge detectors can be qualitative (visual inspection) or quantitative (Precision/Recall, Pratt's Figure of Merit, Boundary Displacement Error) against a ground-truth edge map.
- After edge detection, **line/curve fitting** (least-squares, PCA, polynomial fitting via pseudo-inverse) recovers geometric structures from edge-point sets, enabling boundary detection and higher-level scene understanding.

---

## Detailed notes

### 1. What is an edge?

An **edge pixel** is a pixel at which the image intensity changes significantly. Edges arise from:
- Object/background boundaries (depth discontinuities)
- Surface orientation discontinuities (normals change)
- Reflectance discontinuities (material changes)
- Illumination discontinuities (shadows, highlights)

Edges are the most compact representation of image structure: they remove redundant flat-region data while keeping boundary information.

**Ideal edge models:**
- **Step edge** — intensity jumps instantly from one level to another (ideal, rarely seen in real images).
- **Ramp edge** — intensity rises gradually over several pixels (blur from optics/motion).
- **Roof edge** — a thin line; intensity peaks and falls.

In the presence of noise, smoothing before differentiation is essential.

---

### 2. Image gradient — first-derivative approach

For a continuous 2D image f(x, y), the **gradient** is:

    nabla f = [df/dx, df/dy]

The **gradient magnitude** (edge strength):

    ||nabla f|| = sqrt( (df/dx)^2 + (df/dy)^2 )

Approximation for speed:

    ||nabla f|| ≈ |df/dx| + |df/dy|

The **gradient direction** (angle of maximum intensity change, perpendicular to the edge):

    theta = arctan( (df/dy) / (df/dx) )

For discrete images, derivatives are approximated by finite differences.

---

### 3. Derivative operators (discrete kernels)

#### 3.1 Roberts operator (2x2, diagonal differences)

Detects diagonal edges. Very sensitive to noise because the kernel is tiny.

    Gx = [ 1   0 ]        Gy = [  0   1 ]
         [ 0  -1 ]             [ -1   0 ]

Apply each kernel by convolution; compute magnitude.

#### 3.2 Prewitt operator (3x3)

Averages over 3 rows/columns before differencing — slightly more noise-robust than Roberts.

    Gx = [ -1   0   1 ]       Gy = [ -1  -1  -1 ]
         [ -1   0   1 ]            [  0   0   0 ]
         [ -1   0   1 ]            [  1   1   1 ]

#### 3.3 Sobel operator (3x3) — most commonly used

Weights the central row/column by 2 for better noise suppression.

    Gx = [ -1   0   1 ]       Gy = [ -1  -2  -1 ]
         [ -2   0   2 ]            [  0   0   0 ]
         [ -1   0   1 ]            [  1   2   1 ]

Gradient magnitude:  M(x,y) = sqrt(Gx^2 + Gy^2)
Gradient direction:  theta(x,y) = arctan(Gy / Gx)

**Worked example (Sobel Gx on a 3x3 patch):**

    Patch:   50  50  80       Gx response at centre = (-1)(50) + (0)(50) + (1)(80)
             50  50  80                               + (-2)(50) + (0)(50) + (2)(80)
             50  50  80                               + (-1)(50) + (0)(50) + (1)(80)
                                                    = 30 + 60 + 30 = 120   (strong horizontal edge)

---

### 4. Laplacian — second-derivative approach

The **Laplacian** is the sum of second partial derivatives:

    nabla^2 f = d^2f/dx^2 + d^2f/dy^2

Discrete 3x3 approximation:

    [ 0   1   0 ]
    [ 1  -4   1 ]       (or with diagonals: replace -4 with -8 and add 1s at corners)
    [ 0   1   0 ]

**Key property:** Edges correspond to **zero-crossings** of the Laplacian (where the second derivative changes sign), not to its maxima. This gives sub-pixel edge localization.

**Problem:** The Laplacian amplifies noise severely (double differentiation).

**Solution:** Smooth first with a Gaussian, then apply the Laplacian.

---

### 5. Laplacian-of-Gaussian (LoG) — Marr-Hildreth detector

Convolving the image with a Gaussian g_sigma and then taking the Laplacian:

    LoG(x,y) = nabla^2 [g_sigma(x,y) * I(x,y)]

Because convolution is linear and shift-invariant, this equals:

    [nabla^2 g_sigma(x,y)] * I(x,y)

So you can precompute the LoG kernel once and apply it to the image directly. The LoG kernel in 2D is:

    LoG(x,y) = -1/(pi * sigma^4) * [1 - (x^2+y^2)/(2*sigma^2)] * exp(-(x^2+y^2)/(2*sigma^2))

This is the famous **"Mexican hat"** function (positive central peak, negative surrounding ring).

**Algorithm (Marr-Hildreth):**
1. Convolve image with LoG kernel.
2. Find zero-crossings — pixels where the LoG changes sign between adjacent pixels.
3. Optional: threshold zero-crossings by the gradient magnitude to remove weak responses.

**sigma controls scale:** large sigma = coarse-scale edges, small sigma = fine-scale edges.

---

### 6. The Canny edge detector

#### 6.1 Optimality criteria (Canny 1986)

Canny formulated edge detection mathematically and looked for the optimal filter F. He defined two objective functions:
- **Lambda(F)**: large if F produces good **localization** (detected edge close to true edge).
- **Sigma(F)**: large if F produces good **detection** (high signal-to-noise ratio).

**Problem:** Find the filter F that maximizes the product Lambda(F) * Sigma(F), subject to the constraint that a **single peak** is generated at a step edge (no multiple responses).

This mathematical optimization led to the following three criteria:

1. **Low error rate** — All true edges must be found; no spurious (false) responses. High signal-to-noise ratio: the filter response at the edge should be high relative to noise.

2. **Good localization** — Detected edge pixels must be as close as possible to the centre of the true edge. The distance between detected and true edge positions should be minimized.

3. **Single response per edge** — The detector must not produce multiple edge pixels where only one edge exists. Only one peak per true step edge (prevents thick edges).

Canny showed that the optimal 1D filter satisfying these criteria is well approximated by the **first derivative of a Gaussian**.

#### 6.2 Relationship to the gradient + Laplacian combined approach

One way to use both the gradient (for direction and magnitude) and the Laplacian (for precise localization via zero-crossings):

1. Smooth with Gaussian: `n_sigma * I`
2. Compute gradient: `nabla(n_sigma) * I`
3. Find gradient magnitude: `||nabla(n_sigma) * I||`
4. Find gradient direction: `n_hat = [nabla(n_sigma)*I] / ||nabla(n_sigma)*I||`
5. Compute 1D Laplacian along gradient direction n_hat: `d^2(n_sigma * I) / d(n_hat)^2`
6. Find zero-crossings in the 1D Laplacian to pinpoint edge location.

This approach is equivalent to Canny's non-maximum suppression step (see below).

#### 6.3 The four-stage Canny algorithm

**Step 1: Noise reduction (Gaussian smoothing)**

Convolve the image I with a 2D Gaussian filter with standard deviation sigma:

    I_smooth = G_sigma * I

where G_sigma(x,y) = (1/(2*pi*sigma^2)) * exp(-(x^2+y^2)/(2*sigma^2))

Larger sigma → more smoothing → fewer fine edges detected, more robust to noise.
Smaller sigma → less smoothing → more detail, more noise sensitivity.

Example (Lena image): sigma=1 gives many fine edges, sigma=2 gives cleaner edges with some detail lost, sigma=4 gives only the strongest coarse edges.

**Step 2: Gradient magnitude and direction**

Apply Sobel (or equivalent) operator to the smoothed image:

    Gx = Sobel_x * I_smooth
    Gy = Sobel_y * I_smooth

    Magnitude: M(x,y) = sqrt(Gx^2 + Gy^2)
    Direction: n_hat = [Gx, Gy] / M(x,y)   (unit vector pointing in gradient direction)

**Step 3: Non-maximum suppression (edge thinning)**

The gradient magnitude image has thick ridges (many pixels wide). This step thins them to single-pixel width by keeping only the local maxima along the gradient direction.

For each pixel p:
- Look at the two neighbours of p along the gradient direction n_hat (and its opposite -n_hat).
- If M(p) is less than at least one of those two neighbours, suppress p: set M(p) = 0.
- Only pixels that are strict local maxima along the gradient direction are kept.

This can equivalently be described as computing the 1D Laplacian along the gradient direction and finding zero-crossings.

Visual interpretation: on the gradient image of a curved edge (e.g., a quarter-circle arc), NMS zeroes all pixels except the single brightest pixel perpendicular to the edge direction at each point along the arc. The result is a thin, one-pixel-wide edge curve.

**Step 4: Hysteresis thresholding (double threshold)**

A single threshold would either miss real edges (too high) or include noise (too low). Canny uses two thresholds T0 < T1:

    ||nabla I(x,y)|| < T0          -->  Definitely NOT an edge (suppress)
    ||nabla I(x,y)|| >= T1         -->  Definitely AN EDGE (keep)
    T0 <= ||nabla I(x,y)|| < T1   -->  Edge IF a neighbouring pixel is definitely an edge

The "weak" pixels in the middle band are kept only if they are connected (8-connectivity) to a "strong" pixel. This hysteresis mechanism links edge chains together and rejects isolated noise spikes that happen to fall between T0 and T1.

**Summary table:**

| Step | Operation | Purpose |
|------|-----------|---------|
| 1 | Gaussian smoothing (sigma) | Suppress noise |
| 2 | Sobel gradient (magnitude + direction) | Detect intensity changes |
| 3 | Non-maximum suppression | Thin edges to 1 pixel wide |
| 4 | Hysteresis thresholding (T0, T1) | Link edges, reject false positives |

**Parameters:**
- sigma: controls smoothing scale. Larger sigma = fewer, coarser edges.
- T0 (low threshold): minimum gradient to be considered a candidate edge.
- T1 (high threshold): guaranteed edge threshold. Typically T1 ~ 2-3 x T0.

**Example (Canny parameter effects on a portrait photo):**
- sigma=1, T_low=0.4, T_high=0.8: detailed edges including texture.
- sigma=2, T_low=0.4, T_high=0.8: smoother edges, less texture detail.
- sigma=1, T_low=0.3, T_high=0.7: more edges recovered (lower thresholds).

---

### 7. Edge detection evaluation

#### 7.1 Qualitative evaluation

Visual inspection of the edge map overlaid on the original image. Ask:
- Are edges clean, thin, and well-localized?
- Are object boundaries complete and continuous?
- Is there too much noise (false edges)?
- Do results match human perception?

Quick and intuitive, but subjective. Common practice: compare detectors side-by-side on the same image.

#### 7.2 Quantitative evaluation

Requires a **ground-truth edge map** (manually annotated by humans, multiple annotators with consensus, or synthetic images with known structure).

**Key metrics:**

| Metric | Description |
|--------|-------------|
| **Precision** | Fraction of detected edges that are true edges |
| **Recall** | Fraction of true edges that were detected |
| **F1-score** | Harmonic mean of Precision and Recall |
| **ROC Curve / AUC** | Precision-Recall trade-off across threshold values |
| **Boundary Displacement Error (BDE)** | Average distance between detected and ground-truth edges. Lower = better. |
| **Edge Localization Error** | Position accuracy of detected edges |
| **Pratt's Figure of Merit (FOM)** | See formula below |

**Pratt's Figure of Merit:**

    FOM = 1 / max(N_d, N_g) * sum_{i=1}^{N_d} [ 1 / (1 + alpha * d_i^2) ]

Where:
- N_d = number of detected edge pixels
- N_g = number of ground-truth edge pixels
- d_i = distance from detected edge pixel i to the nearest ground-truth pixel
- alpha = constant, typically 1/9

FOM = 1 is perfect. FOM is penalized by both missing edges (N_d < N_g) and mislocalized edges (large d_i).

**Ground-truth datasets:**
- **BSDS500** — Natural images with human-labeled contours (standard benchmark).
- **BSDS300** — Classic version, still widely cited.
- **NYUDv2** — RGB-D indoor scenes.
- **PASCAL Boundaries** — Object-level boundaries.
- **SBD** — Semantic boundaries from PASCAL VOC.

**Best practices:**
- Align image resolutions between ground-truth and prediction.
- Allow a small positional tolerance (a few pixels) in edge matching.
- Avoid data leakage (do not evaluate on training data).
- Combine quantitative and qualitative evaluation for a full picture.

**Computational metrics** (for real-time systems): runtime per image, memory usage, scalability to high-resolution images.

---

### 8. Line/curve boundary detection

After edge detection, the next step is to fit geometric primitives to the edge-point set. This is called **fitting**.

**Definition:** Fitting is the process of decomposing a set of image tokens (pixels, edge points) into components that belong to lines, circles, or other geometric shapes.

Applications: image segmentation, industrial inspection, autonomous driving (lane detection), image understanding.

#### 8.1 Three fitting sub-problems

1. **Parameter estimation** — The association between edge points and curves is known; recover curve parameters.
2. **Token-curve association** — Know the number of curves but not which point belongs to which curve; solve association and estimation jointly.
3. **Counting** — No prior knowledge: must determine how many curves exist, their associations, and parameters simultaneously.

#### 8.2 Fitting lines to edges — least-squares (vertical distance)

Given edge points (x_i, y_i), fit the line y = mx + c by minimizing the average squared **vertical** distance:

    E = (1/N) * sum_i (y_i - m*x_i - c)^2

Setting dE/dm = 0 and dE/dc = 0 (least-squares):

    dE/dm = (-2/N) * sum_i x_i*(y_i - m*x_i - c) = 0
    dE/dc = (-2/N) * sum_i (y_i - m*x_i - c) = 0

Closed-form solution:

    x_bar = (1/N) * sum_i x_i       y_bar = (1/N) * sum_i y_i

    m = sum_i (x_i - x_bar)(y_i - y_bar) / sum_i (x_i - x_bar)^2

    c = y_bar - m * x_bar

**Problem with this approach:** Fails for **vertical lines** (m → infinity, denominator → 0).

#### 8.3 Fitting lines — perpendicular distance (PCA approach)

Use the **polar (normal) form** of a line:

    x*cos(theta) + y*sin(theta) = rho

Where:
- theta = angle between x-axis and the line's normal vector
- rho = perpendicular distance from the origin to the line

Signed distance from point (x_i, y_i) to the line:

    d_i = x_i*cos(theta) + y_i*sin(theta) - rho

Minimize the sum of squared perpendicular distances:

    E(theta, rho) = sum_{i=1}^{n} (x_i*cos(theta) + y_i*sin(theta) - rho)^2

This works for lines of any orientation including vertical.

**Solution via 2D PCA:**
1. Compute the mean: (x_bar, y_bar).
2. Center the data: x'_i = x_i - x_bar, y'_i = y_i - y_bar.
3. Form the 2x2 covariance matrix from centered data.
4. Compute eigenvectors/eigenvalues — the **eigenvector of the smallest eigenvalue** gives the normal direction theta.
5. Compute rho = x_bar*cos(theta) + y_bar*sin(theta).

Equivalently: y = (-cos(theta)/sin(theta))*x + (rho/sin(theta)).

#### 8.4 Fitting curves (polynomials) to edges

Given edge points (x_i, y_i), fit a polynomial:

    y = f(x) = a*x^3 + b*x^2 + c*x + d

Minimize:

    E = (1/N) * sum_i (y_i - a*x_i^3 - b*x_i^2 - c*x_i - d)^2

Set dE/da = dE/db = dE/dc = dE/dd = 0.

With n points and 4 unknowns (a, b, c, d), for n >> 4 this is an **over-determined linear system**:

    X * a = y

Where X is n x 4 (the Vandermonde-like matrix), a = [a, b, c, d]^T is unknown, y = [y_0, ..., y_n]^T.

Since X is not square, it cannot be directly inverted. Use the **least-squares (pseudo-inverse) solution**:

    X^T X a = X^T y
    a = (X^T X)^{-1} X^T y

The matrix X^+ = (X^T X)^{-1} X^T is the **Moore-Penrose pseudo-inverse** of X.

#### 8.5 The boundary detection problem and the Hough transform

A critical challenge: after running an edge detector, many edge pixels correspond to texture, noise, or irrelevant structure — not the boundary we seek. How do we know which edges belong to the shape of interest?

One classical solution: **The Hough Transform** (Hough 1962, U.S. Patent 3,069,654). This is covered in Part 9.

---

## Key terms (glossary)

- **Edge** — Local spatial discontinuity in image intensity, corresponding to boundaries or surface changes.
- **Gradient** — Vector [df/dx, df/dy]; magnitude = edge strength, direction = perpendicular to edge.
- **Roberts operator** — 2x2 diagonal difference kernel; minimal, very noise-sensitive.
- **Prewitt operator** — 3x3 kernel; separable (smoothing + difference), handles noise better than Roberts.
- **Sobel operator** — 3x3 kernel with central weight 2; most common discrete gradient operator.
- **Laplacian** — Second-derivative operator; edges are zero-crossings; amplifies noise.
- **LoG (Laplacian-of-Gaussian)** — Gaussian smoothing followed by Laplacian; "Mexican hat" kernel; edges at zero-crossings.
- **Marr-Hildreth detector** — Edge detector based on LoG zero-crossings.
- **Canny detector** — Optimal edge detector (1986) satisfying three criteria; four-stage pipeline.
- **Non-maximum suppression (NMS)** — Thins edge ridges by keeping only local maxima along gradient direction.
- **Hysteresis thresholding** — Two-threshold scheme (T0, T1) that links weak edges to strong ones.
- **Step edge** — Idealized instantaneous intensity jump.
- **Zero-crossing** — Point where the Laplacian (or its 1D projection) changes sign; indicates edge centre.
- **Pratt's Figure of Merit (FOM)** — Quantitative metric penalizing both missed and mislocalized edges (range 0-1; 1 = perfect).
- **BDE (Boundary Displacement Error)** — Average spatial distance between detected and ground-truth edges.
- **BSDS500** — Standard benchmark dataset for edge detection evaluation.
- **Fitting** — Process of recovering curve/line parameters from a set of edge points.
- **Pseudo-inverse** — X^+ = (X^T X)^{-1} X^T; least-squares solution for over-determined linear systems.
- **Polar/normal form of a line** — x*cos(theta) + y*sin(theta) = rho; avoids singularity for vertical lines.

---

## Exam targets

1. **Write out all three Canny optimality criteria** by name, with a one-sentence explanation of each. Know that the optimal 1D filter is approximately the first derivative of a Gaussian.

2. **Describe the four Canny stages in order** (smooth → gradient → NMS → hysteresis), with what each stage does and why it is needed.

3. **Explain non-maximum suppression precisely:** for pixel p, compare M(p) with its two neighbours along the gradient direction; suppress if not the local maximum. Know that this is equivalent to finding 1D Laplacian zero-crossings along n_hat.

4. **Explain hysteresis thresholding:** three zones (T < T0: reject; T >= T1: keep; T0 <= T < T1: keep if connected to a strong edge pixel). Know why a single threshold is insufficient.

5. **Write the Sobel kernels (Gx and Gy) as 3x3 matrices** from memory. Be able to apply them to a small pixel patch and compute the magnitude and direction.

6. **State the Laplacian discrete kernel** and explain why edges are zero-crossings (not maxima) of the Laplacian.

7. **Explain LoG:** what it is, why Gaussian smoothing is applied first, and what the "Mexican hat" shape means. Know that nabla^2 [g_sigma * I] = [nabla^2 g_sigma] * I.

8. **Give Pratt's FOM formula**, name all variables, and interpret: what does FOM = 1 mean? What drives FOM down?

9. **Explain the difference between least-squares vertical-distance line fitting and perpendicular-distance fitting (PCA)**. State why vertical fitting fails for vertical lines and how polar form solves it.

10. **Describe the effect of sigma on Canny output:** larger sigma → smoother → fewer/coarser edges. Understand the detection/localization trade-off with sigma.

---

## Pitfalls

- **Gradient direction vs. edge direction:** The gradient vector points **perpendicular** to the edge (across the intensity ramp), not along it. Students often confuse the two.
- **Laplacian edges are zero-crossings, not peaks.** The Laplacian itself has a peak on one side and a trough on the other; the edge is where it crosses zero.
- **NMS direction:** You compare neighbours along the **gradient direction** (perpendicular to the edge), not along the edge itself.
- **Canny step 3 is NMS, not thresholding.** Thresholding is step 4. Common to mix them up.
- **Hysteresis: weak pixels alone are NOT edges.** A weak pixel is kept only if it is 8-connected to a strong pixel. An isolated weak pixel is discarded.
- **LoG and Canny are not the same.** LoG uses zero-crossings of the 2D Laplacian; Canny uses NMS (equivalent to 1D Laplacian zero-crossings along the gradient direction) plus hysteresis. Canny has the explicit optimality derivation.
- **Sobel vs. Prewitt:** Sobel weights the centre pixel by 2 (gives more importance to the immediate neighbours); Prewitt weights uniformly. Sobel is generally preferred in practice.
- **Roberts is 2x2**, not 3x3. It is rarely used in practice because it is extremely noise-sensitive and has no smoothing component.
- **Least-squares vertical fitting fails for vertical lines** (denominator sum_i (x_i - x_bar)^2 → 0). Use perpendicular-distance/PCA formulation instead.
- **Pseudo-inverse**: a = (X^T X)^{-1} X^T y, NOT a = X^{-1} y (X is not square for n >> m).
- **Sigma trade-off in Canny:** increasing sigma improves detection (less noise) but hurts localization (edges are blurred and shifted). This is the inherent detection-localization trade-off captured formally in Canny's Sigma(F)-Lambda(F) product.
