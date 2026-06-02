# Computer Vision — Syllabus Skeleton (Phase 0)

> Course: Computer Vision — ENSIA (Sid-Ahmed Berrani, 2025-2026; Part 13 guest lecture by Luka Čehovin Zajc, Univ. Ljubljana)
> Use this as a **mental scaffold** + quick-scan map. Each part has its own detailed notes file (`0X_partX_topic.md`).

## The arc of the course

The course climbs the classic **vision hierarchy** — pixels → features → understanding:

- **Foundations (P1-P3)**: what CV is, how images form, color.
- **Low-level / image processing (P4-P6)**: enhancement, spatial filtering, frequency-domain filtering. *(image → image)*
- **Mid-level / features (P7-P12)**: features, edges, lines, corners, SIFT detector & descriptor. *(image → features/descriptors)*
- **High-level / advanced (P13-P16)**: motion analysis & tracking, deep learning (segmentation, detection), and model optimization for embedded deployment. *(image → concepts)*

| Vision level | Maps | Parts |
|---|---|---|
| Low (image processing) | image → image | P4, P5, P6 |
| Medium (features) | image → features/descriptors | P7-P12 |
| High (understanding) | image → concepts | P13, P14, P15, P16 |

**Recurring threads**: the gradient $\nabla I$ (edges P7/P8, corners P10, SIFT P11); scale & invariance (features P7, SIFT P11/P12); the spatial ↔ frequency duality (P5 convolution, P6 Fourier); voting/robust fitting (Hough + RANSAC P9).

---

## Part 1 — Introduction to Computer Vision (52 pp)
- **CV defined**: estimating geometric/dynamic properties of the 3-D world from images. The **inverse problem**: 3-D→2-D projection loses information, so recovering 3-D is underdetermined and needs models/priors.
- **Three vision levels**: low (image→image), medium (image→features), high (image→concepts).
- **Human vision & optical illusions** (Müller-Lyer, Adelson checker-shadow, motion aftereffect, pop-out) → vision is active inference using priors; parallels adversarial examples in CV.
- **Applications**: OCR, inspection, surveillance/tracking, biometrics, medical imaging, self-driving, stitching.
- **History**: 1970s line/stereo/flow → 1980s pyramids/variational → 1990s multiview → 2000s SIFT/learning → 2012+ deep learning.

## Part 2 — Image Formation (54 pp)
- **Light**: EM spectrum (visible 400-700 nm); radiance vs luminance vs brightness; BRDF for reflectance.
- **Camera models**: pinhole (inverted, dim; aperture trade-off blur↔diffraction); lenses; perspective projection.
- **Digital sensing pipeline**: photon → photoelectric charge $Q \propto \text{photons} \times \text{exposure}$ → voltage → ADC → digital number; CCD vs CMOS.
- **Image as 2-D function** $I(x,y)$: digitized via **sampling** (grid) + **quantization** (levels). Storage = M × N × b bits.
- **Quality**: spatial resolution (dpi), intensity resolution / bit depth ($L = 2^b$), dynamic range, contrast.

## Part 3 — Color Fundamentals & Basic Image Processing (67 pp)
- **Human color**: trichromacy (S/M/L cones), metamerism, color blindness; hue/saturation/brightness.
- **CIE**: color-matching functions, 1931 chromaticity diagram (horseshoe, gamut triangle, white point).
- **Color models + conversions**: RGB cube; XYZ (device-independent, Y=luminance); HSV/HSI/HSL; CMYK (subtractive, print); CIE Lab (perceptually uniform, ΔE); YCbCr (JPEG/MPEG).
- Comparison table of all spaces (strengths, weaknesses, use cases).

## Part 4 — Image Enhancement: Intensity Transformations (45 pp)
- **Point operations**: $g(x,y)=T[f(x,y)]$ → $s=T(r)$.
- **Basic transforms**: negative $s=L-1-r$; gain/bias $s=\alpha r+\beta$; log $s=c\cdot\log(1+r)$; **power-law/gamma** $s=c\cdot r^{\gamma}$ ($\gamma<1$ brightens, $\gamma>1$ darkens).
- **Gamma correction**: pre-distort by $1/\gamma_{\text{display}}$ ($\approx 2.2$) for perceptual linearity.
- **Contrast stretching** (piecewise-linear) & thresholding.
- **Histogram processing**: histogram $h(r_k)=n_k$; **histogram equalization** $s_k=(L-1)\cdot\text{CDF}(r_k)$; **Otsu** automatic threshold (minimize within-class variance).

## Part 5 — Spatial Filtering (49 pp)
- **Neighborhood operator**: $g(x,y)=\sum w(s,t)\cdot f(x+s,y+t)$; linear + shift-invariant ⇒ convolution.
- **Correlation vs convolution**: convolution flips the kernel 180°.
- **Smoothing**: box average vs **Gaussian** ($\sigma$ controls blur, fewer artifacts).
- **Sharpening**: unsharp masking; **Laplacian** $\nabla^2 f$ (isotropic 2nd derivative); $g=f+c\cdot\nabla^2 f$.
- **Non-linear**: median (kills salt-and-pepper), max/min, alpha-trimmed, **bilateral** (edge-preserving).
- **Template matching**: SSD ↔ correlation; **normalized correlation** for brightness invariance.

## Part 6 — Image Filtering in the Frequency Domain (83 pp)
- **Fourier basics**: sinusoid (amplitude/frequency/phase), Euler, 1-D FT pair.
- **Convolution theorem**: $f \ast h \leftrightarrow F \times H$ (and the dual). Pipeline: FT → multiply by $H(u,v)$ → inverse FT.
- **2-D DFT** for images; DC term $F[0,0]$; log-magnitude spectrum display; orientation ⟂ rule.
- **Filter families**: low-pass **ILPF / BLPF(order n) / GLPF**; high-pass IHPF/BHPF/GHPF — transfer functions + radial profiles.
- **Ringing / Gibbs**: ideal filters ring; Butterworth/Gaussian don't.
- **Phase vs amplitude**: phase encodes *where* structures are (phase alone reconstructs recognizable images).

## Part 7 — Image Features (45 pp)
- **Feature** = local, meaningful, detectable image part. Good features are **repeatable, invariant** (viewpoint/lighting/deformation/occlusion), **distinctive**.
- **Edges**: 4 causes (depth, orientation, reflectance, color discontinuity); 3 profiles (step, ramp, roof).
- **Gradient detection**: $\nabla I$, magnitude $\|\nabla I\|$, direction $\arctan(I_y/I_x)$; Sobel/Prewitt/Roberts; thresholding (single vs hysteresis).
- **Laplacian detection**: $\nabla^2 I$, edges = zero-crossings (location only, no orientation).
- **Noise**: smooth before differentiating; Derivative-of-Gaussian $\nabla(G_{\sigma}\ast f)=(\nabla G_{\sigma})\ast f$; **LoG** (Mexican hat).

## Part 8 — Edge Detection (52 pp)
- **Gradient operators**: Roberts (2×2), Prewitt (3×3 uniform), Sobel (3×3 center-weighted) — kernels + magnitude/direction.
- **LoG / Marr-Hildreth**: $\nabla^2[G_{\sigma}\ast I]$, zero-crossings.
- **Canny — 3 optimality criteria**: low error, good localization, single response.
- **Canny — 4 stages**: Gaussian smooth → Sobel gradient → **non-maximum suppression** → **double-threshold hysteresis** ($T_{\text{low}} < T_{\text{high}}$).
- **Evaluation**: Precision/Recall/F1, Pratt's FOM; BSDS500 dataset.
- **Line/curve fitting**: least-squares (vertical vs perpendicular/PCA); polynomial via pseudo-inverse $a=(X^{\top}X)^{-1}X^{\top}y$.

## Part 9 — The Hough Transform (& RANSAC) (50 pp)
- **History**: Hough 1962; Duda & Hart 1972.
- **Line detection**: slope-intercept $(m,c)$ (m unbounded — problematic) vs **polar** $\rho = x\cos\theta + y\sin\theta$ (bounded). **Point ↔ curve duality**; an **accumulator** array votes.
- **Circle detection**: $(x-a)^2+(y-b)^2=r^2$; 2-D accumulator $A(a,b)$ (r known) or 3-D $A(a,b,r)$ (r unknown).
- **RANSAC** (Fischler & Bolles 1981): sample → fit → count inliers → repeat $N$ times → best model; $N = \log(1-p)/\log(1-(1-e)^s)$; tolerates ~50% outliers. Used for homography/pose.

## Part 10 — Corner Detection (Harris) (28 pp)
- **Corner** = intensity changes in **two** directions (flat=0, edge=1, corner=2).
- **Structure tensor M** (2×2 weighted sum of gradient outer products); eigenvalues $\lambda_1,\lambda_2$ = change strength in principal directions.
- **Harris response**: $R = \det(M) - k\cdot\text{tr}(M)^2$ (k≈0.04-0.06); R>0 corner, R<0 edge, |R|≈0 flat (avoids eigendecomposition).
- **Algorithm**: gradients → build M → R → threshold → non-max suppression.
- **Invariance**: translation + rotation (full); scale + illumination (partial). Variants: **Shi-Tomasi** ($\min(\lambda_1,\lambda_2)>T$), Förstner.

## Part 11 — SIFT: Introduction & Detector (50 pp)
- **Motivation** (Lowe 2004): recognition under scale/rotation/illumination/occlusion via local keypoints. 5 advantages: locality, distinctiveness, quantity, efficiency, extensibility.
- **Scale space**: $L(x,y,\sigma)=G(x,y,\sigma)\ast I$; **Difference-of-Gaussians** $\text{DoG} \approx (s-1)\sigma^2\nabla^2 G$ approximates scale-normalized LoG.
- **Gaussian/DoG pyramid**: within-octave blur ($\sigma$) + octaves (halved resolution).
- **Extrema detection**: compare each pixel to its **26 neighbors** (3×3×3) in DoG; reject low contrast + edge responses (Hessian curvature ratio, r=10).
- **Orientation assignment**: 36-bin gradient histogram → dominant orientation (80% rule spawns extra keypoints) → rotation invariance.

## Part 12 — The SIFT Descriptor & Matching (35 pp)
- **128-D descriptor**: 16×16 window → **4×4 grid** of cells → **8-bin** orientation histogram each → 4×4×8 = **128**.
- **Invariance**: window rotated to keypoint orientation; bins relative to dominant orientation; window scales with $\sigma$.
- **Normalization**: L2-normalize → clip components >0.2 → L2-normalize (illumination invariance).
- **Matching**: Euclidean distance + **Lowe's ratio test** ($d_1/d_2 < 0.8$); NN search (linear or indexed).
- **Applications**: stitching, stabilization, recognition, CBIR (Bag-of-Words/VLAD), forgery detection. Also: **LBP** as a simpler texture descriptor.

## Part 13 — Motion Analysis (86 pp; guest lecture)
- **Optical flow**: brightness-constancy → **OFCE** $I_x u + I_y v + I_t = 0$; the **aperture problem** (one equation, two unknowns).
- **Classical OF**: **Lucas-Kanade** (local least-squares, structure tensor) vs **Horn-Schunck** (global energy + smoothness $\alpha$); pyramidal for large motion.
- **Deep OF**: FlowNet, RAFT (4-D correlation volumes, GRU refinement).
- **Tracking** taxonomy: model-free/based, online/offline, single/multi-object; appearance model + motion model.
- **Appearance models**: NCC template, color-histogram MeanShift, PCA (IVT), correlation filters (MOSSE, KCF), deep (MDNet, SiamFC).
- **Multi-object**: tracking-by-detection + Hungarian matching + Kalman filter + Re-ID.

## Part 14 — Deep Learning for Computer Vision (46 pp)
- **Gestalt theory**: grouping principles (proximity, similarity, common fate/region, symmetry) → motivate **segmentation** (divide image into meaningful regions).
- **Segmentation as clustering**: pixels as `[R,G,B,x,y]` feature vectors; **K-Means** (needs K, init-sensitive) vs **Mean Shift** (KDE mode-seeking, bandwidth W, no K).
- **CNN building blocks**: convolution + receptive field, pooling (max/avg), upsampling / transposed convolution.
- **U-Net** (encoder-decoder + **skip connections**): contracting path (↓spatial, ↑channels) → bottleneck → expanding path (↑spatial, ↓channels); skips concatenate encoder maps to recover detail.
- **Training**: pixel-wise softmax + cross-entropy; heavy elastic-deformation augmentation for tiny datasets.

## Part 15 — DL for CV: Object Localization & Detection (27 pp)
- **Task hierarchy**: image classification (label only) → classification-with-localization (label + 1 bounding box) → object **detection** (label + bbox for *multiple* objects).
- **Output vector** for classification+localization: $y = [P_c, b_x, b_y, b_w, b_h, c_1, c_2, c_3]^\top$ with **conditional multi-task loss** masking regression + class terms when $P_c = 0$.
- **Sliding-window + FC→Conv trick**: replace fully-connected layers with conv equivalents → single forward pass computes all window positions simultaneously (e.g., $16\times 16\times 3$ input → $2\times 2\times 4$ output grid).
- **IoU** (Intersection over Union): evaluation threshold (IoU > 0.5 = good); also used for anchor-to-GT matching in training.
- **Non-Maximum Suppression (NMS)**: discard $p < 0.6$ → keep highest-confidence box → discard overlapping boxes with IoU > 0.5 → repeat per class.
- **Anchor boxes**: one-object-per-cell limitation → output tensor $S \times S \times A \times (5+C)$; YOLO v2+ k-means anchor design with distance $d = 1 - \text{IoU}$.
- **Two-stage (R-CNN family)** vs **single-stage (YOLO/SSD)**: RPN proposes regions → classify (slower, accurate) vs grid + direct prediction (faster, less accurate).

## Part 16 — Optimization of DNN Models for Embedded CV (19 pp)
- **Cloud vs edge deployment trade-offs**: latency, bandwidth, **privacy**, energy, connectivity. Offline-optimize-then-flash vs online cloud inference with comms link.
- **Quantization**: affine mapping $q = \mathrm{round}(r / \text{scale}) + \text{zero\_point}$; **PTQ** (post-training, quick) vs **QAT** (quantization-aware training, more accurate). FP32→INT8 = **4× memory saving**. Domains: Edge AI, mobile, automotive, IoT/MCU.
- **Pruning — unstructured**: zero out individual weights. Needs special inference engine (SparseDNN/EIE) to actually accelerate.
- **Pruning — structured (filter pruning)**: remove whole filters/channels → smaller dense model → runs anywhere with no hardware modification.
- **Filter selection criteria**: L1-norm/SFP, geometric median (**FPGM**), **HRank**, learning-based binary masks.
- **Lecturer's own methods**: **CCFP** (one-shot Correlation Circle Filter Pruning) and **MCFP** (iterative = CCFP + SFP + FPGM); CIFAR-10/ResNet56 + ImageNet/ResNet50 results.
- **Knowledge distillation** and **low-rank factorization** as complementary compression techniques.

---

## Cross-cutting themes (likely exam targets)
- **The gradient $\nabla I$** everywhere: edges (P7/P8), corners/structure tensor (P10), SIFT (P11).
- **Convolution**: spatial filtering (P5) ↔ frequency multiplication (P6, convolution theorem).
- **Scale & invariance**: characteristic scale, DoG pyramid, descriptor design (P11/P12); what each detector is/ isn't invariant to.
- **Voting & robust fitting**: Hough accumulator vs RANSAC (P9) — when to use which.
- **Classic vs deep**: hand-crafted features (SIFT, Harris) vs learned (CNN/U-Net, FlowNet) — trade-offs.
- **Filter design**: smoothing vs sharpening; ideal vs Butterworth/Gaussian (ringing).
- **DL task ladder**: classification → localization → detection → segmentation (semantic, instance).
- **Deployment**: accuracy ↔ size ↔ latency ↔ energy ↔ memory — pruning / quantization / distillation as the levers.

---

## Study plan from here
1. ✅ **Phase 0 — Syllabus map** (this file)
2. **Phase 1 — Per part**: read each `0X_*.md` (bird's eye for a quick pass; detailed notes for depth)
3. **Phase 2 — Algorithms by hand**: be able to *write out* Canny, Harris, Hough voting, the SIFT pipeline, Lucas-Kanade, histogram equalization
4. **Phase 3 — Cross-links**: revise the recurring themes above (gradient, convolution↔frequency, invariance, classic vs deep)
