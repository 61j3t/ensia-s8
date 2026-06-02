# Part 9 — The Hough Transform & RANSAC

## Bird's eye view

- The **Hough Transform** (Hough 1962; generalized by Duda & Hart 1972) is a **voting-based feature extraction** technique that detects geometric shapes (lines, circles, ellipses) in an image even when data is noisy or incomplete.
- The key insight is a **duality between image space and parameter space**: a single edge point in image space maps to an entire curve in parameter space; the intersection of many such curves reveals the shape parameters.
- Lines are first parameterized by slope-intercept ($y = mx + c$), but this is unbounded in $m$; the standard solution is the **polar (normal) parameterization** $\rho = x\cos\theta + y\sin\theta$, which has bounded, finite parameter ranges.
- The **accumulator array** $A(\rho, \theta)$ is incremented (voted into) by every edge point for all $\theta$; peaks in $A$ reveal detected lines. The same voting logic extends to **circles** (3-D accumulator $A(a, b, r)$) and arbitrary shapes.
- **RANSAC** (Fischler & Bolles 1981) is a complementary model-fitting algorithm that handles data with large fractions of outliers by repeatedly sampling minimal subsets and selecting the model with the largest consensus set; required iterations are determined by a closed-form formula.

---

## Detailed notes

### 1. History and motivation

| Year | Event |
|---|---|
| **1962** | Paul Hough files U.S. Patent 3,069,654: "Method and Means for Recognizing Complex Patterns" — the original Hough Transform. |
| **1972** | Richard Duda and Peter Hart publish "Use of the Hough Transformation to Detect Lines and Curves in Pictures" (*Communications of the ACM*) — generalizes the method to digital images and introduces the polar parameterization. |

**Why do we need it?** After edge detection, the image contains a set of edge points $(x_i, y_i)$. The challenge is to group these noisy, incomplete, cluttered boundary pixels into coherent geometric primitives (lines, circles, etc.). Edge-following algorithms break when the boundary is interrupted by noise or occlusion. The Hough Transform avoids this by operating globally through voting rather than locally through connected-component tracing.

**Advantages over edge-following:**
- **Robust to noise and gaps:** a line is detected even if half its pixels are missing.
- **Simple mathematical formulation** for complex recognition tasks.
- **Works in cluttered scenes:** irrelevant edges do not prevent correct detection because they scatter votes across the accumulator rather than concentrating them.
- **Detects multiple instances** of the same shape in a single pass of voting.

**Challenges:**
- Deciding which edge points to use (the edge image already has noise).
- Only part of the shape may be visible (occlusion, truncation at image boundary).
- Noise adds spurious votes, potentially producing false peaks.

**Applications:** lane detection in autonomous driving; medical image analysis (circular tumors, bone boundaries); industrial inspection (regular shapes in parts); document analysis (text baselines, page segmentation); robotics (structured environment recognition); object recognition.

---

### 2. Line detection — slope-intercept parameterization (m, c)

#### 2.1 The duality principle

A straight line in image space is written:

```math
y = mx + c
```
For a fixed edge point $(x_i, y_i)$, any line passing through it satisfies:
```math
y_i = m x_i + c \quad\Rightarrow\quad c = -m x_i + y_i
```
This is a **line equation in parameter space** $(m, c)$: the set of all $(m, c)$ pairs corresponding to lines through $(x_i, y_i)$ forms a straight line in the $m$-$c$ plane.

**Point-Line duality:**

| Image space | Parameter space |
|---|---|
| A **point** $(x_i, y_i)$ | A **line** $c = -m x_i + y_i$ |
| A **line** $y = mx + c$ | A **point** $(m, c)$ |

If two edge points lie on the same image-space line, their corresponding parameter-space lines intersect at the $(m, c)$ of that line. The more collinear edge points exist, the more parameter-space lines pass through the same $(m, c)$ point — which is detected as a peak in the accumulator.

#### 2.2 Line detection algorithm — (m, c) version

1. **Quantize parameter space $(m, c)$:** divide into a discrete grid of cells.
2. **Create accumulator array $A(m, c)$:** a 2-D integer array of the same size as the quantized grid.
3. **Initialize:** set $A(m, c) = 0$ for all cells.
4. **Vote:** for each edge point $(x_i, y_i)$:
   - For each quantized value of $m$, compute $c = -m x_i + y_i$.
   - Increment $A(m, c) \mathrel{+}= 1$.
5. **Find peaks:** identify local maxima in $A(m, c)$. Each peak $(m^{\ast}, c^{\ast})$ corresponds to a detected line $y = m^{\ast}x + c^{\ast}$.

**Worked example (from slides, 3 collinear points):**

Image contains 3 points roughly collinear. After voting:
- Point 1 casts a diagonal stripe of 1s across the accumulator.
- Point 2 casts a different diagonal stripe; these two lines cross at one cell (count = 2).
- Point 3 casts a third stripe that also passes through the same cell (count = 3).
- The cell with count = 3 is the global maximum; the corresponding $(m, c)$ is the detected line.

**Multiple lines:** if the image contains a quadrilateral (four edges), each edge contributes a cluster of votes to a different $(m, c)$ region. The accumulator shows four separated star-burst patterns, each with a peak at the parameters of one of the four lines.

#### 2.3 Problem with (m, c): unbounded slope

The slope $m$ ranges from $-\infty$ to $+\infty$ (for vertical lines $m$ is infinite). Implications:
- Very fine quantization => enormous accumulator array => huge memory.
- Vertical lines cannot be represented at all.
- The algorithm is computationally very expensive.

**Solution:** replace $(m, c)$ with a **polar (normal) parameterization**.

---

### 3. Line detection — polar parameterization (rho, theta)

#### 3.1 The polar line equation

A line in the image can always be described by:
```math
\rho = x\cos\theta + y\sin\theta
```
where:
- $\rho$ = perpendicular distance from the origin to the line (always finite; bounded by the image diagonal).
- $\theta$ = angle of the perpendicular from the x-axis, in $[0, \pi]$.

Both parameters are **bounded and finite**:
```math
0 \leq \theta \leq \pi \\
|\rho| \leq \sqrt{w^2 + h^2} \quad \text{(image diagonal)}
```

**Geometric interpretation:** draw a perpendicular from the image origin to the line. The length of that perpendicular is $\rho$; the angle it makes with the x-axis is $\theta$.

#### 3.2 Point-curve duality in polar space

For a fixed edge point $(x_i, y_i)$, all lines through it satisfy:
```math
\rho = x_i\cos\theta + y_i\sin\theta
```
As $\theta$ varies from $0$ to $\pi$, this traces a **sinusoidal curve** in the $(\theta, \rho)$ plane (the accumulator). Points lying on the same image-space line produce sinusoids that intersect at a single $(\theta^{\ast}, \rho^{\ast})$ in parameter space.

| Image space | $(\theta, \rho)$ parameter space |
|---|---|
| A **point** $(x_i, y_i)$ | A **sinusoidal curve** |
| A **line** | A **point** (intersection of sinusoids) |

#### 3.3 Polar Hough algorithm (step by step)

1. Initialize $A(\rho, \theta) = 0$
2. For each edge point $p_i(x_i, y_i)$ in the image:
   - For $\theta = 0$ to $\pi$ (in discrete steps):
     - Compute $\rho = x_i\cos\theta + y_i\sin\theta$
     - Increment $A(\rho, \theta) \mathrel{+}= 1$
3. Find $(\rho^{\ast}, \theta^{\ast})$ where $A(\rho, \theta)$ is maximum.
4. The detected line is: $\rho^{\ast} = x\cos\theta^{\ast} + y\sin\theta^{\ast}$

**What the accumulator looks like:** a dark 2-D image with $\theta$ on the horizontal axis and $\rho$ on the vertical axis. Each edge point contributes a sinusoidal bright streak. Where many streaks cross, bright spots (peaks) appear — each spot is one detected line.

#### 3.4 Extensions and optimizations

| Optimization | Effect |
|---|---|
| **Use gradient direction** | At each edge point, the gradient direction gives the approximate orientation of the line normal. This constrains $\theta$ to a small range instead of iterating all $[0, \pi]$, reducing computation greatly. |
| **Weight votes by gradient magnitude** | Stronger edges cast higher votes; weak/noisy edges have less influence. |
| **Adjusting accumulator resolution** | High resolution (fine cells): risk of vote dispersion — the true peak may be split across adjacent cells. Low resolution (coarse cells): risk of merging distinct lines. A practical balance is needed. |
| **Counting lines (peaks)** | Count the number of peaks above a threshold. Peaks must be separated using non-maximal suppression (similar to corner detection). |

**Full pipeline (as seen in examples):**

Original image → Gradient image → Edge image (thresholded) → Hough Transform (accumulator) → Detected lines overlaid

The accumulator image shows sweeping bright curves (contributions from long edges) converging to a small number of bright spots.

---

### 4. Circle detection

#### 4.1 Circle parameterization

A circle is described by three parameters:
```math
(x_i - a)^2 + (y_i - b)^2 = r^2
```
where:
- **(a, b)** = center of the circle
- **r** = radius

Finding a circle means finding the three parameters $a$, $b$, $r$ from a set of edge points.

#### 4.2 Case 1 — radius r is known

If $r$ is given (or pre-hypothesized), the accumulator is only 2-D: $A(a, b)$.

**Duality:** each edge point $(x_i, y_i)$ constrains the center to lie on a **circle of radius $r$ centered at $(x_i, y_i)$** in the $(a, b)$ parameter plane. In other words:
```math
(x_i - a)^2 + (y_i - b)^2 = r^2
```
This is a circle of radius $r$ in parameter space, centered at $(x_i, y_i)$.

**Voting procedure:**
- For each edge point $(x_i, y_i)$, draw a circle of radius $r$ in the 2-D accumulator $A(a, b)$ — increment all cells on that circle.
- After all edge points vote, the cell with the highest count is the estimated center $(a^{\ast}, b^{\ast})$.

**What the accumulator looks like:** for 8 edge points scattered on the circumference of one circle, 8 circles of radius $r$ appear in $(a, b)$ space — all passing through the true center, producing a "flower" pattern with a bright peak at $(a^{\ast}, b^{\ast})$.

**Example (coins image):** three coins of different sizes are detected by running the Hough circle transform separately for $r = r_1$ and $r = r_2$. At $r_1$, the accumulator shows a bright peak for the smaller coin; at $r_2$, peaks appear for the two larger coins.

#### 4.3 Case 2 — radius r is unknown

All three parameters $(a, b, r)$ must be found. The accumulator becomes **3-D**: $A(a, b, r)$.

**Duality:** each edge point $(x_i, y_i)$ maps to a **cone** in the 3-D parameter space $(a, b, r)$. The cone equation is:
```math
(x_i - a)^2 + (y_i - b)^2 = r^2
```
(for fixed $(x_i, y_i)$, varying $a$, $b$, $r$ traces a cone with apex at $r = 0$ and axis along the $r$ direction).

**Finding the circle:** the peak of $A(a, b, r)$ gives the center $(a^{\ast}, b^{\ast})$ and radius $r^{\ast}$.

**Cost:** a 3-D accumulator requires substantially more memory and computation than a 2-D one. This is the fundamental curse-of-dimensionality challenge in the Hough approach.

#### 4.4 General shapes — the Generalized Hough Transform

The same voting principle extends to any parameterized shape: ellipses (5 parameters), polygons, or even arbitrary template shapes (Generalized Hough Transform of Ballard 1981). The accumulator dimensionality equals the number of free parameters; beyond ~3-4 parameters the method becomes computationally intractable and RANSAC is preferred.

---

### 5. RANSAC — Random Sample Consensus

#### 5.1 Background

**Reference:** Fischler and Bolles, "Random Sample Consensus: A Paradigm for Model Fitting with Applications to Image Analysis and Automated Cartography," *Communications of the ACM*, Volume 24, Issue 6, 1981.

RANSAC is introduced here as a complementary fitting technique to the Hough Transform, designed for the case where data contains a large fraction of **outliers**.

#### 5.2 Problem setting

**Inliers:** data points whose distribution is explained by the model parameters (up to noise).
**Outliers:** data points that do not fit the model at all (clutter, mismatches, noise).

RANSAC can handle up to approximately **50% outliers**, which classical least-squares fitting cannot do (outliers corrupt the solution dramatically).

**Key idea:** find the best partition of the data into inliers and outliers, then estimate the model from the inlier subset only.

#### 5.3 RANSAC algorithm (6 steps)

1. **Select a random subset** of the original data (size $s$ = minimum number of points needed to fit the model). Call these the *hypothetical inliers*.
2. **Fit the model** to the hypothetical inliers.
3. **Test all remaining points** against the model using a loss function (e.g., distance threshold). Points within the threshold are added to the *consensus set* (inliers).
4. **Repeat** steps 1–3 until $N$ iterations have been performed.
5. **Select the model** that produced the largest consensus set.
6. **Re-fit the model** to the full consensus set associated with the best model (this optional final step improves quality).

**Worked example (line fitting):**
- Iteration A: randomly pick 2 points, fit a line. Count inliers within a distance band: 4 inliers. Store this as the current best.
- Iteration B: pick 2 different points, fit a different line. Count inliers: 13 inliers. This is better; update the best model.
- After $N$ iterations, the model with 13 inliers is selected. Step 6: re-fit a line using all 13 inliers (not just the 2 seed points), yielding a more accurate final model.

#### 5.4 How many iterations N?

**Parameters:**
- $s$ = minimum number of points to fit the model (e.g., $s = 2$ for a line, $s = 3$ for a circle center with known radius).
- $e$ = outlier ratio = (number of outliers) / (total number of data points).
- $p$ = desired probability of success (that at least one trial draws only inliers).

**Probability derivation:**

| Expression | Meaning |
|---|---|
| $(1 - e)$ | Probability that a single randomly drawn point is an inlier |
| $(1 - e)^s$ | Probability that all $s$ points in a trial are inliers |
| $1 - (1 - e)^s$ | Probability that a single trial **fails** (at least one outlier in the sample) |
| $(1 - (1-e)^s)^N$ | Probability that **all N trials** fail |

We require the probability of all-N-failing to equal $(1 - p)$:
```math
\bigl(1 - (1 - e)^s\bigr)^N = 1 - p
```
Solving for $N$:
```math
N = \frac{\log(1 - p)}{\log(1 - (1 - e)^s)}
```
**Illustrative values ($p = 0.99$):**

| e (outlier ratio) | s = 2 | s = 5 | s = 10 |
|---|---|---|---|
| 0.10 | 3 | 6 | 11 |
| 0.30 | 7 | 26 | 161 |
| 0.50 | 17 | 146 | 4714 |

As $e$ and $s$ increase, $N$ grows rapidly — for $e = 0.5$ and $s = 20$, $N$ exceeds 4 million.

**Important:** a smaller value of $s$ (fewer points needed to define the model) means a more efficient RANSAC. This can serve as a decision criterion when choosing between two fitting algorithms.

#### 5.5 RANSAC pros and cons

| Pros | Cons |
|---|---|
| Robust to outliers | Computation time grows rapidly with $e$ and $s$ |
| Simple to understand and implement | Not efficient at detecting multiple model instances simultaneously |
| Works well for $s$ in $[1..10]$ (depending on $e$) | |
| Standard/recommended approach for robust fitting | |

**Applications:** robust line/plane fitting; homography estimation (panorama stitching, AR, image registration); model fitting in object recognition; pose estimation (Perspective-n-Point / PnP problem) in robotics and self-driving.

---

## Key terms (glossary)

| Term | Definition |
|---|---|
| **Hough Transform** | A feature extraction technique using voting in parameter space to detect geometric shapes in images. |
| **Image space** | The 2-D space of pixel coordinates $(x, y)$. |
| **Parameter space** | The space whose axes are the parameters of a shape model (e.g., $m$ and $c$ for a line, $\rho$ and $\theta$ for a polar line). |
| **Accumulator array** | A discrete array indexed by quantized parameter values; each cell counts the number of edge points voting for those parameters. |
| **Point-curve duality** | The correspondence in which a point in image space maps to a curve (line or sinusoid) in parameter space, and a shape in image space maps to a point (intersection) in parameter space. |
| **Slope-intercept parameterization** | Representing a line as $y = mx + c$; $m$ is unbounded, causing problems for vertical lines. |
| **Polar parameterization** | Representing a line as $\rho = x\cos\theta + y\sin\theta$; $\rho$ is finite, $\theta \in [0, \pi]$. |
| **rho** | Perpendicular distance from the origin to a line (in polar Hough). |
| **theta** | Angle of the perpendicular from the x-axis (in polar Hough); ranges $0$ to $\pi$. |
| **Voting / incrementing** | For each edge point and each candidate parameter value, increasing the corresponding accumulator cell by 1. |
| **Peak / local maximum** | A cell in the accumulator with a value higher than all its neighbors; indicates a detected shape. |
| **Non-maximal suppression** | A post-processing step to isolate true peaks from the accumulator and count distinct shapes. |
| **Circle parameterization** | $(x-a)^2 + (y-b)^2 = r^2$; center $(a,b)$ and radius $r$ are the three parameters. |
| **Gradient direction optimization** | Using the edge gradient angle to restrict $\theta$ search in Hough, reducing computation. |
| **RANSAC** | Random Sample Consensus; an iterative algorithm for fitting models to data containing outliers. |
| **Inlier** | A data point consistent with (explained by) the fitted model. |
| **Outlier** | A data point not consistent with the model. |
| **Consensus set** | The set of inliers for a given model hypothesis in RANSAC. |
| **s (RANSAC)** | Minimum number of points needed to fit the model. |
| **e (RANSAC)** | Outlier ratio = # outliers / total # points. |
| **N (RANSAC)** | Number of required iterations: $N = \frac{\log(1-p)}{\log(1-(1-e)^s)}$. |

---

## Exam targets

1. **State the duality principle for lines:** explain both directions — what does a point in image space become in $(m,c)$ space? What does a point in $(m,c)$ space correspond to in image space? Repeat for polar $(\rho, \theta)$ space (point maps to sinusoidal curve).

2. **Write and derive the polar line equation** $\rho = x\cos\theta + y\sin\theta$ and explain why it replaces slope-intercept (what is the problem with $m$ ranging from $-\infty$ to $+\infty$?).

3. **Write out the full Hough line-detection algorithm** for the polar case, step by step (initialize; loop over edge points; loop over $\theta$; compute $\rho$; increment $A(\rho,\theta)$; find maximum).

4. **Describe the accumulator array** for both the $(m,c)$ and the $(\rho,\theta)$ cases: what are the axes, what do individual cells mean, what does a peak mean?

5. **Trace a worked example:** given 3 collinear points, show how the accumulator fills (one sinusoid per point, all three sinusoids cross at the same $(\rho,\theta)$, giving a count of 3 at that cell).

6. **Explain circle detection** with known radius $r$: circle equation, parameter space dimension, what each edge point contributes to $A(a,b)$, what the accumulator looks like, how the center is read off.

7. **Explain circle detection with unknown $r$:** the accumulator becomes 3-D $(a, b, r)$; each edge point casts a cone in 3-D parameter space.

8. **Explain why the $(m,c)$ Hough has a parameterization problem** and how polar form solves it (bounded $\theta$ in $[0,\pi]$, finite $\rho$).

9. **Describe optimizations** of the polar Hough: gradient direction to restrict $\theta$; gradient magnitude weighting; resolution trade-off (high resolution = vote dispersion, low resolution = merging nearby lines).

10. **State and explain the RANSAC algorithm** (6 steps). Know what each step does and why step 6 (re-fitting to the full consensus set) improves accuracy.

11. **Derive the RANSAC iteration formula** $N = \frac{\log(1-p)}{\log(1-(1-e)^s)}$ from first principles (know the four probability expressions and how to solve for N).

12. **Compare Hough and RANSAC**: Hough is exhaustive over a grid → finds all lines simultaneously, memory-intensive. RANSAC is randomized → robust to outliers, finds one model at a time, scales poorly with high $e$ and large $s$.

---

## Pitfalls

- **Slope-intercept cannot represent vertical lines** ($m \to \infty$). Always use the polar form in practice and in exam answers.
- **A peak in the accumulator is NOT guaranteed to be a real line** — noise can produce spurious peaks. A threshold and non-maximal suppression are required.
- **Resolution trade-off is bidirectional:** coarse accumulator merges distinct lines (misses close lines); fine accumulator disperses votes and creates multiple weak peaks for one line. Neither extreme is correct.
- **For circles with unknown $r$, the accumulator is 3-D** — not 2-D. A common mistake is to forget the radius dimension.
- **Each edge point casts a full sinusoidal curve** (in polar Hough) or a full line (in $(m,c)$ Hough), not a single vote — votes go to every quantized $\theta$ value.
- **RANSAC selects the model with the largest consensus set, then re-fits.** The selected model's parameters (from step 5) are based on only $s$ seed points; the final output (step 6) refits to all $N_\text{inlier}$ points for accuracy. Confusing these two steps is a common error.
- **RANSAC is not good at detecting multiple model instances simultaneously** (unlike Hough, which detects all peaks in a single pass). Each RANSAC run finds one dominant model.
- **The RANSAC formula assumes independent draws.** If outliers cluster near inliers the formula still holds but practical convergence may differ.
- **Gradient direction optimization reduces Hough cost but requires a reliable gradient estimate** — applying it to noisy edges can degrade detection.
- **The slide numbering calls this "Part VII" of the lecture series**, not "Part 9" of the course. Your notes use the course numbering (Part 9).
