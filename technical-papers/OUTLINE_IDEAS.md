# Technical Papers: Audio Engineering & DSP

This folder contains formal technical papers and whitepapers bridging theoretical digital signal processing (DSP) and systems engineering with practical C++ production implementation.

## Potential Paper Outlines for Discussion

### 1. Lock-Free Synchronization Paradigms in Real-Time Audio Graphs
**Focus:** The intersection of OS thread scheduling and hard real-time deadlines in audio processing.
**Key Sections:**
- The mathematical constraints of the audio callback (latency, buffer sizes).
- Why standard synchronization primitives (Mutex, Semaphores) violate real-time safety via Priority Inversion.
- Lock-free architectures: Ring buffers (FIFOs) for stream data and atomic pointers for state/parameter updates.
- Double-buffering plugin state changes to avoid allocation and audio-thread stalling.

### 2. Mitigating CPU Spikes via State Ramping and Subnormal Prevention in C++ DSP
**Focus:** Handling mathematical edge cases in floating-point audio processing that cause artifacts and CPU overload.
**Key Sections:**
- The physics of audio artifacts: Why instantaneous signal discontinuities cause broadband frequency splatter (clicks/pops).
- Ramping strategies for parameters, wet/dry mixes, and bypass states (linear vs. multiplicative smoothing).
- The Denormal (Subnormal) problem in IIR filters and feedback loops: when floating-point math switches from hardware to microcode.
- CPU-level mitigation: Utilizing Flush-To-Zero (FTZ) and Denormals-Are-Zero (DAZ) flags via modern RAII patterns (e.g., JUCE's `ScopedNoDenormals`).

### 3. Continuous Latency Compensation in Dynamic Topological Audio Graphs
**Focus:** Maintaining phase coherence in non-linear processing paths.
**Key Sections:**
- The necessity of Automatic Delay Compensation (ADC) in modern DAWs.
- Modeling the audio graph as a Directed Acyclic Graph (DAG).
- Topological sorting and computing the critical path (longest latency path).
- Dynamically resizing delay buffers on non-critical paths without interrupting the audio stream.

---
*Add your notes, thoughts, and new topic ideas below!*
