# Part 2 — Image Formation

## Bird's eye view

- An image is a **2-D function** I(x, y) : Omega ⊂ R² → R, where the value at each point is proportional to the light energy (intensity) collected at that location.
- Image formation starts with **light**: a source illuminates a scene, surfaces reflect light described by a 5-D **BRDF** function; a camera captures the reflected radiance.
- Light has a **dual nature** — it behaves as a particle (photon, travels in straight lines) and as a wave (showing refraction and diffraction); visible light spans roughly 400–700 nm.
- The **pinhole camera model** places a barrier with a tiny aperture between scene and sensor, enforcing a one-to-one ray mapping and producing a sharp (but dim, inverted) image; real cameras add lenses to collect more light without losing sharpness.
- Inside the sensor, photons trigger the **photoelectric effect** (photon → electron → charge → voltage → digital number) via a 6-step pipeline; the two dominant sensor technologies are **CCD** and **CMOS**.
- A continuous image is digitised by two orthogonal operations: **sampling** (domain R² → M×N grid of pixels) and **quantization** (codomain R → {0, 1, …, 2^b − 1} discrete levels).
- Key image quality parameters are **spatial resolution** (M, N — dots per unit distance) and **intensity resolution** (bit depth b, L = 2^b levels); **dynamic range** and **contrast** describe the system's ability to represent extremes of brightness simultaneously.
- The lecture covers the full chain from physics of light → camera optics → sensing → digitisation → the discrete image as a 2-D function — the prerequisite model for all subsequent computer vision algorithms.

---

## Detailed notes

### 1. Why image formation matters

Almost all computer vision algorithms operate on images. Before any processing can happen, we must understand what an image is, how it is created physically, and what mathematical object represents it. The concept introduced in this part underlies every subsequent topic in the course.

---

### 2. The nature of light

#### 2.1 Dual nature

Light behaves both as a **particle** and as a **wave**.

**Particle behaviour (photon model)**
- A photon is an elementary particle with no mass, travelling at the speed of light in vacuum (c ≈ 3 × 10^8 m/s).
- It carries energy and momentum.
- Consequences: travels in straight lines, can reflect off mirrors, casts sharp shadows.

**Wave behaviour**
- **Refraction**: light changes direction when it passes from one medium to another (different speeds). Classic example: a straw appears bent in a glass of water.
- **Diffraction**: light bends around obstacles or spreads out after passing through a narrow slit. Diffraction through a grating produces a pattern of orders (n = 0, ±1, ±2, …) on a screen at distance D, with angular separation theta.
- **Wavelength (lambda)**: distance between two consecutive peaks of the electromagnetic wave.
- **Amplitude**: related to intensity or brightness of the light.
- Different wavelengths correspond to different colours in the visible spectrum.

#### 2.2 The electromagnetic spectrum

The full spectrum runs from gamma rays (highest frequency, shortest lambda ~10^-16 m) through X-rays, UV, visible, IR, microwave, FM/AM radio, to long radio waves (lowest frequency, longest lambda ~10^8 m). The spectrum can be described by either wavelength lambda or frequency v (related by c = lambda × v).

**Visible light** occupies a narrow window:
- Lambda approximately 400 nm (violet) to 700 nm (red)
- Green: approximately 500–570 nm
- The visible band is an extremely small slice of the full EM spectrum

#### 2.3 Light and perception: radiance, luminance, brightness

| Quantity | Definition | Objective? | Eye-dependent? | Instrument-measurable? |
|---|---|---|---|---|
| **Radiance** | Physical energy of light travelling along rays (W/sr/m²) | Yes (physical) | No | Yes |
| **Luminance** | Radiance weighted by the human eye sensitivity curve (lm) | Semi (eye-weighted physical) | Yes (sensitivity curve) | Yes |
| **Brightness** | Psychological perception of light intensity | No (subjective) | Yes + context + cognition | No |

The perception chain:
**Physical world → Radiance → Camera (records as pixel intensities) → Human eye (luminance encoding) → Brain (perceived brightness)**

Key insight: **Brightness is NOT luminance.** Two patches with identical luminance can appear to have very different brightness depending on their surrounding context (simultaneous contrast illusion: a grey square appears lighter on a dark background than on a light background, even though its luminance is identical).

- Brightness has no physical unit, cannot be measured by an instrument, and depends on the observer and context.

---

### 3. The BRDF model

When light hits a surface it scatters and reflects. The most general model for this interaction is the **Bidirectional Reflectance Distribution Function (BRDF)**:

    f_r(theta_i, phi_i, theta_r, phi_r, lambda)

A 5-dimensional function parameterised by:
- (theta_i, phi_i): polar angles of the **incoming** light direction relative to the surface normal
- (theta_r, phi_r): polar angles of the **reflected** (outgoing) direction
- lambda: wavelength

The BRDF describes how much light arriving from direction (theta_i, phi_i) is reflected toward direction (theta_r, phi_r). Surfaces range from perfectly diffuse (Lambertian — scatters equally in all directions) to perfectly specular (mirror — single reflected direction).

---

### 4. Cameras and image formation

#### 4.1 The need for a camera

To see a 3D scene, an eye or camera must capture light reflected from scene surfaces. The scene must be illuminated by one or more light sources. The image is formed on a **sensor plane** via an **optical system**.

#### 4.2 The problem without a barrier

Without any optics, rays from every scene point reach every point on the sensor — rays from different scene points are completely mixed. Result: a completely blurred, unrecognisable image. We need a mechanism to map scene points to unique sensor points.

#### 4.3 The pinhole camera (camera obscura)

**Key idea**: place a **barrier with a small hole (aperture)** between the object and the sensor. Only rays passing through the pinhole reach each sensor location, greatly reducing mixing.

Historical note: Leonardo da Vinci (1452–1519) described the camera obscura in his notebook in 1502: objects illuminated by the sun send their images through the aperture and appear upside-down and smaller on the opposite wall, because of the crossing of rays at the aperture.

**How the pinhole size affects image quality:**

| Pinhole size | Effect |
|---|---|
| Too **large** | Multiple rays from the same scene point hit different sensor locations → blurring |
| **Optimal** small | One-to-one mapping → sharp image, but very little light |
| Too **small** | Light is limited (long exposure needed); **diffraction effects** cause blurring again |

The 4 images on slide 22 (aperture sizes 2 mm, 1 mm, 0.6 mm, 0.35 mm) demonstrate that sharpness first improves then worsens as the aperture is reduced, with 0.35 mm giving the sharpest result in the example shown.

**Three key limitations of the pinhole camera:**
1. The image is very dark (very little light)
2. The image is not perfectly sharp (diffraction)
3. Little control over focus

**The exposure time problem:** small pinholes → little light → very long exposure times → motion blur, low SNR. Increasing hole size increases brightness but reduces sharpness. The solution is **lenses**.

#### 4.4 Using lenses

A lens achieves the same perspective projection as a pinhole but gathers a much greater amount of light. All rays received by the lens from scene point P_0 are refracted (bent) to converge at image point P_i. In the pinhole case only the single ray along the optical axis would have produced P_i; the lens funnels an entire cone of rays to the same point.

The perspective projection model remains identical; lenses add light-gathering without changing the geometric mapping.

---

### 5. Digital image sensing

#### 5.1 The sensing pipeline

Incoming light radiation is converted from physical energy into a digital number through a 6-step pipeline:

1. **Photon arrival at the sensor**: light from the scene reaches the photosensitive material.
2. **Photoelectric effect (photon → electron)**: when a photon hits the photosensitive material (typically silicon), if its energy is sufficient it releases one electron. One photon → at most one electron; number of electrons ∝ number of photons.
3. **Charge accumulation** (integration over exposure time): each pixel integrates charge Q ∝ (number of photons) × (exposure time). Longer exposure → brighter image.
4. **Charge-to-voltage conversion**: V = Q / C, where C is the pixel capacitance. Voltage is a convenient analog representation of light intensity.
5. **Readout and amplification**: introduces read noise.
6. **Analog-to-Digital Conversion (ADC)**: voltage → Digital Number (DN). Produces pixel intensity (e.g., 8-bit → 256 levels, 10-bit, 12-bit, etc.). This is the image we finally process in computer vision.

Full pipeline summary:
**Photons → Electrons → Electric charge → Voltage → Digital number (pixel)**

Sensors are arranged in **linear** (1-D scan) or **2-D arrays** that reflect the grid structure of the final image.

#### 5.2 CCD vs. CMOS sensors

| Property | CCD | CMOS |
|---|---|---|
| ADC location | At the end of each row/column | Directly at each sensing cell |
| Low light performance | Better (lower noise) | Worse (but improving) |
| Speed | Faster readout | Flexible, modern CMOS now competitive |
| Power | Higher | Lower |
| Dominant today? | No (specialised applications) | Yes (most digital cameras) |

Both CCD and CMOS convert photons to electrons using the photoelectric effect. In CCD, charge is shifted row by row to a single amplifier and ADC. In CMOS, each pixel has its own amplifier and ADC circuitry.

#### 5.3 The image formation process (end to end)

The sensor array is placed at the focal plane of the lens. Each sensor cell produces an output proportional to the integral of light received over the exposure time. The scene element → imaging system → (internal) image plane → output digitised image chain produces a rectangular matrix of digital values.

---

### 6. The image as a 2-D function

#### 6.1 Formal definition

An image is modelled as a function:

    I : Omega ⊂ R² → R

where:
- **Domain** Omega is a (usually rectangular) subset of the real image plane (continuous spatial coordinates)
- **Codomain** R is the set of possible intensity values

I(x, y) is proportional to the amount of light energy collected at image plane coordinates (x, y). That energy is called the **intensity** (or grey level for monochromatic images).

**Three ways to represent a digital image:**
1. As a **2-D function / surface**: a 3-D plot of f(x, y) over the image plane — intensity is height. A bright rectangle appears as a plateau, dark regions as valleys.
2. As a **visual intensity array**: the familiar greyscale or colour image.
3. As a **matrix**: each entry is an integer pixel value.

---

### 7. Sampling and quantization

A continuous real image I(x, y) must be converted to a digital image through two operations:

#### 7.1 Sampling (spatial discretisation)

**Definition**: reduces the (continuous) image domain R² to a finite set of M × N spatial coordinates.

    Sampling: R² → M × N

In practice, an **equi-spaced grid** of values (a matrix) is used. This reflects the regular arrangement of cells in a CMOS or CCD sensor. Each sample point is called a **pixel** (picture element).

Visually: a continuous smooth shape on the left becomes a pixelated grid representation on the right when overlaid with a coarse sampling grid.

#### 7.2 Quantization (intensity discretisation)

**Definition**: reduces the continuous sensor response (function codomain) to a finite set of b-bit integer values.

    Quantization: R → {0, 1, 2, …, 2^b - 1}

The continuous intensity range is divided into 2^b equal intervals; each f(x, y) is rounded to the nearest discrete level. This is illustrated by the 3-bit quantization graph: a smooth sinusoidal analog input (blue) becomes a staircase digital output (red) with 8 discrete levels (000 through 111).

**Combined effect**: the continuous blob image, when first sampled (fine grid) and then quantized (coarser grey levels), produces the blocky pixelated representation seen on slides 41.

---

### 8. Digital image parameters

#### 8.1 Spatial resolution

- **Definition**: measure of the smallest discernible detail in an image; usually given as **dots (pixels) per unit distance** (e.g., dpi — dots per inch).
- Must always be stated with respect to spatial units. A 1024 × 1024 pixel image alone is not meaningful without the physical size of each pixel in the real world.
- **M and N** (image dimensions in pixels) determine spatial resolution only when the lens and shooting distance are held constant.
- Example: the same clock photographed at 1250 dpi (very sharp), 300 dpi (good), and 72 dpi (visibly pixelated and blurry).
- Satellite example: Landsat ETM+ (coarsest), ATLAS (medium), QuickBird (finest) — same scene, dramatically different detail.

**Important nuance**: more megapixels does not automatically mean higher spatial resolution in physical terms — it only matters if the lens resolves that level of detail and the subject is at comparable distance.

#### 8.2 Intensity resolution (bit depth)

- **Definition**: smallest discernible change in intensity level; determined by the number of bits b used for quantization.
- Common values: 8-bit (256 levels, most common), 10-bit (1024), 12-bit (4096), 16-bit (65,536).
- L = 2^b is the number of grey levels.
- Low bit depth produces **banding / false contour** artefacts. Example (skull X-ray):
  - 256 levels: smooth, full detail
  - 16 levels: subtle banding visible
  - 4 levels: pronounced posterisation
  - 2 levels (binary): pure black and white, all interior detail lost

#### 8.3 Dynamic range

- **Definition**: the ratio of the maximum measurable intensity to the minimum detectable intensity level in a system.
- Determined by the number of bits: higher bit depth → wider dynamic range.
- **HDR (High Dynamic Range)** images preserve detail in both shadows and highlights simultaneously.
- **LDR (Low Dynamic Range)** images lose detail in either dark or bright regions.
- The **human eye has a dynamic range of about 10^9** — far exceeding any current digital sensor.
- Even if we acquire HDR images, displaying them on a standard screen is problematic (the screen itself has limited dynamic range), requiring tone-mapping compromises.

**Dynamic range vs. Intensity resolution — key distinction:**
- Dynamic range = span from darkest to brightest a system can capture (a ratio, about the range)
- Intensity resolution (bit depth) = how finely that range is divided into discrete steps (about the granularity)

A system can have wide dynamic range but coarse intensity resolution (few steps over a wide range) or narrow dynamic range with fine intensity resolution (many steps over a small range).

#### 8.4 Contrast

**Definition**: the difference between the highest and lowest intensity levels of an image (within a captured scene). Even with HDR acquisition, display contrast is limited by the monitor, requiring compromises (discarding dark details or saturating bright areas).

---

### 9. Memory requirements for a digital image

For a digital image of size M × N pixels with b bits per pixel:

    Storage = M × N × b   bits   =   M × N × b / 8   bytes

For colour images (3 channels RGB):

    Storage = M × N × b × 3   bits

Example: a 1920 × 1080 × 8-bit greyscale image requires 1920 × 1080 × 1 = 2,073,600 bytes ≈ 2 MB (uncompressed).

---

### 10. Worked example: tracing an image from scene to pixel

**Scenario**: a green leaf (reflectance peak ~520 nm) illuminated by sunlight.

1. **Light source**: sun emits broad-spectrum radiation across all visible wavelengths.
2. **BRDF interaction**: leaf surface absorbs most wavelengths; reflects predominantly 500–570 nm (green). BRDF determines how much of the incident radiance at each angle is reflected toward the camera.
3. **Camera optics**: reflected green photons pass through the lens; all rays from a leaf point P_0 converge at image point P_i on the sensor.
4. **Sensing pipeline**: photons hit silicon pixels → photoelectric effect releases electrons → charge accumulates over exposure time → ADC converts to 8-bit digital number DN in [0, 255].
5. **Sampling**: the continuous leaf image is sampled on a regular M × N grid; each cell's accumulated charge gives one pixel value.
6. **Quantization**: each pixel is assigned one of 256 integer levels (0 = black, 255 = white for greyscale; in colour: separate R, G, B channels).
7. **Result**: I[m, n] — a 2-D matrix of integers, the digital image ready for computer vision algorithms.

---

## Key terms (glossary)

- **Photon**: elementary particle of light; no mass; carries energy proportional to frequency (E = hv).
- **Radiance**: physical measure of light energy per unit area per unit solid angle travelling along a ray (W/sr/m²). What the camera sensor measures.
- **Luminance**: radiance weighted by the human eye sensitivity function (lm/m²). What the eye encodes.
- **Brightness**: subjective psychological perception of light intensity. What the brain experiences. No physical unit.
- **BRDF**: Bidirectional Reflectance Distribution Function — f_r(theta_i, phi_i, theta_r, phi_r, lambda); describes how a surface reflects light.
- **Camera obscura**: dark chamber with a pinhole aperture; historical precursor to modern cameras.
- **Pinhole camera**: imaging model where a tiny aperture enforces one-to-one ray mapping; produces sharp but inverted and dim images.
- **Aperture**: size of the hole through which light enters; controls the sharpness/brightness trade-off in a pinhole camera.
- **Photoelectric effect**: phenomenon where photons hitting a photosensitive material release electrons proportional to the number of incident photons.
- **CCD (Charge-Coupled Device)**: image sensor where charge is shifted to a single ADC at the row/column periphery.
- **CMOS (Complementary Metal Oxide Semiconductor)**: image sensor where ADC conversion occurs at each pixel cell; dominant in modern cameras.
- **ADC (Analog-to-Digital Converter)**: converts analog voltage to a digital number (DN); determines bit depth.
- **Pixel**: picture element; the smallest unit of a digital image; one sample in the sampling grid.
- **Sampling**: discretisation of the image domain from R² to an M × N grid.
- **Quantization**: discretisation of the image codomain from R to {0, …, 2^b − 1}.
- **Spatial resolution**: measure of the smallest discernible detail; given as pixels per unit distance (e.g., dpi).
- **Intensity resolution (bit depth)**: number of discrete intensity levels; L = 2^b.
- **Dynamic range**: ratio of maximum to minimum measurable intensity in a system.
- **HDR / LDR**: High / Low Dynamic Range — whether a system preserves detail across both extremes of brightness.
- **Contrast**: difference between the highest and lowest intensity levels in an image.
- **I(x, y)**: the image function — maps 2-D spatial coordinates to intensity values.

---

## Exam targets

1. **Define the image as a 2-D function**: write I : Omega ⊂ R² → R and explain every symbol. State what I(x, y) represents physically (intensity ∝ light energy collected at (x, y)).

2. **Distinguish radiance, luminance, and brightness**: use the summary table. Be precise — luminance is eye-weighted physical; brightness is purely subjective with no units.

3. **Explain the BRDF**: write the 5 arguments f_r(theta_i, phi_i, theta_r, phi_r, lambda) and state what the function expresses. Know why it is 5-D.

4. **Explain the pinhole camera principle**: why is a barrier with a small hole needed? What happens with a large hole (blurring from ray mixing)? What happens with a tiny hole (diffraction)? What are the 3 limitations (dark, not sharp, no focus control)?

5. **Explain why lenses are used**: same perspective projection geometry as pinhole, but gathers far more light by converging an entire cone of rays to each image point.

6. **Describe the 6-step digital sensing pipeline**: Photon arrival → Photoelectric effect → Charge accumulation (Q ∝ photons × time) → Charge-to-voltage (V = Q/C) → Readout/amplification → ADC (voltage → DN). Know the formula V = Q/C and the charge relation Q ∝ photons × exposure time.

7. **Compare CCD vs. CMOS**: ADC location (row-end vs. per-pixel); CCD better in low light, CMOS dominant today.

8. **Define sampling and quantization with equations**:
   - Sampling: R² → M × N (spatial grid)
   - Quantization: R → {0, 1, …, 2^b − 1}

9. **Distinguish spatial resolution vs. intensity resolution vs. dynamic range**: spatial = M, N, pixels per unit distance; intensity = bit depth b, number of levels L = 2^b; dynamic range = ratio max/min measurable intensity.

10. **Compute storage size**: M × N × b bits for grayscale; × 3 for colour.

11. **Describe the effect of reducing bit depth**: 256 levels (smooth) → 16 (banding) → 4 (posterisation) → 2 (binary only). Relate to quantization artefacts.

12. **Explain why "more pixels ≠ more spatial resolution"**: resolution depends on pixel count AND the physical size each pixel represents in the world AND lens quality.

---

## Pitfalls

- **Brightness ≠ Luminance ≠ Radiance**: these are three different quantities at different levels of the perception chain. Confusing them is the most common conceptual error in this part.
- **Dynamic range ≠ bit depth**: dynamic range is the ratio of brightest to darkest the system can capture; bit depth is the granularity of that range once captured. A wide dynamic range with low bit depth gives coarse steps; a narrow range with high bit depth gives fine steps over a small span.
- **Pinhole size trade-off is not monotone**: making the pinhole smaller does NOT always improve sharpness — below a certain size, diffraction causes blurring again. The optimal size is a specific trade-off point.
- **Spatial resolution requires a reference**: a "1024 × 1024 image" has no meaningful resolution without stating the physical area (and hence pixel size) it covers. Resolution = pixels per unit distance, not just pixel count.
- **Sampling and quantization are independent**: sampling is about where you measure (space), quantization is about how precisely you record the value (intensity). They can be varied independently.
- **CCD is NOT obsolete for all uses**: CCD is faster and better in low light; CMOS dominates consumer cameras but CCD still appears in scientific/astronomical imaging.
- **V = Q/C is per-pixel**: each pixel has its own capacitance C and accumulates its own charge Q; the voltage-to-DN conversion then maps this to a pixel intensity independently per cell.
- **The BRDF is not just for mirrors**: it applies to any surface; diffuse surfaces have a flat (constant) BRDF over all outgoing directions, specular surfaces have a narrow peak in the mirror direction.
- **Luminance can be measured; brightness cannot**: exam questions often test this distinction. If asked "can it be measured by an instrument?", radiance = yes, luminance = yes, brightness = no.
