# Part 1 — Introduction to Computer Vision

## Bird's eye view

- **Computer vision** is the enterprise of building machines that can see — formally: a set of computational techniques aiming at estimating or making explicit the **geometric and dynamic properties of the 3D world from digital images**.
- Vision is the most powerful human sense; it operates **without physical contact** and is processed hierarchically by the eye-brain system.
- The field spans three abstraction levels: **low** (image in → image out), **medium** (image in → features/descriptors out), **high** (image in → concepts/understanding out).
- Vision is fundamentally **hard** because it is an **inverse problem**: a 2D image is an information-lossy projection of a 3D world, so many 3D scenes can produce the same image — the solution is inherently underdetermined.
- **Optical illusions** are not flaws but evidence of the brain's **intelligent inference** mechanisms (context, priors, size constancy, color constancy, perceptual completion); studying them informs better CV system design.
- CV is contrasted with **computer graphics**: graphics goes 3D → 2D (information loss is one-way); CV goes 2D → 3D (requires models to compensate for that loss).
- The field has evolved from 1970s line-extraction and 3D-structure inference, through 1980s variational/MRF methods, 1990s multi-view geometry, 2000s SIFT/learning, to **2012-present: deep learning dominance**.

---

## Detailed notes

### 1. What is computer vision?

**Two complementary definitions used in the course:**

1. *"The enterprise of building machines that can see."* (broad)
2. *"A set of computational techniques aiming at estimating or making explicit the geometric and dynamic properties of the 3D world from digital images."* (working definition for the course — practical and precise)

The course prioritizes a **practical approach** while introducing fundamental principles and concepts.

**Nature of the field:**
- Computer vision is an **intellectual frontier**: exciting and disorganized.
- Many useful ideas in CV have **no theoretical grounding**; some well-grounded theories are **useless in practice**.
- This makes it hard to give a single concise definition — the area spans many different problems.

**The core difficulty — the inverse problem:**
- A human perceives the 3D structure of the world with apparent ease, but this masks enormous underlying complexity.
- CV is difficult because it requires **solving an inverse problem**: infer real-world properties (objects, depth, lighting) from a 2D image that contains limited and ambiguous information.
- A 2D image cannot uniquely determine the exact 3D world that produced it without additional assumptions or models.
- Diagram description (slide 28): A 3D object (a box-like shape) is shown being projected onto three separate 2D planes — front view, side view, top view — each projection plane captures a different 2D silhouette. The diagram illustrates how projecting 3D to 2D loses information, and how CV must reverse this process with models.

**CV vs. Computer Graphics (slide 29):**
- Computer graphics: 3D scene → 2D raster image (the forward problem; 3D to 2D implies information loss).
- Computer vision: 2D raster image → 3D understanding (the inverse problem; requires models — e.g., recognizing a house, polygons, lines, edges from a flat image).
- These are inverse processes of each other.

---

### 2. Three levels of computer vision

The pipeline from raw pixel data to semantic understanding is organized into three abstraction levels:

| Level | What it does | Input → Output | Examples |
|---|---|---|---|
| **Low (Image processing)** | Pixel-level operations | image → image | Image restoration, contrast enhancement, noise reduction |
| **Medium** | Extract structure from pixels | image → features/attributes/descriptors | Image segmentation, shape recognition |
| **High** | Semantic interpretation | image → concepts | Scene understanding, object recognition |

**The "semantic gap" problem:** From a signal point of view, an image is just variations in color and brightness (a grid of numbers). A machine sees an iris flower as a matrix of pixel intensity values (e.g., 0, 3, 2, 5, 4, 7, ...). A human immediately and effortlessly distinguishes flower from background, foreground object from leaves. Bridging this gap — from pixels to meaning — is the core challenge of computer vision.

---

### 3. Why vision is so difficult for machines

**The human side:**
- Humans trivially interpret subtle translucency and shading variations, segment a red flower from a complex background, count and classify people in a crowd, recognize known individuals, estimate ages — all effortlessly.
- Perceptual psychologists have spent decades studying how the human visual system achieves this, including through the study of optical illusions.

**The machine side:**
- What we see as a purple iris flower is, for a machine, a 9×10 grid of numbers (example from slides). Tasks that feel trivial to a human — like distinguishing the flower from the green-leaf background — are computationally very hard.
- The gap between the rich semantic interpretation we perform and the raw numeric signal is enormous.

---

### 4. Human vision and the brain — optical illusions

Perceptual psychologists use optical illusions as probes to understand how the brain processes visual information. These reveal the assumptions and strategies built into human vision.

#### 4.1 Named illusions (all from course slides)

**Example 1 — Motion Aftereffect (Waterfall Illusion):**
- After staring at motion in one direction, stationary objects appear to move in the opposite direction.
- Reveals that motion-sensitive neurons adapt and produce opponent signals.

**Example 2 — Muller-Lyer Illusion (1889):**
- Two horizontal lines of identical physical length appear different when one has inward-pointing angle brackets (< >) and the other has outward-pointing brackets (> <).
- The "angles-in" configuration looks shorter.
- Explanation: The brain applies **size constancy**. It interprets the "angles-in" configuration as being farther away (like an inside corner of a room) and the "angles-out" as closer (like an outside corner). Given identical retinal image sizes, it concludes the "far" object must be shorter.
- Diagram description: Two horizontal lines of equal length; top one ends with inward-pointing arrowheads (< — >), bottom one with outward-pointing arrowheads (> — <). Blue dashed vertical lines confirm equal actual lengths. A room-corner illustration shows the real-world analogy.

**Example 3 — Adelson's Checker-Shadow Illusion:**
- On a checkerboard with a cylinder casting a shadow, square A (in the shadow region, a nominally "white" square) and square B (in full light, a nominally "dark" square) appear different in shade — A looks lighter than B. In physical reality they are the **same shade of gray**.
- Explanation: The visual system cannot simply measure luminance; a shadow dims surfaces so a white square in shadow may reflect less light than a black square in full light. The brain uses contextual cues to determine where shadows fall and **compensates for them** — inferring true surface color from context. This is **color constancy** in action.
- Diagram description: A checkerboard in perspective with a green cylinder. Square A and B are labeled and appear strikingly different. When context is removed (circles placed around them), both circles appear identical in gray level.

**Example 4 — The Pop-Out Effect:**
- When one element in a grid differs from all others (e.g., one "U" rotated differently), it immediately and automatically grabs attention — no effortful search needed.
- The effect becomes harder as the distractor similarity to the target increases (shown by four grids of bracket-shapes with varying target-distractor similarity).
- Explanation: The brain is equipped with **specialized neurons, parallel processing**, and attentional mechanisms that prioritize unique or salient visual features. The visual cortex detects these differences pre-attentively.

#### 4.2 What optical illusions teach about vision

**Why illusions matter for CV:**
- They reveal how perception differs from physical reality.
- They expose the **assumptions** used by the human visual system.
- They demonstrate that **vision is an active inference process**, not passive recording.
- They highlight the **limits and biases** of perception.

**Lessons about perception:**
- The brain **fills in missing information** (perceptual completion).
- **Context strongly influences interpretation**.
- Perception is **predictive** (uses priors) rather than purely reactive.
- **Speed and efficiency are prioritized over accuracy**.

**Core visual principles revealed:**
- **Perceptual completion**: seeing shapes not physically present.
- **Color constancy**: automatically correcting for illumination.
- **Depth inference**: interpreting 2D patterns as 3D structures.
- **Selective attention**: not all visual information is processed equally.

**Key takeaway:** Optical illusions are not flaws — they are evidence of intelligent inference. Both human and artificial vision rely on assumptions. Studying illusions helps us build better CV systems.

**Parallels with artificial vision systems:**
- Human illusions ≈ adversarial examples in neural networks (both exploit model assumptions).
- Perceptual biases ≈ dataset and model biases.
- Attention mechanisms exist in both biological and artificial systems.
- Hierarchical processing in the visual cortex resembles deep neural network architectures.

---

### 5. Applications of computer vision

CV is deployed across a wide range of real-world domains:

| Application | Description |
|---|---|
| **OCR** | Reading printed/handwritten text; license plate recognition (slide 31 shows plate "LP53569" detected, segmented character by character, and recognized) |
| **Video OCR** | Detecting and recognizing text overlaid on video frames (e.g., Arabic text in TV broadcasts — ALIF dataset, research by Prof. Berrani) |
| **Machine inspection** | Automated quality control on production lines (e.g., camera above conveyor belt sorting objects; camera inspecting bottle caps) |
| **Surveillance and tracking** | Detecting and tracking people/vehicles in video (slide 34: colored bounding boxes around each detected person in a station; red/yellow/green boxes around vehicles in traffic video) |
| **Biometrics** | Face recognition, iris recognition, fingerprint recognition |
| **Soft biometrics** | Gender detection and apparent age estimation from face images (pipeline: face detection → landmark detection → face alignment → prediction). Prof. Berrani's team won Track 1 of the ChaLearn LAP challenge at CVPR 2016 for apparent age estimation |
| **Face reenactment** | Capturing facial expressions from one actor and applying them to another in real time |
| **Medical imaging** | Analysis of CT scans, MRI, 3D reconstructions (slide 38: 3D vascular reconstruction of skull). Covid-19 diagnosis from CT images using CNNs (ResNet, DenseNet, EfficientNet) — research by Prof. Berrani's group (Eurasip Journal IVP 2024) |
| **Heritage preservation** | Surface damage identification for heritage sites (e.g., Kasbah of Algiers — detecting efflorescence, cracks, mold from photos) |
| **Self-driving cars** | Scene understanding, object detection, lane following (slide 41: Tesla interior with hands-off driving) |
| **Video fingerprinting / copy detection** | Identifying copies of video content even after transformations (watermarks, noise, format change). Pipeline: query flux → compare against reference video database → identify copies |
| **Video macro-segmentation** | Structuring a TV broadcast stream into programs, segments, and repeated sequences (slide 45: a flux divided into repetitions, classification, segmentation, program extraction — e.g., "Journal 20h" vs. "James Bond film") |
| **Consumer applications** | Image stitching (panoramas), image enhancement (deblurring), morphing, face-based authentication |

**Image stitching example (slide 47):** Three overlapping photos of the same brick building are aligned using matched feature points (red and blue dots), then blended into a single wide panoramic image. This requires detecting and matching corresponding points across images.

---

### 6. A brief history of computer vision

The field has evolved through identifiable phases, tracked on a timeline from 1970 to present:

| Era | Key developments |
|---|---|
| **1970s** | Distinguished from image processing by early attempts to infer 3D structure from 2D images: line extraction and labeling, stereo correspondence, optical flow, structure from motion, blocks-world interpretation, generalized cylinders |
| **1980s** | More sophisticated mathematical techniques: variational optimization, Markov Random Fields (MRFs), image pyramids, shape from shading, 3D scanning, scale-space processing |
| **1990s** | Projective invariants, multiview stereo, image segmentation, physics-based vision, factorization methods, graph cuts, face recognition and detection, particle filtering |
| **2000–2010** | Continuous advances across all prior topics + SIFT features, texture synthesis, computational photography, machine learning integration, MRF inference, category recognition |
| **2012–present** | **Deep learning** dominates — convolutional neural networks achieve superhuman performance on many benchmarks; learning-based approaches replace most hand-crafted pipelines |

The timeline diagram in slides 48–52 shows all these topics along a horizontal axis from 1970 to 2000+, with each era's highlights annotated. The key inflection point is 2012 (AlexNet / ImageNet), when deep learning transformed the field.

---

## Key terms (glossary)

| Term | Definition |
|---|---|
| **Computer vision (working definition)** | A set of computational techniques aiming at estimating or making explicit the geometric and dynamic properties of the 3D world from digital images |
| **Inverse problem** | Recovering unknowns (3D scene properties) from insufficient information (2D image), where the solution is underdetermined without additional assumptions |
| **Low-level vision** | Pixel-to-pixel processing: image in, image out (restoration, filtering, noise reduction) |
| **Medium-level vision** | Image to features/descriptors: segmentation, shape recognition |
| **High-level vision** | Image to concepts: scene understanding, semantic recognition |
| **Semantic gap** | The difference between raw pixel values (what a machine sees) and the semantic meaning (what a human understands) |
| **Optical illusion** | A perceptual phenomenon where the brain's interpretation of an image differs from physical reality; reveals the inference mechanisms of human vision |
| **Perceptual completion** | The brain infers and fills in shapes or patterns not physically present in the stimulus |
| **Color constancy** | The brain's ability to perceive the true color of an object regardless of the illumination conditions |
| **Size constancy** | The brain's ability to perceive the true size of an object regardless of its retinal image size, by compensating for estimated distance |
| **Pop-out effect** | The automatic, pre-attentive detection of a uniquely different element in a visual scene, driven by specialized parallel processing in the visual cortex |
| **Computer graphics** | The inverse of CV: 3D scene → 2D image (forward problem, information loss unavoidable) |
| **CV vs. CG** | CV: 2D → 3D (needs models); CG: 3D → 2D (projection, information loss) |
| **OCR** | Optical Character Recognition — converting images of text into machine-readable text |
| **Biometrics** | Identification of individuals using physical/behavioral characteristics (face, iris, fingerprints) |
| **Soft biometrics** | Estimation of attributes like gender, age, ethnicity from biometric data |
| **Video fingerprinting** | Detecting copies of video content in large databases, even after transformations |
| **Adversarial examples** | Inputs designed to fool neural networks, analogous to optical illusions for the human visual system |

---

## Exam targets

1. **State and explain the two definitions of computer vision** used in this course. Be able to quote the precise working definition word for word: "A set of computational techniques aiming at estimating or making explicit the geometric and dynamic properties of the 3D world from digital images."

2. **Explain why computer vision is an inverse problem.** Describe the 3D → 2D projection (information loss) and why reversing it requires additional assumptions or models. Use the CV vs. Computer Graphics diagram to illustrate (CG: 3D → 2D forward direction; CV: 2D → 3D reverse direction requiring models).

3. **Define and distinguish the three levels of vision** (low/medium/high) with their input-output types and concrete examples for each.

4. **Describe three optical illusions from the course**, explaining:
   - What is observed (the perceptual effect)
   - The underlying cognitive/neural mechanism
   - What it reveals about the brain's visual inference strategies
   - The parallel with computer vision systems (e.g., Adelson → color constancy algorithms; Muller-Lyer → size constancy / depth models; pop-out → saliency and attention models)

5. **Explain what optical illusions reveal about the nature of perception**: active inference, context-dependence, priors, speed-accuracy tradeoffs, completion, and constancies.

6. **Draw the CV levels pipeline**: image → [Low] → image → [Medium] → features/descriptors → [High] → concepts.

7. **List at least six real-world applications of CV** and for each state what CV problem it fundamentally solves (e.g., OCR → character classification; surveillance → detection and tracking; medical imaging → classification/segmentation).

8. **Trace the history of CV** through its major eras: 1970s (3D structure inference), 1980s (variational/MRF), 1990s (multiview, segmentation), 2000s (SIFT/learning), 2012+ (deep learning).

9. **Explain the semantic gap**: from the machine's perspective (a matrix of numbers) to the human's perspective (a flower, a person, a scene). Why does this gap make CV hard?

---

## Pitfalls

- **"Computer vision = image processing."** Wrong. Image processing is low-level (image in → image out). CV aims at *understanding* the 3D world — it spans all three levels. Image processing is a tool within CV, not the same thing.
- **"Optical illusions prove the brain is defective."** Wrong. The course explicitly states (citing Michael Bach) that illusions are "not tricks" nor evidence the brain "sucks" — they reveal **intelligent inference strategies** that normally work well but fail in artificial conditions. The exact parallel for machines: adversarial examples.
- **"A 2D image uniquely determines the 3D scene."** Wrong. This is the core error about the inverse problem. Multiple 3D configurations can produce the same 2D image — the problem is fundamentally underdetermined.
- **Confusing CV and computer graphics direction.** CG goes 3D → 2D (easy, forward). CV goes 2D → 3D (hard, inverse). Do not mix these up.
- **Thinking "low-level" means "less important."** Low-level processing is the foundation for all higher-level vision. Without correct pixel-level operations, higher-level results are unreliable.
- **The pop-out effect is only about visual complexity.** Actually it is about *feature saliency*, not number of elements. A target that differs from distractors by a simple feature (e.g., orientation) pops out regardless of display size; a target requiring conjunction of features does not pop out — this is the critical distinction.
- **Muller-Lyer explanation:** Students often say "the brain is tricked by the arrows." The correct explanation involves size constancy applied to a depth interpretation: the brain uses the angle configuration as a depth cue and adjusts perceived size accordingly, resulting in a length illusion for identical-retinal-size lines.
