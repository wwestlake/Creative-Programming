# Why Can't I Allocate Memory Inside an Audio Callback in C++?

You wrote what looks like totally reasonable C++ — a `new` here, a `delete` there — and now your audio is glitching, crackling, or dropping out entirely. Sometimes it's fine. Sometimes it isn't. That's the worst kind of bug, and the heap is almost certainly the culprit.

## The Hard Real-Time Deadline Nobody Warned You About

The audio callback — `processBlock()` in JUCE, `AudioUnitRender` in CoreAudio, whatever your framework calls it — is not a normal function call. The operating system invokes it on a dedicated, high-priority thread at a fixed interval determined by your buffer size and sample rate.

At 44100 Hz with a 512-sample buffer, you have roughly **11.6 ms** to finish and return. At 128 samples, that's **2.9 ms**. At 64 samples (common in pro-audio), **1.45 ms**.

Miss that deadline and the audio driver either plays silence or repeats the last buffer. That's the glitch you hear.

The callback isn't "try your best." It's a hard contract. Every line of code in that function must finish in bounded, predictable time — and `malloc`/`new` cannot promise you that.

## What malloc Actually Does (That Can Destroy Your Deadline)

Calling `new` or `malloc` looks like a simple memory operation. Internally it can do any of the following:

1. **Take a heap lock.** The C runtime heap is shared across all threads. Before it can hand you a chunk of memory, it acquires a mutex. If *any other thread* — your GUI, a plugin scanner, a logger, a garbage collector — is currently allocating or freeing memory, your audio thread blocks on that mutex. For an unpredictable duration.

2. **Ask the OS for more pages.** When the heap has no free block large enough, it calls `sbrk()` or `VirtualAlloc()` — a syscall — to request more virtual memory from the OS. Syscalls can take microseconds to milliseconds. The OS can also decide to run another process first.

3. **Trigger a compaction or sweep.** Some allocators (and languages with GCs running in native plugins) occasionally consolidate free blocks. This is not a constant-time operation.

4. **Touch cold memory pages.** A freshly allocated page may not be mapped to physical RAM yet. The first access triggers a **page fault**, which is another trip into kernel mode.

None of these operations have a guaranteed upper time bound. Any one of them can silently eat your entire audio deadline.

## The Wrong Way

This is what kills audio:

```cpp
// WRONG — allocating inside the audio callback
void MyProcessor::processBlock(juce::AudioBuffer<float>& buffer,
                               juce::MidiBuffer& midiMessages)
{
    // New allocation every callback — heap lock, possible page fault
    auto* tempBuffer = new float[buffer.getNumSamples()];

    for (int ch = 0; ch < buffer.getNumChannels(); ++ch)
    {
        auto* channelData = buffer.getWritePointer(ch);
        for (int i = 0; i < buffer.getNumSamples(); ++i)
            tempBuffer[i] = channelData[i] * gain;

        std::copy(tempBuffer, tempBuffer + buffer.getNumSamples(), channelData);
    }

    delete[] tempBuffer; // Also bad — free() takes the heap lock too

    // Even worse: resizing a container
    std::vector<float> envelope;
    envelope.resize(buffer.getNumSamples()); // allocation!
    envelope.shrink_to_fit();               // deallocation!
}
```

The `std::vector::resize()` call is just as bad as explicit `new`. Any STL container that grows will allocate. `std::string`, `std::map`, `std::function` (with a non-trivial closure) — all potential offenders.

## The Right Way: Pre-Allocate Before the Engine Starts

Allocate everything you could possibly need during `prepareToPlay()`. JUCE calls this before the audio engine starts, at a safe time on a non-realtime thread. Zero allocations allowed after that point.

```cpp
class MyProcessor : public juce::AudioProcessor
{
public:
    // RIGHT — pre-allocate in prepareToPlay
    void prepareToPlay(double sampleRate, int maximumExpectedSamplesPerBlock) override
    {
        // Allocate the worst-case buffer size once
        tempBuffer.resize(maximumExpectedSamplesPerBlock);

        // Pre-size any containers you will need
        envelopeBuffer.resize(maximumExpectedSamplesPerBlock, 0.0f);

        // Pre-allocate filter state, delay lines, FFT buffers — everything
        filter.prepare({ sampleRate, (juce::uint32)maximumExpectedSamplesPerBlock, 1 });
    }

    void processBlock(juce::AudioBuffer<float>& buffer,
                      juce::MidiBuffer& midiMessages) override
    {
        juce::ScopedNoDenormals noDenormals; // Fine — no allocation

        const int numSamples = buffer.getNumSamples();

        // Use the pre-allocated buffer — no heap interaction
        for (int ch = 0; ch < buffer.getNumChannels(); ++ch)
        {
            auto* channelData = buffer.getWritePointer(ch);
            for (int i = 0; i < numSamples; ++i)
                tempBuffer[i] = channelData[i] * gain.load();

            std::copy(tempBuffer.begin(), tempBuffer.begin() + numSamples, channelData);
        }
    }

    void releaseResources() override
    {
        // Free everything when the engine stops — also safe here
        tempBuffer.clear();
        tempBuffer.shrink_to_fit();
        envelopeBuffer.clear();
        envelopeBuffer.shrink_to_fit();
    }

private:
    std::vector<float> tempBuffer;
    std::vector<float> envelopeBuffer;
    juce::dsp::IIR::Filter<float> filter;
    std::atomic<float> gain { 1.0f };
};
```

## When You Need Truly Dynamic Structures: Memory Pools

Some processing genuinely requires dynamic behavior at runtime — variable-count voices in a synth, a growable event queue, a ring buffer of MIDI events. The answer is a **pre-allocated memory pool**.

Allocate a large slab of raw memory once at startup. Hand out chunks from it inside the callback. No OS involvement. Constant time.

```cpp
// A dead-simple fixed-size pool allocator for real-time use
template <typename T, int MaxObjects>
class RTPool
{
public:
    RTPool()
    {
        for (int i = 0; i < MaxObjects; ++i)
            freeList[i] = &storage[i];
        freeCount = MaxObjects;
    }

    T* allocate()
    {
        if (freeCount == 0) return nullptr; // fail gracefully, never block
        return reinterpret_cast<T*>(freeList[--freeCount]);
    }

    void deallocate(T* ptr)
    {
        if (freeCount < MaxObjects)
            freeList[freeCount++] = ptr;
    }

private:
    alignas(T) std::byte storage[MaxObjects][sizeof(T)];
    void* freeList[MaxObjects];
    int freeCount = 0;
};

// Pre-allocated pool for synth voices — constructed before the callback runs
RTPool<SynthVoice, 128> voicePool;

void processBlock(...)
{
    // In the callback — no heap, constant time
    SynthVoice* voice = voicePool.allocate();
    if (voice != nullptr)
    {
        new (voice) SynthVoice(); // placement new — no heap allocation
        // ... use voice ...
        voice->~SynthVoice();
        voicePool.deallocate(voice);
    }
}
```

Note: for multi-threaded access to this pool you would also need a lock-free mechanism — a common choice is `std::atomic` index manipulation or an SPSC ring buffer.

## Passing Data Between Threads: Lock-Free Queues

GUI to audio thread communication is a classic problem. The GUI wants to update a parameter; the audio thread needs to read it. The answer is never a mutex, always a lock-free structure.

```cpp
// For single values: std::atomic
std::atomic<float> cutoffFrequency { 1000.0f };

// GUI thread (safe to write from any thread):
cutoffFrequency.store(newValue, std::memory_order_relaxed);

// Audio thread (safe to read):
float freq = cutoffFrequency.load(std::memory_order_relaxed);
```

For passing whole objects (e.g., new reverb settings, new wavetable data), use a **SPSC lock-free queue** — Single Producer, Single Consumer. JUCE provides `juce::AbstractFifo` for exactly this. The Timur Doumler / Fabian Renn-Giles pattern of using a `RealtimeObject` wrapper is also worth studying.

## The Three Laws of the Audio Callback

Every line of code inside `processBlock()` must satisfy all three:

| Law | What it rules out |
|-----|-------------------|
| **Allocation-free** | `new`, `delete`, `malloc`, `free`, growing STL containers, `std::string` operations, `std::function` with captures |
| **Lock-free** | `std::mutex`, `std::lock_guard`, any blocking synchronization primitive |
| **Syscall-free** | File I/O, network I/O, `sleep()`, `printf()` (may lock internally), `std::cout` |

`juce::ScopedNoDenormals` is safe — it is a thin wrapper around a CPU register flag, no OS interaction. `std::atomic` operations on primitive types are safe. Stack allocations of fixed-size arrays are safe (as long as you don't blow the stack).

## Summary: The Audio Thread Checklist

**Safe in the audio callback:**
- Reading/writing `std::atomic<>` values
- Accessing pre-allocated buffers and arrays
- Stack-allocated fixed-size arrays (small ones)
- `juce::ScopedNoDenormals`
- Lock-free SPSC queues (read side only, if audio is the consumer)
- Placement `new` into pre-allocated pool memory
- Calling DSP functions that themselves follow these rules

**Never safe in the audio callback:**
- `new` / `delete` / `malloc` / `free`
- Any STL container operation that may allocate (`push_back`, `resize`, `insert`, `operator[]` on `std::map`)
- `std::mutex`, `std::lock_guard`, `std::unique_lock`
- `std::cout`, `printf`, `fprintf`, logging frameworks that write to disk
- File reads or writes of any kind
- `std::this_thread::sleep_for()` or any blocking wait
- Dynamic library loading
- Any call into code you do not fully control and have not audited

The rule of thumb: if it could block for an unpredictable amount of time, it has no business inside your audio callback.
