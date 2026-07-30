Audio development is smooth sailing until your plugin crashes a user's DAW because they dropped it on a mono track, or a client suddenly asks for 7.1.4 Dolby Atmos support. If your entire DSP pipeline is hardcoded to "Left" and "Right," you're setting yourself up for failure. 

Here is how to handle multi-channel routing in C++ without breaking your application.

## The Non-Interleaved Reality
In modern C++ audio frameworks like JUCE, audio data is delivered in an `AudioBuffer<float>`. This is a set of non-interleaved arrays—meaning channel 0 (Left) is a contiguous block of floats in memory, channel 1 (Right) is another, and channel N is just another pointer. 

Because channels are completely uncoupled in memory, handling different channel configurations is entirely up to your processing loop.

## The Hardcoded Stereo Trap
Most beginners fall into the trap of assuming every audio bus has exactly two channels. 

**The WRONG way:**
```cpp
void processBlock(juce::AudioBuffer<float>& buffer, juce::MidiBuffer& midiMessages)
{
    // FATAL: Will access violate and crash if the DAW provides a mono buffer
    float* leftChannel = buffer.getWritePointer(0);
    float* rightChannel = buffer.getWritePointer(1);

    for (int sample = 0; sample < buffer.getNumSamples(); ++sample)
    {
        leftChannel[sample] *= 0.5f;
        rightChannel[sample] *= 0.5f;
    }
}
```

**The RIGHT way:**
Stop thinking in Left and Right. Dynamically iterate over whatever channel count the host provides.

```cpp
void processBlock(juce::AudioBuffer<float>& buffer, juce::MidiBuffer& midiMessages)
{
    const int numChannels = buffer.getNumChannels();
    const int numSamples = buffer.getNumSamples();

    for (int channel = 0; channel < numChannels; ++channel)
    {
        float* channelData = buffer.getWritePointer(channel);
        for (int sample = 0; sample < numSamples; ++sample)
        {
            // Safely applies to mono, stereo, 5.1, or 128-channel Ambisonics
            channelData[sample] *= 0.5f;
        }
    }
}
```

## Matrix Routing: N Inputs to M Outputs
When you need to route N inputs to M outputs, you use a routing matrix. A matrix dictates how much of input channel `x` goes to output channel `y`. In C++, this is often represented as a 2D array of gain values.

```cpp
// Routing an N-channel buffer to an M-channel buffer
void routeAudio(const juce::AudioBuffer<float>& input,
                juce::AudioBuffer<float>& output,
                const std::vector<std::vector<float>>& routingMatrix)
{
    // Always clear the destination first!
    output.clear(); 

    for (int inCh = 0; inCh < input.getNumChannels(); ++inCh)
    {
        const float* inData = input.getReadPointer(inCh);

        for (int outCh = 0; outCh < output.getNumChannels(); ++outCh)
        {
            float gain = routingMatrix[inCh][outCh];
            if (gain > 0.0f)
            {
                // addFrom(destChannel, destStartSample, sourceData, numSamples, gain)
                output.addFrom(outCh, 0, inData, input.getNumSamples(), gain);
            }
        }
    }
}
```

## Downmixing and Power Preservation
If you don't have a specific routing matrix but need to collapse a stereo signal to mono, do not just average the two channels (sum and divide by 2). This causes a perceptible drop in loudness due to acoustic power laws.

Instead, preserve perceived power by dividing by the square root of 2 (`~0.707f`).

```cpp
void stereoToMono(juce::AudioBuffer<float>& buffer)
{
    if (buffer.getNumChannels() < 2) return;

    const float* left = buffer.getReadPointer(0);
    const float* right = buffer.getReadPointer(1);
    float* monoOut = buffer.getWritePointer(0); // Overwrite channel 0 with mono mix

    const float invSqrt2 = 0.70710678f; // 1.0f / sqrt(2.0f)

    for (int i = 0; i < buffer.getNumSamples(); ++i)
    {
        monoOut[i] = (left[i] + right[i]) * invSqrt2;
    }

    // Mute the now-unused right channel
    buffer.clear(1, 0, buffer.getNumSamples());
}
```

## Bussing: Accumulating Submixes
When building a mixer where multiple tracks route into a single submix (a bus), you are accumulating samples. The golden rule of bussing is: the bus buffer must be zeroed out before the first track adds to it, and every subsequent track must *add* (not overwrite).

```cpp
void mixTracksToBus(const std::vector<juce::AudioBuffer<float>*>& tracks,
                    juce::AudioBuffer<float>& masterBus)
{
    // 1. Clear the bus before accumulating
    masterBus.clear();

    // 2. Accumulate all tracks into the bus
    for (auto* track : tracks)
    {
        int channelsToMix = juce::jmin(track->getNumChannels(), masterBus.getNumChannels());
        
        for (int ch = 0; ch < channelsToMix; ++ch)
        {
            masterBus.addFrom(ch, 0, track->getReadPointer(ch), track->getNumSamples(), 1.0f);
        }
    }
}
```

## Summary

| Rule | Explanation |
| --- | --- |
| **Iterate, never hardcode** | Always loop up to `buffer.getNumChannels()`. Never blindly request channel 0 or 1. |
| **Clear before accumulating** | When routing multiple signals to one bus, call `buffer.clear()` once before the first `addFrom()`. |
| **Respect equal power** | Use `1.0 / sqrt(2.0)` (~0.707) for stereo-to-mono downmixing, not `0.5`. |
| **Use matrices for N-to-M** | Represent complex routing (like Ambisonics to 5.1) as 2D arrays of gain coefficients. |
