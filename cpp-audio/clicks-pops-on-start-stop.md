# Why does my audio have clicks and pops when I start and stop playback in C++?

You hit play, and there's a sharp crack. You hit stop, same thing. The audio itself sounds fine — it's only the transitions that pop. This is one of the most common audio programming bugs, and it has one root cause: **a discontinuous jump in the waveform**.

---

## What's Actually Happening

Your DAC (digital-to-analog converter) is outputting a stream of sample values — numbers between -1.0 and 1.0 that represent the position of a speaker cone at each instant in time. If one sample is `0.0` and the next is `0.87`, the speaker cone has to teleport. Physically, that means an instantaneous pressure spike — a high-frequency transient that your ear hears as a click or pop.

This is not a bug in your driver, your audio interface, or your OS. It's a math problem. You created a discontinuity in the signal, and physics did the rest.

---

## The Wrong Way

The most common version of this mistake looks like this:

```cpp
// WRONG — instant gain switching in the audio callback
void MyAudioProcessor::processBlock(juce::AudioBuffer<float>& buffer, juce::MidiBuffer&)
{
    if (isPlaying)
        gain = 1.0f;
    else
        gain = 0.0f;   // Jumps from whatever the last sample was to 0.0 instantly

    for (int ch = 0; ch < buffer.getNumChannels(); ++ch)
    {
        auto* data = buffer.getWritePointer(ch);
        for (int i = 0; i < buffer.getNumSamples(); ++i)
            data[i] *= gain;
    }
}
```

Or even more subtly wrong — multiplying by 0 at the block boundary:

```cpp
// ALSO WRONG — silence the buffer when stopped
if (!isPlaying)
    buffer.clear();  // last sample of previous block to 0.0 in one step
```

Both cause the same artifact. The waveform was at some nonzero value at the end of the last block, and now it's at zero. Click.

---

## The Solution: Ramping

The fix is to never jump. Instead, **smoothly interpolate** between your current amplitude and your target amplitude over a short window — typically 5–20 ms. This gives the waveform enough time to gracefully approach silence (or full amplitude) without creating a transient.

### Simple Linear Ramp

Here is a minimal implementation that works well for start/stop:

```cpp
class RampedGain
{
public:
    RampedGain() = default;

    void prepare(double sampleRate, double rampLengthSeconds = 0.010)
    {
        rampSamples = static_cast<int>(sampleRate * rampLengthSeconds);
        currentGain = 0.0f;
        targetGain  = 0.0f;
        samplesRemaining = 0;
    }

    void setTarget(float target)
    {
        targetGain       = target;
        samplesRemaining = rampSamples;
        // stepSize is calculated from wherever we currently are
        stepSize = (targetGain - currentGain) / static_cast<float>(samplesRemaining);
    }

    void start() { setTarget(1.0f); }
    void stop()  { setTarget(0.0f); }

    // Apply gain to a block of samples (single channel)
    void process(float* data, int numSamples)
    {
        for (int i = 0; i < numSamples; ++i)
        {
            if (samplesRemaining > 0)
            {
                currentGain += stepSize;
                --samplesRemaining;

                // Clamp to avoid floating-point drift past the target
                if (samplesRemaining == 0)
                    currentGain = targetGain;
            }

            data[i] *= currentGain;
        }
    }

private:
    float currentGain     = 0.0f;
    float targetGain      = 0.0f;
    float stepSize        = 0.0f;
    int   samplesRemaining = 0;
    int   rampSamples      = 0;
};
```

Usage in a JUCE processor:

```cpp
// In your processor header
RampedGain ramp;

// In prepareToPlay
void prepareToPlay(double sampleRate, int /*samplesPerBlock*/) override
{
    ramp.prepare(sampleRate, 0.010); // 10ms ramp
}

// On transport change (e.g. button press, MIDI, etc.)
void onPlay() { ramp.start(); }
void onStop() { ramp.stop();  }

// In processBlock
void processBlock(juce::AudioBuffer<float>& buffer, juce::MidiBuffer&) override
{
    for (int ch = 0; ch < buffer.getNumChannels(); ++ch)
        ramp.process(buffer.getWritePointer(ch), buffer.getNumSamples());
}
```

---

## The Production Solution: JUCE SmoothedValue

If you are already in JUCE, do not roll your own. `juce::SmoothedValue<float>` handles this correctly and is thread-safe for parameter changes coming from the message thread:

```cpp
juce::SmoothedValue<float, juce::ValueSmoothingTypes::Linear> gainRamp;

void prepareToPlay(double sampleRate, int samplesPerBlock) override
{
    gainRamp.reset(sampleRate, 0.010); // 10ms smoothing time
    gainRamp.setCurrentAndTargetValue(0.0f);
}

void onPlay() { gainRamp.setTargetValue(1.0f); }
void onStop() { gainRamp.setTargetValue(0.0f); }

void processBlock(juce::AudioBuffer<float>& buffer, juce::MidiBuffer&) override
{
    for (int ch = 0; ch < buffer.getNumChannels(); ++ch)
    {
        auto* data = buffer.getWritePointer(ch);
        for (int i = 0; i < buffer.getNumSamples(); ++i)
            data[i] *= gainRamp.getNextValue();
    }
}
```

`SmoothedValue` also supports `Multiplicative` smoothing, which gives a more perceptually linear fade (useful for volume controls rather than gating).

---

## The "Last Sample" Problem

Here is a subtler version of the bug that catches people even after they add ramping:

```cpp
// BUG: You assume the ramp always starts from 0
void onPlay()
{
    currentGain = 0.0f;   // WRONG if we stopped mid-fade
    targetGain  = 1.0f;
}
```

If the user hits stop, the ramp starts moving toward 0. But then they immediately hit play again — maybe 3ms into the fade-out. `currentGain` is now something like `0.72f`. If you reset it to `0.0f` and ramp to `1.0f`, you have created a discontinuity at the reset point. Click.

The correct approach is what the `RampedGain` class above does: **always call `setTarget()` from `currentGain`, never reset it manually**. `SmoothedValue` handles this correctly out of the box — calling `setTargetValue()` mid-ramp picks up from wherever it currently is.

---

## Bonus: DC Offset Clicks

If your audio signal has a DC component — a constant positive or negative bias — and you gate it on or off, you will get clicks even if you think your signal starts at zero.

```cpp
// Your oscillator might produce a signal that "rests" at +0.05 instead of 0.0
// Gating it introduces a 0.05 to 0 jump. Still a click.
```

The fix is either:
1. **High-pass filter** your signal (even a first-order filter at ~5 Hz removes DC bias without affecting audio)
2. Or ensure your ramp starts from the actual last output sample value, not from an assumed rest position

A simple one-pole high-pass to strip DC:

```cpp
// In your processor, per channel
float dcBlocker = 0.0f;  // state per channel
const float dcCoeff = 0.9995f;

// Per sample
float input = rawSample;
float output = input - dcBlocker;
dcBlocker = dcBlocker * dcCoeff + input * (1.0f - dcCoeff);
// 'output' is DC-free
```

---

## The Rule

| Situation | Wrong | Right |
|---|---|---|
| Start playback | `gain = 1.0f` instantly | Ramp from current value to 1.0 over 5-20ms |
| Stop playback | `gain = 0.0f` or `buffer.clear()` | Ramp from current value to 0.0 over 5-20ms |
| Restart mid-fade | Reset `currentGain = 0.0f` | Continue ramp from wherever it currently is |
| DC-biased signal | Gate directly | High-pass filter + ramp |

**The rule: never make a discontinuous jump in an audio signal.**

A click is the waveform screaming that you teleported the speaker cone. Ramp everything — gains, pitches, filter cutoffs, wet/dry levels. If a parameter changes faster than ~5ms, your ears will notice.
