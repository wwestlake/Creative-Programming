You’ve got a massive summing mixer or a high-order IIR filter, and your signal is slowly degrading into noise or blowing up entirely. The culprit is almost always floating-point truncation when dealing with 32-bit (`float`) numbers.

A standard 32-bit float gives you roughly 7 decimal digits of precision. This sounds like plenty until you start accumulating tiny values into a large value. When you add `0.0000001` to `1000.0` using a 32-bit float, the tiny value is truncated into nothingness. In audio, this translates to quantization noise, DC offsets, and unstable feedback loops.

Here is how you handle precision loss in C++ audio processing.

## The Danger of 32-bit State in IIR Filters

IIR (Infinite Impulse Response) filters rely on feedback. If you implement a high-order filter (like a 4th-order Direct Form filter) using entirely 32-bit floats, the roots of the polynomial become extremely sensitive to coefficient quantization. The filter will likely go unstable (blow up) or sound like it's bit-crushing your low end.

Here is the **WRONG** way to implement an IIR filter's state:

```cpp
// WRONG: Using 32-bit floats for internal state and coefficients
class BadFilter 
{
public:
    void process(juce::AudioBuffer<float>& buffer)
    {
        auto* channelData = buffer.getWritePointer(0);
        for (int i = 0; i < buffer.getNumSamples(); ++i)
        {
            float input = channelData[i];
            
            // 32-bit accumulation loses precision fast
            float output = (b0 * input) + (b1 * x1) + (b2 * x2) 
                         - (a1 * y1) - (a2 * y2);
                         
            x2 = x1;
            x1 = input;
            y2 = y1;
            y1 = output;
            
            channelData[i] = output;
        }
    }
private:
    float x1 = 0.0f, x2 = 0.0f;
    float y1 = 0.0f, y2 = 0.0f;
    float a1, a2, b0, b1, b2; // 32-bit coefficients are a recipe for instability
};
```

## The Right Way: Upcast to Double for Math

Audio I/O (from the host or audio interface) is almost always given to you as 32-bit float buffers. The standard practice is to upcast the incoming sample to `double` (64-bit float), perform all your DSP math (which now has ~15 decimal digits of precision), and then downcast back to `float` when writing to the output buffer.

Internal state variables—like filter delays, oscillator phase accumulators, and master summing busses—should absolutely be `double`.

```cpp
// RIGHT: 64-bit internal state, 32-bit I/O
class GoodFilter 
{
public:
    void process(juce::AudioBuffer<float>& buffer)
    {
        auto* channelData = buffer.getWritePointer(0);
        for (int i = 0; i < buffer.getNumSamples(); ++i)
        {
            // Upcast to double
            double input = static_cast<double>(channelData[i]);
            
            // 64-bit precision math
            double output = (b0 * input) + (b1 * x1) + (b2 * x2) 
                          - (a1 * y1) - (a2 * y2);
                          
            x2 = x1;
            x1 = input;
            y2 = y1;
            y1 = output;
            
            // Downcast back to float for the output buffer
            channelData[i] = static_cast<float>(output);
        }
    }
private:
    // Internal state and coefficients are doubles
    double x1 = 0.0, x2 = 0.0;
    double y1 = 0.0, y2 = 0.0;
    double a1, a2, b0, b1, b2; 
};
```

This exact same rule applies to oscillators. If you use a `float` for a phase accumulator, at very high frequencies or over long continuous runs, the increment step might become small enough compared to the current phase value that truncation occurs, throwing the oscillator out of tune. Always use `double` for phase accumulators.

## Denormals (Subnormal Numbers)

When floating-point values get extremely close to zero (around `1e-38` for 32-bit floats), the CPU switches to a different representation called denormalized (or subnormal) numbers. Processing these numbers forces the CPU out of its fast path, causing massive CPU spikes (often 10x-100x slower) that will instantly cause audio dropouts.

This happens all the time in filters or reverbs when the input goes silent and the feedback tail exponentially decays towards zero.

To fix this, you need to disable denormals at the CPU level before processing your audio block. In JUCE, you do this using `juce::ScopedNoDenormals`:

```cpp
void processBlock(juce::AudioBuffer<float>& buffer, juce::MidiBuffer& midiMessages) override
{
    // Disables denormals for the scope of this function
    juce::ScopedNoDenormals noDenormals;
    
    // ... rest of your DSP code ...
}
```

This sets the DAZ (Denormals-Are-Zero) and FTZ (Flush-To-Zero) flags on the CPU, instantly turning those tiny, problematic numbers into plain zeros, saving your CPU cycles.

## Summary

*   **I/O**: Keep it `float` (32-bit) to match the host/API.
*   **State & Math**: Use `double` (64-bit) for filter delays, coefficients, phase accumulators, and summing.
*   **Filters**: Never implement high-order Direct Form IIR filters as a single stage. Cascade 2nd-order biquads (using doubles) to maintain stability.
*   **Denormals**: Always flush to zero (`juce::ScopedNoDenormals`) to prevent CPU spikes on decaying feedback loops.
