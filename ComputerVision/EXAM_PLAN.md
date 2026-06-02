# Computer Vision Exam Plan — Sun 1 June, 09:00

> **Today is Sat 31 May.** ~8-10 productive hours total. **All 16 parts** in scope (Parts 15-16 are extra DL content added late — adjusted plan below).
> Strategy: **strict triage**. Some chapters get deep coverage, some get just the bird's-eye + key formulas. Read each chapter's `0X_*.md` (especially the **Bird's eye view** + **Exam targets** sections — those are the high-value extracts).

---

## Time inventory

| Block | Window | Hours |
|---|---|---|
| Sat 31 May (tonight) | now → midnight | **6-7 h** |
| Sun 1 June (exam day) | 06:30 → 08:30 | **2 h** review |
| **Total** | | **8-10 h** |

---

## Triage — what to actually study

### Tier 1 — HIGH PRIORITY (definitely on exam, 60-90 min each)
The classic CV exam favorites — *specific algorithms with formulas you can be asked to reproduce*.

- **Ch 5 Spatial Filtering** (45 min) — convolution underpins everything else
- **Ch 8 Edge Detection / Canny** (60 min) — Canny's 5 steps are *the* exam classic
- **Ch 9 Hough Transform + RANSAC** (60 min) — voting + RANSAC formula
- **Ch 10 Corner Detection / Harris** (45 min) — structure tensor + Harris response
- **Ch 11 SIFT Detector** (60 min) — scale-space + DoG + keypoint detection
- **Ch 12 SIFT Descriptor** (45 min) — 128-dim descriptor + matching

**Tier 1 total: ~5 hours.**

### Tier 2 — MEDIUM (30-40 min each)

- **Ch 6 Frequency Domain** (40 min) — 2D FT + convolution theorem + filter families
- **Ch 7 Image Features** (30 min) — gradient/Laplacian/LoG, edge profiles
- **Ch 4 Image Enhancement** (25 min) — histogram equalization, gamma transform
- **Ch 13 Motion Analysis** (35 min) — optical flow constraint + Lucas-Kanade
- **Ch 15 DL — Object Localization & Detection** (35 min) — *new* — IoU, NMS, anchors, R-CNN vs YOLO

**Tier 2 total: ~2.5-3 hours.**

### Tier 3 — LOW (15-20 min each)
Concepts only; just read the bird's-eye sections.

- **Ch 1 Introduction** (15 min) — three CV levels (low/mid/high)
- **Ch 2 Image Formation** (15 min) — 2D sampling + image as 2D function
- **Ch 3 Color & Basic Image Processing** (20 min) — RGB / HSV / Lab
- **Ch 14 Deep Learning for CV** (20 min) — Gestalt + segmentation + U-Net
- **Ch 16 DL Model Optimization** (20 min) — *new* — pruning, quantization, edge vs cloud

**Tier 3 total: ~1.5 hours.**

---

## Day-by-day schedule

### Saturday 31 May (tonight, 6-7 h)

| Time | Activity |
|---|---|
| **Block 1 — 90 min** | Tier 1 foundations: **Ch 5 (filtering, 45 min)** → **Ch 8 Canny (45 min)** |
| 15 min break | |
| **Block 2 — 90 min** | **Ch 9 Hough+RANSAC (60 min)** → **Ch 10 Harris (45 min)** (let it spill ~15 min into block 3) |
| 15 min break | |
| **Block 3 — 90 min** | **Ch 11 SIFT detector (60 min)** → **Ch 12 SIFT descriptor (45 min)** |
| 15 min break | |
| **Block 4 — 60-90 min** | Tier 2 sweep: **Ch 6 (40 min)** + **Ch 7 (30 min)** + **Ch 4 (30 min) if time** |

**Sleep by 23:30 latest.** Sleep > extra cramming.

### Sunday 1 June (exam day, 2 h pre-exam)

Wake 06:30. Light breakfast.

| Time | Activity |
|---|---|
| 06:45-07:15 | **Tier 2: Ch 13 Motion (LK, OFCE) + Ch 15 Detection (IoU/NMS/anchors)** — 15 min each |
| 07:15-07:50 | **Tier 3 sweep — bird's-eye sections only**: Ch 1, 2, 3, 14, 16 (~7 min each) |
| 07:50-08:25 | **Cross-chapter cheatsheet pass**: redraw from memory — Canny pipeline, Hough accumulator, Harris response, SIFT pipeline, detection output vector |
| 08:25-08:50 | Travel + settle |
| 09:00 | Exam |

---

## Per-chapter — what to pay attention to

### Ch 1 — Introduction (15 min) — Tier 3
**Core ideas only.** Skip the rest.
- **3 levels of CV** (memorize):
  - **Low (image processing)** — image → image (restoration, enhancement, noise reduction)
  - **Mid** — image → features/attributes/descriptors (segmentation, shape recognition)
  - **High** — image → concepts (scene understanding)
- CV vs Computer Graphics: forward/inverse problem.
- Applications: OCR, surveillance, medical imaging, robotics.

### Ch 2 — Image Formation (15 min) — Tier 3
- Image as a 2D function $f(x, y)$ where $(x,y)$ are spatial coords and $f$ is intensity.
- 2D sampling theorem: same as 1D but extended; spatial Nyquist.
- Pixel = sample of $f$ at grid position.
- Gray-level quantization: $L = 2^b$ levels for $b$ bits.

### Ch 3 — Color Fundamentals & Basic Image Processing (20 min) — Tier 3
- **Tristimulus theory**: any color = mixture of R, G, B.
- **Color spaces**: RGB (additive), CMYK (subtractive, printing), HSV (Hue/Saturation/Value — intuitive), YCbCr (luminance/chrominance — used in JPEG/MPEG).
- HSV conversion from RGB: know roughly the steps (compute V = max, S = (V−min)/V, H from which channel is max).
- Histogram = pixel intensity distribution.

### Ch 4 — Image Enhancement (30 min) — Tier 2
**Spatial-domain operations** $g(x,y) = T[f(x,y)]$.
- **Point operations** (per-pixel):
  - Negative: $s = L - 1 - r$
  - **Log transform**: $s = c \log(1 + r)$ — compresses dynamic range
  - **Power-law (gamma)**: $s = c \cdot r^\gamma$ — gamma < 1 brightens dark, gamma > 1 darkens bright
- **Histogram equalization**: spreads pixel values to flatten histogram → maximize contrast. Formula: $s_k = (L-1) \sum_{j=0}^{k} p_r(r_j)$ (cumulative distribution × max level).
- **Histogram matching/specification**: same idea but match to a target shape.

### Ch 5 — Spatial Filtering (45 min) — Tier 1
**Convolution** is foundational — used by Canny, Harris, SIFT.
- **Convolution formula**: $g[x, y] = \sum_s \sum_t w[s, t] \cdot f[x-s, y-t]$ (kernel $w$ flipped before sliding)
- **Correlation**: same but without flip (often used interchangeably in practice).
- **Smoothing kernels**:
  - **Box filter** (averaging): all weights equal, $1/n^2$
  - **Gaussian**: $G(x, y) = \frac{1}{2\pi\sigma^2} \exp(-\frac{x^2+y^2}{2\sigma^2})$ — most common smoother. **Separable**.
- **Sharpening kernels**:
  - **Laplacian**: 2nd derivative isotropic kernel:
    ```
    [ 0  -1   0]
    [-1   4  -1]
    [ 0  -1   0]
    ```
  - **Unsharp masking**: $g = f + k(f - \bar{f})$ where $\bar{f}$ is blurred version.
- **Order-statistic filters**:
  - **Median filter** — best for salt-and-pepper noise; preserves edges (nonlinear).
- **Kernel separability**: 2D $G(x,y) = G(x) \cdot G(y)$ → can apply two 1D convolutions instead of one 2D → $O(n)$ vs $O(n^2)$ per pixel.

### Ch 6 — Frequency Domain (40 min) — Tier 2
- **2D DFT**: $F(u, v) = \sum_x \sum_y f(x, y) \, e^{-j 2\pi (ux/M + vy/N)}$
- **Convolution theorem**: $f \ast h \leftrightarrow F \cdot H$ — convolution in time = multiplication in frequency
- **Filter families** (the only thing you really need to memorize here):
  - **Ideal LPF**: brick-wall — causes ringing
  - **Butterworth LPF**: $H(u, v) = \frac{1}{1 + (D/D_0)^{2n}}$ — smoother roll-off
  - **Gaussian LPF**: $H = e^{-D^2/(2 D_0^2)}$ — no ringing
- **High-pass = 1 − low-pass** (or band-pass / notch as variations)
- **Why use freq domain?** Some filters easier to design there (e.g., notch out specific frequencies)

### Ch 7 — Image Features (30 min) — Tier 2
- **Feature** = local, meaningful, detectable part of image
- **Edge profiles**: step, ramp, line, roof
- **First derivative (gradient)** — magnitude detects edges, direction perpendicular to edge:
  ```math
  \nabla I = \begin{bmatrix} I_x \\ I_y \end{bmatrix}, \quad |\nabla I| = \sqrt{I_x^2 + I_y^2}
  ```
- **Gradient operators**:
  - **Sobel** (most common):
    ```
    Sx = [-1 0 1; -2 0 2; -1 0 1]
    Sy = [-1 -2 -1; 0 0 0; 1 2 1]
    ```
  - **Prewitt**: uses 1s instead of 2 in the center row/col
  - **Roberts**: 2×2 (older, less robust)
- **Second derivative (Laplacian)** — zero crossings detect edges:
  $\nabla^2 I = I_{xx} + I_{yy}$
- **Laplacian of Gaussian (LoG)** = $\nabla^2 (G \ast I)$ — smooth then take 2nd derivative; reduces noise sensitivity
- **Marr-Hildreth edge detector**: zero crossings of LoG

### Ch 8 — Edge Detection / Canny (60 min) — Tier 1 ★
**The exam classic — know cold.**

**Canny's 3 criteria for an optimal edge detector**:
1. **Low error rate** — find all edges, no spurious
2. **Good localization** — found edges are close to true edges
3. **Single response** — one edge pixel per edge

**Canny algorithm (5 steps — MEMORIZE)**:
1. **Smooth** with Gaussian (parameter $\sigma$)
2. **Compute gradient** (magnitude + direction) — typically Sobel
3. **Non-maximum suppression (NMS)** — thin edges to 1-pixel width; for each pixel, check if it's a local maximum along its gradient direction
4. **Double thresholding** — two thresholds $T_H$ and $T_L$:
   - $|\nabla I| > T_H$ → **strong edge** (keep)
   - $T_L < |\nabla I| < T_H$ → **weak edge** (keep only if connected)
   - $|\nabla I| < T_L$ → discard
5. **Edge tracking by hysteresis** — connect weak edges to strong ones via 8-connectivity

**Why it works**: Smoothing reduces noise; gradient finds intensity changes; NMS thins; thresholds keep real edges and suppress noise; hysteresis recovers genuine continuous edges.

### Ch 9 — Hough Transform + RANSAC (60 min) — Tier 1 ★
Two complementary fitting tools — pay attention to BOTH.

#### Hough Transform
- **Goal**: detect parameterized shapes (lines, circles) in edge images
- **Cartesian line problem**: $y = mx + c$ — vertical lines have $m = \infty$ (bad)
- **Polar parameterization (fix)**: $\rho = x \cos\theta + y \sin\theta$
  - $\theta \in [0, \pi)$ (bounded)
  - $\rho \in [-D, D]$ where $D = $ image diagonal
- **Voting**: each edge point votes for all $(\rho, \theta)$ pairs consistent with it — traces a sinusoid in $(\rho, \theta)$ space
- **Accumulator** = 2D array indexed by $(\rho, \theta)$; peaks = detected lines
- **Generalized Hough Transform** (Ballard, 1981) — for arbitrary shapes
- **Limitations**: accumulator size grows exponentially with parameter count; impractical above 3-4 parameters

#### RANSAC (Random Sample Consensus)
- Use when many outliers (up to ~50%)
- **Algorithm (6 steps)**:
  1. Randomly sample minimal set of size $s$ to define the model (e.g., 2 points for a line)
  2. Fit model
  3. Count inliers (points within tolerance) = **consensus set**
  4. Record hypothesis with inlier count
  5. Repeat $N$ times; keep model with largest consensus
  6. **Re-fit** model using all inliers of winning consensus
- **Iteration count formula**:
  ```math
  N = \frac{\log(1 - p)}{\log(1 - (1 - e)^s)}
  ```
  - $p$ = desired success probability (e.g., 0.99)
  - $e$ = outlier ratio
  - $s$ = sample size
- **Hough vs RANSAC**:
  - Hough: exhaustive grid search, finds ALL lines, memory-intensive
  - RANSAC: randomized, finds ONE dominant model, scales poorly when $e$ high or $s$ large

### Ch 10 — Corner Detection / Harris (45 min) — Tier 1 ★

**Why corners?** They're 2D-localizable (edges only localize in one direction).

**Region taxonomy** based on local intensity variation:
- **Flat region** — tight gradient cluster (both eigenvalues small)
- **Edge region** — thin elongated ellipse (one eigenvalue large)
- **Corner region** — wide isotropic ellipse (**both eigenvalues large**)

**Structure tensor / second-moment matrix** $M$ (over a window $W$):
```math
M = \sum_W w \begin{bmatrix} I_x^2 & I_x I_y \\ I_x I_y & I_y^2 \end{bmatrix}
```

**Eigenvalues** $\lambda_1, \lambda_2$ of $M$ measure intensity-change strength along principal gradient directions.

**Harris response** (avoids explicit eigenvalue computation):
```math
R = \det(M) - k \cdot \mathrm{trace}(M)^2 = \lambda_1 \lambda_2 - k(\lambda_1 + \lambda_2)^2
```
- $k \approx 0.04$ to $0.06$
- $R > T$ → corner
- $R < 0$ → edge
- $|R|$ small → flat

**Harris pipeline**:
1. Compute gradients $I_x, I_y$
2. Build $M$ at every pixel
3. Compute $R$
4. Threshold
5. Non-maximum suppression (keep one peak per cluster)

**Variants**:
- **Shi-Tomasi**: corner if $\min(\lambda_1, \lambda_2) > T$ — simpler, no $k$
- **Förstner**: requires both eigenvalues large AND ratio not too extreme — for precise sub-pixel localization

**Invariances**: rotation ✓, translation ✓, partial illumination ✓; scale ✗ (limited; needs multi-scale → SIFT).

### Ch 11 — SIFT Detector (60 min) — Tier 1 ★
**Scale-Invariant Feature Transform**. Lowe, 2004 — the iconic feature detector.

**Why SIFT?** Harris is not scale-invariant. SIFT adds scale invariance by detecting in scale-space.

**Scale-space**: $L(x, y, \sigma) = G(x, y, \sigma) \ast I(x, y)$ where $G$ is 2D Gaussian.

**Difference of Gaussians (DoG)** — efficient approximation of normalized LoG:
```math
D(x, y, \sigma) = L(x, y, k\sigma) - L(x, y, \sigma)
```

**Pipeline**:
1. Build **Gaussian pyramid** (octaves × scales-per-octave) — each octave halves resolution
2. Compute **DoG** at each scale within each octave
3. **Detect keypoints** = local extrema in 3D neighborhood (8 spatial + 9 above + 9 below = 26 neighbors) in $(x, y, \sigma)$
4. **Sub-pixel refinement**: Taylor expansion around discrete extremum → interpolated location
5. **Edge response elimination**: use Hessian eigenvalue ratio (similar to Harris) — discard keypoints on edges
6. **Orientation assignment**: compute gradient histogram in window around keypoint; dominant orientation = peak; secondary peaks (>80% of main) → duplicate keypoints

**Characteristic scale** = the $\sigma$ at which the normalized 2nd derivative attains an extremum. Proportional to blob size:
```math
\frac{\text{size of blob A}}{\text{size of blob B}} = \frac{\sigma^*_A}{\sigma^*_B}
```

**Output per keypoint**: $(x, y, \sigma, \theta)$ — position, scale, orientation.

### Ch 12 — SIFT Descriptor (45 min) — Tier 1 ★

**Goal**: produce a feature vector at each keypoint that is robust to small translations, illumination, etc.

**Descriptor computation**:
1. Take **16×16 window** around the keypoint
2. Rotate window to align with the keypoint's dominant orientation (achieves rotation invariance)
3. Subdivide into **4×4 grid** of cells (each cell = 4×4 pixels)
4. For each cell, compute **8-bin orientation histogram** (gradients weighted by magnitude + Gaussian centered on keypoint)
5. Concatenate: $4 \times 4 \times 8 = $ **128-dim descriptor**
6. **Normalize** → unit length (lighting invariance)
7. **Clip** large values to 0.2 → re-normalize (suppress strong gradients from non-linear illumination)

**Matching**:
- For each query keypoint, find best + 2nd-best match in target by Euclidean distance
- **Lowe's ratio test**: accept match only if $d_1 / d_2 < 0.7$ — discriminates good matches from ambiguous ones

**Advantages**: locality, distinctiveness, quantity, efficiency, extensibility.

**LBP (Local Binary Patterns)** — alternative descriptor:
- For each pixel, compare to 8 neighbors → produce 8-bit string → histogram over region
- Texture descriptor (faces, materials)

**Other descriptors to know by name**: SURF, HOG, BRIEF, ORB.

### Ch 13 — Motion Analysis (40 min) — Tier 2
**Guest lecture by Luka Čehovin Zajc.** May or may not be heavy on exam — bias your time toward Tier 1 if pressed.

**Optical flow**: estimate motion vector $(u, v)$ for each pixel between consecutive frames.

**Brightness constancy assumption**: $I(x, y, t) = I(x + u, y + v, t + 1)$

**Optical Flow Constraint Equation (OFCE)** — Taylor-expand:
```math
I_x \cdot u + I_y \cdot v + I_t = 0
```

This is **1 equation, 2 unknowns** → the **aperture problem** (can only determine motion perpendicular to the gradient, not along it).

**Lucas-Kanade (LK)** — solve via local least-squares over a patch:
- Assume constant flow $(u, v)$ in a small window
- Stack OFCEs from many pixels → over-determined system $\mathbf{A} \mathbf{v} = \mathbf{b}$
- Solve normal equations: $\mathbf{A}^\top \mathbf{A} \mathbf{v} = \mathbf{A}^\top \mathbf{b}$
- $\mathbf{A}^\top \mathbf{A}$ is the **structure tensor** (same as Harris!) — well-conditioned at corners

**Horn-Schunck** — alternative global method with smoothness regularizer.

**Pyramidal LK** — handle large motions via coarse-to-fine.

**Visual tracking**:
- **Model-free** (no prior class knowledge) vs **model-based**
- **Single-object** vs **multi-object**
- Two components: **appearance model** + **motion model**
- Trackers to know by name: **MeanShift**, **Kalman filter**, **KCF (Kernelized Correlation Filter)**, **SiamFC** (deep), **MOSSE**
- **Multi-object tracking (MOT)**: tracking-by-detection + bipartite matching (Hungarian algorithm)

### Ch 14 — Deep Learning for CV (20 min) — Tier 3
- **Gestalt theory**: humans see objects, not pixels — proximity / similarity / common fate / region / symmetry.
- **Image segmentation goal**: divide image into meaningful regions.
- **Segmentation as clustering**: pixels as $[R, G, B, x, y]$ feature vectors.
  - **K-Means**: needs $K$, sensitive to init.
  - **Mean Shift**: KDE mode-seeking, bandwidth parameter, no $K$ needed.
- **CNN building blocks**: convolution + receptive field, pooling (max/avg), upsampling / transposed conv.
- **U-Net** (encoder-decoder + **skip connections**): contracting path ↓spatial ↑channels → bottleneck → expanding path ↑spatial ↓channels; skips concatenate encoder maps to recover spatial detail. Used heavily in biomedical segmentation.
- **Training**: pixel-wise softmax + cross-entropy; aggressive augmentation (elastic deformation) for small datasets.

### Ch 15 — DL: Object Localization & Detection (35 min) — Tier 2 ★ *new*

**Task ladder**:
- **Classification** — whole image → label
- **Classification + localization** — label + 1 bounding box (single object)
- **Object detection** — label + bbox for **multiple objects** (often different classes)

**Output vector** for classification + localization:
```math
y = [P_c, b_x, b_y, b_w, b_h, c_1, c_2, \ldots, c_C]^\top
```
- $P_c$ = probability that an object is present
- $(b_x, b_y, b_w, b_h)$ = bbox params (normalized)
- $c_i$ = class probabilities
- **Conditional multi-task loss**: when $P_c = 0$, mask out the bbox + class terms.

**Sliding window + FC→Conv trick** (the elegant insight):
- Replace fully-connected layers with equivalent conv layers → a single forward pass over a full image computes all window positions simultaneously (gigantic speed-up).
- Example: 16×16×3 input → 2×2×4 output grid (4 window positions × 4 classes).

**IoU (Intersection over Union)** — the key metric:
```math
\mathrm{IoU} = \frac{|\text{box}_A \cap \text{box}_B|}{|\text{box}_A \cup \text{box}_B|}
```
- Evaluation: IoU > 0.5 = good detection
- Training: assign anchors to ground truth by highest IoU

**Non-Maximum Suppression (NMS)** — algorithm:
1. Discard low-confidence boxes ($p < 0.6$)
2. Pick the highest-confidence remaining box; output it
3. Discard all boxes overlapping it with IoU > 0.5
4. Repeat (per class)

**Anchor boxes** — solve "one-object-per-cell" limitation:
- Output tensor: $S \times S \times A \times (5 + C)$ (S×S grid, A anchors per cell)
- YOLO v2+: k-means anchor design using **distance metric** $d = 1 - \mathrm{IoU}$

**Two-stage vs single-stage detectors**:

| | Two-stage (R-CNN family) | Single-stage (YOLO, SSD) |
|---|---|---|
| Pipeline | RPN → ROI pooling → classify | Grid + direct prediction |
| Speed | Slower | Faster |
| Accuracy | Higher | Lower |
| Examples | R-CNN, Fast R-CNN, Faster R-CNN | YOLO v1-v8, SSD, RetinaNet |

### Ch 16 — DL Model Optimization for Embedded (20 min) — Tier 3 *new*

**Cloud vs edge** — trade-off table:

| Axis | Cloud | Edge |
|---|---|---|
| Latency | network round-trip | local (low) |
| Bandwidth | sends raw data | minimal |
| Privacy | data leaves device | stays local |
| Energy | offloaded | device-bounded |
| Connectivity | needs network | works offline |
| Compute | unlimited | constrained |

**Quantization** — reduce numeric precision:
- Affine map: $q = \mathrm{round}(r/\text{scale}) + \text{zero\_point}$
- **PTQ** (post-training quantization) — quick but slight accuracy drop
- **QAT** (quantization-aware training) — accuracy preserved, slower setup
- FP32 → INT8 = **4× memory saving** + faster inference on int-optimized hardware
- Applications: edge AI, mobile, automotive, IoT/MCUs

**Pruning** — remove weights/channels:
- **Unstructured pruning**: zero individual weights → sparse tensor → needs special engine (SparseDNN/EIE/Tiramisu) to actually speed up. Memory saved on disk, not at inference time.
- **Structured pruning** (filter/channel pruning): remove whole filters → produces a **smaller dense model** that runs anywhere with no hardware mod. **Real-world speedup**.

**Filter selection criteria** (which channels to drop):
- **L1-norm / SFP** (Soft Filter Pruning) — pick lowest-L1 filters
- **FPGM** (Filter Pruning via Geometric Median) — pick filters near the geometric median
- **HRank** — based on feature-map rank
- Learning-based: binary masks learned during training

**Knowledge distillation**: train a small "student" network to mimic a large "teacher" — small model gets close to teacher accuracy.

**Low-rank factorization**: replace a weight matrix $W$ with $W \approx U V^\top$ (smaller matrices), reducing parameters.

**Efficient architectures**: MobileNet (depthwise separable convs), SqueezeNet, EfficientNet.

---

## High-yield cheatsheet (memorize these formulas cold)

**Gradient**: $|\nabla I| = \sqrt{I_x^2 + I_y^2}$, $\theta = \arctan(I_y / I_x)$

**Canny** (5 steps): smooth → gradient → NMS → double-threshold → hysteresis

**Hough polar form**: $\rho = x \cos\theta + y \sin\theta$

**RANSAC**: $N = \log(1-p) / \log(1 - (1-e)^s)$

**Harris response**: $R = \det(M) - k \cdot \mathrm{tr}(M)^2$, $k \approx 0.04$–$0.06$

**Structure tensor**:
```math
M = \sum_W w \begin{bmatrix} I_x^2 & I_x I_y \\ I_x I_y & I_y^2 \end{bmatrix}
```

**DoG ≈ LoG**: $D(\sigma) = L(k\sigma) - L(\sigma)$

**SIFT descriptor**: 4×4 grid × 8 bins = **128-dim**

**Lowe's ratio test threshold**: 0.7

**Optical flow constraint**: $I_x u + I_y v + I_t = 0$

**Lucas-Kanade**: solve $\mathbf{A}^\top \mathbf{A} \mathbf{v} = \mathbf{A}^\top \mathbf{b}$, $\mathbf{A}^\top \mathbf{A}$ = structure tensor

**Detection output vector**: $y = [P_c, b_x, b_y, b_w, b_h, c_1, \ldots, c_C]^\top$

**IoU**: $\mathrm{IoU} = |A \cap B| / |A \cup B|$ — threshold 0.5 = good detection

**NMS threshold**: discard $p < 0.6$ → keep highest → discard overlaps with IoU > 0.5

**Quantization saving**: FP32 → INT8 = **4× memory**, with affine map $q = \mathrm{round}(r/\text{scale}) + \text{zp}$

---

## Tactical advice

1. **Don't read the slides**. Use the `.md` notes — they're far denser. Skim the PDF only when you want to see a specific figure.
2. **Active recall**: after each Tier 1 chapter, close the file and draw the pipeline from memory (Canny's 5 steps, Harris pipeline, SIFT detector pipeline, SIFT descriptor structure).
3. **YouTube backup** — 1 video per Tier 1 chapter max (20 min budget each):
   - "Canny edge detector explained"
   - "Hough transform example"
   - "Harris corner detector explained"
   - "SIFT algorithm step by step"
4. **If you fall behind**, skip Tier 3 entirely (15 min in the morning instead of 1 hour tonight).
5. **Sleep is non-negotiable.** Go to bed by 23:30. Tired exam ≠ knowledge.

**Good luck. The Tier 1 stuff is where 70%+ of the points are.**
