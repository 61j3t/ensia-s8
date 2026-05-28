# Part 13 — Motion Analysis

> Guest lecture by Luka Cehovin Zajc, University of Ljubljana (ViCoS Lab). ENSIA, May 2025.

---

## Bird's eye view

- **Motion analysis** operates at three levels: **optical flow** (pixel displacements between two frames), **point tracking** (following feature points across many frames), and **object tracking** (predicting the state of an object across an entire video).
- The **brightness-constancy assumption** — pixel intensity does not change as the pixel moves — is the fundamental constraint from which all classical optical-flow methods derive.
- The **optical-flow constraint equation** $I_x u + I_y v + I_t = 0$ gives one equation for two unknowns $(u, v)$ at each pixel; this **underdetermination** is the **aperture problem**.
- **Lucas-Kanade** resolves the aperture problem locally (assumes constant velocity in a small patch, solves an overdetermined least-squares system); **Horn-Schunck** resolves it globally (adds a spatial smoothness regularizer, solved iteratively); **pyramidal** extensions handle large displacements.
- **Object tracking** requires two models: an **appearance model** (what the object looks like) and a **motion model** (where it is likely to be); trackers are classified as generative vs. discriminative, and online vs. offline.
- Classical appearance models include SSD/NCC template matching, color histograms with MeanShift, and PCA-based subspace (IVT); discriminative models treat tracking as detection (online Boosting, MOSSE, KCF).
- **Multi-object tracking (MOT)** uses tracking-by-detection with bipartite matching and Kalman-filter motion models; offline MOT uses global graph optimization for occlusion-robust trajectories.
- Modern deep-learning trackers (MDNet, SiamFC, RAFT) dominate benchmarks; the VOT Challenge (since 2013, co-organized by the lecturer) is the standard annual evaluation protocol.

---

## Detailed notes

### 1. The three levels of motion analysis in video

Motion in a video can be studied at increasing levels of abstraction:

| Level | Scope | Output |
|---|---|---|
| **Optical flow** | Image pairs, all pixels | Dense 2D displacement field |
| **Point tracking** | Many frames, many feature points | Trajectories of interest points |
| **Object tracking** | Entire video, entire objects | State (bounding box / segmentation) per frame |

**Why motion matters:** optical flow is a primitive for video compression (inter-frame coding), scene understanding, robot navigation, and action recognition.

---

### 2. Optical flow — problem definition

#### 2.1 Brightness-constancy assumption

The key assumption is that the brightness of a moving pixel is preserved across frames:

$$
I(x, y, t) = I(x + u, y + v, t + 1)
$$

where $(u, v)$ is the displacement (flow vector) at pixel $(x, y)$ between frame $t$ and $t+1$. This assumes:
1. **Constant brightness** — no illumination changes, no specular reflections.
2. **Small displacements** — the Taylor expansion used in the derivation is first-order accurate only for small $(u, v)$.

#### 2.2 Deriving the optical-flow constraint equation (OFCE)

Expand $I(x+u, y+v, t+1)$ by first-order Taylor series around $(x, y, t)$:

$$
I(x+u, y+v, t+1) \approx I(x,y,t) + I_x \cdot u + I_y \cdot v + I_t
$$

Substituting into the brightness-constancy equation and cancelling $I(x,y,t)$:

$$
I_x \cdot u + I_y \cdot v + I_t = 0 \qquad \text{[OFCE]}
$$

where:
- $I_x = \partial I / \partial x$ — spatial gradient in x (computed with finite differences, e.g. Sobel)
- $I_y = \partial I / \partial y$ — spatial gradient in y
- $I_t = I(x,y,t+1) - I(x,y,t)$ — temporal gradient (frame difference)
- $(u, v)$ — the two unknowns (horizontal and vertical flow components)

**The OFCE is one equation in two unknowns** — it is therefore underdetermined at a single pixel.

#### 2.3 The aperture problem

The aperture problem is the fundamental ambiguity in optical flow. When a local patch contains only an edge (a 1D signal), only the component of motion **perpendicular to the edge** can be measured. The component parallel to the edge is invisible. Geometrically, the OFCE constrains $(u, v)$ to lie on a line in velocity space, not at a unique point.

Illustration (from slide): Three scenarios A, B, C show the same circle-inside-a-square viewed through a small aperture. In scenario A the object moves diagonally, but locally it looks the same as scenario B (pure vertical motion) because only the edge is visible. Only when the full boundary is visible (scenario C) does the true motion direction become determinable.

**Solutions:** add extra constraints — either spatial consistency (Lucas-Kanade: assume constant velocity in a neighborhood) or global smoothness (Horn-Schunck).

#### 2.4 Challenges: 3D-to-2D projection

3D motion in the world projects onto the 2D image plane. Depth, perspective, and camera movement all affect the resulting flow field, making it non-trivial to recover true 3D velocities from 2D optical flow alone.

---

### 3. Lucas-Kanade method

#### 3.1 Key assumption

In a local window (neighborhood) of $n$ pixels $q_1, q_2, \ldots, q_n$, the flow $(V_x, V_y)$ is assumed to be **constant**.

This gives a system of $n$ equations (one OFCE per pixel in the window):


$$
\begin{aligned}
I_x(q_1) V_x + I_y(q_1) V_y &= -I_t(q_1) \\
I_x(q_2) V_x + I_y(q_2) V_y &= -I_t(q_2) \\
&\vdots \\
I_x(q_n) V_x + I_y(q_n) V_y &= -I_t(q_n)
\end{aligned}
$$

In matrix form: $\mathbf{A} \mathbf{v} = \mathbf{b}$, where:

$$
\mathbf{A} = \begin{bmatrix} I_x(q_1) & I_y(q_1) \\ I_x(q_2) & I_y(q_2) \\ \vdots & \vdots \\ I_x(q_n) & I_y(q_n) \end{bmatrix}, \quad
\mathbf{v} = \begin{bmatrix} V_x \\ V_y \end{bmatrix}, \quad
\mathbf{b} = \begin{bmatrix} -I_t(q_1) \\ -I_t(q_2) \\ \vdots \\ -I_t(q_n) \end{bmatrix}
$$

#### 3.2 Least-squares solution

The system is overdetermined ($n \gg 2$), so solve via least squares $\mathbf{A}^\top \mathbf{A} \mathbf{v} = \mathbf{A}^\top \mathbf{b}$:

$$
\begin{bmatrix} V_x \\ V_y \end{bmatrix} = \begin{bmatrix} \sum I_x^2 & \sum I_x I_y \\ \sum I_x I_y & \sum I_y^2 \end{bmatrix}^{-1} \begin{bmatrix} -\sum I_x I_t \\ -\sum I_y I_t \end{bmatrix}
$$

where all sums are over pixels $q_i$ in the local window.

The 2×2 matrix $M = \mathbf{A}^\top \mathbf{A}$ is the **structure tensor** (same matrix used in Harris corner detection). Eigenvalues of $M$ determine whether the flow can be solved:
- **Both eigenvalues large** → corner-like region → well-conditioned, unique solution.
- **One large, one small** → edge region → aperture problem, solution only in one direction.
- **Both small** → flat region → no gradient information, cannot estimate flow.

#### 3.3 Algorithm steps

1. Compute spatial gradients $I_x, I_y$ using a derivative filter (e.g. Sobel).
2. Compute temporal gradient $I_t$ as frame difference.
3. For each pixel, assemble $M = \mathbf{A}^\top \mathbf{A}$ and $\mathbf{b} = \mathbf{A}^\top \mathbf{b}$ over the chosen window.
4. Invert $M$ (if well-conditioned) to get $(V_x, V_y)$.

**Properties:**
- Local method (each pixel neighborhood independent).
- Fast, real-time capable.
- Fails for large displacements (small-displacement assumption).
- Fails in uniform / low-texture regions (singular $M$).

**Visual output:** A flow field over a football match scene shows green arrows at each pixel; arrows are longer and consistent where texture is rich, absent in uniform regions.

---

### 4. Horn-Schunck method

#### 4.1 Formulation

Rather than restricting to a local window, Horn-Schunck adds a **global smoothness term**. The energy to minimize is:

$$
E = \iint \left[ (I_x u + I_y v + I_t)^2 + \alpha^2 \left( \|\nabla u\|^2 + \|\nabla v\|^2 \right) \right] dx\, dy
$$

- First term: **brightness constancy** — penalizes OFCE violation.
- Second term: **smoothness** — penalizes large spatial variations in the flow field; $\alpha$ is a regularization parameter balancing the two terms.

#### 4.2 Iterative algorithm

Minimizing $E$ with respect to $u$ and $v$ leads to coupled Euler-Lagrange equations, solved iteratively:

1. Compute local averages $\bar{u}_i$ and $\bar{v}_i$ of the current flow estimates at pixel $i$.
2. Compute the update scalar:

$$
\text{update} = \frac{I_x \bar{u}_i + I_y \bar{v}_i + I_t}{\alpha^2 + I_x^2 + I_y^2}
$$

3. Update the flow:

$$
u_{i+1} = \bar{u}_i - I_x \cdot \text{update}, \qquad v_{i+1} = \bar{v}_i - I_y \cdot \text{update}
$$

4. Repeat until convergence.

#### 4.3 Comparison with Lucas-Kanade

| Feature | Lucas-Kanade | Horn-Schunck |
|---|---|---|
| Motion assumption | Constant in small window | Smooth across entire image |
| Large motion | Weak (improves with pyramid) | Weak (improves with pyramid) |
| Speed | Fast (real-time) | Moderate |
| Noise robustness | Low to moderate | Moderate |
| Occlusion | Poor | Poor |
| Generalization | Low (condition-specific) | Assumes global smoothness |

**Visual output:** Horn-Schunck produces a smoother, more uniform-colored flow field (smooth gradients of hue representing direction); Lucas-Kanade produces a noisier, more locally-varied output.

---

### 5. Pyramidal optical flow (handling large displacements)

Classical optical flow breaks for large displacements because the linearization (Taylor expansion) requires small $(u, v)$. The pyramid approach handles this via **coarse-to-fine estimation**:

**Algorithm:**
1. Build an image pyramid: Level 0 = original; Level 1 = 1/2 resolution; …; Level n = 1/2^n resolution.
2. At the coarsest level (Level n), compute optical flow — displacements are small relative to the downsampled image.
3. Upscale the estimated flow by bilinear interpolation (multiply by 2 to account for resolution change).
4. Use the upscaled flow to **warp** the second image, reducing the residual displacement.
5. Compute flow of the residual at the next finer level.
6. Repeat down to Level 0 (original resolution).

**Result:** Large real-world displacements become small displacements at coarser levels, so the linear approximation holds. Both Lucas-Kanade and Horn-Schunck benefit from this extension.

---

### 6. Deep learning for optical flow

#### 6.1 FlowNet and RAFT

Classical methods rely on hand-crafted assumptions. Deep learning approaches instead train directly on data:

- **FlowNet (2015):** first end-to-end CNN for optical flow. Two streams encode two frames; a correlation layer computes feature similarity; a decoder produces the flow field.
- **RAFT (Recurrent All-Pairs Field Transforms, 2020):** state of the art for several years.
  - Two separate feature encoders extract features from Frame 1 and Frame 2.
  - A **4D correlation volume** is built by computing dot products between all pairs of feature vectors.
  - A context encoder processes Frame 1.
  - An iterative GRU-based update operator (10+ iterations) refines the flow estimate by looking up the correlation volume.
  - Output: a dense optical flow field.

**Training data:** synthetic datasets with ground-truth flow (e.g. FlyingChairs, FlyingThings3D) where flow is known by construction from the rendering engine. The images show chairs/objects from the Sintel or ShapeNet datasets with annotated flow maps shown in false color.

**Advantages of deep learning:** handles large displacements natively (learned representation), robust to noise and occlusion, generalizes across scenes.

---

### 7. Visual object tracking — problem definition

**Formal definition:** Given the state of an object in frame 1, predict the state of that object in every subsequent frame of a video stream.

- **State** = bounding box (axis-aligned or rotated), segmentation mask, or geometric model parameters.
- **Initialization:** the object is marked (ground truth) in frame 1 only.
- **Updating:** the tracker runs frame-by-frame without human intervention.

**Difficulty factors illustrated in slides:**
- **Illumination changes** — lighting conditions shift dramatically.
- **Fast motion** — object moves many pixels between frames, causing blur.
- **Occlusion** — object temporarily hidden behind another object.
- **Articulation** — object deforms (e.g. running person, animal).

---

### 8. Taxonomy of visual object trackers

#### 8.1 Model-free vs. model-based

- **Model-free:** no prior knowledge of object class. Initialized with any bounding box. Output: center, bounding box, or segmentation mask.
- **Model-based:** encodes specific object geometry (6 DoF rigid body, or N DoF articulated model — e.g. human body skeleton, face landmarks).

#### 8.2 Online vs. offline

- **Online tracker:** at frame $t$, the tracker has only seen frames up to $t$. Must make a decision with past and present information only. Suitable for real-time applications.
- **Offline tracker:** at frame $t$, can access frames $t-k \ldots t \ldots t+m$ (future frames). Can use global consistency for better accuracy and occlusion recovery. Not suitable for real-time.

#### 8.3 Single-object vs. multi-object (MOT)

| | Single-object tracking (SOT) | Multi-object tracking (MOT) |
|---|---|---|
| Scope | One object, any class | Many objects, usually one known class |
| Duration | Short-term or long-term | Typically long-term |
| Difficulty | No appearance prior, generality | Object identity maintenance, similar objects, multi-camera |
| Applications | Robotics, sports, surveillance | Autonomous driving, surveillance |

---

### 9. The two models in object tracking

Every tracker combines two components:

1. **Appearance model** — encodes what the object looks like ("what to compare"). Must be invariant enough to handle illumination and deformation, yet discriminative enough to distinguish the object from background.
2. **Motion model** — encodes where the object is likely to be ("where to look"). Predicts the search region based on previous positions.

Tracking reduces to: **search** (motion model proposes candidates) + **match** (appearance model scores candidates) + **update** (refine models with new observation).

---

### 10. Motion models for tracking

A motion model predicts the next position from previous positions.

#### 10.1 Types

- **Random walk (Brownian motion):** next position is sampled from a ball centered on the current position. No velocity assumption. Suitable for erratic motion.
- **Nearly constant velocity:** next position predicted by extrapolating velocity from recent positions. Suitable for smoothly moving objects.

#### 10.2 Recursive Bayes filters

State uncertainty is modeled probabilistically. The posterior $p(x_t \mid z_{1:t})$ (distribution over position given all observations) is updated using:

$$
\text{Posterior} \propto \text{Measurement likelihood} \times \text{Predicted prior}
$$

Visually: measurement distribution (peaked at observed position) convolved with motion distribution (spread by predicted velocity uncertainty) gives posterior (intermediate peak).

**Kalman filter:** assumes Gaussian distributions for all uncertainties. Optimal for linear motion with Gaussian noise. Used extensively in MOT.

**Particle filter:** represents the distribution as a set of weighted samples (particles). Handles multi-modal distributions (e.g. object could be in one of several locations after occlusion).

---

### 11. Appearance models — classical methods

#### 11.1 Template matching (SSD / NCC)

Track by finding the position in the next frame that best matches the object template from the previous frame.

**SSD (Sum of Squared Differences):**

$$
R(x, y) = \sum_{x',y'} \bigl(T(x', y') - I(x+x', y+y')\bigr)^2
$$

Minimum of $R$ = best match position.

**Correlation:**

$$
R(x, y) = \sum_{x',y'} T(x', y') \cdot I(x+x', y+y')
$$

Maximum = best match.

**Normalized Cross-Correlation (NCC):**

$$
R(x, y) = \frac{1}{n \sigma_I \sigma_T} \sum_{x,y} \bigl(I(x,y) - \mu_I\bigr)\bigl(T(x,y) - \mu_T\bigr)
$$

Invariant to linear illumination changes. NCC produces much sharper, more discriminative response maps than SSD or raw correlation (shown in side-by-side comparison of response maps on a face tracking example).

**Case study: NCC tracker**
- At initialization, the visual model $M$ is the object patch.
- At each frame: compute NCC response map $R$ by sliding the model over a search region around the predicted position.
- Take the maximum of $R$ as the new position.
- Optionally update the template.

#### 11.2 Color histogram models and MeanShift

**Motivation:** Color is invariant to many geometric changes (rotation, deformation) unlike raw templates.

**Color model types:**
- **Parametric:** model the color distribution as a Gaussian in RGB/HSV space.
- **Non-parametric (histogram):** bin the pixel colors; robust to outliers.

**Mode-seeking:** find the location in the next frame where the color histogram best matches the model. This is a **gradient ascent** problem on the similarity surface — the algorithm is **MeanShift**:
1. Place a search window near the predicted position.
2. Compute the weighted center of mass (mean shift vector) of pixels whose histogram similarity is high.
3. Move the window to that center.
4. Repeat until convergence.

**Bhattacharyya distance** measures similarity between two histograms $p$ and $q$:

$$
\rho(p, q) = \sum_{i=1}^{B} \sqrt{p_i \cdot q_i}
$$

Higher $\rho$ = more similar. Used as the similarity function in MeanShift tracking.

**Case study: MeanShift tracker (Comaniciu et al., TPAMI 2003)**
- Build a color histogram of the initial object region.
- At each frame: compute **histogram backprojection** (replace each pixel with its probability under the object color model).
- Run MeanShift on the backprojection image to find the new centroid.
- Optionally update the histogram.

**Template vs. Histogram — trade-off:**

| | Template | Histogram |
|---|---|---|
| Specificity | Very high (exact spatial layout) | Lower (bag of colors, no geometry) |
| Deformation handling | Poor | Good |
| Illumination invariance | Poor | Good |
| Use case | Rigid objects, small motion | Deformable objects, rotation |

#### 11.3 Subspace (PCA) appearance models

**Key idea:** treat each object patch as a point in high-dimensional pixel space. Model the set of valid object appearances as a **linear subspace** (manifold).

**PCA (Principal Component Analysis):**
- Decompose the covariance matrix of training patches.
- **Eigenvectors** = principal directions of variation (eigenfaces for faces).
- **Eigenvalues** = variance along each direction.
- Keep top k eigenvectors → low-dimensional subspace.

PCA transform: translate to origin ($t = \mu$), rotate by the eigenvector matrix $R = U$.

**Subspace projection in tracking:**
- Project a candidate patch onto the learned subspace.
- Measure the **reconstruction error** (reprojection error): if the candidate is the object, reconstruction error is low; if it is background, error is high.
- The subspace can be updated incrementally as the object's appearance changes.

**Case study: IVT (Incremental Visual Tracker, Ross et al., IJCV 2008)**
- Model possible appearances with a PCA subspace.
- **Incremental updates:** add new appearance observations to the subspace without recomputing from scratch.
- At each frame: **sample candidate bounding boxes** (particle filter), compute reprojection error for each, select lowest-error candidate as the new position.
- Shown tracking a face across large illumination changes (dark room → bright room).

---

### 12. Discriminative tracking

#### 12.1 Generative vs. discriminative

| | Generative model | Discriminative model |
|---|---|---|
| Goal | Model appearance of the object | Learn boundary between object and background |
| Training signal | Similarity to object examples | Object vs. background classification |
| Advantage | More general | More specific, less data |
| Example | NCC, MeanShift, IVT | Online Boosting, MOSSE, KCF |

#### 12.2 Binary classifier as detector

At each frame, train a binary classifier (object = positive class, background = negative class) on samples drawn from the current frame neighborhood. Apply the classifier densely over a search window — the highest-confidence position is the new object location.

This is **tracking as detection**: instead of storing a fixed template, a classifier is updated each frame.

#### 12.3 Tracking as detection — online Boosting

**Boosting cascade (Haar features):**
- **Haar features:** rectangular difference filters computed efficiently via the integral image. Examples: horizontal/vertical/diagonal edge pairs.
- **Cascade:** many weak classifiers (Haar feature thresholds) combined by AdaBoost. The cascade structure rejects obvious negatives early, so most background patches are rejected after a few stages.

**Online Boosting for tracking (Grabner et al., CVPR/BMVC 2006):**
- Pool of $N$ weak classifiers ($h_1, \ldots, h_N$), each with $M$ possible weak learners.
- At each frame, select the best weak learner per stage using a distribution model.
- Frame loop: current position → evaluate classifier on sub-patches in a search region → build confidence map → find maximum → new position → update classifier.
- The central square around the object is labeled positive (+); surrounding annular region is negative (−) for re-training.

**Case study: Online Boosting tracker** — tracks a Nemo toy in real video; confidence map shown as a 3D surface with a clear maximum at the object location.

#### 12.4 Correlation filter tracking

**Key insight:** template matching is equivalent to correlation; correlation with a linear filter is equivalent to a linear classification. Exploit this to make tracking very fast.

**Formulation:** correlation is convolution with a flipped signal. In the **Fourier domain**, correlation becomes element-wise multiplication:

$$
G = F \star H \quad \Rightarrow \quad \hat{G} = \hat{F} \odot \overline{\hat{H}}
$$

This allows computing correlation over the entire search region in $O(n \log n)$ using FFT, rather than $O(n^2)$ for direct sliding-window comparison.

**Circularity issue:** DFT-based correlation is circular (wraps around). This causes **boundary effects** where the filter "sees" the image as tiling. Fix: apply a **Hanning window** (cosine-tapered window) to the search region patch before computing the FFT, smoothly zeroing the boundaries.

**Correlation as a linear classifier:** template matching maximizes $\langle h, f \rangle$ where $h$ is the filter (template) and $f$ is the feature vector of a candidate patch. This is equivalent to a linear classifier with hyperplane $h^\top f = 0$.

**Discriminative Correlation Filters (DCF):** instead of using the raw object appearance as the filter, **learn** the filter $F$ such that:

$$
\arg\min_{F} \| T \star F - G \|^2
$$

where $T$ is the training example (object patch), and $G$ is the **desired response** — a sharp Gaussian peak centered at the object location. The closed-form solution is computed in the Fourier domain. This is much more discriminative than naive correlation.

**Case study: MOSSE (Bolme et al., CVPR 2010) — Minimum Output Sum of Squared Error:**
- Initialize the filter $F$ to give a Gaussian response to the object patch.
- At each frame: convolve the search region with $F$ (via FFT); location of the maximum is the new object position; update $F$ with the new patch.
- Visual: initialization frame shows a Gaussian response; subsequent frame shows the shifted peak at the new object location (a shifted spike in the response map).

**KCF (Kernelized Correlation Filter):**
- Extends MOSSE to multichannel features (HOG) and a kernel trick for non-linear classification.
- **Multichannel formulation:** extract HOG features (see section 13.1) from the search region; learn a separate filter per HOG channel; average the responses.
- **Scale estimation:** run a second filter at multiple scales to estimate object scale change.

---

### 13. Feature representations for discriminative tracking

#### 13.1 Histogram of Oriented Gradients (HOG)

- Compute gradient magnitude and orientation at every pixel using Sobel filters.
- Divide the image into a dense grid of 16×16 pixel cells.
- Build a histogram of gradient orientations (typically 8 or 9 bins) per cell, weighted by gradient magnitude.
- Concatenate block histograms (normalize across 2×2 cell blocks).
- Result: a rich, locally structured descriptor invariant to small translations and illumination changes.
- HOG is similar to SIFT descriptors but computed on a dense grid rather than at keypoints.

#### 13.2 Deep features for correlation tracking

**Problem with intensity/HOG:** pure intensity values drift with illumination; HOG is low-level and lacks semantic understanding.

**Solution:** use features extracted from a deep CNN backbone.

**Architecture (e.g. Bertinetto et al., CVPR 2017):**
1. Feed Template (object patch) and Search region through the same feature extraction network (shared weights).
2. Compute cross-correlation between template features and search region features across 100+ feature channels.
3. Sum the per-channel correlation maps → final response map; maximum = new object position.

**Limitation of CNN features trained on classification:** the features are discriminative at the category level but not at the instance level — they cannot reliably distinguish two similar objects of the same category.

---

### 14. Case study: MDNet (Nam and Han, CVPR 2016)

**Multi-domain learning for tracking:**
- Architecture: conv1 → conv2 → conv3 → fc4 → fc5 → fc6 (binary output: object / not object).
- **Pre-training:** on many tracking sequences. Each sequence = a domain. Shared layers (conv1–3, fc4–5) learn generic visual features; per-domain fc6 layers learn sequence-specific binary classifiers.
- **Initialization at test time:** fine-tune fc4–5; train fc6 from scratch; train bounding-box regression head.
- **Updates during tracking:** fine-tune all FC layers each N frames. Use **hard negative mining** — select the most confusing background samples to focus training. Maintain short-term and long-term memory pools to prevent concept drift. Stop updating when target is lost.
- **Speed:** ~1 second per frame (slow for real-time use, but high accuracy).

---

### 15. Case study: SiamFC (Bertinetto et al., CVPR 2016)

**Siamese fully-convolutional network:**
- Two identical CNN branches (shared weights) — one for the template, one for the search region.
- The template branch is evaluated once at initialization; the search branch is evaluated every frame.
- Cross-correlation of features → response map → maximum = new position.
- **Template not updated** (avoids drift, maintains discriminability, enables very fast tracking).
- **Scale estimation:** test at 3 pyramid levels (0.96×, 1×, 1.04× scale) and pick best scale.
- **Speed:** ~60 fps on a GPU.
- **Pre-training:** ImageNet VID dataset. Pairs of frames are sampled from the same sequence (within distance d); the network is trained to produce a peak response at the correct relative offset.

---

### 16. Multi-object tracking (MOT)

#### 16.1 Tracking-by-detection paradigm

1. Run a **pre-trained object detector** (category-specific, e.g. YOLO, Faster R-CNN) on every frame to get bounding-box detections.
2. **Associate** detections across frames to form trajectories.

#### 16.2 Frame-to-frame matching

For each pair of consecutive frames, define a **distance matrix** between all detected objects in frame $t$ and all detections in frame $t+1$. Distances can be:
- **IoU (Intersection over Union)** — bounding box overlap.
- **Pixel distance** — Euclidean distance between bounding box centers.
- **3D distance** — using depth sensor data.
- **Appearance similarity** — embedding distance from a re-identification model.

Solve the assignment problem (minimize total matching cost) using the **Hungarian algorithm (bipartite matching)** — assigns each detection in frame $t$ to the best detection in frame $t+1$.

**Issues with frame-to-frame (local) matching:**
- If the detector misses an object in one frame, the trajectory must be terminated; recovery is hard.
- Local greedy decisions can accumulate errors.

#### 16.3 Offline MOT — global graph optimization

- Gather all detections across the entire video.
- Build a **directed graph**: nodes = detections; edges = possible links between detections (weighted by match cost).
- Find the **globally minimum-cost set of trajectories** (a set of node-disjoint paths through the graph).
- Handles occlusions naturally: edges can skip frames when the detector fails (the object is invisible for k frames but can be re-linked).

#### 16.4 Appearance models and motion models for MOT

**Appearance (Re-ID):** describe each detection with a feature embedding (HoG, deep embedding). Match using **cosine distance** or metric-learning distance. The Re-ID problem is a **retrieval** problem: given a new detection, find the stored identity it belongs to.

**Motion models for MOT:**
- **Individual:** Kalman filter with Nearly Constant Velocity model. Update equation: $\mu_{\text{update}} = \mu_{\text{predicted}} + K_g \cdot \text{Innovation}$ where $\text{Innovation} = \text{measurement} - \text{prediction}$ and $K_g$ is the Kalman gain.
- **Group:** Social Force Models model interactions between pedestrians (goal-directed motion + repulsion from others + environmental constraints). Can be pre-trained from data.

---

### 17. Method comparison table (Optical Flow)

| Feature | Lucas-Kanade | Horn-Schunck | Deep Learning (RAFT) |
|---|---|---|---|
| Motion assumption | Constant in small window | Smooth across image | Learned from data |
| Large motion | Weak (pyramid helps) | Weak (pyramid helps) | Strong |
| Speed | Fast (real-time) | Moderate | Varies (GPU) |
| Noise robustness | Low–moderate | Moderate | High |
| Occlusion | Poor | Poor | Data-dependent |
| Generalization | Low | Assumes smoothness | Data-dependent |

---

### 18. Future directions

The field is moving entirely toward **deep learning**:
- Optical flow on large datasets (FlowNet, RAFT, and successors).
- **Point tracking over many frames** (e.g. TAPIR, Co-Tracker) — track arbitrary points through video including during occlusion.
- **Tracking with Transformers** — attention-based architectures (TransT, OSTrack) replace correlation with self/cross-attention.
- **Tracking and segmentation** — predict per-pixel masks rather than bounding boxes (VOS, SAM2/DAM4SAM, CVPR 2025).
- **Multi-object tracking** — joint detection + embedding (JDE, FairMOT), transformer-based (MOTR, TrackFormer).

---

## Key terms (glossary)

- **Optical flow** — the apparent motion of pixel intensity patterns between two consecutive frames, represented as a 2D vector field $(u(x,y), v(x,y))$.
- **Brightness-constancy assumption** — $I(x,y,t) = I(x+u, y+v, t+1)$; intensity of a moving pixel is unchanged.
- **OFCE (Optical-Flow Constraint Equation)** — $I_x u + I_y v + I_t = 0$; one linear constraint on the 2D flow vector at each pixel.
- **Aperture problem** — the fundamental ambiguity: a local edge region only reveals the flow component perpendicular to the edge; motion parallel to the edge is undetectable.
- **Lucas-Kanade** — local optical-flow method assuming constant velocity in a patch; solves an overdetermined least-squares system.
- **Structure tensor** — the 2×2 matrix $M = \mathbf{A}^\top \mathbf{A}$ built from spatial gradients; its eigenvalues characterize the local flow solvability.
- **Horn-Schunck** — global optical-flow method; minimizes a combined brightness-constancy + smoothness energy; iterative solution.
- **Pyramidal optical flow** — coarse-to-fine strategy using image pyramids to handle large displacements.
- **FlowNet / RAFT** — deep-learning optical-flow models trained on synthetic data.
- **SSD / NCC** — Sum of Squared Differences / Normalized Cross-Correlation; simple template-matching similarity measures.
- **Bhattacharyya distance** — $\rho(p,q) = \sum \sqrt{p_i \cdot q_i}$; histogram similarity measure used in MeanShift tracking.
- **MeanShift** — iterative mode-seeking algorithm; moves the search window toward the highest-density region in color probability space.
- **PCA / IVT** — subspace appearance model; object patch is projected onto principal components; reconstruction error used as similarity.
- **Concept drift** — gradual corruption of the appearance model when the tracker incorrectly updates with background.
- **Discriminative Correlation Filter (DCF)** — filter learned to produce a desired Gaussian response to the object; trained via $\arg\min_F \|T \star F - G\|^2$ in Fourier domain (MOSSE, KCF).
- **MOSSE** — Minimum Output Sum of Squared Error; first DCF tracker; real-time capable.
- **KCF** — Kernelized Correlation Filter; extends MOSSE to HOG features and kernel trick.
- **HOG (Histogram of Oriented Gradients)** — gradient-based dense feature; similar to SIFT but computed on a dense grid.
- **Hanning window** — cosine-tapered weighting applied to a patch before FFT to suppress circular boundary effects.
- **Circularity** — property of Fourier-domain correlation where the signal wraps around periodically.
- **SiamFC** — Siamese fully-convolutional network; fast (60fps), static template, pretrained on ImageNet VID.
- **MDNet** — multi-domain CNN tracker; pre-trained on tracking sequences; slow (~1 FPS) but highly accurate.
- **Online tracker** — uses only past and present frames; suitable for real-time.
- **Offline tracker** — uses future frames; better accuracy, occlusion recovery.
- **VOT Challenge** — annual Visual Object Tracking competition; standardized benchmarks since 2013.
- **Tracking-by-detection (MOT)** — detect objects per frame with a category detector; link detections across frames.
- **Bipartite matching / Hungarian algorithm** — optimal assignment of detections across frames by minimizing total cost.
- **Kalman filter** — recursive Bayesian filter assuming Gaussian noise; used for individual motion in MOT.
- **Particle filter** — Monte Carlo Bayesian filter; represents distribution with weighted samples; handles multi-modal uncertainty.
- **Re-identification (Re-ID)** — matching a detected object to a stored identity using appearance embeddings; cosine distance or metric learning.
- **Social Force Model** — physical model of pedestrian motion considering goals, environment, and inter-person repulsion.
- **IoU (Intersection over Union)** — area of overlap divided by area of union of two bounding boxes; used as a detection matching metric.
- **Hard negative mining** — selecting the most confusing background examples to train a classifier more effectively.

---

## Exam targets

1. **Derive the OFCE** from the brightness-constancy assumption using a first-order Taylor expansion. Write out the full equation $I_x u + I_y v + I_t = 0$ and define each term.

2. **Explain the aperture problem** — what ambiguity it creates, why a single edge gives only one constraint, and how it relates to the eigenvalues of the structure tensor $M$.

3. **Derive the Lucas-Kanade least-squares solution** — set up the overdetermined system $\mathbf{A}\mathbf{v} = \mathbf{b}$ for a patch of $n$ pixels, write the normal equations, and give the closed-form solution for $[V_x, V_y]^\top$.

4. **State the Horn-Schunck energy functional** (both the data term and the smoothness term), explain the role of $\alpha$, and write out the iterative update equations.

5. **Compare Lucas-Kanade and Horn-Schunck** across: motion assumption, large motion, speed, noise robustness, and occlusion.

6. **Explain pyramidal optical flow** — why it is needed, what a coarse-to-fine strategy is, what "warp" means in this context, and how it applies to both LK and HS.

7. **Define the three levels of motion analysis** (optical flow, point tracking, object tracking) and give the scope and output of each.

8. **Contrast template matching (NCC) and color histogram (MeanShift)** as appearance models — formula for each similarity measure, when each fails, and what the Bhattacharyya distance measures.

9. **Explain the structure tensor and its role** in determining where Lucas-Kanade can and cannot estimate flow.

10. **Describe discriminative correlation filters** — the MOSSE objective $\arg\min_F \|T \star F - G\|^2$, why the FFT is used, what circularity causes, and how the Hanning window fixes it.

11. **Explain SiamFC** — network architecture, why the template is not updated, how scale is estimated, pre-training procedure.

12. **Describe the tracking-by-detection paradigm for MOT** — detection + association + bipartite matching. What is the Hungarian algorithm's role?

13. **Compare online vs. offline tracking** — what information each uses, trade-offs in accuracy and real-time applicability.

14. **Explain concept drift** — what causes it and how trackers mitigate it (hard negative mining, conservative update, short/long-term memory).

---

## Pitfalls

- **Brightness constancy is not satisfied** in practice whenever there is illumination change, specular reflection, or camera noise. All classical optical-flow methods degrade under these conditions; deep learning methods are more robust.
- **OFCE gives one equation for two unknowns** — students often forget that this means a single pixel's flow is underdetermined. Always state the aperture problem explicitly.
- **Lucas-Kanade fails at edges, not just flat regions.** The structure tensor $M$ has one large and one small eigenvalue at an edge, making the system ill-conditioned (rank-deficient), not just at uniform regions.
- **Small-displacement assumption** is violated in fast motion. LK/HS without pyramids silently produce wrong results — never assume they work at arbitrary speeds.
- **Circular correlation in the Fourier domain** produces artifacts from boundary wrap-around. The Hanning window is the standard fix; without it, DCF trackers fail at the edges of the search region.
- **Template tracking does not update** in basic NCC/SiamFC → fails on appearance change, but avoids concept drift. Updating → adapts to appearance change but risks drifting to background. Both are legitimate design choices with different failure modes.
- **Concept drift** is a real failure mode for online discriminative trackers: if the classifier is incorrectly updated with background, future frames will be classified against an incorrect model. Hard negative mining and conservative updates are countermeasures.
- **SOT vs. MOT are different problems** — do not conflate them. MOT requires re-identification (who is this person across frames), not just localization (where is this object).
- **Bhattacharyya distance** $\rho(p,q) = \sum \sqrt{p_i \cdot q_i}$ is a **similarity** measure (higher = more similar), not a distance in the strict sense. The Bhattacharyya **distance** is $D_B = -\ln(\rho)$. The lecture uses $\rho$ directly.
- **IVT uses reprojection error as similarity** — low error means the candidate is well-reconstructed by the object subspace (good match); students sometimes invert this.
- **The Hungarian algorithm** solves bipartite matching optimally in polynomial time. In MOT, missed detections mean a trajectory must be ended (or kept alive with a grace period); this is a design choice, not an inherent limitation of the algorithm.
