# How do I safely communicate between the UI thread and audio thread in C++?

*Originally published on Quora. Code drawn from a real production NLE audio mixer.*

---

This is one of those problems that bites every developer who writes their first audio application.
You write a perfectly reasonable looking mixer, you update a volume value from a slider callback,
and somewhere between there and the audio callback you get a crash — or worse, a subtle glitch
that only happens under load.

Here is what is actually going on, and what to do about it.

---

## The Core Problem

Real-time audio runs on a dedicated thread with extremely tight timing constraints. On most systems,
your audio callback has somewhere between 1 and 10 milliseconds to produce the next buffer of samples.
If it misses that window, you get a dropout — an audible click, glitch, or silence.

The UI thread, on the other hand, is a normal thread. It handles mouse clicks, repaints, file I/O,
layout — all the things that can take arbitrary amounts of time.

These two threads need to share data — volume levels, mute states, effect parameters — but they
cannot safely do so using conventional synchronization (like a `std::mutex`) without risking
exactly the problem you are trying to avoid.

**Why you cannot lock a mutex in the audio callback:**

If the UI thread holds the mutex and your OS decides to pause it for any reason — a page fault,
a scheduler preemption, a system call — your audio thread blocks waiting for the lock. That burns
your entire timing budget and you get a dropout.

---

## The Wrong Way

```cpp
// DON'T do this
std::mutex paramMutex;
float volumeDb = 0.0f;

// UI thread - slider changed
void sliderValueChanged(Slider* s)
{
    std::lock_guard<std::mutex> lock(paramMutex);  // holds the lock...
    volumeDb = static_cast<float>(s->getValue());
}

// Audio thread - callback
void processBlock(AudioBuffer<float>& buffer)
{
    std::lock_guard<std::mutex> lock(paramMutex);  // ...could block here
    applyGain(buffer, volumeDb);
}
```

This looks safe. It is not. Under any real system load, the UI thread can hold that lock at exactly
the wrong moment and starve your audio callback.

---

## The Right Way: Atomic Values for Simple Parameters

For simple scalar parameters — volume, pan, mute state — `std::atomic` is your best tool.
Reads and writes are guaranteed to be atomic (no torn reads) and the default
`memory_order_seq_cst` is safe enough for audio parameter communication.

```cpp
#include <atomic>

class AudioChannelStrip
{
public:
    // Written only from the UI thread
    void setVolumeDb(float db)
    {
        volumeDb.store(db, std::memory_order_relaxed);
    }

    void setMuted(bool muted)
    {
        isMuted.store(muted, std::memory_order_relaxed);
    }

    // Called only from the audio thread
    void processBlock(float* samples, int numSamples)
    {
        if (isMuted.load(std::memory_order_relaxed))
        {
            std::fill(samples, samples + numSamples, 0.0f);
            return;
        }

        float gain = dbToLinear(volumeDb.load(std::memory_order_relaxed));
        for (int i = 0; i < numSamples; ++i)
            samples[i] *= gain;
    }

private:
    std::atomic<float> volumeDb { 0.0f };
    std::atomic<bool>  isMuted  { false };
};
```

No locks. No blocking. The audio thread reads atomically and moves on.

---

## The Right Way: Polling on a Timer for UI Updates

The other direction — getting data **out** of the audio thread and onto the UI — has its own
trap. You cannot call `repaint()` from the audio thread. You cannot touch any UI component.

The correct pattern is a **timer on the UI thread** that polls audio state periodically.
This is exactly what we do in production for peak level metering:

```cpp
// From a real production NLE audio mixer
// The panel inherits privately from juce::Timer

class MovieAudioMixerPanel : public juce::Component,
                              private juce::Timer
{
public:
    MovieAudioMixerPanel()
    {
        startTimer(80);  // poll at ~12 Hz — plenty for metering
    }

    ~MovieAudioMixerPanel()
    {
        stopTimer();  // always stop before destruction
    }

private:
    void timerCallback() override
    {
        // This fires on the UI thread — safe to update UI state here
        for (auto& ch : channels)
        {
            // Read peak values written by audio thread via atomics
            ch.peakMeterL = audioEngine.getPeakLevel(ch.trackId, Channel::Left);
            ch.peakMeterR = audioEngine.getPeakLevel(ch.trackId, Channel::Right);
        }

        // Trigger a repaint — safe because we're on the message thread
        for (auto& strip : stripComponents)
            strip->repaint();
    }

    std::vector<AudioChannelStripState> channels;
    std::vector<std::unique_ptr<ChannelStripComponent>> stripComponents;
};
```

The key insight: **the audio thread writes to atomics, the UI thread reads them on its own schedule**.
Neither thread waits for the other. The meter display might be 80ms behind reality — nobody will
ever notice.

---

## When You Need More: Lock-Free Queues

Sometimes you need to send richer data — a whole preset, a list of new plugin parameters,
an asset to be loaded. Atomics are not enough.

In that case, reach for a lock-free FIFO. The audio thread should **only consume** from the queue;
the UI thread **only produces**. Never the other way around.

```cpp
#include <juce_audio_basics/juce_audio_basics.h>

// juce::AbstractFifo gives you a lock-free index manager
class ParameterQueue
{
public:
    struct Update
    {
        int    channelIndex;
        float  newVolumeDb;
        float  newPan;
    };

    // Called from UI thread
    void push(const Update& update)
    {
        int start, size;
        fifo.prepareToWrite(1, start, size);
        if (size > 0)
            buffer[start] = update;
        fifo.finishedWrite(size);
    }

    // Called from audio thread
    bool pop(Update& update)
    {
        int start, size;
        fifo.prepareToRead(1, start, size);
        if (size <= 0)
            return false;
        update = buffer[start];
        fifo.finishedRead(size);
        return true;
    }

private:
    static constexpr int Capacity = 64;
    juce::AbstractFifo fifo { Capacity };
    std::array<Update, Capacity> buffer;
};
```

The audio callback drains this queue at the start of each block and applies any pending parameter
changes. No locks, no blocking, no dropouts.

---

## The Rules, Summarized

| What you want to do | Safe approach |
|---|---|
| UI → Audio: send a simple float/bool | `std::atomic<T>` |
| UI → Audio: send a complex update | Lock-free FIFO (producer: UI, consumer: audio) |
| Audio → UI: report a level or state | `std::atomic<T>` written by audio, polled by UI timer |
| Audio → UI: trigger a repaint | Never directly. Use a timer on the UI thread. |
| Either direction: `std::mutex` | **Never in the audio callback.** |

The fundamental rule is simple: **the audio thread must never wait**.
Everything else follows from that.

---

*Code examples adapted from the Creation Suite NLE — a real production C++ creative application
built with JUCE. The audio mixer panel uses exactly these patterns for its channel strips,
peak metering, and VST3 parameter automation.*
