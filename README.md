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

