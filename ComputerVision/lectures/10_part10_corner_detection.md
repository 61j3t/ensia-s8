# Part 10 — Corner Detection (Harris Detector)

## Bird's eye view

- A **corner** (interest point) is a location where intensity changes rapidly in **two independent directions** simultaneously — distinguishing it from a flat region (no change) or an edge (change in only one direction).
- The key insight: classify regions by analysing the **distribution of gradient vectors** (Ix, Iy) inside a local window — flat regions have a tight isotropic cluster, edges a thin elongated ellipse, corners a wide isotropic ellipse.
- The **structure tensor M** (second-moment matrix) is a 2×2 weighted sum of outer products of gradient vectors over a local window; its **eigenvalues λ1, λ2** measure the strength of intensity change along the two principal gradient directions.
- The **Harris Corner Response** R = det(M) − k · trace(M)² avoids computing eigenvalues explicitly; R > threshold signals a corner, R < 0 signals an edge, |R| small signals a flat region.
- The full Harris algorithm: compute gradients → build M at every pixel → compute R → threshold → (optionally) non-maximal suppression to reduce each cluster to a single point.
- Harris is **invariant to translation and rotation**, **partially invariant to scaling** (requires multi-scale analysis for large changes), and **partially invariant to illumination** (uses relative, not absolute, intensity changes).
- Variants: **Shi-Tomasi** (corner if min(λ1, λ2) > threshold — simpler, no heuristic k); **Forstner** (requires both eigenvalues large AND their ratio not too extreme, for precise localisation).

---

## Detailed notes

### 1. What is a corner?

A **corner** is defined as a point in the image where two edges meet — equivalently, a location where image intensity changes rapidly in at least two distinct directions within a small region.

Three region types can be distinguished visually and analytically:

| Region type | Description |
|---|---|
| **Flat** | No significant intensity change in any direction. |
| **Edge** | Significant intensity change in one direction only; the edge runs along the other direction. |
| **Corner** | Significant intensity change in two (or more) distinct directions simultaneously. |

**Why corners matter:**
- Edge detectors perform poorly at corners (the gradient direction is ill-defined exactly at the corner point).
- Corners provide **repeatable, localised interest points** for matching across images — essential for stereo, motion estimation, and recognition.

### 2. Intuition: gradients around the three region types

Exactly at a corner the gradient is ill-defined. However, in the **region around** a corner, gradients have two or more different well-defined orientations. Corner detection therefore works on a **local window** rather than a single pixel.

**Key observation (slide 5–7):** Looking at the partial derivatives Ix = dI/dx and Iy = dI/dy for the three region types:

- **Flat region:** Both Ix and Iy are uniformly small (near-zero noise). In the (Ix, Iy) scatter plot, points cluster tightly near the origin in a small circle.
- **Edge region:** One of Ix or Iy is large while the other is near-zero (depending on edge orientation). The scatter is an elongated ellipse pointing along one axis.
- **Corner region:** Both Ix and Iy take significant non-zero values in different sub-areas of the window. The scatter is a wide, roughly circular (isotropic) spread.

The **distribution of gradient vectors in the window characterises the region type**. The problem reduces to quantifying that distribution with a small number of parameters.

### 3. From gradient distribution to the structure tensor

**Step 1 — PCA interpretation.**
The problem of finding the dominant directions of the gradient distribution is equivalent to a **Principal Component Analysis (PCA)** on the gradient vectors (Ix(x,y), Iy(x,y)) computed at all pixels inside the window. PCA yields two eigenvalues:

- **λ1**: variance (inertia) in the direction of maximum spread — the principal axis.
- **λ2**: variance in the direction of minimum spread (perpendicular to the first).

**Step 2 — The structure tensor M (second-moment matrix).**
Rather than running PCA directly, the Harris detector encodes the gradient distribution in the **structure tensor M**, a 2×2 symmetric matrix computed for each pixel using a weighted sum over a local window:

```
M = sum_{x,y in window}  w(x,y) * [ Ix²(x,y)        Ix(x,y)·Iy(x,y) ]
                                    [ Ix(x,y)·Iy(x,y)  Iy²(x,y)       ]
```

Where:
- **Ix(x,y)** and **Iy(x,y)** are the image partial derivatives (gradients) in x and y.
- **w(x,y)** is a weighting function — typically a **Gaussian kernel**, which gives more weight to pixels near the centre of the window. The Gaussian width determines the **scale** at which corners are detected.
- The window is typically 3×3 or 5×5 pixels, or a Gaussian-weighted neighbourhood.

M is computed **at every pixel** in the image (every pixel is a candidate corner).

**Relationship to covariance:** M is mathematically very similar to the covariance matrix of gradient vectors in the window:
```
Cov(Ix, Iy) = E[(∇I)(∇I)^T]
```
This is why the eigenvalues of M directly describe the shape of the gradient distribution ellipse.

**Reference:** C.G. Harris and M.J. Stephens, "A Combined Corner and Edge Detector," Alvey Vision Conference, 1988.

### 4. Eigenvalue interpretation of M

The eigenvalues λ1 and λ2 of M give the **strength of intensity change in the two orthogonal principal directions**.

| λ1, λ2 values | Region type | Gradient scatter shape |
|---|---|---|
| Both small (λ1 ≈ λ2 ≈ 0) | **Flat** | Small circle near origin |
| One large, one small (λ1 >> λ2 or λ2 >> λ1) | **Edge** | Thin elongated ellipse |
| Both large (λ1 ≈ λ2, both high) | **Corner** | Large, roughly circular ellipse |

**Decision diagram (the λ1–λ2 space):**
Imagine a plot with λ1 on the horizontal axis and λ2 on the vertical axis:
- **Bottom-left region** (both small): flat — no features.
- **Lower-right region** (λ1 >> λ2): edge along the y-direction.
- **Upper-left region** (λ2 >> λ1): edge along the x-direction.
- **Upper-right region** (both large, roughly equal): corner.

### 5. The Harris corner response function R

Computing eigenvalues explicitly is expensive. Harris proposes an empirically designed scalar **Corner Response Function** that avoids eigendecomposition:

```
R = det(M) − k · (trace(M))²
```

Where:
- **det(M) = λ1 · λ2** (product of eigenvalues — also equals IxIx·IyIy − (IxIy)² in terms of the matrix entries)
- **trace(M) = λ1 + λ2** (sum of eigenvalues — also equals IxIx + IyIy)
- **k** is a sensitivity constant, empirically chosen in the range **[0.04, 0.06]**

**Interpreting R:**

| R value | Meaning |
|---|---|
| R large and positive | **Corner** (both λ1, λ2 large → det large, trace² is also large but det dominates) |
| R large and negative | **Edge** (one eigenvalue dominates → det ≈ 0, trace² large → R negative) |
| |R| small | **Flat region** (both eigenvalues small → both terms near zero) |

Expanding R in terms of eigenvalues:
```
R = λ1·λ2 − k·(λ1 + λ2)²
```

The k term penalises large traces (i.e., edge-like responses where one eigenvalue dominates). A larger k makes the detector less sensitive (fewer corners detected); a smaller k makes it more sensitive.

**Why not just use det(M)?** det(M) = λ1·λ2 alone would be large for a corner, but would be near zero for an edge (one eigenvalue small). The trace² penalty ensures that when one eigenvalue dominates (edge), R is driven negative rather than simply being small. This provides cleaner separation.

**Harris = PCA on local gradient vectors, looking for locations where gradient energy is high in multiple directions.**

### 6. The Harris algorithm (step-by-step)

1. **Compute image gradients** at every pixel:
   - Ix(x,y) = dI/dx (e.g. using Sobel or finite differences)
   - Iy(x,y) = dI/dy

2. **Compute the structure tensor M at every pixel:**
   ```
   M = sum_{x,y in window}  w(x,y) * [ Ix²    Ix·Iy ]
                                       [ Ix·Iy  Iy²   ]
   ```
   The Gaussian window size (sigma) determines the **scale** of the detected corners.

3. **Compute the corner response score R at every pixel:**
   ```
   R = det(M) − k · (trace(M))²
   ```

4. **Threshold R:** keep only pixels where R > T (user-chosen threshold).
   - This produces clusters of high-R pixels at each real corner.

5. **(Optional) Non-maximal suppression (NMS):** reduce each cluster to a single point.
   - Slide a window of size k over the R-map.
   - At each position, if the centre pixel has the maximum R value within the window, keep it (label positive); otherwise suppress it (label negative).
   - Result: one detected corner point per cluster.

**BBC logo example (slides 16–19):**
- Input: the BBC logo (white letters on black background, with rectangular borders — lots of right-angle corners).
- R-map: the map shows very high R values (bright yellow in a heat-map) at the sharp corners of the letter outlines and the surrounding rectangles; edges appear as moderate negative R (dark red/red in the map); flat areas (black background) are near-zero.
- After thresholding R > T: a sparse binary map with small clusters of white pixels at each corner.
- After NMS: a single bright dot per corner, correctly localised at rectangle corners and letter junctions.

**Architectural image example (slide 21):**
- Input: grayscale photo of a building facade with many right-angle structural elements.
- R-map: high-response points visible at every junction of horizontal/vertical structural lines.
- After thresholding and NMS: detected corners (orange dots) placed precisely at structural junctions.

### 7. Properties and invariances of the Harris detector

**Invariance** in computer vision means the detector's output remains unchanged (or reliably consistent) when the image undergoes certain transformations.

#### 7.1 Translation invariance — YES

The corner response R depends on **local image gradients** (derivatives of intensity), not on the absolute pixel coordinates. Shifting the image shifts the detected corners by the same amount — the set of corners is preserved.

#### 7.2 Rotation invariance — YES

Harris uses the **eigenvalues of M** (via det and trace). Eigenvalues are invariant to rotation of the coordinate system because the structure tensor M transforms as M' = R·M·R^T under rotation R, and eigenvalues are preserved under this similarity transform. The shape of the gradient distribution ellipse does not change under rotation — only its orientation changes, which does not affect λ1 or λ2.

#### 7.3 Scale invariance — PARTIAL (limited)

Harris is **partially invariant to scaling**. For small scaling changes, gradients change similarly and the detector performs well. However, for large scale changes, a corner at one scale may appear as a smooth edge or flat region at another scale. Full scale invariance requires **multi-scale analysis** (running the detector at multiple resolutions / Gaussian pyramid levels).

#### 7.4 Illumination invariance — PARTIAL

Harris is **somewhat invariant to illumination changes** because gradients measure **relative** (local) intensity changes rather than absolute intensity values. A global brightness shift (adding a constant to all pixels) does not affect gradients, so corners are detected consistently.

However, **large non-linear illumination changes** (e.g., hard shadows, different camera exposure) alter local contrast ratios and can reduce detector performance.

### 8. Performance evaluation metrics

Two standard metrics for assessing corner detector quality:

**Repeatability (Consistency):** The ability to reliably detect the same features across multiple images of the same scene under varying conditions (rotation, scaling, illumination). Measured by applying the detector to multiple transformed images and counting how many of the same corners are re-detected.

**Localization Accuracy:** The precision with which the detector identifies the exact position of each corner. Measured by comparing detected positions to ground-truth corner positions.

Additional metrics: **precision** (fraction of detections that are true corners) and **recall** (fraction of true corners that are detected).

### 9. Variant detectors based on the structure tensor

All three detectors below share the same structure tensor M and its eigenvalues.

| Detector | Criterion | Main idea | Special feature |
|---|---|---|---|
| **Harris (1988)** | R = λ1·λ2 − k·(λ1+λ2)² > T | Combines eigenvalues with an empirical heuristic | Sensitive to parameter k |
| **Shi-Tomasi (1994)** | min(λ1, λ2) > threshold | Smallest eigenvalue must be large enough | Simpler, no heuristic k; more robust for tracking |
| **Forstner (1987)** | Both eigenvalues large AND their ratio not too extreme | Measures "interest" and "accuracy" separately | Designed for precise photogrammetric localisation; ensures isotropy |

**Shi-Tomasi detail:** Instead of the Harris response R, Shi and Tomasi declare a point a corner if **λ_min > threshold**, where λ_min = min(λ1, λ2). This directly asks "is the weakest direction also strong?" — a corner must have strong gradients in all directions. Simpler and avoids the k parameter entirely.

**Forstner detail:** A point is a good feature if (a) both eigenvalues are large (like Harris and Shi-Tomasi) and (b) their ratio λ1/λ2 is not too extreme (ensuring isotropy — the feature is well-localised in all directions). Forstner was designed specifically for precise spatial localisation in photogrammetry.

---

## Key terms (glossary)

- **Corner / Interest point** — location where intensity changes rapidly in two independent directions.
- **Flat region** — region with no significant gradient in any direction; both λ1, λ2 small.
- **Edge region** — region with strong gradient in one direction; one eigenvalue large, one small.
- **Structure tensor / Second-moment matrix M** — 2×2 symmetric matrix summarising the distribution of gradient vectors in a local window; M = sum w(x,y) · [[Ix², Ix·Iy],[Ix·Iy, Iy²]].
- **Weighting function w(x,y)** — typically a Gaussian; gives more importance to pixels near the window centre.
- **Eigenvalues λ1, λ2 of M** — strengths of intensity change in the two principal (orthogonal) gradient directions.
- **Corner Response Function R** — scalar score R = det(M) − k·trace(M)²; large positive = corner, negative = edge, near zero = flat.
- **det(M)** — λ1·λ2; product of eigenvalues.
- **trace(M)** — λ1+λ2; sum of eigenvalues.
- **k (sensitivity constant)** — empirical parameter in [0.04, 0.06] controlling the balance between corner and edge detection.
- **Non-Maximal Suppression (NMS)** — post-processing step that reduces clusters of high-R pixels to a single local maximum point.
- **Invariance** — the property that detections remain reliable under specific image transformations.
- **Repeatability** — ability of a detector to find the same features across different views or conditions.
- **Localization accuracy** — precision with which the detector pinpoints the exact corner position.
- **Shi-Tomasi criterion** — a corner if min(λ1, λ2) > threshold; avoids the k heuristic.
- **Forstner detector** — detects corners requiring both eigenvalues large and their ratio balanced.
- **PCA on gradients** — conceptual interpretation of what M computes; λ1/λ2 are the principal component variances of the gradient vector cloud.

---

## Exam targets

1. **Define a corner** and distinguish it from a flat region and an edge using the concept of intensity changes in two directions.

2. **Explain the gradient distribution perspective:** describe what the (Ix, Iy) scatter plot looks like for flat, edge, and corner regions, and what λ1/λ2 signify geometrically.

3. **Write and explain the structure tensor M:**
   ```
   M = sum_{x,y in window} w(x,y) * [ Ix²    Ix·Iy ]
                                      [ Ix·Iy  Iy²   ]
   ```
   State what w(x,y) is, why a Gaussian is used, and how M relates to the covariance of gradient vectors.

4. **Write the eigenvalue classification table:**

   | λ1, λ2 | Region |
   |---|---|
   | Both small | Flat |
   | One large, one small | Edge |
   | Both large | Corner |

5. **Write and derive the corner response R:**
   ```
   R = det(M) − k · (trace(M))²
     = λ1·λ2 − k·(λ1+λ2)²
   ```
   Explain the sign of R for each region type. State the range of k.

6. **Describe the full Harris algorithm** in 4 steps: gradient computation → M at each pixel → compute R → threshold + NMS.

7. **Explain Non-Maximal Suppression:** why it is needed (corners detected as clusters), how it works (sliding window, keep local maximum).

8. **State the invariance properties of Harris** and justify each:
   - Translation: YES (R depends on local derivatives, not position)
   - Rotation: YES (eigenvalues invariant to coordinate rotation)
   - Scaling: PARTIAL (multi-scale needed for large changes)
   - Illumination: PARTIAL (gradients use relative changes; fails for non-linear shifts)

9. **Compare Harris, Shi-Tomasi, and Forstner** — criterion, main idea, and key difference.

---

## Pitfalls

- **Confusing det(M) and trace(M) in the formula.** The formula is R = det(M) − k·trace(M)² — det comes first, trace is squared. Not the other way round.
- **Forgetting that M is computed per pixel, not globally.** Every pixel gets its own M using the gradients in its surrounding window.
- **Misidentifying edges from R.** An edge gives R < 0 (negative), not just small R. Flat gives |R| small. Corner gives R large and positive. Know all three cases.
- **The sign of R for edges.** When one eigenvalue dominates (edge), det ≈ 0 but trace is large, so −k·trace² dominates and R is negative.
- **k has no units and is empirical.** Do not try to derive k from first principles — it is chosen experimentally in [0.04, 0.06].
- **Harris is NOT scale invariant.** A common mistake is claiming full scale invariance. Harris requires multi-scale analysis for large scale changes.
- **Harris is NOT fully illumination invariant.** It handles additive global shifts (because gradients filter out constants) but not multiplicative or non-linear changes (shadows, exposure changes).
- **Shi-Tomasi vs Harris:** Shi-Tomasi does NOT use R. It directly thresholds min(λ1, λ2). This means it does need to compute eigenvalues explicitly (or equivalent), unlike Harris which avoids that via det and trace.
- **Non-maximal suppression is optional but important.** Without it, one physical corner produces many nearby detections (a cluster of pixels above threshold). NMS collapses the cluster to one point.
- **The weighting function w(x,y) is not uniform.** A flat (box) window is the simplest but a Gaussian window is preferred — it reduces the impact of noise at the window edges and makes the response smoother. The Gaussian sigma determines detection scale.
- **Gradient direction is ill-defined exactly at the corner point** itself. Harris works around this by using the distribution in the surrounding region — not the corner pixel in isolation.
