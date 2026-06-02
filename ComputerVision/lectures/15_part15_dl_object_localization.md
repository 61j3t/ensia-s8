# Part 15 — Deep Learning for Computer Vision: Object Localization & Detection

> Lecture XVI, Sid-Ahmed Berrani. ENSIA Computer Vision, 2025-2026.

---

## Bird's eye view

- **Three problem levels:** image classification (one label) → classification with localization (one label + one bounding box) → object detection (multiple labels + multiple bounding boxes, possibly of different categories).
- **Bounding box regression** extends a standard CNN by adding a second output head that predicts four real-valued offsets $(b_x, b_y, b_w, b_h)$ alongside the softmax class output; both are trained jointly using a combined loss.
- **Sliding-window detection** runs a trained classifier at every position and scale of an image; computational cost is prohibitive unless the fully-connected layers are converted to convolutional layers, allowing a single forward pass to produce a spatial grid of predictions.
- **IoU (Intersection over Union)** is the canonical evaluation metric: a predicted box is a good detection when $\text{IoU} > 0.5$; IoU also drives the Non-Maximum Suppression (NMS) algorithm that collapses duplicate predictions of the same object.
- **Anchor boxes** (also called default boxes or prior boxes) break the one-object-per-cell limitation of basic grid detectors: each grid cell predicts one output vector per anchor, allowing multiple objects of different aspect ratios to be detected at the same location. Anchors are learned from training data via k-means clustering in YOLO v2+.
- The complete YOLO-style pipeline is: divide image into an $S \times S$ grid; for each cell and each anchor predict $(P_c, b_x, b_y, b_w, b_h, c_1, \ldots, c_C)$; apply NMS per class.
- **Landmark detection** is a natural extension: instead of four box coordinates, the network predicts $2L$ coordinates for $L$ semantic keypoints (e.g. 64 face landmarks).

---

## Detailed notes

### 1. The problem hierarchy: classification → localization → detection

The lecture opens by placing object localization in a spectrum of four increasingly complex tasks:

| Task | Number of objects | Output |
|---|---|---|
| Image classification | 1 (assumed) | Class label |
| Classification with localization | 1 | Class label + 1 bounding box |
| Object detection | Multiple, different categories | Class label + bounding box per object |
| (Segmentation — mentioned implicitly) | Multiple | Pixel-level masks |

**Key distinction:** classification with localization still assumes a single dominant object; detection must handle an unknown, variable number of objects from potentially different categories in the same image.

---

### 2. Classification with localization — output vector and loss

#### 2.1 Network architecture

A standard CNN backbone (e.g. AlexNet, VGG) is extended with **two output heads** after the final pooling/FC layers:

1. **Classification head:** softmax over $C$ classes (e.g. 4: pedestrian, car, motorcycle, background).
2. **Regression head:** 4 real-valued outputs $(b_x, b_y, b_w, b_h)$ describing the bounding box.

Slide 4 shows: input image → convolutional layers (blue cuboid) → further layers → two output nodes (softmax with 4 values, and bounding box with 4 values).

#### 2.2 Output vector structure

The full prediction vector for one image is:

```math
y = \begin{bmatrix} P_c \\ b_x \\ b_y \\ b_w \\ b_h \\ c_1 \\ c_2 \\ c_3 \end{bmatrix}
```

where:
- $P_c$ — probability that any object of interest is present (objectness score). $P_c = 1$ means object present; $P_c = 0$ means background (all other entries are "don't care" and ignored in the loss).
- $b_x, b_y$ — coordinates of the **center** of the bounding box (normalized relative to the image or grid cell).
- $b_w, b_h$ — **width and height** of the bounding box (normalized relative to the image or cell).
- $c_1, c_2, c_3$ — one-hot class indicators (if $P_c = 1$, exactly one $c_i = 1$). With 4 classes including background, 3 binary indicators suffice.

**Example (slide 5):** image of a car on a snowy road produces $y = [1, b_x, b_y, b_w, b_h, 0, 1, 0]^\top$ (object present, class = car = $c_2$). An image of an empty road produces $y = [0, ?, ?, ?, ?, ?, ?, ?]^\top$ — the question marks mean those values are irrelevant and must not contribute to the loss.

#### 2.3 Loss function

The loss is a **conditional, multi-task loss**:

```math
\mathcal{L} = \begin{cases}
\mathcal{L}_\text{cls}(\hat{c}, c) + \mathcal{L}_\text{reg}(\hat{b}, b) & \text{if } P_c = 1 \\
\mathcal{L}_\text{obj}(\hat{P}_c, 0) & \text{if } P_c = 0
\end{cases}
```

- $\mathcal{L}_\text{cls}$: cross-entropy or squared error on the class predictions.
- $\mathcal{L}_\text{reg}$: mean squared error (or smooth-$\ell_1$) on the four bounding-box coordinates.
- $\mathcal{L}_\text{obj}$: binary cross-entropy on $P_c$.

When $P_c = 0$ (background), only the objectness loss is computed; the regression and class losses are masked out because the box coordinates are meaningless for a background patch.

---

### 3. Landmark detection

Landmark detection is a direct generalization of bounding-box regression: instead of predicting 4 box coordinates the network predicts $2L$ coordinate pairs $(l_{1x}, l_{1y}), \ldots, (l_{Lx}, l_{Ly})$ for $L$ semantic keypoints.

**Example (slide 7):** a CNN for face analysis outputs:
- Face / Not face (binary)
- $l_{1x}, l_{1y}$ through $l_{64x}, l_{64y}$ — 64 facial landmark positions (eye corners, lip corners, nose tip, etc.)

The same bounding-box pipeline on the left (car in snowy scene, $b_x, b_y, b_h, b_w$) contrasts with the face landmark pipeline on the right (face with 64 red dots scattered over facial features).

**Applications:** facial expression recognition, pose estimation, gaze tracking, AR filters.

---

### 4. Object detection — sliding-window approach

#### 4.1 Motivation: from localization to detection

For detection with multiple objects, the approach is:
1. Train a CNN on **tightly cropped** positive (object) and negative (background) image patches — a binary or multiclass classifier on a single patch.
2. At inference time, **slide the trained window** over the full image at multiple scales and positions; classify each crop.

Slide 8 shows a training set where positive examples are closely cropped cars (label = 1) and negative examples are background scenes (label = 0), all fed through a single CNN.

#### 4.2 The computational cost problem

Naive sliding window requires running the CNN separately for every $(x, y, \text{scale})$ combination. With a stride of 1 pixel over a 1000×1000 image at 5 scales, this is millions of forward passes — completely intractable.

Slide 9 illustrates: three snapshots of the same road scene showing small, medium, and large sliding window sizes (orange squares) being dragged across the image at different positions, making the cost explicit.

#### 4.3 Convolutional implementation of sliding window

**Key insight:** replace all fully-connected layers with equivalent convolutional layers. Then a single forward pass over the full image computes all sliding-window positions simultaneously.

**Conversion (slide 10 and 11):** consider a small CNN trained on $14 \times 14 \times 3$ crops:

```
14×14×3 → [conv 5×5] → 10×10×16 → [maxpool 2×2] → 5×5×16 → [FC 400] → [FC 400] → softmax(4)
```

The FC layers are converted to convolutions:
- The first FC (5×5×16 → 400) becomes a $5 \times 5$ conv with 400 filters → output $1 \times 1 \times 400$.
- The second FC (400 → 400) becomes a $1 \times 1$ conv with 400 filters → $1 \times 1 \times 400$.
- The output FC (400 → 4) becomes a $1 \times 1$ conv with 4 filters → $1 \times 1 \times 4$.

Now feed the full image (e.g. $16 \times 16 \times 3$) through this fully-convolutional network:

```
16×16×3 → [conv 5×5] → 12×12×16 → [maxpool 2×2] → 6×6×16 → 2×2×400 → 2×2×400 → 2×2×4
```

The $2 \times 2 \times 4$ output contains **4 class predictions for 4 sliding-window positions**, all computed in one forward pass. Each spatial position in the output feature map corresponds to one window position in the input.

Slide 12 shows the full diagram with both the original (top row: $14 \times 14 \times 3$ → $1 \times 1 \times 4$) and the larger-image version (bottom row: $16 \times 16 \times 3$ → $2 \times 2 \times 4$), making the correspondence explicit.

Slide 13 shows the resulting $3 \times 3$ output grid overlaid on the snowy car image, with the car-containing cell highlighted in red — the network correctly identifies which spatial region contains the car.

**Remaining limitation:** the bounding boxes predicted are fixed to the grid cell positions; the network cannot predict a box that spans the exact pixel boundaries of the object. This motivates anchor-box-based prediction of offsets.

---

### 5. Evaluating detections: Intersection over Union (IoU)

**Definition:**

```math
\text{IoU} = \frac{\text{Area of Intersection}}{\text{Area of Union}}
```

Slide 14 illustrates: the ground-truth box (blue/purple) and the predicted box (red/orange) partly overlap around a car in a snowy scene. The IoU is the ratio of the overlapping rectangle area to the total area covered by both boxes.

**Threshold:** by convention, $\text{IoU} > 0.5$ is considered a good (correct) detection. Some benchmarks (e.g. COCO) average over thresholds from 0.5 to 0.95.

**Range:** $\text{IoU} \in [0, 1]$. $\text{IoU} = 1$ means perfect overlap (predicted = ground truth). $\text{IoU} = 0$ means no overlap.

**Use in training:** IoU is used to match predicted anchors to ground-truth boxes during training — the anchor with the highest IoU to a ground-truth box is assigned to that ground truth.

---

### 6. Non-Maximum Suppression (NMS)

#### 6.1 The multiple-detections problem

When the convolutional sliding window runs over a $19 \times 19$ grid with anchor boxes, many adjacent cells can all fire high-confidence predictions for the same physical object. Slide 15 shows two cars in a road scene with 5 overlapping predicted boxes per car, each with a different confidence score (e.g. 0.8, 0.7, 0.9, 0.6, 0.7).

#### 6.2 NMS algorithm

Each prediction is a 5-d vector:

```math
\text{prediction} = \begin{bmatrix} p \\ b_x \\ b_y \\ b_w \\ b_h \end{bmatrix}
```

where $p$ is the objectness probability (or class-specific confidence = $P_c \times P(\text{class} \vert \text{object})$).

**Algorithm (slide 17):**

1. **Threshold:** discard all predictions with $p < 0.6$ (low-confidence boxes are removed immediately).
2. **Greedy suppression loop** — while any boxes remain:
   a. Pick the box with the **highest** $p$ value. Output this box as a confirmed detection.
   b. Discard all remaining boxes that have **IoU $> 0.5$** with the just-output box (they are likely duplicates of the same object).
3. Repeat until no boxes remain.

**Multi-class NMS:** run the algorithm independently for each class.

**Visual result:** from the messy cloud of overlapping boxes (slide 15, right), NMS produces one clean box per car (slide 16 shows the desired final output).

---

### 7. YOLO-style object detection with a grid

#### 7.1 Basic grid detector (one anchor per cell)

Divide the input image into an $S \times S$ grid (e.g. $19 \times 19$). For each cell, predict one output vector:

```math
y = \begin{bmatrix} P_c \\ b_x \\ b_y \\ b_w \\ b_h \\ c_1 \\ c_2 \\ c_3 \end{bmatrix}
```

- An object is assigned to the cell whose center falls inside the object's bounding box.
- $b_x, b_y$ are the center coordinates of the box **relative to the cell** (values in $[0, 1]$).
- $b_w, b_h$ are the width and height of the box **relative to the full image** (can be $> 1$ if the object spans multiple cells).

Slide 18 shows a $3 \times 3$ grid overlaid on a scene with a person and a car. The output $y$ vector is shown for the cell containing the person — the bounding box extends across multiple cells but is encoded relative to the containing cell.

**Limitation (slide 18):** each grid cell can predict only one object. If two objects' centers fall in the same cell, only one can be detected. This is solved by anchor boxes.

#### 7.2 YOLO key idea

"You Only Look Once" — the entire image is processed in a **single forward pass** of the network. Rather than running a classifier on thousands of crops, YOLO frames detection as a regression problem over a spatial grid, predicting boxes and classes directly from image pixels.

---

### 8. Anchor boxes

#### 8.1 Motivation

A single bounding box per grid cell cannot handle:
- Two objects whose centers fall in the same cell (e.g. a person standing next to a car).
- Objects with very different aspect ratios (a tall thin person vs. a wide flat car).

Slides 21-22 spell this out: objects can have different scales (small, medium, large) and aspect ratios (tall, wide, square); a single box per location is not flexible enough.

#### 8.2 Definition

**Anchor boxes** (also called **default boxes** or **prior boxes**) are a set of **predefined bounding boxes of various sizes and aspect ratios**. They act as reference templates that the model learns to adjust (via offset regression) to match target objects.

- Anchor box 1 (slide 19): tall and narrow (e.g. aspect ratio 0.5, for pedestrians).
- Anchor box 2 (slide 19): wide and short (e.g. aspect ratio 2.0, for cars).

#### 8.3 Output vector with anchors

With $A$ anchors per cell, the output vector for each cell has $A$ blocks, one per anchor:

```math
y = \begin{bmatrix} P_c \\ b_x \\ b_y \\ b_w \\ b_h \\ c_1 \\ c_2 \\ c_3 \\ \hline P_c \\ b_x \\ b_y \\ b_w \\ b_h \\ c_1 \\ c_2 \\ c_3 \end{bmatrix}
\leftarrow \text{Anchor box 1}
\quad
\leftarrow \text{Anchor box 2}
```

Slide 19 shows the stacked vector; slide 20 shows a concrete filled example where anchor box 1 (pedestrian-shaped) detects the person ($P_c = 1$, $c_1 = 1$, $c_2 = 0$, $c_3 = 0$) and anchor box 2 (car-shaped) detects the car ($P_c = 1$, $c_1 = 0$, $c_2 = 1$, $c_3 = 0$).

#### 8.4 Training with anchors

During training (slides 23-24), the ground-truth bounding boxes are matched to anchors using IoU:
- Each ground-truth box is assigned to the anchor (across all cells) that has the **highest IoU** with it.
- That anchor's output in the cell containing the object's center is trained to predict the correct box.
- All other anchors for that cell have $P_c = 0$ and their regression targets are "don't care".

Example (slide 23-24): a $3 \times 3$ grid over a snowy road scene with a truck. The truck is in the bottom-center cell. Its ground-truth box has high IoU with anchor box 2 (wide). The output for anchor box 2 in that cell is set to $[1, b_x, b_y, b_w, b_h, 0, 1, 0]^\top$ (car); anchor box 1 in the same cell gets $[0, ?, ?, ?, ?, ?, ?, ?]^\top$. All other cells get all-zero $P_c$ entries.

#### 8.5 Inference with anchors

During inference (slide 25):
- The network outputs box proposals: for each of the $S \times S$ cells and each of the $A$ anchors, one predicted box.
- Total raw predictions: $S \times S \times A$ boxes.
- NMS is then applied per class to remove duplicates.

Slide 25 shows the grid overlaid on a pedestrian street scene: before NMS (left) there are many overlapping blue boxes; after NMS (right) only the clean detected box per object remains.

#### 8.6 Designing anchor boxes: k-means clustering (YOLO v2+)

Earlier YOLO versions (v1) used manually designed anchors. From YOLO v2 onwards (slides 26-27), anchors are **learned from the training data**:

**Algorithm:**
1. Collect all ground-truth bounding boxes from the entire training dataset. Represent each box by its (width, height) pair normalized relative to the image or feature map.
2. Run **k-means clustering** on the set of (width, height) pairs.
3. The $k$ cluster centroids become the $k$ anchor box shapes.

**Custom distance metric:** YOLO does not use standard Euclidean distance. Instead it uses an IoU-based distance:

```math
d(\text{box}, \text{centroid}) = 1 - \text{IoU}(\text{box}, \text{centroid})
```

This avoids penalizing large boxes more than small ones (Euclidean distance is scale-dependent), making the clustering scale-invariant.

**Result:** the chosen $k$ anchors are the best representative shapes for the dataset's object size distribution.

---

### 9. Complete YOLO detection pipeline summary

The full pipeline, combining all concepts from the lecture:

1. **Input:** image of arbitrary size (resized to e.g. $416 \times 416$).
2. **Backbone CNN:** processes the full image in one forward pass (fully convolutional after FC-to-conv conversion).
3. **Grid division:** the output feature map represents an $S \times S$ grid (e.g. $13 \times 13$ or $19 \times 19$).
4. **Per-cell prediction:** for each cell and each of $A$ anchors, predict $(P_c, b_x, b_y, b_w, b_h, c_1, \ldots, c_C)$.
5. **Threshold:** discard boxes with $P_c < $ threshold (e.g. 0.6).
6. **NMS per class:** greedily keep the highest-confidence box; discard overlapping boxes ($\text{IoU} > 0.5$).
7. **Output:** final set of bounding boxes, each with a class label and confidence score.

**Total output tensor shape:** $S \times S \times A \times (5 + C)$ where 5 = $(P_c, b_x, b_y, b_w, b_h)$ and $C$ = number of classes.

---

### 10. Loss function for YOLO-style detection

The loss combines three components (applied only to anchors responsible for a detection; background anchors contribute only to the objectness term):

```math
\mathcal{L} = \mathcal{L}_\text{obj} + \mathcal{L}_\text{noobj} + \mathcal{L}_\text{cls} + \mathcal{L}_\text{reg}
```

- **$\mathcal{L}_\text{obj}$** — binary cross-entropy on $P_c$ for cells/anchors that **do** contain an object. Encourages the network to output $P_c \approx 1$ when an object is present.
- **$\mathcal{L}_\text{noobj}$** — binary cross-entropy on $P_c$ for cells/anchors that do **not** contain an object. Encourages $P_c \approx 0$. Usually weighted down (there are far more background anchors than object anchors).
- **$\mathcal{L}_\text{cls}$** — cross-entropy (or MSE) on the class predictions $(c_1, \ldots, c_C)$, computed only for anchors with an object ($P_c = 1$).
- **$\mathcal{L}_\text{reg}$** — MSE on the bounding box coordinates $(b_x, b_y, b_w, b_h)$, computed only for anchors with an object. Often uses the square root of width and height to reduce the influence of large boxes.

Combined (simplified form):

```math
\mathcal{L}_\text{reg} = \sum_{i,a} \mathbf{1}_{ia}^\text{obj} \left[ (b_x - \hat{b}_x)^2 + (b_y - \hat{b}_y)^2 + (\sqrt{b_w} - \sqrt{\hat{b}_w})^2 + (\sqrt{b_h} - \sqrt{\hat{b}_h})^2 \right]
```

where $\mathbf{1}_{ia}^\text{obj}$ is 1 if anchor $a$ in cell $i$ is responsible for an object, 0 otherwise.

---

### 11. Contextual note: R-CNN family (two-stage detectors)

The lecture focuses on YOLO-style (single-stage) detection but the broader context includes the influential two-stage family:

| Method | Key idea | Speed | Accuracy |
|---|---|---|---|
| R-CNN (2014) | Selective Search → 2000 region proposals → warp each to fixed size → CNN per crop → SVM | Very slow (~47s/image) | High |
| Fast R-CNN (2015) | Run CNN once on full image → extract features at proposal locations via RoI Pooling → classify + regress | ~2s/image | Higher |
| Faster R-CNN (2016) | Replace Selective Search with a Region Proposal Network (RPN) trained end-to-end on the same CNN features | ~0.2s/image | Higher |

**Key distinction from YOLO:**
- Two-stage detectors first generate region proposals (where objects might be) then classify them — more accurate but slower.
- Single-stage detectors (YOLO, SSD) predict boxes and classes directly from a grid in one shot — faster but historically less accurate on small objects.

Both Faster R-CNN and YOLO use anchor boxes internally. The RPN in Faster R-CNN generates proposals by classifying anchors as object/background and regressing box offsets; this is conceptually identical to the YOLO grid head.

---

## Key terms (glossary)

- **Image classification** — single label output for the whole image; no spatial localization.
- **Classification with localization** — single label + one bounding box $(b_x, b_y, b_w, b_h)$; assumes one dominant object.
- **Object detection** — multiple bounding boxes + labels for an unknown number of objects of potentially different classes.
- **Bounding box regression** — predicting the four real-valued coordinates $(b_x, b_y, b_w, b_h)$ of the tightest axis-aligned box around an object; trained with MSE or smooth-$\ell_1$ loss.
- **$P_c$ (objectness score)** — probability that any object of interest is present in the region; gates the class and regression losses.
- **Landmark detection** — predicting $2L$ coordinates of $L$ semantic keypoints instead of a bounding box; generalizes the regression head.
- **Sliding-window detection** — exhaustively classify every crop of every size and position in an image; correct but computationally intractable without the convolutional trick.
- **FC-to-conv conversion** — replacing fully-connected layers with equivalent convolutional layers so the network can process arbitrary-size inputs and produce a spatial grid of predictions in one forward pass.
- **IoU (Intersection over Union)** — $\frac{\text{area of intersection}}{\text{area of union}}$; ranges in $[0, 1]$; threshold 0.5 is the standard "good detection" criterion; also used as the anchor-matching criterion during training.
- **Non-Maximum Suppression (NMS)** — post-processing algorithm that removes duplicate detections of the same object by iteratively keeping the highest-confidence box and discarding all overlapping boxes with $\text{IoU} > 0.5$.
- **Grid cell** — one spatial location in the $S \times S$ output feature map; each cell is responsible for detecting objects whose centers fall within it.
- **Anchor box (default box / prior box)** — a predefined bounding box template of a specific size and aspect ratio, placed at each grid cell; the network predicts offsets from the anchor to the true object box.
- **Aspect ratio** — the width-to-height ratio of a bounding box; anchor boxes cover multiple aspect ratios (e.g. tall/narrow for pedestrians, wide/short for cars).
- **k-means clustering for anchors** — YOLO v2+ procedure: collect all ground-truth (width, height) pairs from the training set, cluster with k-means using IoU-based distance $d = 1 - \text{IoU}$, use the $k$ centroids as anchor shapes.
- **YOLO (You Only Look Once)** — single-stage object detector that frames detection as a grid regression problem, processing the entire image in one forward pass; produces $S \times S \times A \times (5 + C)$ output tensor.
- **Single-stage detector** — architecture that predicts class labels and bounding boxes directly from the feature map in one pass (YOLO, SSD); fast but less accurate on small/dense objects.
- **Two-stage detector** — architecture that first generates region proposals then classifies each (R-CNN family); slower but more accurate.
- **R-CNN** — Region-based CNN; uses Selective Search for proposals, warps and classifies each crop independently; slow.
- **Fast R-CNN** — runs CNN once on the full image; uses RoI Pooling to extract fixed-size features at proposal locations.
- **Faster R-CNN** — adds a Region Proposal Network (RPN) sharing the backbone CNN; end-to-end trainable, fast.
- **RPN (Region Proposal Network)** — a small fully-convolutional network that slides over the feature map and classifies each anchor as object/background while regressing box offsets; produces the proposals for the second stage.
- **RoI Pooling** — operation that extracts a fixed-size ($h \times w$) feature map from an arbitrary-sized region of the feature map by dividing the region into a grid and max-pooling each cell.
- **Multi-task loss** — combined loss over multiple output heads (objectness + classification + regression), weighted to balance their contributions.
- **Background class** — the "no object" category; when $P_c = 0$, the box coordinates and class probabilities are irrelevant and masked out from the loss.

---

## Exam targets

1. **Distinguish the three task levels** — image classification, classification with localization, and object detection. State the output format and number of objects for each. Be able to give concrete examples (car on a snowy road vs. traffic scene with multiple cars).

2. **Write the full output vector $y$** for classification with localization (slide 5 format): $[P_c, b_x, b_y, b_w, b_h, c_1, c_2, c_3]^\top$. Explain what each entry means and what happens to the vector when $P_c = 0$.

3. **Describe the loss function** for classification with localization: why it is conditional on $P_c$; what $\mathcal{L}_\text{cls}$, $\mathcal{L}_\text{reg}$, and $\mathcal{L}_\text{obj}$ are; why the box regression loss is masked when $P_c = 0$.

4. **Explain the FC-to-conv conversion** — given a CNN trained on $14 \times 14 \times 3$ patches, show step by step how the FC layers become convolutional and how feeding a $16 \times 16 \times 3$ image produces a $2 \times 2 \times 4$ output corresponding to 4 sliding-window positions.

5. **Define IoU** with the formula, state the threshold for a "good detection", and explain its two roles: (a) evaluation metric, (b) anchor-to-ground-truth matching criterion during training.

6. **State and execute the NMS algorithm**: given a set of boxes with confidence scores, walk through the steps (threshold $p < 0.6$, pick highest, discard $\text{IoU} > 0.5$, repeat). Be able to apply it manually to a small example.

7. **Explain the anchor-box limitation** of the basic grid detector (one object per cell) and describe how anchor boxes resolve it: one output block per anchor, each cell produces $A$ predictions, objects are assigned to the anchor with the highest IoU.

8. **Draw or describe the output tensor** for a YOLO-style detector: shape $S \times S \times A \times (5 + C)$, what each dimension represents, and how NMS is applied after.

9. **Describe anchor box design by k-means** (YOLO v2+): collect ground-truth (width, height) pairs; run k-means with $d = 1 - \text{IoU}$ as the distance; centroids become anchor shapes. Why IoU-based distance instead of Euclidean?

10. **Contrast single-stage (YOLO) and two-stage (Faster R-CNN) detectors** — architecture difference, speed/accuracy trade-off, and how anchors appear in both (RPN in Faster R-CNN, grid head in YOLO).

11. **Explain landmark detection** as a generalization of bounding-box regression: the network outputs $2L$ coordinate pairs; label consistency requirement (landmark $i$ must correspond to the same semantic point across all training images).

---

## Pitfalls

- **$P_c = 0$ does NOT mean the box coordinates are zero** — it means they are irrelevant and must be masked from the loss. A common mistake is to include regression or classification losses for background examples.
- **$b_x, b_y$ are relative to the grid cell, but $b_w, b_h$ are relative to the full image** in YOLO — these are different reference frames. Do not mix them up.
- **IoU > 0.5 is convention, not physics** — the threshold can be varied. COCO mAP averages over IoU thresholds 0.5, 0.55, …, 0.95. Always state which threshold you are using.
- **NMS is run per class independently** — two boxes of different classes with high overlap are NOT suppressed against each other. Suppression only applies within the same class.
- **The FC-to-conv trick does not change the learned weights** — the convolutional filter at each position computes exactly the same dot product as the original FC neuron; the only change is that the same operation is now applied at multiple spatial positions.
- **Anchor box 1 vs. anchor box 2 assignment is by highest IoU, not by class** — a pedestrian-shaped anchor detects pedestrians because pedestrian ground-truth boxes have higher IoU with the tall narrow anchor, not because of any explicit class assignment.
- **k-means for anchors uses IoU distance, not Euclidean** — Euclidean distance in (width, height) space penalizes large boxes disproportionately. IoU is scale-invariant and measures actual shape overlap, making it the correct metric for anchor shape clustering.
- **Each grid cell predicts $A$ boxes, not 1** — with anchor boxes, the output per cell scales linearly with the number of anchors. A $19 \times 19$ grid with 5 anchors produces $19 \times 19 \times 5 = 1805$ raw box proposals before NMS.
- **NMS threshold is separate from the confidence threshold** — first discard low-confidence boxes ($p < 0.6$), then apply NMS ($\text{IoU} > 0.5$). Confusing the two thresholds is a common error.
- **Landmark detection requires consistent labeling** — landmark $l_1$ must always be, for example, the left eye corner across every training image. If the labeling is inconsistent, the network cannot converge on a meaningful landmark.
- **YOLO assigns an object to exactly one cell** (the one containing the object's center) and exactly one anchor (the one with the highest IoU). If two objects have centers in the same cell and similar aspect ratios, one will be missed — this is a known limitation at high object density.
