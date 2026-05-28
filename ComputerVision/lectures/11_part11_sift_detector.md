# Part 11 — SIFT: Introduction & Detector

---

## Bird's eye view

- **SIFT** (Scale-Invariant Feature Transform) was introduced by David G. Lowe (IJCV, 2004, ~78 000 citations) and transformed object recognition by providing features invariant to **scale, rotation, and illumination**.
- The method has two stages: a **detector** (find keypoints + assign scale and orientation) and a **descriptor** (encode local appearance as a feature vector). This part covers only the detector.
- Blob detection is the core idea: SIFT treats interest points as **blobs** — regions visually distinct from their surroundings — localized in both space and scale.
- The **scale space** $L(x,y,\sigma) = G(x,y,\sigma) * I(x,y)$ is searched for extrema via the **Difference-of-Gaussians (DoG)**, which efficiently approximates the Normalized Laplacian of Gaussian (NLoG).
- The Gaussian pyramid is organized into **octaves** (each halving image resolution) with several **blur levels** (controlled by $\sigma$) per octave; DoG images are computed between adjacent blur levels.
- **Extrema detection** compares each DoG sample against its 26 neighbors (3×3 in the same scale ± 3×3 in adjacent scales) to find scale-space extrema as keypoint candidates.
- Each keypoint is assigned a **characteristic scale** ($\sigma^*$ at which the NLoG response peaks) and a **principal orientation** from a weighted gradient histogram, giving full invariance to scale and rotation.
- Knowing scale and orientation allows rotation and scale variations between images to be **undone**, enabling reliable matching; the ratio of blob sizes between two views equals $\sigma_1/\sigma_2$.

---

## Detailed notes

### 1. Motivation — why recognition is hard

Computer vision operates at three levels: low (image → image, e.g. restoration, denoising), medium (image → features/descriptors, e.g. segmentation, shape), and high (image → concepts, e.g. scene understanding). SIFT lives at the **medium level**.

**The recognition challenge:** Given a template/query image (e.g. a DVD cover), find it inside a cluttered "rich" target scene containing many objects photographed under different conditions. The query and the target version of the same object differ in:

- **Scale** — object appears larger or smaller
- **Rotation** — object photographed at a different angle
- **Illumination** — different lighting conditions (brightness, shadows)
- **Occlusion** — part of the object is hidden

**Solution strategy:** Rather than comparing whole images, detect **local interest points** in both images, describe each with an invariant feature vector (signature/descriptor), and match descriptors across images.

**General idea (Lowe):** Transform image content into local feature coordinates that are invariant to translation, rotation, scale, and other imaging parameters. The slide shows a toy truck detected from two very different viewpoints: keypoints (yellow boxes) from the template are extracted as patches, and the same patches are found in the target image despite viewpoint change.

---

### 2. SIFT — overview and invariances

**Reference:** David G. Lowe, *Distinctive image features from scale-invariant keypoints*, International Journal of Computer Vision, 60(2), 2004.

**Invariances provided:**
- Scale
- Rotation
- Illumination (partial — robust to moderate lighting change)

While still providing **very good spatial localization** (well-defined position).

**SIFT consists of two parts:**
1. A **detector** of interest areas: keypoint localization (covered here)
2. A **local signature computation method** (the descriptor, covered in Part 12)

**Terminology:** Signature = descriptor = feature vector (all synonyms used in the course).

**Advantages of SIFT features:**

| Property | Meaning |
|---|---|
| **Locality** | Features are local → robust to occlusion and clutter; no prior segmentation needed |
| **Distinctiveness** | Individual features can be matched against large databases |
| **Quantity** | Many features generated even for small objects |
| **Efficiency** | Close to real-time performance |
| **Extensibility** | Framework easily extended to other feature types |

---

### 3. What is an interest point?

An **interest point** (keypoint) must:
- Contain **rich image content** within its local window (brightness/color variations)
- Have a **well-defined representation** (signature) that enables comparison/matching with other points
- Have a **well-defined position** in the image (localizability)
- Be **invariant to rotation and scaling**
- Be **insensitive to lighting changes**

**Why edges/lines are poor interest points:** A point on a straight edge is ambiguous — you cannot tell where along the edge it lies after a translation parallel to the edge. The visual content of an edge patch looks almost identical to a shifted patch on the same edge.

**What are blobs?**

A **blob** is a region in an image that is visually distinct from its surroundings, typically in terms of brightness, color, or texture. Blobs can represent corners, spots, junctions, or object parts.

Formally: blobs are areas where **image intensity remains relatively constant** inside but changes significantly at the boundaries.

**Characteristics of blobs:**
- Localized in both **space** and **scale**
- Not necessarily circular (though often approximated as circular for analysis)
- **Scale-dependent**: the same feature looks different at different scales

**Why blobs are good interest points:** They have a fixed position, consistent shape/appearance, and a definite size. This makes them reliable across viewpoints. The goal is:

1. Locate the blob (position $x^*$, $y^*$)
2. Determine its size (scale $\sigma^*$)
3. Determine its orientation
4. Formulate an invariant description

---

### 4. Principle of blob detection — 1D case

Before extending to 2D images, the course builds intuition with 1D signals.

**1D blob structures** look like localized bumps or dips (flat-topped or peaked, positive or negative) of varying widths. They represent intensity structures at a given scale.

**Detection strategy using the normalized 2nd derivative:**

Define the **normalized ($\sigma$-normalized) 2nd derivative** of the Gaussian-smoothed signal:

$$
\sigma^2 \cdot \frac{\partial^2 n_{\sigma}}{\partial x^2} * f(x)
$$

where $n_{\sigma}$ is the 1D Gaussian with standard deviation $\sigma$, and $*$ denotes convolution.

**Why normalize by $\sigma^2$?** Without normalization, the response magnitude falls off as $\sigma$ increases (larger blobs produce weaker raw 2nd derivatives). The $\sigma^2$ factor compensates, making the response comparable across scales.

**Key result from the 1D blob analysis (slides 20–27):**

The course shows three 1D blobs (A, B, C) of increasing width. For each blob, at a fixed $\sigma$:
- Row 1: $f(x)$ — the original signal
- Row 2: $n_{\sigma}$ — Gaussian at that $\sigma$
- Row 3: $\partial^2 n_{\sigma}/\partial x^2$ — 2nd derivative of Gaussian (a "W" shaped kernel)
- Row 4: $(\partial^2 n_{\sigma}/\partial x^2) * f(x)$ — convolution result (unnormalized)
- Row 5: $\sigma^2 \cdot (\partial^2 n_{\sigma}/\partial x^2) * f(x)$ — $\sigma$-normalized result

As $\sigma$ increases across the slides, for a given blob:
- When $\sigma$ is small relative to the blob width: the convolution produces oscillating responses, and the normalized 2nd derivative shows a **clear minimum** (negative peak) right at the blob center.
- When $\sigma$ matches the blob width: the minimum is strongest.
- When $\sigma$ is large relative to the blob: the response flattens out (the blob is "subsumed" by the large Gaussian).

**Conclusion:** Each blob is detected as a **minimum or maximum of the $\sigma$-normalized 2nd derivative in the $(x, \sigma)$ joint space.** The blob is localized at:
- $x^*$ = position of the extremum → **blob position**
- $\sigma^*$ = $\sigma$ at which the extremum occurs → **characteristic scale** = **blob size**

**Characteristic scale definition:** The $\sigma$ at which the $\sigma$-normalized 2nd derivative of the signal attains its minimum/maximum.

**Critical property:** The characteristic scale is **proportional to the blob size**:

$$
\frac{\text{size of blob A}}{\text{size of blob B}} = \frac{\sigma^*_A}{\sigma^*_B} \qquad , \qquad \frac{\text{size of blob B}}{\text{size of blob C}} = \frac{\sigma^*_B}{\sigma^*_C}
$$

**1D blob detection algorithm:**

1. Given a 1D signal $f(x)$
2. Compute $\sigma^2 \cdot (\partial^2 n_{\sigma}/\partial x^2) * f(x)$ at $k$ different scales $(\sigma_0, \sigma_1, \ldots, \sigma_k)$
3. Find $(x^*, \sigma^*) = \arg\max_{(x,\sigma)} \left|\sigma^2 \cdot \frac{\partial^2 n_{\sigma}}{\partial x^2} * f(x)\right|$
   - $x^*$ = the blob position
   - $\sigma^*$ = the characteristic scale (blob size)

---

### 5. Extension to 2D — blob detection in images

In 2D, the 2nd derivative generalizes to the **Laplacian**:

$$
\nabla^2 I = \frac{\partial^2 I}{\partial x^2} + \frac{\partial^2 I}{\partial y^2}
$$

The **Normalized Laplacian of Gaussian (NLoG)** is used:

$$
\text{NLoG} = \sigma^2 \cdot \nabla^2 n_{\sigma}
$$

where $n_{\sigma}$ is a 2D isotropic Gaussian. Visually:
- The Gaussian $n_{\sigma}$ is a smooth bell-shaped surface (peak in center)
- The LoG $\nabla^2 n_{\sigma}$ is a "Mexican hat" shape (negative central region, positive ring around it)
- The NLoG $\sigma^2 \nabla^2 n_{\sigma}$ is the $\sigma$-normalized version with preserved amplitude across scales

**2D blob detection algorithm:**

1. Given an image $I(x,y)$
2. Convolve $I$ with the NLoG at $k$ different scales $(\sigma_0, \sigma_1, \ldots, \sigma_k)$
3. Find $(x^*, y^*, \sigma^*) = \arg\max_{(x,y,\sigma)} \left|\sigma^2 \cdot \nabla^2 n_{\sigma} * I(x,y)\right|$
   - $(x^*, y^*)$ = the blob position in the image
   - $\sigma^*$ = the characteristic scale (blob size)

**Scale space $S(x,y,\sigma)$:** A stack of images created by filtering with different values of $\sigma$:

$$
S(x, y, \sigma) = n(x, y, \sigma) * I(x, y)
$$

(Reference: Tony Lindeberg, Feature Detection with Automatic Scale Selection, IJCV, 1998.)

As $\sigma$ increases, the image becomes progressively blurrier (higher scale, lower resolution appearance). The NLoG response is computed across this whole stack.

**Illustration:** Applying NLoG to an image of a person at scales $\sigma_0$ through $\sigma_3$, the response $|\sigma^2 \nabla^2 S|$ at a specific point (e.g. the nose) peaks at $\sigma_1$ — this is the characteristic scale for that blob. At a nearby non-blob point, no clear peak exists across scales ("no significant maximum => no blob").

---

### 6. DoG — the efficient approximation used by SIFT

Computing NLoG at many scales is expensive. Lowe's key insight: the **Difference of Gaussians (DoG)** is an efficient approximation.

**DoG definition:**

$$
\text{DoG} = (n_{s\sigma} - n_{\sigma}) \approx (s - 1) \cdot \sigma^2 \cdot \nabla^2 n_{\sigma}
$$

where $s > 1$ is the scale factor between adjacent Gaussian levels (a constant multiplicative step).

Therefore:

$$
\text{DoG} \approx (s - 1) \cdot \text{NLoG}
$$

The shape of DoG and the Laplacian are nearly identical (the slide overlays the two curves and shows excellent agreement). Since $(s - 1)$ is just a constant scaling factor, **extrema of DoG coincide with extrema of NLoG** — the same keypoints are found.

**Why DoG is preferred:** Gaussian smoothing at adjacent scales is computed anyway to build the Gaussian pyramid. Subtracting adjacent Gaussian images is essentially free compared to explicitly computing second derivatives.

---

### 7. The Gaussian pyramid and DoG pyramid

SIFT builds a **scale-space pyramid** with two nested levels of scale:

**Level 1 — Gaussian smoothing scale ($\sigma$):**
- At a fixed image resolution, apply Gaussian blurs with increasing $\sigma$ values.
- This simulates observing the image at different "levels of detail" without changing pixel count.
- Produces the **Gaussian pyramid within one octave**.

**Level 2 — Image resolution scale (octaves):**
- After covering a range of $\sigma$ values within an octave, **downsample the image by factor of 2** (width and height halved).
- This starts a **new octave** — a coarser version of the image.
- In each new octave, $\sigma$ starts again from a base level but represents a **higher absolute scale** because the pixel spacing is coarser.

**Structure within one octave (fixed spatial resolution):**

$$
\text{Gaussian images at scales:} \quad \sigma,\ s\sigma,\ s^2\sigma,\ \ldots,\ s^k\sigma
$$

$$
\text{DoG images (differences):} \quad D_1 = s\sigma - \sigma,\quad D_2 = s^2\sigma - s\sigma,\quad \ldots
$$

Each octave produces $(k - 1)$ DoG images from $k$ Gaussian images. Extrema can only be found in the interior DoG levels (need one above and one below for the 3D neighborhood check), so typically 3–4 DoG images per octave are used for detection.

**Pyramid diagram description:** The full pyramid looks like a stack of layers. On the left side are the Gaussian blurred images (getting progressively blurrier within each octave, then restarting at coarser resolution). On the right side are the DoG images (one fewer than the Gaussians per octave). Arrows show subtraction between adjacent Gaussian layers. The process repeats across multiple octaves.

**Within a single octave (detailed illustration, slides 41–42):**

Starting from the original image, blurs are applied at scales $\sigma$, $s\sigma$, $s^2\sigma$, …, $s^k\sigma$. Pairwise differences produce the DoG stack. Local extrema are found in the DoG stack (highlighted yellow region = the layers used for 3D extrema search). The final output is a set of **interest point candidates** (red dots scattered across the image at different DoG levels), which after thresholding become the SIFT interest points (green dots).

**Key formula — displayed blob radius:**

Each keypoint has a characteristic scale $\sigma$ and comes from a specific octave. When visualizing the blob back in the original image, the circle radius is:

$$
r_{\text{display}} \approx \sqrt{2} \cdot \sigma \cdot 2^{\text{octave}}
$$

This accounts for the downsampling factor of the octave.

---

### 8. Scale-space extrema detection (3×3×3 neighborhood)

Once the DoG pyramid is built, **local extrema** are detected across the 3D scale-space $(x, y, \text{scale})$:

- Each DoG sample point is compared to its **26 neighbors**:
  - 8 neighbors in the same scale level (3×3 grid minus center)
  - 9 neighbors in the scale level above (3×3 grid)
  - 9 neighbors in the scale level below (3×3 grid)
  - Total: 8 + 9 + 9 = **26 neighbors**

- A point is a candidate keypoint if it is **larger than all 26 neighbors** (local maximum) or **smaller than all 26 neighbors** (local minimum).

- A **threshold** is applied to reject extrema with small DoG responses (they are likely to be noise or low-contrast regions).

- The surviving candidates are the **SIFT interest point candidates**.

**Pyramid figure description:** Three adjacent horizontal "grids" represent three DoG layers at the same octave. The candidate point X sits in the middle layer. Green oval marks cover all neighbors in the top and bottom layers. The X is compared against all of them simultaneously.

---

### 9. Keypoint localization — sub-pixel/sub-scale refinement

Detected extrema are at discrete pixel and scale locations. Lowe refines them using a **Taylor expansion** of the DoG function $D(x, y, \sigma)$ around the candidate:

$$
D(\mathbf{x}) \approx D + \left(\frac{\partial D}{\partial \mathbf{x}}\right)^T \mathbf{x} + \frac{1}{2} \mathbf{x}^T \frac{\partial^2 D}{\partial \mathbf{x}^2} \mathbf{x}
$$

Setting the derivative to zero gives the refined offset $\hat{\mathbf{x}}$. If $|\hat{\mathbf{x}}| > 0.5$ in any dimension, the candidate is moved to the adjacent sample and the process is repeated.

**Rejection criteria after refinement:**

1. **Low contrast:** If $|D(\hat{\mathbf{x}})| < \text{threshold}$ (Lowe uses 0.03), the keypoint is discarded as an unstable low-contrast point.

2. **Edge response:** Blobs on edges (elongated structures) are unstable — small position errors along the edge lead to large descriptor changes. SIFT checks the **ratio of principal curvatures** using the Hessian matrix $H$ of the DoG image:

$$
H = \begin{pmatrix} D_{xx} & D_{xy} \\ D_{xy} & D_{yy} \end{pmatrix}
$$

The ratio of eigenvalues $\lambda_1/\lambda_2$ can be computed from the trace and determinant. The keypoint is rejected if:

$$
\frac{(\text{Tr}(H))^2}{\text{Det}(H)} > \frac{(r+1)^2}{r}
$$

where $r$ is the threshold ratio (Lowe uses $r = 10$). This eliminates edge-like responses while retaining blob-like responses.

---

### 10. Orientation assignment

After localizing the keypoint at $(x^*, y^*, \sigma^*)$, SIFT assigns a **principal orientation** to make descriptors rotation-invariant.

**Procedure:**

1. Take the **Gaussian-smoothed image** at the scale closest to $\sigma^*$.
2. Compute the **gradient magnitude and direction** at each pixel in a neighborhood around the keypoint:

$$
m(x, y) = \sqrt{(L(x+1,y) - L(x-1,y))^2 + (L(x,y+1) - L(x,y-1))^2}
$$

$$
\theta(x, y) = \tan^{-1}\!\left(\frac{\partial I/\partial y}{\partial I/\partial x}\right)
$$

3. Build an **orientation histogram** with 36 bins (one per 10°, covering 0°–360°). Each gradient vote is weighted by its magnitude and by a Gaussian centered on the keypoint (to downweight distant pixels).

4. The **dominant peak** of the histogram gives the keypoint's principal orientation.

5. If any other peak is within **80% of the dominant peak's height**, an additional keypoint is created at that orientation. This explains why one detected blob can spawn multiple keypoints (accounting for multiple dominant gradients in ambiguous structures).

**The orientation histogram figure description (slide 46):** On the right, a circular neighborhood (blue circle) around the keypoint contains a 6×6 grid of gradient arrows, each pointing in the local gradient direction. On the left, the resulting bar chart has 8 bins (simplified visualization) with the arrow directions indicated below. The dominant bin determines the assigned orientation.

**Why this achieves rotation invariance:** The descriptor is subsequently computed relative to this assigned orientation, so rotating the image only changes the assigned angle — the descriptor itself remains consistent.

---

### 11. Scale and orientation — the payoff

Once each keypoint has:
- Position $(x^*, y^*)$
- Characteristic scale $\sigma^*$ (from the DoG extremum)
- Principal orientation $\theta^*$ (from the gradient histogram)

The local **coordinate frame** of the keypoint is fully defined. All subsequent description is computed relative to this frame. Consequently:

- **Scale invariance:** Two views of the same blob at different scales produce keypoints with different $\sigma^*$ values. The ratio $\sigma_1/\sigma_2$ equals the actual scale ratio between the two images. The descriptor region is normalized by $\sigma^*$ before computing.

- **Rotation invariance:** Two views of the same blob at different orientations produce keypoints with different $\theta^*$ values. The descriptor is computed in the rotated frame, so rotation differences are factored out.

**Visualization (slides 47–48):** Two images of the same DVD cover, one upright and one rotated ~45°. The orange circle marks the same large blob in both images. In the upright image the orientation arrow points upward; in the rotated image the arrow has rotated accordingly. After "undoing" the rotation (normalizing by $\theta^*$), both blobs produce the same canonical patch.

**Scale ratio between images (slides 49–50):** The same flame/person blob appears at scale $\sigma_1$ in the cover photo and $\sigma_2$ in the full scene photo. The ratio of blob sizes equals $\sigma_1/\sigma_2$, giving a direct estimate of the scale change between the two images.

---

### 12. Complete SIFT detector pipeline — summary

```
Input image I(x, y)
        |
        v
[Step 1] Build Gaussian pyramid
  - Multiple octaves (each octave: image downsampled by 2)
  - Within each octave: blur at scales σ, sσ, s²σ, ..., s^k σ
        |
        v
[Step 2] Compute DoG pyramid
  - For each adjacent pair of Gaussian images: D = G(·,sσ) − G(·,σ)
  - DoG ≈ (s−1) · NLoG  (efficient NLoG approximation)
        |
        v
[Step 3] Scale-space extrema detection
  - Find local min/max in 3×3×3 neighborhood (26 neighbors)
  - Apply contrast threshold (|D(x̂)| > 0.03)
  - Apply edge test (curvature ratio, Hessian, r=10)
  → Keypoint candidates
        |
        v
[Step 4] Keypoint localization (sub-pixel refinement)
  - Taylor expansion of D(x,y,σ) to find refined (x̂, ŷ, σ̂)
  → Precise keypoint positions
        |
        v
[Step 5] Orientation assignment
  - Compute gradient histogram (36 bins) in σ-scaled neighborhood
  - Weight by gradient magnitude × Gaussian window
  - Assign dominant peak as principal orientation θ*
  - Create extra keypoint for secondary peaks > 80% of dominant
        |
        v
Output: Set of keypoints, each described by (x*, y*, σ*, θ*)
```

---

## Key terms (glossary)

| Term | Definition |
|---|---|
| **SIFT** | Scale-Invariant Feature Transform; Lowe 2004 method for detecting and describing local image features |
| **Keypoint / interest point** | A local image region selected for its richness, distinctiveness, localizability, and invariance properties |
| **Blob** | A region visually distinct from its surroundings (in brightness/color/texture), localized in space and scale |
| **Scale space** | $S(x,y,\sigma) = n(x,y,\sigma) * I(x,y)$; the family of images obtained by convolving $I$ with Gaussians of increasing $\sigma$ |
| **$\sigma$ (sigma)** | Standard deviation of the Gaussian kernel; controls the smoothing scale |
| **Octave** | One level of the image resolution pyramid; each octave halves image dimensions |
| **Gaussian pyramid** | The full multi-octave, multi-blur-level stack of Gaussian-smoothed images |
| **DoG (Difference of Gaussians)** | $D = G(\cdot,s\sigma) - G(\cdot,\sigma)$; efficient approximation of the normalized Laplacian of Gaussian |
| **NLoG (Normalized Laplacian of Gaussian)** | $\sigma^2 \nabla^2 n_{\sigma}$; the operator used for scale-normalized blob detection |
| **Characteristic scale** | $\sigma^*$ at which the NLoG (or DoG) response is extremized; proportional to blob size |
| **Scale-space extremum** | A DoG sample that is a local min or max among all 26 neighbors in the 3D $(x,y,\sigma)$ space |
| **26-neighbor check** | The 3×3×3 neighborhood comparison: 8 in-plane + 9 above + 9 below |
| **Hessian edge test** | Uses $\text{Tr}(H)^2/\text{Det}(H)$ to reject keypoints on edges (unstable elongated structures) |
| **Orientation histogram** | 36-bin gradient direction histogram over a Gaussian-weighted neighborhood around the keypoint; peak gives principal orientation |
| **Principal orientation $\theta^*$** | The dominant gradient direction assigned to a keypoint; enables rotation invariance in descriptor computation |
| **$r_{\text{display}}$** | Visual radius of a detected blob: $\approx \sqrt{2} \cdot \sigma \cdot 2^{\text{octave}}$ |
| **Ratio of blob sizes** | $\sigma_1/\sigma_2$; gives the relative scale factor between the same blob in two images |
| **Signature / descriptor / feature vector** | Synonymous terms for the numerical representation of a keypoint (computed by the descriptor stage, Part 12) |

---

## Exam targets

1. **State what SIFT is and its two components.** Explain the difference between the detector and the descriptor. Know that signature = descriptor = feature vector.

2. **List and explain the four invariances/properties of SIFT** and the five advantages (Locality, Distinctiveness, Quantity, Efficiency, Extensibility). Be ready to justify each.

3. **Define a blob** and list three characteristics. Explain why edges are poor interest points but blobs are good ones.

4. **Derive the 1D blob detection principle.** Explain the $\sigma$-normalized 2nd derivative, what "characteristic scale" means, and why $\sigma^2$ normalization is necessary. Write the algorithm formally: given $f(x)$, compute at $k$ scales, find $(x^*, \sigma^*) = \arg\max \left|\sigma^2 (\partial^2 n_{\sigma}/\partial x^2) * f(x)\right|$.

5. **Write the NLoG formula in 2D** and the 2D blob detection algorithm. Write the scale-space equation $S(x,y,\sigma) = n(x,y,\sigma) * I(x,y)$.

6. **Derive why $\text{DoG} \approx (s-1)\text{NLoG}$.** Write the DoG equation. Explain why DoG is used in practice instead of NLoG directly.

7. **Describe the full Gaussian/DoG pyramid construction** — explain the two types of scale (Gaussian $\sigma$ and octave), how many blur levels per octave, and how DoG images are generated.

8. **Explain the 26-neighbor extrema check:** draw the 3×3×3 neighborhood, state the count (8+9+9=26), and describe the threshold rejection.

9. **Describe keypoint refinement**: the Taylor expansion approach, and the two rejection tests (low contrast threshold and Hessian-based edge test). Write the curvature ratio formula.

10. **Explain orientation assignment step-by-step:** gradient computation, 36-bin histogram with Gaussian weighting, peak selection, and the 80% rule for secondary orientations. Write the gradient direction formula $\theta = \tan^{-1}(\partial I/\partial y \;/\; \partial I/\partial x)$.

11. **Reproduce the complete detector pipeline** as a numbered sequence of steps (pyramid construction → DoG → extrema → refinement → orientation).

12. **Explain what scale and orientation give us:** why knowing both allows rotation and scale variations to be "undone," and how the ratio $\sigma_1/\sigma_2$ gives the scale ratio between views.

---

## Pitfalls

- **DoG is not the same as NLoG** — it approximates it up to the constant factor $(s-1)$. Extrema locations are the same; magnitudes differ by that factor. Do not confuse them.

- **$\sigma$ is not the same as octave.** $\sigma$ controls Gaussian smoothing within a fixed resolution image. Octave controls image resolution (downsampling by 2). SIFT uses both simultaneously. Mixing them up is a very common error.

- **The 26-neighbor check is 3×3×3 minus 1, not 3×3×3.** Count: 9+8+9 = 26. The center point is the candidate itself.

- **Characteristic scale is proportional to blob SIZE, not to position.** $\sigma^*$ tells you how large the blob is, not where it is. Position comes from $(x^*, y^*)$.

- **Orientation histogram has 36 bins (10° each), not 8.** The 8-bin diagram in slides is a simplified illustration. In the actual SIFT algorithm, 36 bins are used for the orientation assignment stage.

- **The 80% rule creates multiple keypoints from one blob.** When a secondary peak in the orientation histogram exceeds 80% of the primary peak, a separate keypoint is generated with the secondary orientation. This is not an error — it is intentional for robustness.

- **Edge points are explicitly rejected** by the Hessian test. An edge produces a large curvature ratio (one principal curvature much larger than the other). The ratio threshold $r=10$ means the ratio of eigenvalues must not exceed 10 for the point to be kept.

- **Blob radius in visualization includes the octave factor.** The formula $r_{\text{display}} \approx \sqrt{2} \cdot \sigma \cdot 2^{\text{octave}}$ means blobs from coarser octaves appear larger in the original image even if their within-octave $\sigma$ is the same.

- **"Interest point" and "keypoint" are synonymous** in this context. Do not confuse them with "descriptor" — the descriptor is what is computed AT the interest point, not the point itself.

- **Low contrast points are rejected before the edge test**, not after. Lowe's threshold is $|D(\hat{\mathbf{x}})| < 0.03$ (using normalized 0–1 pixel values).
