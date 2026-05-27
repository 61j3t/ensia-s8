# Part 14 — Deep Learning for Computer Vision

## Bird's eye view

- **Gestalt theory** explains why humans perceive whole objects, not isolated pixels; the brain organises visual input into groups using principles such as proximity, similarity, common fate, common region, and symmetry.
- **Image segmentation** is the task of dividing an image into meaningful regions; it is inherently subjective (different annotators produce different boundaries) and is formally a **clustering problem** in a pixel feature space.
- **Classical segmentation** (K-Means, Mean Shift) works bottom-up from low-level pixel features; it fails on complex scenes because hand-crafted rules cannot generalise.
- **Deep learning segmentation** works top-down, learning hierarchical representations from data; key architectures are FCN, **U-Net**, DeepLab, and Mask R-CNN.
- **U-Net** is an encoder–decoder CNN with skip connections that concatenate fine spatial detail from the encoder directly into the decoder — enabling pixel-precise segmentation even from very small datasets.
- Core CNN building blocks: **convolution** (local weighted sum with a sliding kernel), **pooling** (spatial subsampling), **upsampling / transposed convolution** (learnable spatial expansion in the decoder).
- Training U-Net uses **pixel-wise softmax** followed by **cross-entropy loss** over every pixel; data augmentation (elastic deformation, flips, rotations) is essential when data are scarce.

---

## Detailed notes

### 1. Image perception by humans — Gestalt theory

**Core claim:** Human visual perception does not process scenes as isolated pixels. Instead we spontaneously organise visual information into **meaningful structures**: objects, groups, contours, foreground and background.

> *"We do not see pixels; we see objects. We perceive objects in their entirety before their individual parts."*

This is directly motivating for image segmentation: the goal is to **divide an image into meaningful regions**, mirroring what the brain does effortlessly.

#### 1.1 Gestalt principles

| Principle | Description | Visual example |
|---|---|---|
| **Proximity** | Closer elements are grouped together | A row of equally-spaced cloud shapes reads as one group; pairs of closely-spaced clouds read as three pairs |
| **Similarity** | Elements that share colour, shape, or texture are grouped | Blue clouds grouped together; orange and purple clouds grouped separately regardless of their position |
| **Common fate** | Elements that move (or change) together are grouped | Clouds all moving in the same direction form one perceptual group; those moving differently form another |
| **Common region / Connectivity** | Elements enclosed in the same region, or connected by a line, are grouped | Pairs of clouds enclosed by a red oval form groups; clouds connected by a red bar also group together |
| **Symmetry** | Parallel and symmetrical features are grouped | Pairs of mirrored curved lines are perceived as enclosing shapes (e.g., two facing parentheses read as one unit) |

**Exam note:** Be ready to name all five principles and give a brief definition of each. The slide examples (cloud-shape diagrams) are a favourite for matching questions.

---

### 2. Image segmentation — problem formulation

#### 2.1 What is segmentation?

**Goal:** partition an image into regions such that each region is visually and/or semantically coherent.

- It is a **highly subjective** process: D. Martin et al. (2001) showed that three different human annotators produce noticeably different boundary drawings for the same image (horse/cattle scene, office scene).
- Formally, image segmentation = **a clustering problem**.
- Each pixel is represented as a **feature vector**: at minimum [R, G, B], but can be extended with spatial coordinates (x, y), depth (d), motion (m), texture (t), etc.
- **Pixel similarity** = distance between feature vectors (typically L2 / Euclidean distance).

**Figure description (baboon image):** A photograph of a mandrill's colourful face (red nose, blue cheeks) is plotted as a 3-D scatter in RGB space. The red-nose pixels cluster as a tight red cloud near (R≈200, G≈50, B≈50), the blue-cheek pixels cluster separately, and the dark fur pixels cluster near the origin. Segmenting the image = finding and labelling these clusters in feature space.

#### 2.2 Two main segmentation strategies

| Strategy | Governing idea | Approach | Examples |
|---|---|---|---|
| **Bottom-up** | Pixels belong together because they **look similar** | Color similarity, texture similarity, local continuity, edge detection, region growing, clustering | K-Means, Mean Shift |
| **Top-down** | Pixels belong together because they come from the **same object** | Object-level reasoning; guided by semantics, prior knowledge, learned object representations | FCN, U-Net, DeepLab, Mask R-CNN |

Bottom-up = classical CV; top-down = deep learning.

---

### 3. Classical segmentation method 1 — K-Means

#### 3.1 Algorithm recap

1. Choose K (the number of segments).
2. Initialise K cluster centres randomly in feature space.
3. Assign each pixel to its nearest centre (L2 distance).
4. Recompute each centre as the mean of assigned pixels.
5. Repeat steps 3–4 until convergence.

**Feature vector choices:**
- `[R, G, B]` — colour only; segments purely by colour.
- `[R, G, B, x, y]` — colour + position; enforces spatial compactness (neighbouring pixels are preferred).

#### 3.2 Worked example — mandrill image

| K | Feature vector | Result |
|---|---|---|
| 2 | [R, G, B] | Two broad clusters: a tan/brown region (dark fur + background) and a light/grey region; coarse, missing colour detail |
| 8 | [R, G, B] | Better separation: distinct regions for red nose, blue face, orange/yellow areas, dark fur |

**Figure description (peppers, K=16):** Left column: original photo of mixed bell peppers (red, green, yellow, orange) on a purple cloth. Top-right: K-Means with `[R, G, B]` only — produces regions that are colour-correct but spatially fragmented (the same yellow colour on two different peppers merges into one region). Bottom-right: K-Means with `[R, G, B, x, y]` — regions are more spatially coherent; individual peppers are better delineated because spatial proximity is penalised.

#### 3.3 Properties of K-Means

| Pro | Con |
|---|---|
| Simple and reasonably fast | Must choose K in advance |
| Deterministic once initialised | Sensitive to initialisation (different random seeds → different results) |
| | Sensitive to outliers |

---

### 4. Classical segmentation method 2 — Mean Shift

#### 4.1 Motivation

Mean Shift avoids having to specify K. Instead it finds the **modes (peaks)** of the density of pixels in feature space using **kernel density estimation (KDE)**.

**Key idea:** Points iteratively move toward regions of **high local density** in feature space.

#### 4.2 Algorithm

**Given:** Distribution of N pixels in feature space, e.g. `f = (x, y, R, G, B)`.
**Task:** Find density modes (clusters) in that feature space.

Steps:
1. **Initialise:** for each pixel i, set mean `m_i = f_i` (pixel's own feature vector).
2. **Repeat for each mean m_i:**
   - a. Place a kernel window of size **W** around `m_i`.
   - b. Compute the local centroid `m` of all points inside the window; assign `m_i = m`.
   - c. Stop if `|| m_i^(t+1) - m_i^(t) || < ε` (shift is smaller than threshold).
3. **Assign:** pixels whose trajectories converge to the **same mode** belong to the same cluster (segment).

**Intuition (density landscape):** Each hill in the density surface represents one cluster. Pixels "climb" the nearest hill by repeatedly computing the local centroid. Red dots on the 3-D density surface diagram mark the peaks (modes), and the arrows show pixels flowing uphill toward them.

#### 4.3 Comparison: K-Means vs Mean Shift on the same data

**Figure description:** Three scatter plots of the same 2-D point cloud in the shape of a "Mickey Mouse" outline (a large central blob with two smaller blobs at top-left). K-Means with k=3 divides the large central region in a geometrically straight-line way, slightly mis-assigning boundary points. Mean Shift naturally discovers the true density peaks and follows the irregular contours of each blob, producing more accurate clusters.

**Figure description (peppers, comparison):** Input image of peppers. K-Means (k=16) result: fragmented, some incorrect region merges at boundaries. Mean Shift (W=21) result: smoother region boundaries, individual peppers better separated, more visually natural.

**Figure description (landscape):** Three Mean Shift segmentations of lake/tree/sky scenes show clear sky, water, tree and land regions delineated cleanly with white boundary outlines.

#### 4.4 Properties of Mean Shift

| Pro | Con |
|---|---|
| No K to choose | Simple but computationally expensive (O(N²) per iteration) |
| Finds arbitrary number of segments | Results depend heavily on bandwidth W |
| No initialisation required | |
| Robust to outliers | |

---

### 5. Limitations of classical methods and the case for deep learning

**Main limitation of classical CV:**
Hand-crafted rules (colour thresholds, local similarity heuristics) often fail on:
- Complex, cluttered scenes
- Illumination changes
- Texture variations
- Object variability (e.g., "cat" has infinite appearances)

**Classical methods rely on:** manually designed features, local similarity rules, fixed heuristics.

**Deep learning methods learn:** object representations, semantic understanding, hierarchical visual features — all directly from data.

Key DL segmentation architectures:
- **FCN** (Fully Convolutional Network)
- **U-Net** (Ronneberger, Fischer, Brox — 2015, ~140,000 citations)
- **DeepLab**
- **Mask R-CNN**

> *"Instead of defining segmentation rules manually, the model learns them directly from data."*

---

### 6. Theoretical refresher — CNN building blocks

#### 6.1 Convolution

**What it is:** A local operation where a small **kernel** (filter) slides over an image (or feature map), computing a **weighted sum** at each position.

- Each output neuron depends only on a small region of the input: the **local receptive field**.
- The kernel weights are **learned** during training.
- Output is called a **feature map**.

**Figure description:** A 6×6 binary input matrix is convolved with a 3×3 kernel (values: 1,1,0 / 0,1,0 / 0,0,1). The sliding window produces a 4×4 output feature map (values 4,2,2,3 / 2,3,4,2 / 3,2,4,4 / 2,2,5,3). Each output cell = dot product of the kernel with the corresponding 3×3 patch in the input.

#### 6.2 Receptive field

The **receptive field** of a neuron in layer L is the region of the original input image that influences that neuron's output.

- Deeper layers have larger receptive fields: a 3×3 conv applied twice gives an effective 5×5 receptive field on the input.
- Larger receptive field = more global context for predictions.
- **Importance:** the receptive field determines the context available to each neuron when making a prediction. Larger is needed to understand objects; smaller captures fine local detail.

**Figure description:** A diagram shows three feature map grids (Layer 1, 2, 3). The highlighted region in Layer 1 (large red/orange patch) maps to a medium region in Layer 2 (orange) which maps to a single cell in Layer 3 (red). This illustrates how a single deep neuron aggregates a wide area of the input.

#### 6.3 Pooling (subsampling)

**What it is:** Reduces the **spatial size** of feature maps by summarising patches into a single value.

- **Max pooling:** take the maximum value in each patch (most common).
- **Average pooling:** take the mean value in each patch.

**Example:** A 4×4 feature map with 2×2 max-pooling becomes a 2×2 map. From patch [29,15,0,100] → output 100; from [28,184,70,38] → 184; etc.

**Purpose:** Reduces computational cost; provides some translation invariance; increases the effective receptive field.

#### 6.4 Upsampling

**What it is:** Increases the spatial resolution of feature maps — the opposite of pooling. Needed in decoder paths to recover spatial detail.

Methods:
| Method | Description |
|---|---|
| **Nearest Neighbor** | Each pixel is replicated to fill the 2×2 block |
| **Bed of Nails** | Values placed at corners; rest filled with zeros |
| **Max Unpooling** | Values placed back in their original max-pool positions; rest zeros |
| **Bilinear / Bicubic / Lanczos Interpolation** | Smooth interpolation using surrounding values |
| **Transposed convolution** | Learnable upsampling — used in U-Net decoder |

#### 6.5 Transposed convolution (learnable upsampling)

**What it is:** Used in the decoder to upsample feature maps by **learning** how to expand spatial resolution.

**How it works internally:**
1. Insert zeros between input elements (according to stride), creating a "sparser" intermediate map.
2. Apply a learnable convolutional kernel over this expanded map, producing a higher-resolution output.
3. Because weights are trained, the layer learns to upsample in a more expressive way than fixed interpolation.

**Example (stride 2):** A 2×2 input [0,1 / 2,3] with kernel [1,4 / 2,3] and stride 2 produces a 4×4 output. The zero-insertion step places each input value at stride-spaced positions, then the kernel fills in the surrounding cells. Result: [0,0,1,4 / 0,0,2,3 / 2,8,3,12 / 4,6,6,9].

**Caveat:** Output size also depends on padding and output_padding; transposed convolutions may introduce checkerboard artifacts if not used carefully.

---

### 7. U-Net architecture

**Origin:** Ronneberger, Fischer, Brox (2015) — designed for biomedical image segmentation. ~31 million trainable parameters.

#### 7.1 Overall structure

U-Net has a characteristic **U-shape** with three parts:
1. **Encoder (contracting path)** — extracts features, loses spatial resolution.
2. **Bottleneck** — most compressed representation; deepest point of the U.
3. **Decoder (expanding path)** — restores spatial resolution using upsampling.
4. **Skip connections** — bridge encoder to decoder at each resolution level.

**Figure description (full U-Net diagram):** The architecture is shown as a series of blue rectangular blocks (feature maps) arranged in a U shape. Left side (encoder): input tile (572×572, 1 channel) → conv 3×3 ReLU → 570×570, 64 ch → conv → 568×568, 64 ch → max pool 2×2 (red arrow down) → 284×284, 128 ch → ... continues down to 32×32, 1024 ch at the bottleneck. Right side (decoder): up-conv 2×2 (green arrows up) interleaved with conv blocks, going back up to 388×388, 2 ch (output segmentation map). Gray horizontal arrows represent skip connections (copy and crop) connecting each encoder level to its mirror decoder level. Final layer is conv 1×1 (dark green arrow) mapping 64 channels to 2 output classes.

#### 7.2 Encoder (contracting path)

- Consists of repeated: **3×3 convolution + ReLU**, **3×3 convolution + ReLU**, then **2×2 max pooling** (stride 2).
- At each downsampling step: spatial dimensions halve, number of channels **doubles** (64 → 128 → 256 → 512).
- **Why double channels?** As spatial maps shrink, the network compensates by increasing channels to learn richer, more abstract patterns. Doubling channels at each stage lets the encoder capture progressively complex features: edges → textures → shapes → objects.

#### 7.3 Bottleneck

- The most spatially compressed representation (e.g., 32×32 or 28×28 feature maps for a 512×512 input).
- Two 3×3 conv layers at 1024 channels.
- No pooling after; transitions directly to the decoder via up-conv.

**Diagram (encoder to bottleneck):** A wide tall rectangle (encoder feature map) transitions via a rightward arrow to a short wide rectangle (bottleneck) — indicating reduction in spatial height and width while maintaining or increasing channel depth.

#### 7.4 Decoder (expanding path)

- Consists of repeated: **2×2 up-convolution** (transposed conv, halves channels, doubles spatial size), **concatenation with skip connection**, **3×3 conv + ReLU**, **3×3 conv + ReLU**.
- At each upsampling step: spatial dimensions double, channels **halve** (1024 → 512 → 256 → 128 → 64).
- **Why halve channels?** The decoder is converting abstract features back into fine spatial details; fewer channels needed. Halving keeps computation balanced and transitions gradually from abstract to spatially precise.
- Final layer: **1×1 convolution** maps the 64-channel feature map to the number of output classes (2 in the original paper for binary segmentation).

**Diagram (bottleneck to decoder):** A short wide rectangle (bottleneck) transitions to a tall wide rectangle (decoder feature map) — the reverse of the encoder transition, indicating spatial expansion.

#### 7.5 Skip connections

**Purpose:** Transfer high-resolution spatial detail from the encoder directly to the corresponding level of the decoder, preventing the loss of fine-grained information that occurs as features travel through the bottleneck.

**Mechanism:** The encoder feature map (before max pooling) is **copied and cropped** to match the decoder feature map size, then **concatenated along the channel axis** (not added — concatenated, so both sets of features are preserved).

**Why they matter:**
- Shallow encoder layers encode: fine location, local detail, precise boundaries.
- Deep encoder / bottleneck encodes: coarse, semantic, global, abstract information.
- The decoder needs both: semantic understanding (from the bottleneck) + precise spatial location (from skip connections).

**Diagram (skip architecture):** A horizontal timeline from "Shallow" (left) to "Deep" (right) shows: image → conv1 → pool1 → conv2 → pool2 → conv3 → pool3 → ... The left side is labelled Fine / Location / Local / Detail; the right side is labelled Coarse / Semantic / Global / Abstract. Skip connections bridge left-side detail into the right-side decoder.

#### 7.6 Image size impact on U-Net

| Input size | Bottleneck size | Impact |
|---|---|---|
| 256×256 | 16×16 | Less memory, faster; smaller receptive field may miss large context |
| 512×512 | 32×32 | Standard config; more context captured; more memory; many implementations require input to be a multiple of 2^(pool depth) |

**Key insight:** Increasing input size scales spatial dimensions throughout the entire network (encoder, bottleneck, decoder), increasing memory and compute cost proportionally. However, **parameter count stays roughly the same** because U-Net uses convolutions with fixed filter counts (parameters depend on kernel size and channel count, not spatial resolution). Larger inputs capture more context but require careful design of padding and pooling.

---

### 8. U-Net training

#### 8.1 Dataset

Original paper used **electron microscopy (EM) images** from the ISBI challenge: grayscale images of biological tissue with cell membrane annotations.

**Figure description (dataset example):** Left: a grayscale EM microscopy image showing tightly packed cells with dark boundaries (membranes) and lighter interiors. Right: the corresponding binary annotation — black lines (membranes) on a white background, where each enclosed region is one cell to be segmented.

#### 8.2 Data augmentation

Because medical datasets are extremely small (only ~30 training images), U-Net relied heavily on augmentation:
- **Elastic deformations** — the most important augmentation; simulates biological tissue deformation.
- **Random rotations, translations, scalings**
- **Grey-value variations**
- **Mirroring (flip)**

Elastic deformation was key to teaching the network to generalise to the natural variability of tissue morphology.

#### 8.3 Training hyperparameters

| Parameter | Value |
|---|---|
| Optimizer | SGD (Stochastic Gradient Descent), via Caffe framework |
| Momentum | 0.99 (very high; updates strongly influenced by past gradients, stabilises training) |
| Learning rate | Started high, manually reduced during training |
| Batch size | 1 image per iteration |
| Input tile size | 512×512 |

#### 8.4 Loss function

U-Net uses **pixel-wise softmax** followed by **cross-entropy loss**:

**Softmax** converts raw logits `a_k(x)` into class probabilities for pixel x:

```
p_k(x) = exp(a_k(x)) / sum_{k'} exp(a_{k'}(x))
```

**Cross-entropy loss** (minimised over all pixels):

```
H(p, q) = - sum_x  p(x) * log q(x)
```

where `p(x)` is the ground-truth distribution (one-hot) and `q(x)` is the predicted distribution. The loss is computed **for every pixel** in the segmentation map — this makes U-Net a fully dense prediction model.

---

## Key terms (glossary)

- **Gestalt theory** — psychological theory stating humans perceive whole objects, not isolated elements; visual grouping follows principles of proximity, similarity, common fate, common region, and symmetry.
- **Proximity** — Gestalt principle: spatially closer elements are grouped together.
- **Similarity** — Gestalt principle: elements sharing visual properties (colour, shape) are grouped.
- **Common fate** — Gestalt principle: elements with the same motion or change are grouped.
- **Common region / Connectivity** — Gestalt principle: elements enclosed in the same area or connected by a line are grouped.
- **Symmetry** — Gestalt principle: parallel and symmetrical features are grouped.
- **Image segmentation** — task of partitioning an image into meaningful, coherent regions (= clustering in pixel feature space).
- **Bottom-up segmentation** — grouping based on pixel-level similarity (classical CV: K-Means, Mean Shift).
- **Top-down segmentation** — grouping guided by semantic/object knowledge (deep learning: U-Net, Mask R-CNN).
- **K-Means** — iterative clustering algorithm requiring the user to specify K; sensitive to initialisation and outliers.
- **Mean Shift** — non-parametric clustering via kernel density estimation; finds modes automatically; no K required but computationally expensive.
- **Kernel window W** — bandwidth parameter in Mean Shift; controls the scale of the density estimation.
- **Convolution** — sliding-kernel operation producing a feature map; weights are learned.
- **Receptive field** — the region of the input image that influences a given neuron's output; grows with depth.
- **Pooling** — spatial subsampling (max or average) that reduces feature map size.
- **Upsampling** — spatial resolution increase (nearest-neighbor, bilinear, or transposed convolution).
- **Transposed convolution** — learnable upsampling via zero-insertion + convolution; used in U-Net decoder.
- **Encoder (contracting path)** — U-Net's left half; convolution + max-pooling, channels double at each step.
- **Bottleneck** — deepest, most compressed U-Net layer; highest semantic abstraction.
- **Decoder (expanding path)** — U-Net's right half; up-convolution + concatenation, channels halve at each step.
- **Skip connections** — direct paths from encoder feature maps to the corresponding decoder level; concatenated along the channel axis to restore spatial detail.
- **Pixel-wise softmax** — softmax applied independently to each pixel's class logits.
- **Cross-entropy loss** — loss function summed over all pixels: `H(p,q) = -sum_x p(x) log q(x)`.
- **Elastic deformation** — augmentation simulating non-rigid tissue deformation; key for biomedical U-Net training.
- **FCN** — Fully Convolutional Network; early deep segmentation model, predecessor to U-Net.
- **Mask R-CNN** — instance segmentation network combining object detection with per-object pixel masks.

---

## Exam targets

1. **Name and define all five Gestalt principles** with a brief visual description of each. Explain why Gestalt theory motivates image segmentation.

2. **Formulate image segmentation as a clustering problem.** What is a pixel feature vector? What features can be included beyond [R, G, B]? How is pixel similarity defined?

3. **Contrast bottom-up vs top-down segmentation.** Give two classical and two deep learning methods for each.

4. **Describe K-Means for segmentation:** algorithm steps, the role of K, and the effect of adding spatial coordinates (x, y) to the feature vector. Explain why K=2 gives a coarser result than K=8.

5. **Describe Mean Shift for segmentation:** the kernel density estimation idea, the hill-climbing metaphor, the full algorithm (initialise, kernel window, centroid update, convergence criterion `|| m_i^(t+1) - m_i^(t) || < ε`), and cluster assignment. Compare to K-Means.

6. **Compare K-Means and Mean Shift** on the four properties: ease of use, sensitivity to initialisation, handling of outliers, number of clusters.

7. **Explain the four main limitations of classical segmentation** that motivate the move to deep learning.

8. **Define convolution** in the context of CNNs. What is a feature map? What is the local receptive field? How does the effective receptive field grow with depth?

9. **Explain max pooling and average pooling** with the worked numerical example (4×4 → 2×2 with a 2×2 pool).

10. **Explain transposed convolution:** mechanism (zero insertion + learned kernel), why it is called "learnable upsampling," and its advantage over fixed interpolation. Be able to trace through the 2×2 → 4×4 example.

11. **Draw and label the U-Net architecture:** encoder (contracting path), bottleneck, decoder (expanding path), skip connections, final 1×1 conv. Indicate where channels double (encoder) and halve (decoder), and where spatial dimensions halve (pooling) and double (up-conv).

12. **Explain skip connections in U-Net:** mechanism (copy-and-crop + channel concatenation), what information they transfer (fine spatial detail from the encoder), and why they are necessary (bottleneck loses spatial precision).

13. **Trace image sizes through U-Net** for a 256×256 input: after pool1 → 128×128; pool2 → 64×64; pool3 → 32×32; pool4 → 16×16 (bottleneck); up1 → 32×32; up2 → 64×64; up3 → 128×128; up4 → 256×256 (output).

14. **Write the U-Net loss function:** pixel-wise softmax formula `p_k(x) = e^{a_k(x)} / sum_{k'} e^{a_{k'}(x)}`; cross-entropy `H(p,q) = -sum_x p(x) log q(x)`. Explain why the loss is computed per pixel.

15. **Describe U-Net training settings:** optimizer (SGD), momentum (0.99), batch size (1), data augmentation (especially elastic deformation), and why augmentation was critical (only ~30 training images).

---

## Pitfalls

- **Gestalt is not just a list — know the mechanism.** Proximity groups by distance; similarity by shared visual properties; common fate by shared motion/change; common region by enclosure or connection; symmetry by parallel/mirror structure.
- **Segmentation subjectivity.** The correct answer to "how should this image be segmented" is not unique. Different humans draw different boundaries. This matters for evaluating algorithms.
- **K-Means with [R,G,B,x,y] is NOT the same as [R,G,B].** Adding spatial coordinates enforces spatial compactness: a red pixel at position (10,10) and a red pixel at (400,400) will be assigned to different clusters because the L2 distance in 5-D feature space is large even if colours match.
- **Mean Shift bandwidth W is not K.** You do not choose the number of clusters; you choose the scale of the kernel. Different W values give different numbers of segments.
- **Transposed convolution ≠ deconvolution** in the mathematical sense. It is better described as a stride-1 convolution applied to a zero-padded input — the weights are learned, not inverted.
- **Skip connections concatenate, they do not add.** This is distinct from ResNet residual connections (which add). Concatenation doubles the channel count at the concatenation point before the following conv reduces it.
- **U-Net parameter count does not depend on input size.** Only memory and FLOPs scale with input size, not the number of trainable weights.
- **Pixel-wise softmax is applied independently to each pixel.** It does not model spatial relationships between pixels at inference time; spatial context is captured via the convolutional receptive field, not by the softmax itself.
- **The bottleneck is the worst place for spatial precision.** It is the best place for semantic understanding. Spatial precision is recovered via the decoder + skip connections — not from the bottleneck alone.
- **K-Means is sensitive to initialisation; Mean Shift is not.** This is a key exam differentiator.
