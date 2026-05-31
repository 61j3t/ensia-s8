# Part 3 — Color Fundamentals & Basic Image Processing

## Bird's eye view

- **Color is not a physical property** of the world — it is a consequence of how the eye senses light. The eye compresses the infinite-dimensional spectrum into just three numbers (S, M, L cone responses), making color perception many-to-one (metamerism).
- **Three perceptual descriptors** — hue (what color), saturation (purity), brightness/luminance (how light) — underlie all color models and serve as the bridge between physics and perception.
- **A color model (color space)** is a mathematical coordinate system + subspace where each color is a single point. Different tasks call for different spaces: RGB for displays, HSV/HSI for human-intuitive manipulation, CIE XYZ as the device-independent reference, CMYK for printing, YCbCr for video compression, CIE Lab for perceptually uniform color difference.
- The **RGB cube** is the workhorse model: each pixel is a triplet (R, G, B) in $[0, 255]^3$. The diagonal from (0,0,0) black to (255,255,255) white is the grayscale axis.
- **HSV conversion from RGB** is the most tested computation: normalize to $[0,1]$, then $V = \max(r,g,b)$, $S = (V - \min)/V$, H determined by which channel dominates — full worked example for RGB(255,0,0) → HSV(0°, 100%, 100%).
- The **CIE 1931 chromaticity diagram** (horseshoe) separates color from brightness; a device's gamut is the triangle formed by its three primaries inside the diagram.
- The slide set covers only color models (pages 1–67) — sampling, quantization, and basic pixel operations referenced in the part title appear to be in a companion part; all slide content is captured here.

---

## Detailed notes

### 1. Color Fundamentals: Physical Basis

#### 1.1 Light and the spectrum

- We perceive **light radiation** characterized by intensity and wavelength.
- **1660 Newton**: white light through a prism separates into a rainbow of colors. Conclusion: white is not a pure color — it is the composition of all colors.
- The **electromagnetic spectrum** (figure description): a broad band from gamma rays (~0.001 nm) through X-rays, ultraviolet, **visible spectrum (~400–700 nm)**, infrared, microwaves, TV, radio.
- Within the visible range, wavelength maps to perceived color: ~400 nm = violet/blue → ~500 nm = cyan/green → ~600 nm = orange/red → 700 nm = deep red.
- The prism diagram shows white light entering from the left; the prism fans it into a rainbow with ultraviolet at the bottom edge and infrared at the top edge.

#### 1.2 Object color perception

- Colors perceived in an object are determined by the **wavelengths reflected** from its surface.
- A body that reflects a **limited range** of the spectrum appears as a shade of that color.
- Key cases (figure description):
  - **Blue object**: reflects mainly blue (B) wavelengths → appears blue.
  - **Red object**: reflects mainly red (R) wavelengths → appears red.
  - **White object**: lit by red-only light, reflects red → appears red (color depends on illuminant).
  - **Black object**: absorbs all wavelengths → reflects nothing → appears black.
- This is why illuminant matters: the same object can look different under different light sources (see also: metamerism).

---

### 2. Human Vision and Color Perception

#### 2.1 Photoreceptors: rods and cones

The retina contains two types of photoreceptor cells:

| Cell type | Function | Color? |
|---|---|---|
| **Rods** | Low-light (scotopic) vision | No — achromatic |
| **Cones** | Bright-light (photopic) vision | Yes — chromatic |

- The human retina contains approximately **6–7 million cones** total.

#### 2.2 The three cone types (S, M, L)

Each cone type has a broad spectral sensitivity curve (absorption vs. wavelength, peaks shown):

| Cone | Sensitivity peak | Proportion of all cones |
|---|---|---|
| **S (Short-wavelength)** | Blue — ~420–445 nm | 2–5% |
| **M (Medium-wavelength)** | Green — ~530–540 nm | 32–40% |
| **L (Long-wavelength)** | Red — ~560–580 nm | 55–65% |

Figure description: Three overlapping bell curves on a 400–700 nm axis. The blue (S) curve peaks earliest at ~445 nm; the green (M) peaks at ~535 nm; the red (L) peaks at ~575 nm. The curves overlap substantially — this overlap is the cause of metamerism.

Key insight: **2% of cones are blue-sensitive (S) yet they are the most sensitive per-cone**; 33% respond to green; 65% to red.

#### 2.3 Tristimulus values

A color is perceived through **simultaneous stimulation** of all three cone types. The brain receives three numbers — the **tristimulus values** S, M, L:

$$S = \int I(\lambda)\, S(\lambda)\, d\lambda$$
$$M = \int I(\lambda)\, M(\lambda)\, d\lambda$$
$$L = \int I(\lambda)\, L(\lambda)\, d\lambda$$
Where:
- $I(\lambda)$ = incoming light spectrum (spectral power distribution).
- $S(\lambda), M(\lambda), L(\lambda)$ = cone sensitivity functions.

**Critical insight**: The eye does NOT measure wavelength directly. It measures **three integrals of the spectrum**. The original spectral shape is lost once reduced to $(S, M, L)$.

#### 2.4 Color blindness (color vision deficiency)

- **Color blindness** = seeing colors differently than most people; difficulty distinguishing certain colors.
- Most common type: **red-green confusion** (trouble distinguishing red from green).
- Another type: **blue-yellow confusion**.

Types illustrated:
- **Monochromacy**: no color at all (no functioning cones, or only one type) — sees in pure grayscale.
- **Tritanopia**: M-cone absent → loss of green distinction.
- **Protanopia**: L-cone absent → loss of red distinction (red-green confusion).
- **Deuteranopia**: M-cone absent (related; causes red-green confusion).

**The Ishihara Test** (1917, Dr. Shinobu Ishihara):
- Circular plates filled with colored dots; a number is hidden in a specific color.
- Normal vision: sees the number easily.
- Red-green color blindness (protanopia/deuteranopia): may fail to see or see a different number.

---

### 3. Color Matching and the CIE Standard

#### 3.1 Color matching experiment

Goal: characterize every **monochromatic (single-wavelength) color** as a mixture of three suitably chosen primaries.

In the **1930s**, the **CIE** (Commission Internationale de l'Eclairage — International Commission on Illumination) defined three primary colors:
1. **RED: 700.0 nm**
2. **GREEN: 546.1 nm**
3. **BLUE: 435.8 nm**

Experimental setup (figure description): One side of a split screen shows an RGB tristimulus source (red, green, blue lamps combined). The other side shows a test lamp (single wavelength). An observer adjusts the intensities of the RGB lamps to match the test color. Results averaged over 17 British subjects → the **CIE Standard Observer**.

#### 3.2 CIE RGB color matching functions

The resulting **CIE RGB color matching functions** $\bar{r}(\lambda), \bar{g}(\lambda), \bar{b}(\lambda)$ (figure description):
- A plot from 360–760 nm on the x-axis, matching intensity on the y-axis.
- The **blue ($\bar{b}$)** curve peaks sharply at ~440 nm (peak ~0.31).
- The **green ($\bar{g}$)** curve peaks at ~520–560 nm (peak ~0.21).
- The **red ($\bar{r}$)** curve has a large peak at ~600 nm (peak ~0.34) and a **negative lobe** around 450 nm (dips to ~−0.1).

**Important limitation**: Almost all colors can be matched, EXCEPT some shades between blue and green. For these, **a negative amount of red is needed** — physically this means adding red to the test side, not the reference side. The reason: the sensitivity regions of the three cone classes overlap, so green light activates the L (red) cone more than cyan test light does. To balance the response, you must subtract some red.

---

### 4. Color Characteristics: Hue, Saturation, Brightness

The three characteristics used to distinguish colors:

#### 4.1 Brightness (Luminance)

- **Brightness**: achromatic notion of intensity — how much light is emitted or reflected.
- Determines how dark or light a color appears without changing its hue.
- Formula (Lightness in HSL): $L = \frac{M + m}{2}$, where $M = \max(R,G,B)$, $m = \min(R,G,B)$.

#### 4.2 Hue

- **Hue**: the dominant wavelength in a mixture of light waves — the basic color identity (red, green, blue, etc.).
- Hue is the foundation of color perception; it dates to Newton's color wheel.
- A bright pink and a deep maroon share the **same hue (red)** but differ in saturation and luminance.
- On the **color wheel** (figure description): a circular disc with R at top, G at bottom-right, B at bottom-left. Between them: Yellow (R-G), Cyan (G-B), Magenta (B-R). Any position 0°–360° is a hue.
- Formula: $H = 0$ if $R = G = B$ (achromatic); otherwise a value 0–255 (or 0–360°) split into three strips: G→B, B→R, R→G gradients.

#### 4.3 Saturation

- **Saturation**: the "relative purity" — how much white light is mixed with the hue. From gray (0 saturation) to pure vivid color (full saturation).
- High saturation: vivid, bold colors. Low saturation: muted, washed-out, closer to gray.
- Formula: $S = 0$ if $R = G = B$; otherwise:
  - If $L < 128$: $S = \frac{255 \times (M - m)}{M + m}$
  - If $L \geq 128$: $S = \frac{255 \times (M - m)}{511 - (M + m)}$
  - Where $M = \max(R,G,B)$, $m = \min(R,G,B)$, $L = (M+m)/2$.

#### 4.4 Relationship (figure description)

A diagram showing a hue bar (rainbow strip) at the top. From one chosen blue hue:
- Moving **left** (less saturation): colors become increasingly gray until fully achromatic.
- Moving **down** (more brightness): colors go from dark/near-black to bright.

Summary table:

| Attribute | Controls | Changes |
|---|---|---|
| Hue | Color identity | Which color it is |
| Saturation | Purity | How vivid vs. washed out |
| Luminance | Brightness | How dark vs. light |

---

### 5. Color Models (Color Spaces)

**Definition**: A color model (also called color space or color system) is:
- A mathematical structure for representing colors as sets of numbers.
- A specification of (1) a coordinate system and (2) a subspace where each color is a single point.
- It maps physical stimuli to perceived color and allows color reproduction under controlled conditions.

**Why not just use Pantone?** Pantone provides discrete, enumerated color swatches for human communication. It cannot answer: how close are two colors? how to interpolate? how to convert camera → screen → printer? Color spaces are needed for **numerical representation, computation, and perception modeling**.

---

### 6. The RGB Color Model

#### 6.1 Structure: the color cube

RGB = additive color model. Each color is a point $(R, G, B)$ in $[0, 255]^3$ (or normalized $[0, 1]^3$).

**The RGB cube** (figure description):
- A cube with three axes: R (red), G (green), B (blue).
- Corners:
  - **(0,0,0)** = Black (no light)
  - **(1,0,0)** = Red
  - **(0,1,0)** = Green
  - **(0,0,1)** = Blue
  - **(1,1,0)** = Yellow (R+G)
  - **(0,1,1)** = Cyan (G+B)
  - **(1,0,1)** = Magenta (R+B)
  - **(1,1,1)** = White (all channels max)
- The **main diagonal** from (0,0,0) to (1,1,1) is the **grayscale axis** — equal R, G, B = gray.

#### 6.2 Applications

- Used in all **electronic displays**: LCD, OLED, CRT, televisions, computer monitors, digital cameras.
- Each pixel has **three subpixels** (R, G, B); adjusting their intensities produces millions of colors.
- **RGB is device-dependent**: different displays use slightly different RGB primaries, leading to variations in color reproduction.

#### 6.3 Problems with RGB

**Problem 1 — Limited gamut**:
- Only a subset of human-perceivable colors can be represented.
- Colors that fall outside the gamut include: pure cyan, pure yellow, pure magenta; highly saturated neon/laser colors; some fluorescent colors.
- The exact gamut depends on the physical display hardware.

**Problem 2 — Perceptually non-linear**:
- Two points at distance $d$ in one part of RGB space may be perceptually very different.
- Two other points at the same distance $d$ in another part may look identical.
- RGB is not aligned with human color perception.

**Problem 3 — Unintuitive**:
- It is not easy to determine which $(R, G, B)$ values produce a given perceived color (e.g., "rust orange" requires knowing specific R, G, B ratios).

---

### 7. CIE XYZ Color Space

#### 7.1 Motivation

To eliminate the **negative values** in the CIE RGB color matching functions, the CIE defined a new color space using **three virtual (hypothetical) primaries: X, Y, Z**.

These primaries are **mathematical constructs** — they do not correspond to real physical light sources. You cannot build a lamp that emits "pure X light."

**RGB → XYZ conversion matrix**:
$$\begin{bmatrix} X \\\\ Y \\\\ Z \end{bmatrix} = \frac{1}{0.17697} \begin{bmatrix} 0.49 & 0.31 & 0.20 \\\\ 0.17697 & 0.81240 & 0.01063 \\\\ 0.00 & 0.01 & 0.99 \end{bmatrix} \begin{bmatrix} R \\\\ G \\\\ B \end{bmatrix}$$
Or compactly: $[X, Y, Z]^T = M \cdot [R, G, B]^T$

Key property: **Y corresponds to luminance**.

#### 7.2 Properties of XYZ

Advantages:
- **All matching functions are non-negative** — no negative primary amounts needed.
- **Device-independent**: does not depend on LEDs, phosphors, or sensors.
- **Covers all visible colors**.
- Serves as the **reference/foundation** for all other color spaces (RGB, Lab, etc.).

Limitations:
- Many XYZ points do not correspond to physically visible colors.
- Not realizable physically (this is actually a strength for standardization).
- Not perceptually uniform.

**Rule**: Use XYZ to define and convert colors — not to display them.

#### 7.3 Chromaticity coordinates

Dividing XYZ values by their sum removes the luminance component:
$$x = \frac{X}{X+Y+Z}, \quad y = \frac{Y}{X+Y+Z}, \quad z = \frac{Z}{X+Y+Z}$$
Note: $x + y + z = 1$, so $z = 1 - x - y$. Only **two coordinates $(x, y)$ are needed** to fully describe chromaticity. This gives the **CIE 1931 chromaticity diagram**.

---

### 8. The CIE 1931 Chromaticity Diagram

#### 8.1 What it is

- A **2D representation** of all colors perceivable by the average human eye, plotted in $(x, y)$ space.
- Each point corresponds to a chromaticity (color) **independent of brightness** — luminance information (Y) is discarded.
- XYZ tells us *how much light* AND *what color*. The chromaticity diagram tells us *what color only*.

#### 8.2 Key features (figure description)

The diagram is a horseshoe/tongue shape:

1. **Chromaticity coordinates $(x, y)$**: x-axis (0–0.8), y-axis (0–0.9).
2. **Spectral locus (outer curved edge)**: represents pure monochromatic (single-wavelength) colors. Each point on the curved boundary corresponds to a single wavelength from ~400 nm (violet, bottom-left) going clockwise through green (top, ~520 nm), yellow, orange, to red (~700 nm, bottom-right).
3. **Purple line (straight bottom edge)**: connects the red and blue ends of the spectrum. Represents **non-spectral purples** — colors that do not appear in the rainbow and can only be produced by mixing red and blue.
4. **White point (D65 Standard Illuminant)**: located near the center of the diagram (~$x=0.31$, $y=0.33$), labeled as "Point of Equal Energy."
5. **Gamut triangle**: any display device is represented by a triangle whose vertices are its three primaries (R, G, B). Colors inside the triangle are reproducible; colors outside are out-of-gamut.

#### 8.3 Color mixing property

A **straight line segment** joining any two points in the diagram defines all colors obtainable by additively mixing those two colors. Any color inside a triangle formed by three primaries can be reproduced by mixing those primaries.

#### 8.4 Gamut

The gamut of a device = the convex envelope of its primary colors inside the chromaticity diagram.

Figure description: The chromaticity diagram with several overlapping triangles labeled sRGB, Adobe RGB, Apple RGB, ColorMatch RGB, Wide Gamut RGB. Each triangle has vertices at the device's R, G, B primary positions. Wide Gamut RGB has the largest triangle; sRGB the smallest. All triangles sit inside the full horseshoe (the human visual range). Color space A and Color Space B can have overlapping or non-overlapping gamuts, requiring conversion methods.

#### 8.5 Important limitation

The chromaticity diagram is **not perceptually uniform**: equal distances in $(x, y)$ space do NOT correspond to equal perceived color differences. This motivated the development of perceptually uniform spaces (CIE Lab).

#### 8.6 Color model as a triangle

A color model is defined by:
- A **triangle** in the chromaticity diagram (the three primaries as vertices), and
- A **white point**.

Choosing a different triangle = defining a different color model. The same physical color will have different coordinates in different models.

---

### 9. HSV (Hue, Saturation, Value) Color Model

#### 9.1 Overview

HSV is an alternative to RGB designed to align with human color perception. It separates color identity from brightness.

Three components:
1. **H (Hue)**: Color type — angle on the color wheel (0°–360°). Represents which color it is.
2. **S (Saturation)**: Purity of the color — how vivid vs. gray (0–100% or 0–1).
3. **V (Value)**: Brightness — 0% = completely black, 100% = full brightness.

**Geometry** (figure description): HSV is typically represented as an **inverted cone/cylinder**:
- Hue H is the angle around the central vertical axis (like spokes of a wheel).
- Saturation S is the distance from the center axis (center = gray, edge = full saturation).
- Value V is the height from bottom (dark/black) to top (bright).

#### 9.2 HSV in practice

Standard ranges:
- **Hue**: 0°–360°
- **Saturation**: 0–100%
- **Value**: 0–100%

In **OpenCV** (slightly different encoding):
- **H**: 0–179 (scaled from 360°, using 7 bits). Mapping: 0=Red, 30=Yellow, 60=Green, 120=Cyan, 150=Blue, 180=Red (wraps).
- **S**: 0–255 (8 bits). 0 = no saturation (grayscale); 255 = full saturation.
- **V**: 0–255 (8 bits). 0 = black; 255 = maximum brightness.

#### 9.3 Why use HSV?

- More **intuitive for humans**: we think of colors as "red, somewhat vivid, somewhat dark."
- **Separates chromatic content from brightness** — easy to adjust one without affecting others.
- **Better for color manipulation**: change hue without changing brightness; threshold on saturation to find vivid colors.
- Used in **image segmentation**, **color filtering**, and **color adjustments** in OpenCV.

---

### 10. RGB → HSV Conversion (Full Algorithm)

**Step 1**: Normalize RGB to $[0, 1]$:
$$r = \frac{R}{255}, \quad g = \frac{G}{255}, \quad b = \frac{B}{255}$$
**Step 2**: Compute Value:
$$V = \max(r, g, b)$$
**Step 3**: Compute Saturation:
- If $V = 0$ (color is black): $S = 0$.
- Otherwise: $S = \frac{V - \min(r, g, b)}{V}$

**Step 4**: Compute Hue:
- Let $\Delta = V - \min(r, g, b)$
- If $\Delta = 0$: $H = 0$ (achromatic).
- Otherwise:
$$H = \begin{cases} 0° + 60° \times \frac{g - b}{\Delta} & \text{if } V = r \text{ (dominant channel is red)} \\\\ 120° + 60° \times \frac{b - r}{\Delta} & \text{if } V = g \text{ (dominant channel is green)} \\\\ 240° + 60° \times \frac{r - g}{\Delta} & \text{if } V = b \text{ (dominant channel is blue)} \end{cases}$$
- If $H$ is negative, add 360° to bring into $[0°, 360°]$.

**Intuition**:
- $V = \max(r,g,b)$: how bright is the color? (dominated by brightest channel)
- $S$: how different are the channels from each other? Equal = gray = no saturation.
- $H$: which channel dominates, and by how much? Maps to an angle on the color wheel.

**RGB → HSV is not creating new information; it reorganizes color information into perceptual dimensions.**

#### Worked example: RGB(255, 0, 0) → HSV
$$\text{Normalize:} \quad r = 1,\; g = 0,\; b = 0$$
$$V = \max(1, 0, 0) = 1$$
$$S = \frac{1 - 0}{1} = 1$$
$$\Delta = 1 - 0 = 1$$
$$V = r, \text{ so } H = 0° + 60° \times \frac{0 - 0}{1} = 0°$$
**Result: HSV(0°, 100%, 100%) — pure red**

---

### 11. HSV → RGB Conversion (Full Algorithm)

A four-step process ($H$ in $[0°, 360°]$, $S$ and $V$ in $[0, 1]$):

**Step 1 — Input normalization**: ensure $H \in [0°, 360°]$, $S$ and $V \in [0, 1]$.

**Step 2 — Compute base values**:
$$C = V \times S$$
$$X = C \times \left(1 - \left|\left(\frac{H}{60°}\right) \bmod 2 - 1\right|\right)$$
$$m = V - C$$
**Step 3 — Assign pre-shift RGB based on hue sector**:

| Hue range | $(R', G', B')$ |
|---|---|
| $0° \leq H < 60°$ | $(C, X, 0)$ |
| $60° \leq H < 120°$ | $(X, C, 0)$ |
| $120° \leq H < 180°$ | $(0, C, X)$ |
| $180° \leq H < 240°$ | $(0, X, C)$ |
| $240° \leq H < 300°$ | $(X, 0, C)$ |
| $300° \leq H < 360°$ | $(C, 0, X)$ |

**Step 4 — Adjust to $[0, 255]$**:
$$R = (R' + m) \times 255, \quad G = (G' + m) \times 255, \quad B = (B' + m) \times 255$$

---

### 12. HSI and HSL Color Models

Both HSI and HSL are **perceptual color spaces** that separate chromatic components (Hue, Saturation) from brightness-related components, similar to HSV but differing in how brightness is computed.

**Geometry comparison**:
- **HSV**: cone shape — the top face is the full-saturation plane; apex is black.
- **HSI**: double-cone (bicone) shape — top apex = white, bottom apex = black, widest cross-section at the equator (mid-intensity). At I=0.75 the triangle is a light-colored triangle; at I=0.5 it is a vivid mid-tone triangle.
- **HSL**: similar bicone with lightness axis.

**How brightness is computed (key difference)**:

| Component | Formula | Interpretation |
|---|---|---|
| **V (HSV)** | $\max(R, G, B) / 255$ | Brightness from brightest channel |
| **L (HSL)** | $(\max + \min) / (2 \times 255)$ | Midpoint — perceptual brightness (closer to human vision) |
| **I (HSI)** | $(R + G + B) / (3 \times 255)$ | Average — overall intensity (better for grayscale) |

---

### 13. CMYK Color Model

**CMYK** = Cyan, Magenta, Yellow, Key (Black).

- **Subtractive** color model: colors are created by subtracting (absorbing) wavelengths from white light/background.
- Opposite of RGB (additive). In printing, inks subtract from white paper.
- Components:
  - **C (Cyan)**: absorbs red.
  - **M (Magenta)**: absorbs green.
  - **Y (Yellow)**: absorbs blue.
  - **K (Black)**: used to deepen colors (true black is hard to achieve with CMY alone; combining CMY gives muddy dark brown).
- Applications: color printing, publishing, graphic design.
- **Device-dependent**: varies with ink properties and paper.

---

### 14. CIE Lab Color Model

- **Perceptually uniform**: designed so that equal distances in Lab space correspond to approximately equal perceived color differences (unlike RGB and XYZ).
- **Device-independent**: based on human perception, not hardware.
- Components:
  - **L**: Lightness (0 = black, 100 = white).
  - **A**: Green–red axis (negative = green side, positive = red side).
  - **B**: Blue–yellow axis (negative = blue side, positive = yellow side).
- Applications: color correction, image processing, quality control (measuring color difference $\Delta E$).
- Conversion: Lab ↔ XYZ ↔ RGB.

---

### 15. YCbCr Color Model

- Used in **video compression**: JPEG, MPEG, video streaming, digital TV.
- Separates luminance from chrominance — exploits the fact that **humans are more sensitive to brightness than to color**.
- Components:
  - **Y**: Luminance (brightness) — the grayscale channel.
  - **Cb**: Blue chrominance (difference from blue channel).
  - **Cr**: Red chrominance (difference from red channel).
- Applications: efficient compression (compress chrominance more than luminance), broadcasting.

---

### 16. Color Space Comparison Table

| Color Space | Strengths | Weaknesses | Typical Use |
|---|---|---|---|
| **RGB** | Simple, matches hardware, no conversion | Not perceptual, brightness/color mixed, illumination-sensitive | Cameras, displays, raw processing |
| **HSV/HSI/HSL** | Intuitive, separates color from brightness, easy thresholding | Not perceptually uniform, hue unstable near gray, device-dependent | Color segmentation, tracking, CV heuristics |
| **CIE XYZ** | Device-independent, based on vision experiments, foundation of all spaces | Not perceptually uniform, not displayable, includes non-visible colors | Color definition, conversion, standards |
| **Chromaticity (x,y)** | Removes brightness, useful for color comparison | Loses luminance, not perceptually uniform | Illumination analysis, color consistency |
| **CIE Lab** | Approx. perceptually uniform, meaningful $\Delta E$ distances, separates L from chroma | Computationally heavier, requires XYZ conversion | Color difference, quality control, correction |
| **CMYK** | Matches subtractive printing process, efficient ink usage | Device-dependent, not perceptual, limited gamut | Printing, publishing, graphic arts |
| **YCbCr** | Efficient compression, separates brightness from color, matches human sensitivity | Not perceptual, not intuitive, slightly device-dependent | JPEG, MPEG, video, TV |

---

## Key terms (glossary)

- **Wavelength** — property of light determining color; visible range ~400–700 nm.
- **Tristimulus values (S, M, L)** — three numbers describing how strongly each cone class is stimulated; the signal sent to the brain.
- **Trichromacy** — color perception through three cone types; the biological basis of all three-primary color models.
- **Metamerism** — two physically different spectra that produce identical tristimulus values (S,M,L) and therefore look the same color. Color is many-to-one from spectrum to perception.
- **Hue** — the dominant wavelength / color identity (red, green, blue, etc.); angle 0°–360° on the color wheel.
- **Saturation** — purity of the hue; relative amount of white mixed in. 0 = gray, 1 = pure vivid color.
- **Brightness / Luminance / Lightness / Value / Intensity** — different measures of how light/dark a color is ($V = \max$ channel, $L =$ midpoint, $I =$ average).
- **Color model / Color space** — a coordinate system + subspace where each color is a single point.
- **RGB** — additive model; color as $(R,G,B) \in [0,255]^3$; cube geometry; device-dependent.
- **HSV** — Hue-Saturation-Value; cone geometry; perceptually intuitive; used for color manipulation.
- **HSI** — Hue-Saturation-Intensity; bicone geometry; $I = \text{average of RGB}$.
- **HSL** — Hue-Saturation-Lightness; $L = (\max+\min)/2$; closer to human brightness perception.
- **CIE XYZ** — device-independent reference space with virtual primaries; all matching functions non-negative; Y = luminance.
- **Chromaticity coordinates** — $(x, y) =$ normalized XYZ; removes luminance; maps color independent of brightness.
- **CIE 1931 chromaticity diagram** — horseshoe-shaped 2D plot of all human-perceivable colors in $(x,y)$ space; spectral locus on outer curved edge.
- **Spectral locus** — the curved outer boundary of the chromaticity diagram; represents pure monochromatic colors 400–700 nm.
- **Gamut** — the set of colors a device can reproduce; represented as a triangle in the chromaticity diagram.
- **CMYK** — subtractive model for printing; Cyan, Magenta, Yellow, Key(Black).
- **CIE Lab** — perceptually uniform space; L = lightness, a = green-red, b = blue-yellow.
- **YCbCr** — luminance + chrominance; used in video/image compression (JPEG, MPEG).
- **Perceptually uniform** — equal distances in color space correspond to equal perceived color differences (Lab approximates this; RGB and XYZ do not).
- **Color blindness (color vision deficiency)** — absence or malfunction of one or more cone types.
- **Protanopia** — missing L-cone; red-green confusion.
- **Tritanopia** — missing M-cone.
- **Monochromacy** — no color perception (no functioning cones or only one type).
- **Ishihara test** — standard test for color blindness using plates with hidden numbers in colored dots.

---

## Exam targets

1. **Explain trichromacy**: three cone types (S/M/L), peak sensitivities, proportions (2%, 33%, 65%), and the tristimulus integral equations. State what information is lost.

2. **Define metamerism** and explain why it exists (compression from infinite-dimensional spectrum to 3 numbers). State the implication: color alone is unreliable in computer vision.

3. **State the three color characteristics** (hue, saturation, brightness) and give their precise definitions. Know the formulas: $L = (M+m)/2$, $S$ formula for HSL, $H = 0$ if $R=G=B$.

4. **Describe the RGB cube** — corners and their colors, the diagonal grayscale axis, device-dependence, and the two key problems (limited gamut, perceptually non-linear).

5. **RGB → HSV conversion** — reproduce the full algorithm (normalize, $V = \max$, $S = (V-\min)/V$, $H$ piecewise formula) and the worked example for RGB(255,0,0) → HSV(0°,100%,100%).

6. **HSV → RGB conversion** — reproduce the four-step algorithm (compute $C$, $X$, $m$; assign sector; final adjustment).

7. **Explain CIE XYZ**: why virtual primaries, the RGB→XYZ matrix, what Y represents, why device-independent, and when to use it (define/convert, not display).

8. **Describe the CIE 1931 chromaticity diagram**: horseshoe shape, spectral locus, purple line, white point, gamut triangle. Explain what chromaticity coordinates $(x,y)$ are and how they're derived. State the key limitation (not perceptually uniform).

9. **Compare HSV, HSI, HSL**: how $V$, $L$, $I$ differ (max vs. midpoint vs. average).

10. **Complete color space comparison**: for each of RGB, HSV, CIE XYZ, chromaticity $(x,y)$, CIE Lab, CMYK, YCbCr — state strengths, weaknesses, and typical application.

11. **Color blindness types**: protanopia (missing L-cone), tritanopia (missing M-cone), monochromacy. Describe the Ishihara test.

---

## Pitfalls

- **The eye does NOT measure wavelength**: it measures three integrals (S, M, L). Students often confuse "wavelength = color" — this is only approximately true.
- **Metamerism means RGB alone is not a reliable color descriptor**: two physically different objects may produce the same $(R,G,B)$ under one light and different values under another.
- **HSV V ≠ luminance**: $V = \max(R,G,B)$ which is not a perceptual brightness; $L$ (HSL) and $I$ (HSI) are different from $V$ and from each other.
- **Hue is undefined when $S = 0$** (achromatic/gray pixels). Any $H$ value is valid for gray. Many algorithms break on this edge case.
- **CIE XYZ primaries are NOT real lights**: they are mathematical constructs. You cannot build a "pure X lamp." This is often confused with real physical primaries.
- **The chromaticity diagram is NOT perceptually uniform**: green occupies a disproportionately large area relative to perceptual differences; red and blue differences are compressed.
- **Standard observer = only 17 British subjects**: the CIE 1931 standard is an approximation of a very small and non-representative sample.
- **OpenCV HSV ranges differ from standard**: H is 0–179 (not 0–360), S and V are 0–255 (not 0–100%). Confusing these ranges causes incorrect color thresholding.
- **CMYK is subtractive, RGB is additive**: mixing all RGB primaries → white; mixing all CMY primaries → (muddy) black/dark. Never confuse the mixing models.
- **Negative RGB matching coefficients**: the CIE RGB matching function $\bar{r}(\lambda)$ goes negative near 450 nm — this is not an error, it reflects the real overlap of cone sensitivities.
- **Gamut is device-specific**: sRGB, Adobe RGB, and Apple RGB all have different gamuts; colors in one may be out-of-gamut in another. Always specify which RGB standard you are using.
