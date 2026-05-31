# Chapter 1 — Overview of Digital Signal Processing

**Course:** Signals, Systems and Transforms — Dr Nafaa NACEREDDINE (ENSIA)

---

## Bird's eye view

- A **signal** is anything that conveys information; it can be 1-D (speech, ECG), 2-D (image), or 3-D (wind field). Three types exist: continuous-time $x(t)$, discrete-time $x(nT)$, and digital/quantized $x_Q(nT)$.
- A **system** is a mathematical model that maps an input signal to an output signal; a system that processes digital signals is called a **digital filter** or **digital signal processor**.
- **Processing** = passing a signal through a system to perform a function. Two paradigms: (i) analog processing of analog signal; (ii) **digital processing of analog signal** via ADC → DSP → DAC chain.
- DSP wins over analog processing on flexibility, reliability, cost, security, and simplicity (only additions and multiplications needed).
- Key course convention: **discrete-time signal = digital signal** (quantizer assumed to have infinite resolution); we always work with $x_Q(nT)$.
- DSP application domains span speech (compression, synthesis, recognition, enhancement), audio, image/video, communications, biomedical, bioinformatics, and finance.

---

## Detailed notes

### 1. Signal

**Definition:** Anything that conveys *information*.

**Examples given in lecture:**
- Speech, Electrocardiogram (ECG), Radar pulse, DNA sequence, Stock price, Image, Video

**Signal dimensionality:**
- 1-D: speech (function of time $t$)
- 2-D: image (function of two spatial variables)
- 3-D: wind (function of latitude, longitude, elevation)

**Three types of time-domain signals:**

| Type | Notation | Time | Amplitude |
|---|---|---|---|
| Continuous-time (analog) | $x(t)$ | Continuous range | Any real value |
| Discrete-time | $x(nT)$ | Discrete instants $t = \ldots, -T, 0, T, 2T, \ldots$ | Any real value |
| Digital (quantized) | $x_Q(nT)$ | Discrete instants only | Confined to a **finite set of numbers** |

Where $T$ is the **sampling period** and $n$ is an integer index.

**Fig. 1.1 (Speech waveform):** Shows the vowel of "a" plotted over approximately 22 ms. The waveform is quasi-periodic with amplitude roughly between -0.6 and +0.85, illustrating that speech is a 1-D analog signal of time.

**Fig. 1.2 (ECG waveform):** Shows ~2.6 seconds of ECG data with three sharp QRS peaks (amplitude ~240) separated by about 0.8 s, superimposed on lower-amplitude P and T waves. Illustrates a biomedical 1-D signal.

**Fig. 1.3 (Radar pulse pair):** Upper plot shows a clean transmitted sinusoidal burst (zero after t ≈ 0.45). Lower plot shows the received signal — a delayed, noise-corrupted version of the transmitted pulse, delayed by $\tau$ (round-trip time to target). Used to estimate target distance.

**Fig. 1.4 (Relationships between $x(t)$, $x(nT)$, and $x_Q(nT)$):**

The block diagram shows the ADC pipeline:

```
x(t)          sample at t=nT        quantizer          x_Q(nT)        Digital
[analog] ——————————————————> x(nT) ——————————————> [quantized] ——> Signal
signal                    [sampled]                  signal         Processor
```

Three waveforms below illustrate:
1. $x(t)$: smooth continuous curve (time and amplitude both continuous)
2. $x(nT)$: impulse train at t = 0, T, 2T, ... with continuous amplitude values (time discrete, amplitude continuous)
3. $x_Q(nT)$: impulse train at same instants but amplitudes snapped to nearest integer level (both time and amplitude discrete)

**Quantization worked example (4-bit representation):**

- At $n = 0$: $x(0)$ is close to 2, so $x_Q(0) = 2$ → binary: `0010`
- At $n = 1$: $x(T) \in (3, 4)$, so $x_Q(T) = 3$ → binary: `0011`

With 4-bit two's complement, $x_Q(nT)$ is restricted to integers in $[-8, 7]$.

**Course convention:** Throughout the course, **discrete-time signal = digital signal**. This is equivalent to assuming the quantizer has **infinite resolution** (no quantization error). We always work with $x_Q(nT)$.

---

### 2. System

**Definition:** A mathematical model or abstraction of a physical process that **relates input to output**.

Any system that processes digital signals is called a:
- **Digital system**
- **Digital filter**
- **Digital (signal) processor**

**Examples of systems:**

| System | Input | Output |
|---|---|---|
| Grading system | Coursework + exam marks | Grade |
| Squaring system | 5 | 25 |
| Amplifier | $\cos(\omega t)$ | $10\cos(\omega t)$ |
| Communication system (mobile phone) | Voice | CDMA signal |
| Noise reduction system | Noisy speech | Noise-reduced speech |
| Feature extraction system | $\cos(\omega t)$ | $\omega$ (the frequency) |

---

### 3. Processing

**Definition:** Performing a particular function by passing a signal through a system.

Two processing paradigms:

**Fig. 1.5 — Analog processing of analog signal:**
```
analog input ——> [analog signal processor] ——> analog output
```
The entire chain stays in the continuous/analog domain.

**Fig. 1.6 — Digital processing of analog signal:**
```
analog input ——> [ADC] ——> [digital signal processor] ——> [DAC] ——> analog output
```
The real-world analog signal is first converted to digital (ADC = Analog-to-Digital Converter), processed digitally, then converted back (DAC = Digital-to-Analog Converter). This is the dominant modern approach.

---

### 4. Advantages of DSP over Analog Signal Processing

| Advantage | Detail |
|---|---|
| **Flexibility** | DSP operations can be reconfigured simply by changing the program (software-defined) |
| **Development ease** | Can develop and simulate using PC tools, e.g., MATLAB |
| **Reliability** | Processing of 0s and 1s is almost immune to noise; data stored without deterioration |
| **Lower cost** | Enabled by advances in VLSI (Very Large Scale Integration) technology |
| **Security** | Encryption and scrambling can be introduced in the digital domain |
| **Simplicity** | Core operations are only **additions** and **multiplications** |

---

### 5. DSP Application Areas

#### 5.1 Speech
- **Compression:** Linear Predictive Coding (LPC) is a standard coding scheme for compressing speech data
- **Synthesis:** Computer production of speech signals, e.g., text-to-speech (Microsoft TTS engine)
- **Recognition:** Automatic telephone number enquiry systems, voice assistants
- **Enhancement:** Noise reduction for noisy speech signals

#### 5.2 Audio
- **Compression:** MP3 is the coding standard for compressing audio data
- **Generation:** Synthesis of musical instrument sounds (piano, cello, guitar, flute) by computer
- **Production:** Songs with low-cost electronic piano keyboard quality
- **Transcription:** Automatic music transcription (writing a piece of music from a recording)

#### 5.3 Image and Video
- **Compression:** JPEG (image) and MPEG (video) are the coding standards
- **Recognition:** Face recognition, palm recognition, fingerprint recognition
- **Enhancement:** Image quality improvement
- **3-D reconstruction:** Construction of 3-D objects from 2-D images

#### 5.4 Communications
- Encoding and decoding of digital communication signals (e.g., channel coding, modulation)

#### 5.5 Astronomy
- Finding the periods of orbits from time-series measurements

#### 5.6 Biomedical Engineering
- Medical care and diagnosis
- Analysis of ECG (electrocardiogram), EEG (electroencephalogram), NMR (nuclear magnetic resonance) data

#### 5.7 Bioinformatics
- DNA sequence analysis
- Extracting, processing, and interpreting information in genomic and proteomic data

#### 5.8 Finance
- Market risk management
- Trading algorithm design
- Investment portfolio analysis

---

## Key terms (glossary)

- **Signal** — Anything that conveys information; a function of one or more independent variables.
- **Continuous-time signal $x(t)$** — Defined for all values of continuous time $t$; amplitude takes any value.
- **Discrete-time signal $x(nT)$** — Defined only at discrete instants $t = nT$ ($n$ integer, $T$ = sampling period); amplitude still continuous.
- **Digital signal $x_Q(nT)$** — Both time and amplitude are discrete; amplitude confined to a finite set of numbers (quantized).
- **Sampling period $T$** — Time interval between successive samples; inverse is the sampling frequency $f_s = 1/T$.
- **Quantization** — Process of mapping a continuous amplitude to the nearest value in a finite discrete set; introduces quantization error.
- **System** — A mapping from an input signal to an output signal; a mathematical model of a physical process.
- **Digital filter / Digital processor** — A system that processes digital signals.
- **ADC (Analog-to-Digital Converter)** — Samples and quantizes an analog signal to produce a digital signal.
- **DAC (Digital-to-Analog Converter)** — Reconstructs an analog signal from a digital one.
- **DSP (Digital Signal Processing)** — The study and application of systems that process digital signals.
- **LPC (Linear Predictive Coding)** — A DSP-based speech compression technique.
- **VLSI (Very Large Scale Integration)** — IC fabrication technology enabling low-cost digital processors.
- **Two's complement** — Binary number representation used for signed integers; with 4 bits, range is $[-8, 7]$.

---

## Exam targets

1. **Classify a given signal** as continuous-time, discrete-time, or digital. State the key difference: time discreteness vs. amplitude discreteness vs. both.
2. **Draw and label Fig. 1.4** (the ADC pipeline block diagram) from memory: analog → sample → quantize → digital processor. Annotate each signal with its correct notation ($x(t)$, $x(nT)$, $x_Q(nT)$).
3. **Explain the course convention** that discrete-time = digital (infinite-resolution quantizer assumption) and why this simplification is made.
4. **Work through a quantization example**: given a sampled value and a bit width, determine the quantized output in decimal and binary (two's complement).
5. **Compare analog processing vs. digital processing of analog signals** using the block diagrams (Fig. 1.5 vs. Fig. 1.6). Know why the ADC→DSP→DAC chain is preferred.
6. **List and justify the 6 advantages of DSP** over analog processing (flexibility, ease of development, reliability, lower cost, security, simplicity).
7. **Give two concrete DSP application examples** for each domain: speech, audio, image/video, biomedical, communications, bioinformatics, finance.

---

## Pitfalls

- **Discrete-time ≠ digital in general.** Discrete-time means only time is discretized; digital means both time AND amplitude are discretized. The course treats them as equivalent (infinite-resolution assumption) — state this assumption explicitly in an exam answer.
- **$x(nT)$ vs $x[n]$:** The lecture uses $x(nT)$ with explicit sampling period $T$. Later chapters may drop $T$ and write $x[n]$. These are the same signal under the convention that $n$ indexes samples.
- **Quantization range with b bits (two's complement):** Range is $[-2^{b-1}, 2^{b-1} - 1]$. For b=4: $[-8, 7]$. Do not confuse with unsigned range $[0, 2^b - 1]$.
- **The ADC chain order matters:** Sampling comes before quantization. A sampled-but-not-quantized signal $x(nT)$ is NOT yet digital.
- **LPC is for speech compression, MP3 is for audio compression.** These are distinct standards and distinct application areas — do not conflate them.
- **A digital system is also called a digital filter** even if it performs recognition or feature extraction — the term "filter" is used generically for any digital processor.
