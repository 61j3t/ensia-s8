# Part 7 — Image Features

## Bird's eye view

- A **feature** is a local, meaningful, detectable part of an image — specifically a location of sudden intensity change that carries high information content.
- Good features must be **invariant** to viewpoint, lighting, object deformation, and partial occlusion, and must be **unique** and **easy to extract**.
- **Edges** are the primary feature type: pixels where intensity changes abruptly, caused by depth discontinuities, surface orientation changes, reflectance changes, or color changes.
- Three edge profile models exist — **step**, **ramp**, and **roof** — and noise deforms all three away from their ideal shapes, making smoothing mandatory before differentiation.
- **Gradient-based detection** finds edges as local maxima of the gradient magnitude; the 2D gradient gives both strength ($S = \|\nabla I\|$) and direction ($\theta = \arctan(\partial I/\partial y \div \partial I/\partial x)$).
- **Laplacian-based detection** uses the second derivative; edges appear as **zero-crossings** where the Laplacian changes sign; it provides location only, not orientation.
- Noise completely corrupts raw derivatives; the standard fix is to smooth first with a Gaussian, or equivalently convolve with the **Derivative of Gaussian** (DoG) or **Laplacian of Gaussian** (LoG) — a single precomputed kernel.
- The lecture covers seven topics: edges, gradient detection, Laplacian detection, Canny detector, line/curve detection, Hough transform, and Harris corner detector (topics 4–7 are referenced in the roadmap on slide 7 but the detailed slides in this PDF end at topic III / noise treatment — the remainder form a natural sequel).

---

## Detailed notes

### 1. What is an image feature?

**Definition (two complementary views):**
- A feature is a **local, meaningful, detectable part of the image** (e.g., a corner marked with a red dot on a 3-D scene of blocks).
- A feature is a **location of sudden change** in intensity — a point where the image function changes rapidly.

**Why use features instead of raw pixels?**

| Reason | Explanation |
|---|---|
| High information content | Edges and corners encode structural information densely |
| Invariance | Features can be made robust to viewpoint and illumination change |
| Computational efficiency | A sparse binary edge map requires far fewer operations than a full grayscale image |
| Biological plausibility | Early mammalian vision involves detection of edges and local features |

**Properties a good feature must have:**

A good feature is **invariant** to:
- Viewpoint changes
- Lighting/illumination conditions
- Object deformations
- Partial occlusions

And should be:
- **Unique** (distinctive — not repeated uniformly across the image)
- **Easy to find and extract** (repeatable / detectable)

These properties map to the canonical four: **repeatability, invariance, robustness (to noise/deformation), distinctiveness**.

---

### 2. Feature types — focus on Edges

#### 2.1 What is an edge?

**Formal definition:** Edge pixels are pixels at which the intensity of an image function changes abruptly.

**Causes of edges in a real scene** (illustrated with a street-scene aerial photograph):

| Cause | Description |
|---|---|
| **Depth discontinuity** | Two surfaces at different depths produce a sharp intensity boundary |
| **Surface orientation discontinuity** | A fold or crease in a single surface changes how light reflects |
| **Reflectance discontinuity** | Change of material (e.g. tarmac → painted line) |
| **Surface color discontinuity** | Different pigment on the same surface |

An artist's sketch of a sculpture (Henry Moore, 1964) demonstrates that a few edge strokes convey 3-D structure, shading, and highlights — confirming that edges carry most of the scene's information.

#### 2.2 Edge detection objective and output

**Objective:** Convert a 2D image into a set of points where intensity changes rapidly.

An edge detector produces three quantities per detected edge point:
1. **Edge position** — the (x, y) coordinates
2. **Edge magnitude** (strength) — how abrupt the change is
3. **Edge orientation** — the angle at which the edge is aligned (orthogonal to the gradient direction)

**Distinction between direction and orientation:**
- **Direction** refers to the way intensity changes from one side to the other (i.e., the gradient vector direction $\alpha$).
- **Orientation** is the angle of the edge itself, which is perpendicular: $\alpha - 90°$.

**Performance requirements for a good edge detector:**
1. High detection rate (find all real edges, no missed edges)
2. Good localization (detected position close to true edge position)
3. Low sensitivity to noise

#### 2.3 Edge profile models

Real images contain edges that deviate from perfect mathematical ideals due to noise and optical blur. Three canonical models:

| Model | Intensity profile | Shape | Real-world examples |
|---|---|---|---|
| **Step edge** | Abrupt jump from one level to another | Heaviside-like step function | Boundary between a black object and white background |
| **Ramp edge** | Gradual transition over several pixels | Linear or sigmoid ramp | Shadow boundaries, out-of-focus regions. Note: slope is inversely proportional to degree of blurring |
| **Roof edge** | Intensity rises to a peak then falls symmetrically | Peaked / triangular profile | Thin structures: wires, roads in aerial images, fine hair |

Noise deforms all three away from their ideal shapes — the same CT scan cross-section can show step-like edges at bone boundaries, ramp edges at soft-tissue transitions, and roof edges at narrow structures.

---

### 3. Edge detection — I: Gradient-based

#### 3.1 1-D intuition

In 1-D, an edge = rapid change in intensity f(x). The **first derivative** $df/dx$ peaks at the edge location:
- Local **extrema** of $df/dx$ indicate edges.
- Local **maxima** of $|df/dx|$ indicate edges with both location and strength.

#### 3.2 2-D gradient operator

For a 2-D image $I(x, y)$, the **gradient** is:

$$\nabla I = \left[\frac{\partial I}{\partial x},\ \frac{\partial I}{\partial y}\right]$$
- A vertical edge (transition along x): $\nabla I = [\partial I/\partial x,\ 0]$
- A horizontal edge (transition along y): $\nabla I = [0,\ \partial I/\partial y]$
- A diagonal edge: $\nabla I = [\partial I/\partial x,\ \partial I/\partial y]$

**Gradient magnitude (edge strength):**
$$S = \|\nabla I\| = \sqrt{\left(\frac{\partial I}{\partial x}\right)^2 + \left(\frac{\partial I}{\partial y}\right)^2}$$
**Gradient direction:**
$$\theta = \arctan\!\left(\frac{\partial I/\partial y}{\partial I/\partial x}\right)$$
The gradient vector points in the direction of steepest intensity increase; the edge orientation is perpendicular to this.

#### 3.3 Discrete approximation (convolution kernels)

For a discrete image with pixel spacing $\varepsilon$, partial derivatives are approximated by finite differences over a 2×2 neighbourhood:
$$\frac{\partial I}{\partial x} \approx \frac{1}{2\varepsilon}\left[(I_{i+1,j+1} - I_{i,j+1}) + (I_{i+1,j} - I_{i,j})\right]$$
$$\frac{\partial I}{\partial y} \approx \frac{1}{2\varepsilon}\left[(I_{i+1,j+1} - I_{i+1,j}) + (I_{i,j+1} - I_{i,j})\right]$$

Implemented as convolution with 2×2 kernels (up to $1/2\varepsilon$ scaling):

```
∂I/∂x kernel:  [-1  1]      ∂I/∂y kernel:  [1   1]
               [-1  1]                      [-1  -1]
```

#### 3.4 Gradient operator families

A range of gradient operators has been proposed, trading off localization against noise sensitivity:

| Operator | Size | Localization | Noise sensitivity | Detection quality |
|---|---|---|---|---|
| Roberts | 2×2 | Best | Most sensitive | Poorest |
| Prewitt | 3×3 | Good | Moderate | Moderate |
| Sobel (3×3) | 3×3 | Good | Moderate | Good |
| Sobel (5×5) | 5×5 | Poorest | Least sensitive | Best |

**Sobel 3×3 kernels (most commonly used):**

```
∂I/∂x:  [-1  0  1]      ∂I/∂y:  [ 1  2  1]
         [-2  0  2]               [ 0  0  0]
         [-1  0  1]               [-1 -2 -1]
```

Key trade-off: larger kernels average over more pixels → less noise but worse localization. Smaller kernels localize well but amplify noise.

#### 3.5 Thresholding

After computing $\|\nabla I(x,y)\|$, a decision must be made about which pixels are edges.

**Single threshold T:**
- $\|\nabla I(x,y)\| < T$ → definitely NOT an edge
- $\|\nabla I(x,y)\| \geq T$ → definitely an edge

**Hysteresis (two thresholds $T_0 < T_1$):**
- $\|\nabla I(x,y)\| < T_0$ → definitely NOT an edge
- $\|\nabla I(x,y)\| \geq T_1$ → definitely an edge
- $T_0 \leq \|\nabla I(x,y)\| < T_1$ → edge **only if a neighbouring pixel is definitely an edge**

**Threshold pitfalls:**
- Too high → real edges missed
- Too low → noise treated as genuine edges

Hysteresis avoids both extremes by allowing weak responses to "connect" to confirmed strong edges.

---

### 4. Edge detection — II: Laplacian-based

#### 4.1 Second derivative intuition

The **second derivative** $d^2f/dx^2$ has **zero-crossings** (sign changes from positive to negative or vice versa) exactly at edge locations — where the first derivative has its extrema. This gives a mathematically precise edge locator.

#### 4.2 The 2D Laplacian

The **Laplacian** is the sum of the pure second partial derivatives:
$$\nabla^2 I = \frac{\partial^2 I}{\partial x^2} + \frac{\partial^2 I}{\partial y^2}$$
It measures how much a pixel's intensity differs from its neighbours in **all directions** simultaneously.

**Edge detection rule:** Edges are **zero-crossings** in the Laplacian image — points where the sign of $\nabla^2 I$ changes.

**Important limitation:** The Laplacian does **not** provide the direction/orientation of edges. It gives location only.

#### 4.3 Discrete Laplacian
$$\frac{\partial^2 I}{\partial x^2} \approx \frac{1}{\varepsilon^2}(I_{i-1,j} - 2I_{i,j} + I_{i+1,j})$$
$$\frac{\partial^2 I}{\partial y^2} \approx \frac{1}{\varepsilon^2}(I_{i,j-1} - 2I_{i,j} + I_{i,j+1})$$

Three common discrete Laplacian convolution kernels:

```
Standard 4-neighbour:      Isotropic (1/6 scaled):      8-neighbour:
[0   1  0]                 [1   4   1]                  [-1  -1  -1]
[1  -4  1]           (1/6)*[4  -20  4]                  [-1   8  -1]
[0   1  0]                 [1   4   1]                  [-1  -1  -1]
```

#### 4.4 Zero-crossing detection procedure

**Step 1 — Apply the Laplacian operator:**
- Convolve the image with a Laplacian kernel.
- Result: a 2D image of Laplacian values, some positive, some negative.

**Step 2 — Check each pixel and its 8-connected neighbours** (horizontal, vertical, diagonal):
- If the **sign of the Laplacian value changes** between a pixel and one of its neighbours,
- AND the **absolute difference is large enough** (guard against tiny sign flips due to noise),
- → A **zero-crossing** (edge) is detected.

---

### 5. Edge detection — III: Derivatives and Noise

#### 5.1 The noise problem

Raw gradient computation on a noisy image completely obscures the true edge. The noise introduces rapid fluctuations that have the same derivative signature as real edges — the edge signal is lost in the noise floor.

**Consequence:** Image smoothing is a **mandatory step** before applying any derivative-based edge detector.

#### 5.2 Solution: Smooth then differentiate

**Method 1 — Smooth then differentiate (two separate operations):**

Convolve the image with a Gaussian $n_\sigma$, then take the gradient:
$$\nabla(n_\sigma \ast f)$$
This works but requires two passes.

**Method 2 — Derivative of Gaussian (single operation):**

Since both $\nabla$ and Gaussian convolution are linear:
$$\nabla(n_\sigma \ast f) = (\nabla n_\sigma) \ast f$$
Therefore, pre-compute $\nabla n_\sigma$ once and convolve directly with f. This single kernel performs smoothing and differentiation simultaneously. The result for a step-edge signal: the noisy f(x) produces a clean, localised peak in $(\nabla n_\sigma) \ast f$ at the true edge location.

**Method 3 — Laplacian of Gaussian (LoG):**

Analogously:
$$\nabla^2(n_\sigma \ast f) = (\nabla^2 n_\sigma) \ast f$$
The **Laplacian of Gaussian** kernel is:
$$\nabla^2 G(x,y) = \frac{x^2 + y^2 - 2\sigma^2}{\sigma^4} \cdot \exp\!\left(-\frac{x^2+y^2}{2\sigma^2}\right)$$

Its 1-D cross-section has a distinctive **inverted Mexican hat (sombrero)** shape with two zero-crossings at $\pm\sqrt{2}\,\sigma$ from the centre. In 2D it looks like an inverted sombrero.

A practical 5×5 discrete approximation of LoG:

```
[ 0   0  -1   0   0]
[ 0  -1  -2  -1   0]
[-1  -2  16  -2  -1]
[ 0  -1  -2  -1   0]
[ 0   0  -1   0   0]
```

#### 5.3 Gradient vs. Laplacian comparison

| Property | Gradient ($\nabla$) | Laplacian ($\nabla^2$) |
|---|---|---|
| **Output** | Location + magnitude + orientation | Location only |
| **Detection criterion** | Threshold on maxima | Zero-crossings |
| **Operation type** | Non-linear (sqrt) — requires two convolutions | Linear — requires one convolution |
| **Noise handling** | Use Derivative of Gaussian | Use Laplacian of Gaussian (LoG) |
| **Shape of smoothed kernel** | Two directional lobes (one per axis) | Inverted sombrero / Mexican hat |

---

## Key terms (glossary)

- **Feature** — A local, meaningful, detectable part of an image; a location of sudden intensity change.
- **Repeatability** — A feature detected in one image is detected at the corresponding location in another image under varying conditions.
- **Invariance** — Insensitivity of the feature to transformations (viewpoint, illumination, scale, rotation).
- **Distinctiveness** — The feature descriptor is unique enough to distinguish it from others.
- **Edge** — A pixel where image intensity changes abruptly; characterised by position, magnitude, and orientation.
- **Step edge** — Abrupt intensity jump; modelled as a Heaviside function.
- **Ramp edge** — Gradual transition; slope inversely proportional to blurring degree.
- **Roof edge** — Peaked profile; represents narrow ridges or thin structures.
- **Gradient** ($\nabla I$) — Vector $[\partial I/\partial x,\ \partial I/\partial y]$; points in direction of steepest ascent.
- **Gradient magnitude** — $S = \|\nabla I\| = \sqrt{(\partial I/\partial x)^2 + (\partial I/\partial y)^2}$; measures edge strength.
- **Edge direction** — The way intensity changes across the edge (gradient direction, angle $\alpha$).
- **Edge orientation** — The alignment angle of the edge itself (perpendicular to gradient, $\alpha - 90°$).
- **Roberts / Prewitt / Sobel operators** — Discrete convolution kernels approximating image gradients.
- **Thresholding** — Deciding which pixels are edges based on gradient magnitude.
- **Hysteresis thresholding** — Two-threshold scheme: weak responses confirmed as edges only if adjacent to a strong edge.
- **Laplacian** ($\nabla^2 I$) — Sum of second partial derivatives $\partial^2 I/\partial x^2 + \partial^2 I/\partial y^2$; measures deviation from local neighbourhood mean in all directions.
- **Zero-crossing** — A point where the Laplacian changes sign; indicates an edge.
- **Derivative of Gaussian (DoG)** — Kernel $\nabla n_\sigma$; combines smoothing and first-derivative in one convolution.
- **Laplacian of Gaussian (LoG)** — Kernel $\nabla^2 G$; combines Gaussian smoothing and Laplacian in one convolution; "Mexican hat" shape.
- **$\sigma$ (sigma)** — Standard deviation of the Gaussian; controls the scale at which features are detected.

---

## Exam targets

1. **Define a feature** in two ways: as a local/meaningful/detectable image part AND as a location of sudden change. State the four desirable properties.

2. **List the four causes of edges** (depth discontinuity, surface orientation discontinuity, reflectance discontinuity, surface color discontinuity) and give a real-world example of each.

3. **Describe the three edge profile models** (step, ramp, roof) — what intensity profile each has, what real-world situation it models, and why noise causes deviations.

4. **State what an edge detector outputs** (position, magnitude, orientation). Distinguish direction from orientation (orientation = direction $- 90°$).

5. **Write and interpret the 2D gradient formulas:**
   - $\nabla I = [\partial I/\partial x,\ \partial I/\partial y]$
   - $S = \sqrt{(\partial I/\partial x)^2 + (\partial I/\partial y)^2}$
   - $\theta = \arctan(\partial I/\partial y\ /\ \partial I/\partial x)$

6. **Write the discrete convolution kernels** for the Sobel 3×3 operator (both $\partial/\partial x$ and $\partial/\partial y$ versions). Explain the trade-off between kernel size, localization, and noise sensitivity.

7. **Explain single vs. hysteresis thresholding** — write the decision rule for each, and explain the failure modes of setting T too high or too low.

8. **Define the Laplacian** $\nabla^2 I = \partial^2 I/\partial x^2 + \partial^2 I/\partial y^2$. Explain why edges appear as zero-crossings. State the key limitation (no orientation).

9. **Explain the noise problem** for derivative-based detectors (noise = rapid changes = same signature as edges). State the solution (smooth first).

10. **Derive the Derivative of Gaussian trick**: $\nabla(n_\sigma \ast f) = (\nabla n_\sigma) \ast f$ — why this works (linearity of both operations) and why it is efficient (one precomputed kernel).

11. **State the LoG formula** and describe its shape (inverted sombrero / Mexican hat). Give the zero-crossing positions ($\pm\sqrt{2}\,\sigma$ from centre in 1-D).

12. **Compare gradient vs. Laplacian** in a table: output information, detection criterion, linearity, number of convolutions required.

---

## Pitfalls

- **Direction $\neq$ orientation.** Direction is the gradient vector direction (how intensity changes); orientation is the edge alignment angle, which is perpendicular ($\alpha - 90°$). Mixing them up loses marks.
- **The Laplacian does not give edge orientation.** Only gradient-based methods provide orientation. A common error is claiming Laplacian gives full edge information.
- **Zero-crossing detection requires checking the absolute difference**, not just the sign change — otherwise noise-induced micro sign-flips produce false edges.
- **Single threshold is not the same as hysteresis.** In hysteresis, the middle-band pixel is ONLY an edge if it has a confirmed strong-edge neighbour. It is not automatically accepted.
- **Larger gradient operators = better detection but worse localization**, not both simultaneously. Do not say "Sobel 5×5 is strictly better than 3×3" — it has worse localization.
- **The slope of a ramp edge being inversely proportional to blurring** is a precise statement: a sharp physical edge blurred by the optics produces a gentle ramp, not a missing edge.
- **DoG and LoG are single convolutions**, not two separate steps. The efficiency gain comes precisely from pre-computing the combined kernel. Do not describe them as "smooth then differentiate separately."
- **Roof edges are detected differently from step edges.** The first derivative of a roof edge gives two peaks (not extrema that look like step edges), and the Laplacian zero-crossings are also different in number. Thin-line detectors are therefore a separate topic.
