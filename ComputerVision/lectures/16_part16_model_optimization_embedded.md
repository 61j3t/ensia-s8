# Part 16 — Optimization of Deep Neural Network Models for Embedded CV Applications

> Lecturer: Sid-Ahmed Berrani. ENSIA Computer Vision, 2025-2026.

---

## Bird's eye view

- The central deployment question is **where** to run inference: on the **cloud** (unlimited compute, high bandwidth cost, latency, privacy risk) or on **embedded/edge devices** (constrained compute and memory, low latency, offline, private). Neural network optimization bridges the gap by shrinking models before deploying them to the edge.
- **Quantization** keeps the network architecture identical but reduces numerical precision (FP32 → INT8/FP16), yielding up to 4x smaller weights and lower memory bandwidth with minimal accuracy loss. Two workflows: **Post-Training Quantization (PTQ)** (fast, no retraining) and **Quantization-Aware Training (QAT)** (better accuracy for sensitive models).
- **Pruning** removes redundant weights (unstructured) or entire filters/neurons (structured). Unstructured pruning achieves higher sparsity but requires specialized sparse-execution hardware/software; structured pruning produces a smaller dense model that runs on any hardware.
- **Criteria-based pruning** scores each filter by importance (L1 norm, rank, geometric median) and removes the lowest-scoring ones; **learning-based pruning** trains a binary mask jointly with the model to decide which filters to keep.
- The lecturer's own research introduces **Correlation Circle Filter Pruning (CCFP)** — a one-shot structured pruning method using filter-correlation geometry — and **Multi-Criteria Filter Pruning (MCFP)** which combines CCFP with magnitude/geometric-median criteria iteratively; on ResNet50/ImageNet MCFP achieves ~55% FLOPs reduction with only 0.56% top-1 drop.
- **Key trade-off axis**: one-shot pruning (prune once, cheap) vs. iterative pruning (prune-retrain cycles, more accurate at high compression ratios). The iterative MCFP curve dominates the one-shot CCFP point on the accuracy-vs-FLOPs-reduction graph.
- Evaluation metrics across all techniques: **Top-1/Top-5 accuracy drop**, **FLOPs reduction (%)**, **parameter reduction (%)**, **inference latency**, and **memory footprint**.

---

## Detailed notes

### 1. Deployment decision: cloud vs. embedded devices

#### 1.1 The two deployment scenarios

The slide (slide 3) shows two architectures side by side:

**Scenario A — Offline / edge deployment:**
- A full-size model is trained on the cloud, then **compressed offline** (neural network optimization phase involving a speed gauge, a smaller network, and human review).
- The compressed model is then flashed to edge devices (smartphone, robotic arm, drone, autonomous car) and runs entirely locally.
- No communication channel needed during inference.

**Scenario B — Online / cloud inference:**
- The model stays on the cloud. During inference, the edge device sends data over a **communication link** (bidirectional arrow through a cloud symbol) and receives predictions back.
- Requires reliable, low-latency network connectivity.

#### 1.2 Trade-off table

| Criterion | Cloud inference | Embedded / edge inference |
|---|---|---|
| Latency | High (network round-trip) | Low (local) |
| Bandwidth | High (send raw data each frame) | None (data stays on device) |
| Privacy | Risk (data leaves device) | High (data never leaves) |
| Energy | High (network + server) | Lower (local chip) |
| Compute available | Essentially unlimited | Severely constrained |
| Model size constraint | None | Tight (MB to KB range) |
| Connectivity required | Yes | No |
| Typical use cases | Batch analytics, training | Autonomous vehicles, IoT, drones, phones |

**Conclusion:** When latency, privacy, energy, or connectivity are critical, the model must be compressed and deployed on the embedded device. Neural network optimization is what makes this feasible.

---

### 2. Quantization

#### 2.1 Principle

Quantization **preserves the network architecture** (same layers, same connectivity) but **reduces the number of bits** used to represent weights and activations. Lower precision means:
- Smaller model file (4x smaller: FP32 → INT8 uses 32/8 = 4x fewer bytes per value).
- Lower memory bandwidth during inference.
- Lower inference latency on hardware with native INT8/FP16 kernels.
- Lower energy per prediction.

The slide shows the pipeline: **FP32 model → Quantize → INT8/FP16 model**, with a gauge annotation showing **4x smaller weights** when going from FP32 to INT8.

*FP32 = 32-bit Floating Point (standard training precision).*

#### 2.2 How quantization maps real numbers to integers

A floating-point value is approximated using an integer representation plus two calibration parameters:
- **scale**: a floating-point multiplier that maps the integer range to the real range.
- **zero_point**: an integer offset so that real zero maps exactly to an integer.

The mapping formulas:

```math
\text{real} \approx \text{scale} \times (\text{integer} - \text{zero\_point})
```

```math
\text{integer} = \text{round}\!\left(\frac{\text{real}}{\text{scale}} + \text{zero\_point}\right)
```

**Worked example (from slide):**
- Real value: $0.73$
- INT8 value: $93$
- Reconstructed real value: $\approx 0.729$
- Quantization error: $|0.73 - 0.729| = 0.001$ (small; the price paid for efficiency)

**Practical notes:**
- **Weights** are quantized **once** (offline) and stored in compressed form.
- **Activations** vary per input; they are quantized **at inference time** using range statistics collected during a **calibration** step (run representative inputs through the FP32 model, observe activation ranges, compute scale/zero_point).
- The approximation error is the price paid for compression. Neural networks are generally robust to this error because many weights contribute to each output.

#### 2.3 Two quantization workflows

**Post-Training Quantization (PTQ):**
```
Train FP32 → Calibrate (collect activation ranges) → Export INT8
```
- Fast conversion, no full retraining required.
- Best when the accuracy drop is acceptable (typically < 1–2% for most CNNs).
- Choose PTQ when you have a tight deployment deadline or limited training data.

**Quantization-Aware Training (QAT):**
```
Train with fake quantization → Fine-tune → Export INT8
```
- "Fake quantization": during forward passes, simulate INT8 rounding while keeping FP32 for gradient updates (straight-through estimator).
- More work, but usually better accuracy for accuracy-sensitive models or aggressive quantization levels.
- Choose QAT when PTQ causes unacceptable accuracy loss, particularly for small models or binary networks.

#### 2.4 Applications and hardware considerations

Quantization matters most when compute, memory, or energy are constrained:

| Domain | Hardware | Why quantization helps |
|---|---|---|
| Edge AI | Cameras, drones, robots | Size and energy |
| Mobile | Phones, tablets, NPUs | Battery life, memory |
| Cloud inference | GPU clusters | Higher throughput per GPU (INT8 tensor cores) |
| Automotive | Embedded ECUs | Real-time perception, power budget |
| IoT / MCU | Microcontrollers | Tiny models under tight memory (< 1 MB) |

**Quantization improves simultaneously:** model size, memory bandwidth, inference latency, and energy per prediction.

**Key takeaway:** Quantization **preserves model structure** but **changes the precision of the math**.

Works best when the target hardware has optimized INT8/FP16 kernels (e.g., NVIDIA Tensor Cores, ARM Cortex-M with CMSIS-NN, Google Edge TPU, Apple Neural Engine).

---

### 3. Pruning

#### 3.1 Core idea

Pruning removes weights or entire structures from a trained network that contribute little to accuracy, producing a smaller and potentially faster model. After pruning, the model is typically **fine-tuned** (retrained briefly) to recover accuracy.

#### 3.2 Unstructured vs. structured pruning

The slide (slide 8) shows the contrast visually: a fully connected network on the left splits into two paths.

**Unstructured pruning (weight-level):**
- Removes individual weights (connections) wherever they are small or unimportant.
- The network diagram retains the same number of neurons but many connections are shown as dashed red lines (zeroed out).
- Result: a **sparse weight matrix** (many zeros), but the matrix shape is unchanged.

**Structured pruning (neuron/filter-level):**
- Removes entire neurons (in FC layers) or entire filters (in convolutional layers).
- The network diagram shows a smaller network with fewer nodes — actual shape changes.
- Result: a **smaller dense model**.

#### 3.3 Unstructured pruning — properties

| Property | Detail |
|---|---|
| Pruning rate achievable | Very high (can zero out 90%+ of weights) |
| Effect on model shape | None — same tensor dimensions |
| Speedup on standard hardware | None without specialized support |
| Specialized support needed | SparseDNN [1] (CPU sparse inference), Tiramisu [2] (polyhedral compiler), EIE [3] (Efficient Inference Engine hardware) |
| Key limitation | Weights are zeroed but **still exist in memory** — standard dense matrix multiplications must still be executed unless the hardware/software understands sparsity |

#### 3.4 Structured pruning — properties

| Property | Detail |
|---|---|
| Pruning rate achievable | Moderate (practical limit ~50–65% FLOPs in published results) |
| Effect on model shape | Reduces the number of filters/neurons — tensor dimensions change |
| Speedup on standard hardware | Yes — produces a smaller dense model that any hardware can run faster |
| Specialized support needed | None |
| Accuracy trade-off | Often less accuracy loss than unstructured pruning at the same compression level |

**Why structured pruning is preferred for deployment:** the pruned model is a standard smaller dense network. It runs on any CPU, GPU, or NPU without modification. No sparse library or custom hardware required.

---

### 4. Criteria-based pruning methods (structured, filter-level)

#### 4.1 Pipeline

The slide (slide 10) shows the process for a convolutional layer with three filter groups (Filter1, Filter2, Filter3):
1. Feed an input image through the layer.
2. **Compute an importance/similarity score** for each filter.
3. Rank filters by score (bar chart on the right: Filter2 has the highest score, Filter1 medium, Filter3 very low).
4. Remove the lowest-scoring filter(s) (Filter3 shown with dashed border = pruned).
5. Fine-tune the pruned network.

#### 4.2 Common importance criteria

- **L1-norm of filter weights:** $s_i = \sum_{j} |W_{ij}|$. Filters with small total magnitude are assumed to contribute less. Simple and parameter-free.
- **L2-norm (SFP — Soft Filter Pruning):** Similar to L1 but uses $\ell_2$; classic baseline.
- **Geometric Median (FPGM):** Filters closest to the geometric median of the filter set are considered redundant (they can be approximated by others). Removes the most replaceable filters.
- **Hrank:** Prunes filters whose output feature maps have low rank (measured by singular value decomposition of the activation maps). Low rank → redundant spatial patterns.
- **LFPC / DPMS / CIFP / FSCL:** various criteria from the literature combining magnitude, rank, or learned signals.

#### 4.3 One-shot vs. iterative schedules

- **One-shot pruning:** compute scores once, remove all filters at the target pruning rate in a single step, then fine-tune.
  - Fast (one pass).
  - Less accurate at high compression ratios (large accuracy drop).
- **Iterative pruning:** remove a small fraction of filters, fine-tune, recompute scores, remove more, fine-tune again — repeat.
  - Slower (multiple prune-finetune cycles).
  - Significantly better accuracy at high compression ratios.

The plot on slide 17 (Pruning ResNet50 on ImageNet) illustrates this clearly:
- X-axis: FLOPs reduction (%).
- Y-axis: Top-1 accuracy.
- Dashed horizontal line: baseline accuracy (76.15%).
- Green curve (Iterative MCFP): starts at ~76.6% accuracy with 24% FLOPs reduction and gracefully degrades to ~75.0% at 62% FLOPs reduction — always **above or near baseline**.
- Red dot (One-Shot CCFP): at 55% FLOPs reduction, accuracy drops to ~73.9% — significantly below the iterative curve and below baseline.
- **Conclusion: iterative pruning dominates one-shot at high compression ratios.**

---

### 5. Learning-based pruning methods

#### 5.1 Concept

Instead of hand-designing an importance score, learning-based methods **train a binary mask** jointly with the network. A set of learnable parameters (shown as a gear icon in slide 11) produces a binary mask vector (e.g. [1, 1, 0]) indicating which filters to keep (1) or prune (0).

**Process:**
1. Initialize the network with all filters active.
2. During training, the mask parameters are learned alongside the weights.
3. After training, apply the final binary mask to remove pruned filters.
4. Optional fine-tune.

#### 5.2 Advantages

1. **Adaptive pruning:** the decision of which filters to prune is data-driven, not heuristic.
2. **Pruning precision:** the model learns to preserve the most task-relevant features, since the mask is trained end-to-end with the task loss.
3. **Performance optimization:** can optimize specific metrics (accuracy, latency) by incorporating them into the training objective.

#### 5.3 Disadvantages

1. **Complexity and resource-intensive:** requires additional memory for mask parameters and higher computational cost during training.
2. **Incompatibility with pretrained models:** not easily applied to an already-trained model; the mask must be learned from the start (or requires expensive retraining).
3. **Longer training time:** extensive training and hyperparameter tuning needed.

---

### 6. CCFP and MCFP — the lecturer's research contributions

#### 6.1 Correlation Circle Filter Pruning (CCFP) — one-shot

**Key idea:** Measure the **pairwise correlation** between filters within each convolutional layer. Filters that are highly correlated with others are redundant — the network already has other filters computing similar feature detectors. These can be safely removed.

**Method:**
- Compute a filter correlation matrix (circle/graph of filter relationships).
- Identify redundant filter clusters using correlation geometry.
- Remove one-shot: prune all redundant filters in a single pass.
- Fine-tune to recover accuracy.

**Results on CIFAR-10 (ResNet56, one-shot, slide 14):**

| Method | Top-1 (%) | Top-1 drop (%) | FLOPs reduction (%) | Param reduction (%) |
|---|---|---|---|---|
| SFP | 93.59 | 0.24 | 52.6 | — |
| FPGM | 93.59 | 0.10 | 52.6 | — |
| Hrank | 93.46 | 0.09 | 50.0 | 42.4 |
| ResRep | 93.71 | 0.00 | 52.91 | — |
| CCFP (Ours) | 92.98 | 0.15 | 50.35 | 52.65 |
| CCFP (Ours) | 92.98 | 0.39 | 59.86 | 64.77 |

**Results on ImageNet (ResNet50, one-shot, slide 15):**

| Method | Top-1 (%) | Top-5 (%) | Top-1 drop (%) | FLOPs red. (%) | Param red. (%) |
|---|---|---|---|---|---|
| FPGM | 76.15 | 92.87 | 1.32 | 53.5 | — |
| ResRep | 76.15 | 92.87 | 0.18 | 56.11 | — |
| CCFP (Ours) | 76.13 | 92.86 | 1.90 | 54.41 | 36.38 |
| CCFP (Ours) | 76.13 | 92.86 | 2.30 | **64.11** | 45.4 |

CCFP achieves the highest FLOPs reduction (64.11%) on ImageNet at a moderate accuracy cost — competitive with one-shot baselines.

#### 6.2 Multi-Criteria Filter Pruning (MCFP) — iterative

**Formula:** $\text{MCFP} = \text{CCFP} + \text{SFP} + \text{FPGM}$ (combining correlation-circle criterion with magnitude-based SFP and geometric-median-based FPGM).

**Rationale:** different criteria capture different notions of redundancy. Combining them (multi-criteria) produces more robust filter rankings than any single criterion.

**Results on ImageNet (ResNet50, iterative, slide 18):**

| Method | Top-1 (%) | Top-1 drop (%) | FLOPs red. (%) | Param red. (%) |
|---|---|---|---|---|
| SFP | 76.15 | 14.01 | 41.8 | — |
| FPGM | 76.15 | 1.32 | 53.5 | — |
| CCFP (one-shot) | 76.13 | 1.90 | 54.41 | 36.38 |
| **MCFP (Ours)** | 76.13 | **0.56** | 55.21 | 48.79 |
| **MCFP (Ours)** | 76.13 | **1.16** | 62.04 | 54.81 |

MCFP achieves only 0.56% top-1 accuracy drop at 55% FLOPs reduction — the best accuracy-efficiency trade-off among all compared methods on ResNet50/ImageNet.

**Key insight from the comparison:**
- Iterative MCFP clearly dominates one-shot CCFP on the accuracy vs. FLOPs trade-off graph.
- One-shot is cheaper (one prune-then-finetune cycle) but pays a high accuracy penalty at high compression.
- Iterative is expensive (many cycles) but reaches much higher FLOPs reduction with much smaller accuracy drop.

#### 6.3 Published papers (lecturer's lab)

- Bellebna & Berrani, "Iterative multi-criteria filter pruning for efficient CNN deployment," *Journal of Supercomputing* 82(3):170 (2026).
- Yagoub, Mers, Bellebna, Berrani, "Rethinking Deep Neural Networks Pruning: A Cost-Driven Methodology," CSA 2026.
- Bellebna & Berrani, "Pruning Schedules Decide the Outcome: One-Shot vs. Iterative Filter Pruning of CNN Models," CSA 2026.

---

### 7. Metrics and benchmarks

When reporting and comparing compressed models, always report all of the following:

| Metric | What it measures | How computed |
|---|---|---|
| **Top-1 / Top-5 accuracy** | Task performance | Standard ImageNet / CIFAR evaluation |
| **Top-1 / Top-5 accuracy drop** | Accuracy cost of compression | Baseline accuracy minus pruned accuracy |
| **FLOPs reduction (%)** | Computational saving | $(1 - \text{FLOPs}_\text{pruned}/\text{FLOPs}_\text{baseline}) \times 100$ |
| **Parameter reduction (%)** | Memory/storage saving | $(1 - \text{Params}_\text{pruned}/\text{Params}_\text{baseline}) \times 100$ |
| **Inference latency** | Wall-clock speed on target hardware | Milliseconds per image on the deployed device |
| **Memory footprint** | RAM required during inference | Peak activation + weight memory |
| **Energy per prediction** | Battery/power impact | Joules per inference on embedded hardware |

**FLOPs vs. parameters vs. latency:** FLOPs reduction and parameter reduction are not the same. Parameter reduction measures storage; FLOPs reduction measures compute. Latency on real hardware depends on both plus memory access patterns and hardware-specific factors (cache, vectorization).

---

### 8. Broader context: other compression techniques (not detailed in slides, but exam-relevant background)

The slides focus on quantization and pruning. For completeness, two related techniques often appear in deployment contexts:

#### 8.1 Knowledge distillation (KD)

A large **teacher** network (high accuracy, high cost) trains a small **student** network. The student is trained to match:
- The teacher's **soft output probabilities** (class logits after softmax with temperature $T > 1$), which carry more information than one-hot labels.
- Optionally, intermediate feature maps (feature-level distillation).

Loss:

```math
\mathcal{L}_\text{KD} = (1 - \alpha) \mathcal{L}_\text{CE}(y, \hat{y}_s) + \alpha \cdot T^2 \cdot \mathcal{L}_\text{CE}\!\left(\sigma\!\left(\frac{z_t}{T}\right), \sigma\!\left(\frac{z_s}{T}\right)\right)
```

where $z_t$ and $z_s$ are teacher and student logits, $\sigma$ is softmax, $T$ is temperature, and $\alpha$ balances the two terms.

**When to use:** when designing a compact model from scratch for a target device; the student can be any efficient architecture (MobileNet, etc.).

#### 8.2 Low-rank factorization

A weight tensor (e.g. a convolutional filter of shape $C_\text{out} \times C_\text{in} \times k \times k$) is approximated as a product of two lower-rank tensors:

```math
W \approx U \cdot V^\top
```

This reduces the number of parameters from $C_\text{out} \times C_\text{in} \times k^2$ to $(C_\text{out} + C_\text{in}) \times r \times k^2$ where $r \ll \min(C_\text{out}, C_\text{in})$.

**When to use:** effective for fully-connected layers and large convolutional kernels; less common than pruning/quantization for CNNs.

#### 8.3 Efficient architecture design (MobileNet / SqueezeNet)

Rather than compressing a large network, design a small network from scratch using efficient building blocks:

**Depthwise Separable Convolution (MobileNet):**
- Replace a standard $k \times k$ convolution with two operations:
  1. **Depthwise convolution:** apply one $k \times k$ filter per input channel (no cross-channel mixing).
  2. **Pointwise convolution:** apply $1 \times 1$ convolution to mix channels.
- Computational reduction factor:

```math
\frac{\text{Depthwise Separable FLOPs}}{\text{Standard Conv FLOPs}} = \frac{1}{N} + \frac{1}{k^2}
```

where $N$ = number of output channels and $k$ = kernel size. For $k=3, N=256$: approximately 8–9x fewer operations.

**SqueezeNet:** uses "Fire modules" (squeeze layer of $1 \times 1$ filters followed by an expand layer of $1 \times 1$ and $3 \times 3$ filters in parallel) to achieve AlexNet-level accuracy at 50x fewer parameters.

---

### 9. Summary comparison table

| Technique | What changes | Hardware needed | Accuracy impact | Implementation complexity |
|---|---|---|---|---|
| Quantization (PTQ) | Precision of values | INT8/FP16 kernels on target | Small (< 1–2%) | Low |
| Quantization (QAT) | Precision + retraining | INT8/FP16 kernels | Very small | Medium |
| Unstructured pruning | Individual weights zeroed | Sparse software (SparseDNN, EIE) | Small–medium | Medium |
| Structured pruning (one-shot) | Filters removed | Standard | Medium | Low–medium |
| Structured pruning (iterative) | Filters removed | Standard | Small | High |
| Knowledge distillation | New student model trained | Standard | Depends on student arch | High |
| Low-rank factorization | Weight tensors decomposed | Standard | Small | Medium |
| Efficient architectures | Architecture redesigned | Standard | Depends on design | High (design effort) |

---

## Key terms (glossary)

- **Quantization** — representing neural network weights and activations with fewer bits (e.g., INT8 instead of FP32) to reduce model size and speed up inference.
- **FP32** — 32-bit floating-point; standard training precision (4 bytes per value).
- **INT8** — 8-bit integer; common quantization target (1 byte per value; 4x smaller than FP32).
- **Scale / zero_point** — calibration parameters that define the affine mapping between real-valued floats and integers in quantization: $\text{real} \approx \text{scale} \times (\text{integer} - \text{zero\_point})$.
- **PTQ (Post-Training Quantization)** — quantize a fully-trained FP32 model using a calibration dataset; no retraining. Fast but may lose more accuracy.
- **QAT (Quantization-Aware Training)** — simulate INT8 rounding during FP32 training (fake quantization); export INT8. More accurate for sensitive models.
- **Calibration** — passing a representative dataset through the FP32 model to collect activation range statistics (min/max) needed to set scale/zero_point for activations.
- **Pruning** — removing weights or structures from a neural network that contribute little to accuracy, to reduce model size and/or compute.
- **Unstructured pruning** — zeroing individual weights anywhere in the weight tensor; creates sparse matrices; requires sparse software/hardware to accelerate.
- **Structured pruning** — removing entire filters (in CNNs) or neurons (in FC layers); produces a smaller dense model that runs on any hardware.
- **Filter importance score** — a scalar criterion used to rank how essential each filter is; low score → candidate for pruning. Examples: L1-norm (SFP), geometric median (FPGM), rank of activation maps (Hrank).
- **One-shot pruning** — compute scores, prune all at once to target compression rate, fine-tune once.
- **Iterative pruning** — alternating cycles of pruning a small fraction and fine-tuning; recovers accuracy better at high compression.
- **CCFP (Correlation Circle Filter Pruning)** — structured one-shot filter pruning using pairwise filter correlations to identify redundant filters; proposed by Bellebna & Berrani.
- **MCFP (Multi-Criteria Filter Pruning)** — iterative pruning combining CCFP + SFP + FPGM criteria; achieves best accuracy-efficiency trade-off in the lecturer's experiments.
- **FLOPs (Floating Point Operations)** — count of multiply-add operations in a forward pass; primary measure of computational cost. Note: "FLOPs reduction (%)" = percentage of operations eliminated by pruning.
- **Parameters** — total number of trainable weight values in the model; determines storage/memory footprint.
- **Inference latency** — wall-clock time to process one input on the target hardware.
- **SparseDNN [1]** — software framework for fast sparse deep learning inference on CPUs, enabling speedup from unstructured pruning.
- **Tiramisu [2]** — polyhedral compiler for expressing fast and portable sparse code.
- **EIE [3]** — Efficient Inference Engine; hardware accelerator designed for compressed (sparse) deep neural networks.
- **Knowledge distillation** — training a small "student" network to mimic the outputs (and optionally intermediate representations) of a large "teacher" network.
- **Temperature (KD)** — softmax temperature $T > 1$ used in knowledge distillation to soften probability distributions and reveal inter-class similarity information.
- **Depthwise separable convolution** — factorization of a standard convolution into a per-channel depthwise conv + a $1 \times 1$ pointwise conv; used in MobileNet; ~8–9x fewer FLOPs than standard conv.
- **Low-rank factorization** — approximating a weight matrix $W$ as a product of two low-rank matrices $U \cdot V^\top$ to reduce parameter count.
- **NPU (Neural Processing Unit)** — dedicated on-chip hardware accelerator optimized for matrix multiply and convolution operations in DNNs; found in mobile SoCs (Apple Neural Engine, Qualcomm Hexagon, etc.).
- **Edge AI** — running AI inference on embedded/edge devices (cameras, drones, robots) rather than in the cloud.
- **IoT / MCU** — Internet of Things microcontrollers with very tight memory (kilobytes to a few megabytes); require extremely compact models.

---

## Exam targets

1. **Explain the cloud vs. embedded deployment trade-off.** Draw the two architecture diagrams (offline optimization then edge deployment; vs. cloud inference with communication). For each, list latency, bandwidth, privacy, and energy implications.

2. **State the quantization mapping formulas** — both directions ($\text{real} \to \text{integer}$ and $\text{integer} \to \text{real}$) — and apply them to a worked example. Explain what scale and zero_point represent.

3. **Calculate the memory saving from FP32 → INT8.** A model with $M$ parameters uses $4M$ bytes at FP32 and $M$ bytes at INT8; give the 4x figure and explain why.

4. **Contrast PTQ and QAT:** pipeline steps, when each is preferred, and what "fake quantization" means in QAT.

5. **List at least four domains where quantization matters** (Edge AI, Mobile, Automotive, IoT/MCU) and explain what constraint drives its use in each.

6. **Distinguish unstructured and structured pruning.** Draw a small network diagram and show what each type removes. Explain why unstructured pruning does not automatically accelerate on standard hardware (weights still exist; dense matmul is still executed).

7. **Explain why structured pruning is preferred for embedded deployment.** Specifically: it produces a smaller dense model with changed tensor shapes that runs on any hardware without specialized software.

8. **Describe the criteria-based pruning pipeline** for a CNN layer with multiple filters: compute scores, rank, remove lowest, fine-tune. Give two concrete criteria (e.g., L1-norm and geometric median FPGM) and explain the intuition of each.

9. **Compare one-shot vs. iterative pruning.** Draw or describe the accuracy-vs-FLOPs-reduction curve (green iterative curve vs. red one-shot dot from slide 17). Explain why iterative dominates at high compression: each fine-tune step lets the network adapt before the next round of pruning.

10. **Describe CCFP conceptually.** What makes two filters "correlated"? Why does high correlation imply redundancy? What is one-shot about it?

11. **Explain MCFP and its formula** (MCFP = CCFP + SFP + FPGM). Why does combining criteria improve over any single criterion? What does the iterative schedule add?

12. **Read a pruning results table.** Given Top-1 (%), Top-1 drop (%), FLOPs reduction (%), and Parameter reduction (%), identify the method with the best accuracy-efficiency trade-off. Specifically: small accuracy drop AND large FLOPs reduction simultaneously.

13. **Define FLOPs and parameters** as evaluation metrics and explain why they are not the same and why both must be reported.

14. **Explain knowledge distillation** at a conceptual level: teacher, student, soft labels, temperature. Write the loss $\mathcal{L}_\text{KD}$ and explain each term.

15. **Explain depthwise separable convolutions** (MobileNet): the two-step decomposition (depthwise then pointwise), the computational reduction formula, and the typical speedup for a 3x3 conv.

---

## Pitfalls

- **Quantization ≠ pruning.** Quantization keeps all weights but reduces precision; pruning removes weights/filters entirely. They are complementary and can be combined (quantize a pruned model), but confusing them in an exam is a common error.
- **FP32 → INT8 is 4x smaller, not 8x.** FP32 = 4 bytes; INT8 = 1 byte; ratio = 4. Students sometimes compute 32/8 = 4 bits and confuse bits with bytes.
- **Unstructured pruning does NOT automatically accelerate inference on a standard GPU or CPU.** The zeroed weights still occupy memory and the dense matrix multiplication is still executed — the zeros just produce zero contributions but are not skipped. Acceleration requires sparse execution support (SparseDNN, EIE, etc.).
- **Structured pruning changes tensor dimensions.** After removing K filters from a layer with N filters, the output tensor has N-K channels. All subsequent layers must be updated accordingly. This is a real implementation concern — not just zeroing a mask.
- **One-shot pruning is not always worse than iterative.** At moderate compression ratios (low FLOPs reduction), one-shot can be competitive. It only falls significantly behind at high compression. Always check the operating point.
- **FLOPs reduction ≠ latency reduction ≠ parameter reduction.** A model with 50% fewer FLOPs may not run 50% faster on a given device due to memory bottlenecks, layer overhead, and hardware pipeline effects. Always benchmark on the actual target hardware.
- **PTQ is not always sufficient.** For small models (MobileNet-scale) or aggressive quantization (INT4, binary), PTQ can cause unacceptable accuracy loss. In those cases, QAT is required.
- **Negative accuracy drop in the tables means accuracy improved.** In the CCFP/MCFP tables, some methods show negative Top-1 drop (e.g. -0.45% for PvLA, -0.27% for CIFP on CIFAR-10). This means the pruned model is slightly more accurate than the baseline, possible because pruning acts as regularization.
- **Iterative MCFP = CCFP + SFP + FPGM, not MCFP alone.** The note on slide 18 explicitly states "Multi-Criteria (MCFP) = CCFP + Orange cells (SFP, FPGM)". SFP and FPGM are cited methods; CCFP is the new contribution; combining all three gives MCFP.
- **Knowledge distillation and low-rank factorization are not covered deeply in these slides** — the lecturer focused on quantization and pruning. Know the concepts for context but focus exam preparation on quantization and structured filter pruning.
- **Calibration is not training.** In PTQ, calibration only computes scale/zero_point from a small representative dataset — it does not update model weights. Confusing calibration with fine-tuning is a common misstatement.
