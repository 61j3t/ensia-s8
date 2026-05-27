# Chapter 7 — Short-Time Fourier Transform (STFT) & Spectrogram

## Bird's eye view

- **Why STFT?** The standard DFT/DTFT gives a single global spectrum for the entire signal — it tells you *what* frequencies exist but not *when*. Speech is non-stationary: the sound "sh-o-p" has completely different spectral content in each segment.
- **Core idea:** Slide a short analysis window w[n] along the signal and apply the DFT to each windowed segment independently. The result is a 2-D complex matrix S[m, k] indexed by frame m (time) and bin k (frequency).
- **Spectrogram = |STFT|²** — the squared magnitude of S[m, k], displayed as a 2-D heat-map with time on the x-axis, frequency on the y-axis, and colour/intensity showing energy.
- **Fundamental trade-off:** Long window → better frequency resolution, poor time resolution (narrowband spectrogram, horizontal striations at harmonics). Short window → better time resolution, poor frequency resolution (wideband spectrogram, vertical striations at pitch periods). Governed by the **uncertainty principle**: Δt · Δf ≥ 1/(4π).
- **Window type matters:** The rectangular window has the narrowest main lobe (best frequency resolution) but the highest sidelobes (worst spectral leakage). Hann, Hamming, Bartlett, and Blackman windows trade resolution for leakage suppression.
- **Practical speech parameters:** Frame size 20–40 ms, overlap 5–10 ms, Hamming window standard; wideband uses L ≈ 6 ms (96 samples at 16 kHz), narrowband uses L ≈ 60 ms (960 samples at 16 kHz).
- **Speech features visible in the spectrogram:** voiced/unvoiced regions, formant trajectories (wideband), individual harmonics and fundamental frequency (narrowband), consonant bursts, silence, and phoneme boundaries.

---

## Detailed notes

### 1. Context and motivation

#### 1.1 The DTFT and its limitation

The **Discrete-Time Fourier Transform (DTFT)** relates the time-domain sequence x[n] to its spectrum X(ω):

    X(ω) = sum over n from -inf to +inf of x[n] · e^{-jωn}

The DTFT characterises the *global* spectral composition of a signal. This is adequate for **stationary signals** (whose statistical properties do not change over time), but speech is fundamentally **non-stationary**:

- A speech record of several seconds contains silences, voiced vowels, unvoiced fricatives, stop consonants, transitions — each with entirely different frequency content.
- Zooming into two 20-ms windows taken at different positions in the same word reveals completely different waveforms (e.g., a smooth sinusoidal pattern in the vowel region vs. a noisy irregular pattern in a fricative region).
- Consequently, the DFT of the full signal averages over all these different spectral characters, losing all temporal information about *when* each frequency component occurs.

**Key problem illustrated:** Consider two signals both containing sinusoids at 1 Hz and 5 Hz. In the first, the two tones occur sequentially (one after the other); in the second, they are superposed simultaneously. Both produce *identical* magnitude spectra under a global DFT. The DFT cannot distinguish these cases. We know *what* but not *when*.

Similarly, the word "shop" contains three phonetically and spectrally distinct segments: "sh" (broadband noise energy concentrated above ~3 kHz), "o" (periodic vowel with clear harmonic structure and formants), and "p" (a burst transient). A single DFT over the whole word mixes all three.

#### 1.2 The STFT solution

**Solution: consider small segments of the signal and apply the FT locally.**

The idea is to treat each short segment as approximately stationary (quasi-stationarity assumption — valid for speech over windows of 20–40 ms) and compute its spectrum independently. Sliding this analysis window step-by-step through the signal produces a sequence of local spectra, one per frame, which together form the STFT.

**STFT intuition (visual):** Imagine placing a narrow rectangular box (red frame) at the start of a speech waveform. Extract the samples inside, compute their DFT to get a local spectrum, then shift the box rightward by the hop size H and repeat. Each position of the window yields one column of the spectrogram.

---

### 2. Windowing

#### 2.1 Concept

**Windowing** is the process of multiplying the signal x[n] by a window function w[n] to extract and taper a short segment. The window is non-zero only over a finite support of M+1 samples, forcing the signal to zero outside the analysis region.

For a rectangular window: w[n] = 1 for 0 ≤ n ≤ M, and w[n] = 0 elsewhere.

The windowed signal is: x_w[n] = w[n] · x[n]

Key terminology:
- **Window size (= frame size when no zero-padding):** the number of non-zero samples in the window, M+1.
- **Frame size:** the total number of samples in the analysis block (may be larger than window size if zero-padding is used).
- **Window size ≠ frame size** when **zero-padding** is applied — the window selects M+1 samples but the DFT is computed on a longer zero-padded block, increasing frequency sampling density (but not true resolution).

#### 2.2 Overlapping frames and hop size

When the window slides through the signal, consecutive frames typically overlap to ensure smooth temporal coverage.

- **Hop size H:** the number of samples the window shifts between successive frames.
- **Overlapping size = window size − hop size.**

**Visual:** Successive frames (m=1, m=2, m=3, …) are shown as the window box slides rightward in steps of H samples, with each new frame sharing (window size − H) samples with the previous one. A hop size of H/2 (50% overlap) is very common.

---

### 3. STFT definition

#### 3.1 From DFT to STFT

Recall the **N-point DFT**:

    X[k] = sum_{n=0}^{N-1} x[n] · e^{-j(2πkn/N)}

The **STFT** extends this by introducing a sliding window w[n − mH] centred at time frame m:

    S[m, k] = sum_{n=0}^{N-1} x[n] · w[n − mH] · e^{-j(2πkn/N)}

where:
- **m** = time (frame) index (yellow)
- **k** = frequency bin index (red)
- **N** = number of points used in the DFT (frame size)
- **w[n − mH]** = the window centred at sample mH, i.e., the window starting at frame m times hop size (green)
- **e^{-j(2πkn/N)}** = the DFT kernel (green)

**Interpretation:** For each time frame m, the STFT picks up N samples of x starting at position mH, multiplies them by the window, and applies the N-point DFT. The index m marches forward in steps of H samples through the signal (m=1, m=2, m=3 illustrated sequentially in the slides).

#### 3.2 Output dimensions

**DFT output:** a spectral vector of N complex Fourier coefficients (# frequency bins).

**STFT output:** a spectral **matrix** of dimensions (# frequency bins) × (# frames), where each column is the DFT of one frame.

The number of frequency bins and frames are computed as:

    # frequency bins = framesize / 2 + 1

    # frames = (samples − framesize) / hopsize + 1

**Worked example (from slides):**
- Signal = 10,000 samples; frame size = 1000; hop size = 500
- # frequency bins = 1000/2 + 1 = **501** (covering 0 to sampling_rate/2)
- # frames = (10000 − 1000)/500 + 1 = **19**

#### 3.3 Practical parameter choices

| Parameter | Typical values |
|---|---|
| Frame size | 512, 1024, 2048, 4096, 8192 samples |
| Hop size | 256, 512, 1024, 2048, 4096 (= 1/2, 1/4, 1/8 of frame size) |
| Window function | Hamming (default), Hann, Blackman, rectangular |

For **speech**: frame changes occur at 20–40 ms timescales, so target T = 20–40 ms with overlap 5–10 ms. Window size in ms: T = M × T_s, where T_s = 1/f_s is the sampling period.

---

### 4. Window functions

#### 4.1 Rectangular window and its spectrum

For the rectangular window w_R[n] = 1, 0 ≤ n ≤ M, the DTFT is:

    W_R(ω) = sum_{n=0}^{M} e^{-jωn}
            = e^{-jωM/2} · [ sin(ω(M+1)/2) / sin(ω/2) ]

The magnitude spectrum |W_R(ω)| has:
- A **main lobe** centred at ω = 0 with width Δω_m = 4π/(M+1)
- **Sidelobes** at lower amplitude; first zero crossing at ω = 2π/(M+1)
- Peak sidelobe level: **α = −13 dB** (independent of M)

As M increases, the main lobe narrows (better frequency resolution) but the sidelobe level stays constant at −13 dB.

#### 4.2 Time-frequency trade-off and spectral leakage

When the window is applied, two key spectral features govern its quality:

- **Main lobe width:** indicates frequency resolution. A **narrower** main lobe → better ability to distinguish close frequencies.
- **Sidelobe attenuation:** indicates leakage suppression. **Lower** sidelobes → less spectral leakage.

**Fundamental trade-off:**
- Narrow main lobe → higher sidelobes (example: rectangular window) → good frequency resolution but poor leakage suppression.
- Wide main lobe → lower sidelobes (example: Blackman window) → poor frequency resolution but better leakage suppression.

This trade-off is **unavoidable** — it is a direct consequence of the **uncertainty principle** in time-frequency analysis: you cannot simultaneously localise a signal perfectly in both time and frequency.

**Uncertainty principle:**  Δt · Δf ≥ 1/(4π)

The "blurring" in the STFT time-frequency plane can be visualised as disks of constant area: a tall narrow disk corresponds to good time resolution / poor frequency resolution; a wide flat disk corresponds to good frequency resolution / poor time resolution. The area of the disk is constant.

#### 4.3 Standard window functions (comparison at same length M)

All formulas below use w_R[n] as the rectangular window (defining the support):

**Bartlett (triangular) window:**

    w[n] = (1 − |2n/M − 1|) · w_R[n]

**Hann window:**

    w[n] = (1/2)(1 − cos(2πn/M)) · w_R[n]

**Hamming window:**

    w[n] = (0.54 − 0.46 cos(2πn/M)) · w_R[n]

**Blackman window:**

    w[n] = (0.42 − 0.5 cos(2πn/M) + 0.08 cos(4πn/M)) · w_R[n]

**Visual comparison (same M):** In the time domain, the rectangular window is flat (amplitude 1 throughout), while Bartlett is triangular, Hann and Hamming are bell-shaped (Hamming has a non-zero pedestal), and Blackman is the broadest bell. In the frequency domain, the rectangular window has the narrowest main lobe but the highest visible sidelobes (around −13 dB). The Hann, Hamming and Blackman windows progressively increase the main lobe width while suppressing sidelobes to much lower levels (Blackman achieves around −60 dB sidelobe attenuation), which is critical for detecting weak spectral components near strong ones.

| Window | Main lobe width | Peak sidelobe level | Tradeoff |
|---|---|---|---|
| Rectangular | Narrowest | −13 dB | Best freq. res., worst leakage |
| Bartlett | Wider | Better | Intermediate |
| Hann | Wider | ~−32 dB | Good balance |
| Hamming | Wider | ~−43 dB | Commonly used default |
| Blackman | Widest | ~−58 dB | Best leakage, worst freq. res. |

---

### 5. The spectrogram

#### 5.1 Definition

The **spectrogram** Y[m, k] is defined as the squared magnitude of the STFT:

    Y[m, k] = |S[m, k]|²

It represents the **power spectral density** (energy per frequency bin) at each time frame. It is a real-valued, non-negative 2-D function.

In decibels (the standard display form):

    Y[m, k] (dB) = 10 log₁₀ Y[m, k]
                 = 20 log₁₀ |S[m, k]|

**Construction (visual from slides):** The procedure is:
1. Slide the short-time window (frame m=1, m=2, m=3, …).
2. For each frame, compute the magnitude spectrum |S[m,k]|.
3. Stack these spectra as vertical columns side by side.
4. The result is a 2-D grey-scale (or colour) image: x-axis = time (frames), y-axis = frequency (kHz), pixel intensity = energy in dB.

#### 5.2 DTFT vs. STFT illustration

For a frequency-modulated signal (chirp — a sine of constant amplitude but linearly increasing frequency):
- The **DTFT** gives a single, broad spectrum showing all the frequencies present, but with no temporal information about when the frequency was low vs. high.
- The **STFT / spectrogram** shows a diagonal dark stripe rising from bottom-left to top-right: at early times, the dominant frequency is low; at later times, it is high. This is invisible in the global DFT.

#### 5.3 Spectrogram examples

**Example — sea lions:** The time waveform shows complex bursting calls over 40 seconds. The global spectrum (DFT) shows energy concentrated below ~200 Hz. The magnitude spectrogram (linear) is nearly all dark because most energy is at low frequencies and the dynamic range overwhelms detail. The **dB spectrogram** reveals rich harmonic structure: horizontal bands of energy corresponding to harmonics of f₀, with energy bursting and decaying in a periodic rhythmic pattern across the full time axis.

**Example — speech ("This is a test", male, F_s = 16 kHz):**
- Waveform shows alternating voiced (high amplitude, periodic) and unvoiced (noisy) segments with silence gaps.
- **Wideband spectrogram** (L = 6 ms, 96 samples, Hamming, N = 512, R = 10 samples, 2250 frames): shows broad formant bands (F1, F2, F3) as smooth coloured horizontal bands; vertical striations appear at individual pitch periods; fine harmonic structure not resolved; transients and fricatives clearly localised in time.
- **Narrowband spectrogram** (L = 60 ms, 960 samples, Hamming, N = 1024, R = 96 samples, 235 frames): horizontal striations at each harmonic multiple of f₀; fundamental frequency and all harmonics clearly resolved; time localisation poor (formant transitions are blurred); voiced vs. unvoiced regions still visible.

**Example — speech ("She had your dark suit in"):**
- Narrowband spectrogram (L = 800 samples ≈ 50 ms, 1024-point FFT): resolves individual harmonics as horizontal lines; spectrum slices at /IY/, /AE/, and /S/ show: /IY/ — harmonic peaks at ~1 kHz with a large peak near 2 kHz; /AE/ — strong low-frequency harmonics with envelope peaks indicating formants; /S/ — flat noisy spectrum concentrated at high frequencies (above ~4 kHz), no harmonic structure.
- Wideband spectrogram (L = 80 samples ≈ 5 ms, 1024-point FFT): smooth formant bands; spectrum slices show the spectral envelope (sound envelope) as a smooth curve over the harmonics.

**Example — "Matlab" pronunciation (f_s = 7418 Hz):**
- Short window (100 samples, T = 13.5 ms, overlap 90, frame 128): well-resolved formant arcs visible; individual phonemes distinguishable in time; some harmonic blurring.
- Medium window (300 samples, T = 40.4 ms, overlap 250, frame 512): clearer harmonic structure; smooth formant arcs; better balanced.
- Long window (40 samples, T = 5.4 ms, overlap 20, frame 128): time resolution very fine; formants smeared in frequency; horizontal structure disappears.

---

### 6. The role of window length in the spectrogram

#### 6.1 Fundamental frequency and harmonics

For voiced speech, the signal has a **fundamental frequency f₀** (pitch) and harmonics:

    f_n = n · f₀    (nth order harmonic)

The magnitude spectrum of the STFT at a voiced frame shows a series of peaks at f₀, 2f₀, 3f₀, … modulated by the **vocal tract envelope** (the smooth curve connecting the harmonic peaks, corresponding to the filter/formant structure).

Noise-free sounds (intonation, voiced vowels, melodic instruments) exhibit this clean harmonic comb structure. Fricatives and unvoiced sounds do not — they show a noisy broadband spectrum.

**Pronunciation example (three phonemes I, A, S):**
- /I/ and /A/: clear harmonic peaks at low frequencies, smooth envelope; energy concentrated below 4 kHz.
- /S/: no harmonic structure; energy concentrated at high frequencies (broadband noise above ~4 kHz), characteristic of a fricative.

#### 6.2 Wide window → Narrowband spectrogram

When the analysis window is **wide** (long duration, many samples):
- Many pitch periods fit inside the window.
- The DFT has high frequency resolution: individual harmonics at f₀, 2f₀, 3f₀ are resolved as separate spectral lines.
- The window captures many signal cycles, smearing time variations → **poor time resolution**.
- In the spectrogram: **horizontal striations** — each harmonic appears as a horizontal band; the fundamental and its overtones are separately visible.
- The narrowband spectrogram can show pitch frequency directly.

**Wide window ⟺ Narrowband spectrogram** (good frequency resolution, poor time resolution)

Narrowband spectrogram parameters (example): L = 800 samples (50 ms), 1024-point FFT, overlap = 890 samples.

#### 6.3 Short temporal window → Wideband spectrogram

When the analysis window is **short** (brief duration):
- Only a fraction of a pitch period or a single pitch period fits inside.
- The DFT has poor frequency resolution: harmonics are not resolved; the spectrum shows a smooth spectral envelope.
- Few samples → much better temporal tracking of signal changes → **good time resolution**.
- In the spectrogram: **vertical striations** — each glottal pulse (one pitch period) appears as a vertical line or stripe; individual pitch periods resolved in time.
- Smoothing effect: the short window smooths the harmonic structure, revealing the vocal tract formant envelope.

**Short window ⟺ Wideband spectrogram** (good time resolution, poor frequency resolution)

Wideband spectrogram parameters (example): L = 80 samples (5 ms), 1024-point FFT, overlap = 75 samples.

**Practical guideline for speech:** speech spectral characteristics change on a 20–40 ms timescale, so the analysis window T should target 20–40 ms. Overlap should be 5–10 ms. Window size in samples: M = T × f_s.

#### 6.4 Side-by-side comparison

**Test signal:** two sine functions at 400 Hz and 450 Hz, plus two impulse spikes at t = 0.45 s and t = 0.5 s.

| | Wide window (128 ms) | Short window (32 ms) |
|---|---|---|
| 400/450 Hz sines | Clearly resolved as two separate horizontal bands | Merged into one broad band (not resolved) |
| Impulses at 0.45 s and 0.5 s | Impulses smeared into broad vertical blobs (poor time localisation) | Impulses clearly separated as sharp vertical lines |

This is the clearest illustration of the trade-off: no single window length resolves both the closely-spaced tones AND the closely-spaced impulses simultaneously.

#### 6.5 Wideband vs. narrowband features summary

| Feature | Wideband spectrogram | Narrowband spectrogram |
|---|---|---|
| Window length | Short (5–10 ms) | Long (40–100 ms) |
| Time resolution | High | Low |
| Frequency resolution | Low | High |
| Voiced regions | Vertical striations at pitch periods | Horizontal striations at harmonics |
| Formants | Broad bands visible | Still visible as envelope |
| Harmonic structure | Not resolved (smoothed away) | Individual harmonics resolved |
| Fundamental f₀ | Not directly readable | Directly readable |
| Unvoiced regions | No vertical striations | No strong harmonic structure |

**Wideband spectrogram:** follows broad spectral peaks (formants) over time; resolves individual pitch periods as vertical striations (since the impulse response of the analysing filter is comparable in duration to a pitch period); for unvoiced speech there are no vertical pitch striations.

**Narrowband spectrogram:** individual harmonics resolved in voiced regions; formant frequencies still evident as regions of high harmonic energy; usually can see f₀ directly; unvoiced regions show no strong structure.

**Note:** The condition determining a good window length depends on the Nyquist criterion (f_s).

---

### 7. The uncertainty principle

The time-frequency uncertainty principle states:

    Δt · Δf ≥ 1/(4π)

where Δt is the time localisation uncertainty and Δf is the frequency localisation uncertainty. Their product is always bounded below — you cannot make both arbitrarily small simultaneously.

**Consequences:**
- Increasing frame size → better Δf → worse Δt (the "blurring disk" in time-frequency space becomes wider in time and narrower in frequency).
- Decreasing frame size → better Δt → worse Δf.
- The area of the resolution cell in the time-frequency plane is **constant** — only its shape changes.

This is the fundamental reason why narrowband and wideband spectrograms each reveal complementary aspects of the signal and why no single spectrogram can show both fine harmonic structure and fine temporal detail.

---

## Key terms (glossary)

- **STFT (Short-Time Fourier Transform):** A DFT applied to overlapping windowed segments of a signal, producing a 2-D complex time-frequency representation S[m, k].
- **Spectrogram:** The squared magnitude of the STFT, Y[m, k] = |S[m, k]|², displayed as a 2-D energy map.
- **Non-stationary signal:** A signal whose spectral content changes over time (speech is a prime example).
- **Quasi-stationarity:** The assumption that speech is approximately stationary over short intervals (20–40 ms), which justifies STFT analysis.
- **Window function w[n]:** A finite-duration weighting sequence used to extract and taper a local segment of the signal before DFT.
- **Frame size (N):** Number of samples in one DFT analysis block (including any zero-padding).
- **Window size (M+1):** Number of non-zero samples of the window.
- **Hop size (H):** Number of samples the window advances between consecutive frames. Overlap = window size − hop size.
- **Zero-padding:** Appending zeros to a windowed frame to make its length N > M+1, which increases frequency-bin density without improving true resolution.
- **Main lobe:** The central peak of the window's frequency-domain magnitude spectrum; its width determines frequency resolution.
- **Sidelobe:** Secondary peaks of the window spectrum; their level determines spectral leakage.
- **Spectral leakage:** Energy from one frequency component "leaking" into adjacent bins due to the finite window — reduced by tapered windows (Hann, Hamming, Blackman).
- **Narrowband spectrogram:** Spectrogram computed with a long window; shows resolved harmonics as horizontal striations.
- **Wideband spectrogram:** Spectrogram computed with a short window; shows resolved pitch periods as vertical striations and smooth formant bands.
- **Fundamental frequency f₀:** The lowest harmonic of a periodic voiced sound; harmonics are at f_n = n·f₀.
- **Formants:** Resonance peaks of the vocal tract filter, visible as broad horizontal bands in the wideband spectrogram.
- **Frequency bin:** One output sample of the DFT corresponding to frequency k·F_s/N.
- **Uncertainty principle:** Δt · Δf ≥ 1/(4π) — the inescapable lower bound on the product of time and frequency localisation.
- **Chirp:** A signal with constant amplitude and linearly increasing frequency — a classic STFT demonstration signal (shows a diagonal stripe in the spectrogram).
- **Hamming window:** w[n] = (0.54 − 0.46 cos(2πn/M))·w_R[n]; the standard default for speech analysis.
- **Hann window:** w[n] = (1/2)(1 − cos(2πn/M))·w_R[n]; good leakage suppression.
- **Blackman window:** w[n] = (0.42 − 0.5 cos(2πn/M) + 0.08 cos(4πn/M))·w_R[n]; best leakage suppression, widest main lobe.

---

## Exam targets

1. **Explain why the standard DFT/DTFT is insufficient for speech analysis.** Use the non-stationarity argument and the classical counterexample of two signals with identical spectra but different temporal arrangement.

2. **Write and annotate the STFT formula** S[m, k] = Σ x[n]·w[n−mH]·e^{−j2πkn/N}. Identify each term (time index m, frequency index k, hop size H, window w, DFT kernel) and explain what the summation is computing.

3. **Derive the output dimensions of the STFT matrix:**
   - # frequency bins = framesize/2 + 1
   - # frames = (total samples − framesize)/hopsize + 1
   
   Apply to a numerical example (e.g., 10K samples, frame=1000, hop=500 → 501 bins, 19 frames).

4. **State and apply the uncertainty principle** Δt·Δf ≥ 1/(4π). Explain what it means for the STFT and why narrowband and wideband spectrograms cannot simultaneously achieve both resolutions.

5. **Give the formulas for at least three window functions** (rectangular, Hann, Hamming, Blackman/Bartlett). Compare them in terms of main lobe width and sidelobe attenuation.

6. **Define the spectrogram** Y[m,k] = |S[m,k]|². Explain the dB conversion (10 log₁₀ Y or 20 log₁₀ |S|) and why dB display is necessary in practice.

7. **Contrast narrowband and wideband spectrograms:** give the identifying visual feature of each (horizontal vs. vertical striations), the window length used, and what information each reveals about voiced speech (harmonics vs. formants/pitch periods).

8. **Explain the windowing trade-off** (main lobe width vs. sidelobe level), and why the rectangular window's −13 dB sidelobe is independent of M. State why Hamming is preferred in speech.

9. **Given a set of STFT parameters** (f_s, window type, L in ms or samples, hop size), compute: T in ms, overlap in samples, # frequency bins, # frames, and state whether the result is wideband or narrowband. Apply to the "This is a test" example (wideband: L=6 ms, 96 samples; narrowband: L=60 ms, 960 samples).

10. **Describe what phonemes look like in a spectrogram**: voiced vowel (horizontal harmonic bands + formant envelope), fricative /S/ (broadband noise at high frequencies), stop consonant (silence + burst transient).

---

## Pitfalls

- **Spectrogram ≠ STFT.** The STFT S[m,k] is complex. The spectrogram Y[m,k] = |S[m,k]|² is real and discards phase. You cannot recover the signal from the spectrogram alone (phase information is lost).

- **"Narrowband" refers to the analysis bandwidth, not signal bandwidth.** A narrowband spectrogram uses a *long* window → *narrow* analysis bandwidth → resolves harmonics. The naming is counterintuitive: long window = narrowband, short window = wideband.

- **Increasing frame size improves frequency resolution, not zero-padding.** Zero-padding increases the density of frequency samples (smoother-looking spectrum) but does not reveal any new spectral information. True frequency resolution is set by the window length M, not N.

- **The rectangular window's sidelobe level (−13 dB) is independent of M.** Increasing M narrows the main lobe but does not reduce the sidelobe level. Other window types are needed to suppress leakage.

- **Window size ≠ frame size.** Window size is M+1 (non-zero samples of w[n]). Frame size is N (samples fed into the DFT). They are equal without zero-padding, but differ when zero-padding is applied.

- **Overlap is not hop size.** Overlap = window size − hop size. A hop size H = window_size/2 gives 50% overlap, meaning each frame shares half its samples with the previous one.

- **The uncertainty principle applies to the STFT, not just quantum mechanics.** Δt·Δf ≥ 1/(4π) is a purely mathematical consequence of Fourier analysis; it is not a technological limitation — it cannot be circumvented by better hardware or algorithms.

- **Horizontal striations → narrowband (long window); vertical striations → wideband (short window).** Students often confuse which visual pattern corresponds to which window length. Mnemonic: long window = more periods = harmonic resolution = horizontal lines.

- **The good window length for speech depends on f_s (Nyquist).** The rule T = M × T_s = M/f_s ties the window duration in milliseconds to the sampling rate. At f_s = 16 kHz, 6 ms corresponds to 96 samples; at f_s = 8 kHz, 6 ms is only 48 samples.

- **Phase of S[m,k] is discarded in the spectrogram.** This matters for signal reconstruction (e.g., in vocoding, phase must be retained or estimated separately).
