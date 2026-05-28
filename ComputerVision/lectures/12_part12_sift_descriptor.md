# Part 12 — The SIFT Descriptor & Matching

## Bird's eye view

- Each detected SIFT keypoint already has a position (x, y), a scale, and a dominant orientation. The next step is computing a **128-dimensional feature vector** (the descriptor/signature) that uniquely identifies it.
- The descriptor is built inside a **16×16 pixel window** centred on the keypoint, sampled at the keypoint's scale from the **Gaussian-blurred image** of that octave (not the DoG, not the original).
- The window is divided into a **4×4 grid of cells**. For each cell an **8-bin gradient orientation histogram** is computed. Concatenating all histograms gives **4×4×8 = 128 values**.
- Orientation invariance: all gradient orientations are expressed **relative to the keypoint's dominant orientation**, so rotating the image just rotates the whole window — the relative angles stay constant.
- Scale invariance: the physical size of the 16×16 window is adapted to the keypoint's scale (proportional to sigma), so the same region of the scene is always covered.
- **Normalization** (L2 norm → clamp at 0.2 → re-normalize) makes the descriptor robust to illumination changes.
- Matching is done by **Euclidean distance** between 128-D vectors; **Lowe's ratio test** (ratio of best-to-second-best distance < 0.8) filters out ambiguous matches.
- Applications span image stitching, video stabilization, object recognition, content-based retrieval, image registration, and copy-move forgery detection.

---

## Detailed notes

### 1. From keypoint to descriptor — the setup

After the SIFT keypoint detection pipeline (DoG extrema → subpixel refinement → orientation assignment), each keypoint carries:
1. Coordinates (x, y) in image space
2. A scale (sigma at which it was detected)
3. A dominant orientation (computed from a gradient histogram over the surrounding region)

The goal now is to compute a **signature/descriptor/feature vector** for each keypoint that uniquely identifies it with respect to all other keypoints across different images.

**Critical implementation detail:** the descriptor is computed in the **Gaussian-blurred image** at the keypoint's own scale and octave — not in the DoG image used for detection, and not in the original unblurred image.

---

### 2. Building the 128-D descriptor — step by step

#### 2.1 The 16×16 sampling window

- A **16×16 pixel window** is placed around the keypoint.
- The window is **rotated** to align with the keypoint's dominant orientation. This rotation is what gives rotation invariance.
- Its physical size in the original image is scaled relative to the keypoint's sigma: larger sigma → larger physical window, ensuring scale invariance.

**Figure description (slide 3):** A 16×16 grid of small squares is shown. The keypoint (red dot) sits at the centre (between cells 8 and 9 in both directions). The entire grid represents the neighbourhood that will be summarized into the descriptor.

#### 2.2 Dividing into a 4×4 grid of cells

- The 16×16 window is divided into **16 cells, each 4×4 pixels**.
- This provides **spatial granularity**: instead of one global histogram that would lose all spatial information, we get 16 local histograms capturing gradient distributions in different parts of the neighbourhood.

**Figure description (slide 4):** The same 16×16 grid; one 4×4 cell in the top-left corner is highlighted with an orange border. An arrow points to a bar chart on a dark background showing 8 blue bars of varying heights, each corresponding to one of the 8 orientation bins. The arrows below the chart iconically show the 8 directions (N, NE, E, SE, S, SW, W, NW).

#### 2.3 The 8-bin orientation histogram per cell

For each of the 16 cells:
1. **Compute gradient** at every pixel inside the cell: magnitude $m(x,y)$ and orientation $\theta(x,y)$.
2. **Optionally apply Gaussian weighting** centred at the keypoint, so pixels near the keypoint contribute more than those at the periphery.
3. Each pixel **votes for a bin**:
   - The 360° range is split into **8 bins at 45° intervals**: 0°, 45°, 90°, 135°, 180°, 225°, 270°, 315°.
   - **Vote weight = gradient magnitude × Gaussian weight**
   - The orientation used for binning is **relative to the keypoint's dominant orientation** — this is what achieves rotation invariance.

#### 2.4 Concatenation to form the 128-D vector

- There are 16 cells, each producing an 8-bin histogram.
- Concatenating all 16 histograms gives: **16 × 8 = 128 values**.
- This is the raw SIFT descriptor vector.

**Why 4×4 cells of 8 bins and not some other configuration?** Lowe's experiments showed this strikes the best balance between distinctiveness (enough spatial/orientation resolution) and robustness (large enough cells to be stable under small geometric distortions).

**Figure description (slide 6):** On the left, the 16×16 window with keypoint at centre. A thick arrow points right to a 4×4 grid of cells where each cell now displays a small starburst/compass-rose icon showing the dominant gradient directions. The keypoint's red dot is visible in the corresponding cell of the 4×4 output grid, illustrating that the spatial layout is preserved.

---

### 3. Descriptor normalization

#### 3.1 First normalization (L2, for illumination invariance)

After concatenation, the raw 128-D vector is normalized to unit length:

$$
\mathbf{d} \leftarrow \frac{\mathbf{d}}{\|\mathbf{d}\|_2}
$$

where $\|\cdot\|_2$ is the Euclidean (L2) norm: $\sqrt{\sum_{k=1}^{128} d_k^2}$.

**Why this achieves illumination invariance:**
- A global brightness change (e.g., multiply all pixel values by a constant c) scales all gradient magnitudes by c.
- After L2 normalization the factor c cancels out completely.
- Only the **relative distribution of gradient strengths** matters, not their absolute values.

#### 3.2 Clipping and re-normalization (for nonlinear illumination changes)

After the first normalization:
1. **Clamp**: any component value > 0.2 is set to 0.2.
2. **Re-normalize** to unit length again.

**Why clipping is needed:**
- A single strong edge crossing a cell (large gradient magnitude in one bin) would dominate the descriptor even after L2 normalization.
- Clamping prevents a few large-magnitude gradients from disproportionately influencing the descriptor.
- The result is more **balanced and distinctive** descriptors.
- This also helps robustness to camera saturation and other nonlinear intensity changes.

**Summary of the full normalization pipeline:**

| Step | Operation | Purpose |
|---|---|---|
| 1 | L2-normalize raw 128-D vector | Linear illumination invariance |
| 2 | Clamp values > 0.2 to 0.2 | Prevent strong edges from dominating |
| 3 | L2-normalize again | Restore unit-length property |

---

### 4. Descriptor matching

#### 4.1 Distance measures between two descriptors

Let H1 and H2 be two 128-D SIFT descriptors.

**Euclidean distance** (most common for SIFT):

$$
d(H_1, H_2) = \sqrt{\sum_{k} \bigl(H_1(k) - H_2(k)\bigr)^2}
$$

Smaller distance = more similar descriptors = better match.

**Histogram intersection** (alternative):

$$
d(H_1, H_2) = \sum_{k} \min\bigl(H_1(k),\, H_2(k)\bigr)
$$

Larger intersection = better match (this is a similarity measure, not a distance).

#### 4.2 Nearest-neighbor matching

Given a query keypoint descriptor Q from image A, find the descriptor D* in image B that minimizes d(Q, D*). This is the **nearest neighbor** of Q in the 128-D descriptor space.

**The problem:** in cluttered or repetitive scenes, the nearest neighbor may just be the least-bad wrong match. The raw distance alone does not tell you whether the match is reliable.

#### 4.3 Lowe's ratio test

Lowe's key insight: a match is reliable if the nearest neighbor is **much closer** than the second-nearest neighbor. Formally:

$$
\frac{d(Q,\, D_{1\text{st}})}{d(Q,\, D_{2\text{nd}})} < \text{threshold}
$$

where $D_{1\text{st}}$ is the nearest neighbor and $D_{2\text{nd}}$ is the second-nearest neighbor.

**Lowe's recommended threshold: 0.8**

- If the ratio < 0.8: accept the match (the nearest neighbor is distinctively closer than the runner-up).
- If the ratio >= 0.8: reject the match (the descriptor is ambiguous — two database descriptors are similarly close, so the correspondence is unreliable).

**Worked example:**

Suppose query descriptor Q has:
- d(Q, D_1st) = 0.35
- d(Q, D_2nd) = 0.50

Ratio = 0.35 / 0.50 = 0.70 < 0.8  →  **accept**

Now suppose:
- d(Q, D_1st) = 0.48
- d(Q, D_2nd) = 0.52

Ratio = 0.48 / 0.52 = 0.92 >= 0.8  →  **reject** (the two candidates are indistinguishably close)

Lowe reported that the ratio test eliminates ~90% of false matches while discarding fewer than 5% of correct matches.

#### 4.4 Efficient nearest-neighbor search in 128-D space

**Sequential scan:** Compare the query descriptor to every descriptor in the database. Correct, but O(N) per query — slow for large databases.

**Multidimensional indexing:** Partition the descriptor space into cells, use geometrical rules to prune cells that cannot contain the nearest neighbor. Examples: R-Tree, SS-Tree, SR-Tree. This reduces the number of comparisons per query significantly.

For CBIR applications with local features:
- The query image produces a set of descriptors {Q1, Q2, ..., Qn}.
- Each Qi is matched to k nearest neighbors in the database (k-NN search).
- Images accumulate votes: if database image X is a NN of many query descriptors, X scores high.
- A statistical threshold (e.g., Page-Hinkley test) decides the cut-off between retrieved and non-retrieved images.

---

### 5. Other image descriptors (Section XIII — context)

The slides briefly introduce **Local Binary Pattern (LBP)** as an alternative to SIFT:

- LBP is a **texture descriptor**, conceptually simpler and faster than SIFT.
- For each pixel: take a 3×3 neighbourhood. Compare each of the 8 neighbours to the centre pixel. Write 1 if neighbour >= centre, else 0. This gives an 8-bit binary number (0–255).
- Over a region (e.g., 16×16), compute the **histogram of LBP values** → this histogram is the descriptor.
- Captures texture patterns (edges, corners, flat areas).
- Variants: rotation-invariant LBP, multi-scale LBP (larger radii).
- Trade-off vs SIFT: LBP is faster and simpler but less robust and distinctive in general image matching tasks.

---

### 6. Applications of image feature matching (Section XIV)

#### 6.1 Image stitching / panorama creation
- **Goal:** combine overlapping photos into a seamless panorama.
- SIFT matches features across overlapping regions → estimates the **homography** (projective transform) between images → warps and blends images.

#### 6.2 Video stabilization
- **Goal:** remove jitter from hand-held videos.
- SIFT tracks keypoints between consecutive frames → estimates motion model (affine transform) → smooths the trajectory and warps frames accordingly.

#### 6.3 Object / instance recognition
- **Goal:** recognize a known object (logo, book cover, product) in a new scene.
- SIFT descriptors from a query image are compared against a database of object descriptors. Robust to scale, viewpoint, and illumination changes.

#### 6.4 Content-based image retrieval (CBIR)
- **Goal:** retrieve visually similar images from a large database given a query image ("query by example").
- Pipeline: extract SIFT descriptors from query → k-NN search against pre-indexed DB descriptors → rank images by number of matching descriptors → apply decision threshold.
- Aggregation techniques: **Bag-of-Words (BoW)** or **VLAD** (Vector of Locally Aggregated Descriptors) compress many local descriptors into a single image-level vector for efficient search.
- Applications: reverse image search, product search in e-commerce, medical image retrieval, museum/artwork archives, mobile visual search (point camera at a movie poster → retrieve metadata).

#### 6.5 Image fingerprinting — copy-move forgery detection
- **Goal:** detect regions of an image that were copied and pasted elsewhere in the same image.
- SIFT matches features within a single image → identifies pairs of highly similar regions → consistent affine transformation among matches confirms forgery.

#### 6.6 Image registration
- **Goal:** align two or more images of the same scene (taken at different times, viewpoints, or modalities).
- SIFT detects and matches features in both images → estimates geometric transform (rigid, affine, or projective).
- Applications: medical imaging (aligning CT and MRI scans), satellite image alignment (multi-temporal or multi-spectral).

---

## Key terms (glossary)

- **SIFT descriptor** — 128-D feature vector summarizing gradient orientation distributions in a 16×16 neighbourhood around a keypoint.
- **16×16 window** — the spatial extent around a keypoint in which the descriptor is computed, at the keypoint's scale.
- **4×4 cell grid** — the 16×16 window divided into 16 sub-regions, each 4×4 pixels, to provide spatial granularity.
- **8-bin orientation histogram** — the per-cell histogram with bins at 0°, 45°, 90°, ..., 315° (45° intervals).
- **Vote weight** — gradient magnitude × Gaussian weight; the contribution of each pixel to its histogram bin.
- **Gaussian weighting** — downweighting of pixels far from the keypoint centre; gives the descriptor centre-emphasis and smoothness.
- **Rotation invariance** — achieved by expressing all orientations relative to the keypoint's dominant orientation before binning.
- **Scale invariance** — achieved by scaling the 16×16 window proportionally to the keypoint's sigma.
- **L2 normalization** — dividing the descriptor vector by its Euclidean norm to achieve unit length; cancels linear brightness changes.
- **Clipping (clamping)** — setting any descriptor component > 0.2 to 0.2 before re-normalizing; prevents dominant gradients from overwhelming the descriptor.
- **Euclidean distance** — $d(H_1,H_2) = \sqrt{\sum_{k}(H_1(k)-H_2(k))^2}$; standard metric for comparing 128-D SIFT vectors.
- **Histogram intersection** — $d(H_1,H_2) = \sum_{k}\min(H_1(k),H_2(k))$; similarity measure (larger = better match).
- **Nearest-neighbor (NN) matching** — finding the descriptor in a database that minimizes Euclidean distance to a query descriptor.
- **Lowe's ratio test** — accept a match only if $d_{1\text{st}} / d_{2\text{nd}} < 0.8$; filters ambiguous matches where two database descriptors are similarly close.
- **Bag-of-Words (BoW)** — aggregation technique that quantizes local descriptors into a visual vocabulary and represents images as frequency histograms over visual words.
- **VLAD** — Vector of Locally Aggregated Descriptors; aggregates residuals of descriptors relative to cluster centroids into a compact image representation.
- **LBP (Local Binary Pattern)** — texture descriptor encoding local intensity structure as an 8-bit binary number per pixel, faster but less distinctive than SIFT.
- **Homography** — projective transformation relating two images of the same planar scene; estimated from SIFT correspondences for image stitching.
- **Page-Hinkley test** — statistical decision criterion used in CBIR to determine the cut-off between retrieved and non-retrieved images based on voting counts.

---

## Exam targets

1. **Derive the 128-D dimension.** Explain the 16×16 window → 4×4 grid of 4×4 cells → 8-bin histogram per cell → 16 × 8 = 128. Know why each design choice was made.

2. **Explain how rotation invariance is built into the descriptor.** Key: orientations are measured relative to the keypoint's dominant orientation, and the window itself is rotated before sampling.

3. **Explain how scale invariance is built into the descriptor.** Key: the window is computed in the Gaussian-blurred image at the keypoint's own octave/scale, and its physical size scales with sigma.

4. **Write out the full normalization pipeline** (L2-normalize → clamp at 0.2 → L2-normalize again) and justify each step in terms of what illumination change it addresses.

5. **State Lowe's ratio test** with the threshold value (0.8) and explain intuitively why comparing to the second-nearest neighbor is better than using an absolute distance threshold.

6. **Apply Lowe's ratio test numerically.** Given two candidate distances, compute the ratio and decide accept/reject.

7. **Compare Euclidean distance and histogram intersection** for descriptor matching: which is a distance (smaller = better), which is a similarity (larger = better).

8. **Describe the CBIR pipeline** using SIFT: extract descriptors from query → k-NN search in DB → vote aggregation → threshold decision.

9. **List four applications** of SIFT-based feature matching and briefly describe how SIFT contributes to each.

10. **Describe LBP** at a level sufficient to contrast it with SIFT: mechanism (binary comparison to neighbours → 8-bit code → histogram), strengths (speed, simplicity), and weaknesses (less distinctive).

---

## Pitfalls

- **The descriptor is NOT computed in the DoG image.** Detection uses DoG; the descriptor is computed in the Gaussian-blurred image at the corresponding scale. Confusing these is a common error.

- **16×16 pixels = 16 cells of 4×4, not 4 cells of 16×16.** The grid structure is 4×4 cells where each cell is 4×4 pixels. Getting this inverted loses the spatial granularity argument.

- **128 = 16 cells × 8 bins, not 16 bins or 4 bins.** The 8 bins cover 360°/8 = 45° each. Common trap: writing 128 = 16 × 16 (wrong) or 128 = 4 × 4 × 8 (correct, since 4×4 = 16 cells).

- **Rotation invariance requires two things, not one:** (a) subtracting the dominant orientation from all gradient orientations AND (b) rotating the sampling window. Mentioning only one is incomplete.

- **Lowe's threshold is 0.8, not 0.5 or 1.0.** The ratio test rejects matches where the ratio >= 0.8; it does NOT threshold the absolute distance.

- **The ratio test compares nearest to second-nearest, not nearest to average.** The second-nearest neighbor specifically represents the "best alternative match."

- **L2 normalization alone is not sufficient** — the clipping step is needed to handle nonlinear illumination effects (e.g., strong edges, camera saturation). Both steps together form the complete normalization.

- **Histogram intersection is a similarity, not a distance.** For histogram intersection, larger value means better match (opposite of Euclidean distance). Do not apply a "smaller is better" rule here.

- **The Gaussian weight is centred at the keypoint, not at each cell's centre.** The weighting is global across the entire 16×16 window, tapering from the centre.
