# ECE6255 Team 5's Term Project
## Table of Contents
1. [Introduction](#introduction)
1. [Procedure](#procedure)
    1. [Transimitter/Scrambling](#transmitterscrambling)
    1. [Receiver/Descrambling](#receiverdescrambling)


## Introduction
In speech communication, one way to maintain privacy of conversation is to employ a digital voice encoder, which turns the voice signal into a bit stream, and a digital encryption system, which encrypts the bit stream into another bit stream that cannot be decoded back into a useful bit stream without the encryption key. This is in the class of “secure communication” and requires a sophisticated encryption system to do the job right. In this challenge, we explore a much simpler type of secure voice communication, often called “voice scrambling”, which is only designed to make the voice not readily intelligible. The level of sophistication in voice scrambling is not very high and thus not very secure; however, for tactical or temporary privacy, it may suffice.
In voice scrambling, the spectrum of a voice signal is split by the scrambler into a number of bands, which are then permuted to form a scrambled spectrum according to some permutation pattern. At the descrambler, which knows the permutation matrix, the received signal undergoes the reverse of the band-splitting and permutation procedure to reconstruct an intelligible voice signal.
Band-splitting is executed by filtering and the degree of privacy of a voice scrambler depends on the number of bands that is implemented in it. However, since band-splitting filters can never be perfect (of the pure brick wall type), the number of bands is limited by the amount of sidelobes, which affects the quality of the reconstructed signal. The degree of privacy also depends on how often the permutation is altered. Without time-varying permutation, some people claim to be able to learn to understand the scrambled voice without de-permutation. Time-varying permutation of the speech spectrum nonetheless induces what is called bandwidth expansion, resulting in a loss of voice quality upon reconstruction, even with perfect knowledge of the permutation matrix. One way to reduce this type of degradation is to change the permutation scheme during pauses or gaps in a speech utterance.
This challenge aims at developing a voice scrambling system, both the transmitter/scrambler and the receiver/descrambler, that allows proper time variation of the spectrum permutation. The system should allow user to specify the number of spectral channels/bands for scrambling and a nominal, permissible rate of permutation change, say from 0 (fixed permutation) to 2 per second in fractional steps (e.g., 1/2, one change in permutation every two seconds, 2/5, two changes every 5 seconds, and so on).

## Procedure
### Transmitter/Scrambling
``` python
# Input arguments
sample_rate: float
samples: array-like  # audio to be scrambled
B: int  # number of filter bands
P_rate: str  # Permissible rate of permutations 

frame_len: float  # frame length in seconds (maybe milliseconds?)

# Output arguments
output: array-like  # scrambled audio
P: array-like  # array of permutation matrices
P_idx: array-like  # array of frame indices where permutation matrix changed
```

1. Divide vocal audio into frames of length $L$
1. Compute the permutation matrix $M$
1. For each frame:
    1. Check if pause or gap in speech utterance **AND** rate of permutation is obeyed
        1. Recompute the permutation matrix $M$
    1. Split frame's spectrum into $B$ bands using a FIR bandpass filter
    1. For each bandpass filter's contents $x_{B_i}$:
        1. Baseband contents to 0Hz
        1. Shift bandpassed contents according to permutation matrix slot $M_i$
        1. Recombine slots for transmission
1. Transmit fully scrambled audio (retain permutation matrices ($P$) and frame index of when matrix is changed ($P_{idx}$) for our verification, **DO NOT** transmit with scambled audio)

### Receiver/Descrambling
We can either compute the permutation matrix based on a psudeorandom number generator where both the transmitter and receiver start on the same seed or just use the output $P$ and $P_{idx}$ from the Transmitter 
Steps are same as Transmitter but reversed and permutation matrix slot is derived from the transpose $P^T$.

## Theoretical Foundation

### Audio Acquisition and Pre-Processing

Audio is captured at a **16 kHz sample rate** in mono (single channel), which is the standard for speech processing -- sufficient to capture frequencies up to 8 kHz (by Nyquist's theorem), covering the full range of human speech.

Two pre-processing steps are applied before feature extraction:

**Silence Trimming.** An energy-based voice activity detector segments the signal into 30 ms frames and computes the root-mean-square (RMS) energy of each frame:

$$E_i = \sqrt{\frac{1}{N} \sum_{n=1}^{N} x_i[n]^2}$$

where $x_i[n]$ is the $n$-th sample in frame $i$ and $N$ is the frame length. Frames with $E_i$ below a threshold $\tau = 0.01$ are classified as silence. Leading and trailing silence frames are removed, keeping a one-frame margin on each side.

**Pre-Emphasis Filtering.** A first-order high-pass filter boosts high-frequency content that is attenuated by the vocal tract:

$$y[n] = x[n] - \alpha \cdot x[n-1], \quad \alpha = 0.97$$

This flattens the spectral tilt inherent in voiced speech, improving the quality of subsequent spectral analysis.

### Mel-Frequency Cepstral Coefficients (MFCCs)

MFCCs are the most widely used feature representation in speech recognition. They model the short-time power spectrum of a signal on a perceptually motivated mel-frequency scale.

The extraction pipeline proceeds as follows:

1. **Framing**: The pre-emphasized signal is divided into overlapping frames of 25 ms (400 samples at 16 kHz) with a 10 ms hop (160 samples), corresponding to a standard configuration that captures the quasi-stationary nature of speech.

2. **Windowing and FFT**: Each frame is multiplied by a Hann window and transformed via a 512-point Short-Time Fourier Transform (STFT) to obtain the power spectrum.

3. **Mel Filterbank**: The power spectrum is passed through a bank of triangular bandpass filters spaced according to the mel scale:

$$m = 2595 \cdot \log_{10}\left(1 + \frac{f}{700}\right)$$

This scale approximates the human auditory system's frequency resolution -- high resolution at low frequencies, lower resolution at high frequencies.

4. **Log Compression**: The log of each filterbank energy is taken, modeling the approximately logarithmic loudness perception of the ear.

5. **Discrete Cosine Transform (DCT)**: The DCT of the log filterbank energies yields the cepstral coefficients. The first 13 coefficients are retained (C0 through C12), which capture the broad spectral shape while discarding fine harmonic detail.

6. **Delta and Delta-Delta Features**: First-order (velocity) and second-order (acceleration) temporal derivatives are appended to capture dynamic speech characteristics:

$$\Delta c_t = \frac{\sum_{k=1}^{K} k \cdot (c_{t+k} - c_{t-k})}{2 \sum_{k=1}^{K} k^2}$$

The delta-delta coefficients are computed identically over the delta sequence. This yields a **39-dimensional feature vector** per frame (13 static + 13 delta + 13 delta-delta), which is the standard configuration used in classical speech recognition systems.

### Dynamic Time Warping (DTW)

DTW is an algorithm for measuring the similarity between two temporal sequences that may vary in speed. It is particularly well suited for speech because speakers naturally vary in pronunciation rate -- the same word spoken quickly and slowly will have different sequence lengths but similar spectral trajectories.

Given two MFCC sequences $\mathbf{A} = (a_1, \ldots, a_M)$ and $\mathbf{B} = (b_1, \ldots, b_N)$ where each $a_i, b_j \in \mathbb{R}^{39}$, DTW finds the alignment path $W = (w_1, \ldots, w_L)$ that minimizes:

$$D(\mathbf{A}, \mathbf{B}) = \min_W \sum_{l=1}^{L} d(a_{w_l^a}, b_{w_l^b})$$

subject to the constraints:
- **Boundary**: $w_1 = (1,1)$ and $w_L = (M, N)$
- **Monotonicity**: $w_l^a \leq w_{l+1}^a$ and $w_l^b \leq w_{l+1}^b$
- **Continuity**: $w_{l+1} - w_l \in \{(1,0), (0,1), (1,1)\}$

The local distance $d(a_i, b_j)$ is the Euclidean distance between the two feature vectors. The optimal path is found via dynamic programming with $O(M \times N)$ time and space complexity.

The **normalized distance** (total path cost divided by path length $L$) is used to avoid bias toward shorter sequences:

$$\hat{D}(\mathbf{A}, \mathbf{B}) = \frac{D(\mathbf{A}, \mathbf{B})}{L}$$

### Classification by Minimum Average Distance

For each UID class $c$ with $K_c$ training templates $\{T_c^1, \ldots, T_c^{K_c}\}$, the score for an unknown utterance $U$ is the average normalized DTW distance:

$$S(c) = \frac{1}{K_c} \sum_{k=1}^{K_c} \hat{D}(U, T_c^k)$$

The predicted class is:

$$\hat{c} = \arg\min_c S(c)$$

Averaging over multiple templates per class provides robustness against intra-speaker variability (natural variations in pronunciation speed, emphasis, and loudness across repetitions).

