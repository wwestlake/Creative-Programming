You’ve built a seemingly simple audio app or plugin, but the moment you launch it, your CPU fan spins up like a jet engine and Task Manager or Activity Monitor shows it chewing through 100% of a CPU core. 

You aren't even doing heavy DSP yet. So what is burning all those cycles?

In audio development, extreme CPU usage is rarely caused by your actual math. It usually comes down to architectural flaws that keep a thread spinning endlessly when it should be sleeping. Here is how to track down and fix the four most common culprits.

## 1. Busy-Waiting and Spinlocks

The most common way to accidentally burn 100% of a CPU core is to implement a naïve wait loop on a background thread. If you need a background thread to process data only when the audio thread delivers it, you might be tempted to do this:

```cpp
// THE WRONG WAY
void run() override
{
    while (! threadShouldExit())
    {
        if (dataReady.load()) 
        {
            processData();
            dataReady.store(false);
        }
        // This spins millions of times a second, burning 100% of a core!
    }
}
```

This is called "busy-waiting." The thread is asking "is it ready?" millions of times per second, which keeps the CPU core fully powered up and doing pointless work.

**The Fix:** You need to put the thread to sleep until there is actually work to do. In JUCE, `juce::WaitableEvent` is perfect for this.

```cpp
// THE RIGHT WAY
juce::WaitableEvent dataReadyEvent;

// In your audio thread:
dataReadyEvent.signal(); // Wakes up the background thread

// In your background thread:
void run() override
{
    while (! threadShouldExit())
    {
        // Thread sleeps here consuming 0% CPU until signaled
        if (dataReadyEvent.wait(100)) 
        {
            processData();
        }
    }
}
```

## 2. Abusive UI Polling and Repainting

If your UI needs to display meters, scopes, or playheads, you are probably using a `juce::Timer` to query the audio thread's state and call `repaint()`. 

Calling `repaint()` too often will crush your UI thread. If you set your timer interval to 1ms (1000 frames per second), the OS rendering engine will desperately try to keep up, maxing out the CPU and making your UI feel sluggish and unresponsive.

**The Fix:** Human eyes can't process updates faster than 60Hz. Cap your UI refresh rates.

```cpp
// THE WRONG WAY
startTimer(1); // 1ms = 1000 FPS. Do not do this!

// THE RIGHT WAY
startTimerHz(30); // 30 FPS is smooth enough for most meters
// or
startTimerHz(60); // 60 FPS for very smooth scopes/playheads
```

## 3. The Denormal Trap

If your app runs perfectly fine most of the time, but the CPU spikes to 100% specifically when a reverb, delay, or filter tail fades out to silence, you are dealing with denormalized numbers. 

When floating-point values get extremely close to zero, the CPU switches into a high-precision mode that is 10x to 100x slower to process. We covered this entirely in a previous article on Denormals, but the quick fix is to flush these tiny numbers to zero at the hardware level.

In JUCE, add this to the top of your `processBlock`:
```cpp
juce::ScopedNoDenormals noDenormals;
```

## 4. Microscopic Buffer Sizes

Sometimes the bottleneck isn't your code, but the overhead of moving data around. 

If your audio interface is set to request tiny buffer sizes—like 32 or 16 samples—your `processBlock` function is being called thousands of times per second. Even if your DSP math is trivial, the overhead of the function call, setting up pointers, and iterating through the loop completely dwarfs the actual work being done.

**The Fix:** Unless you are tracking live instruments and desperately need sub-2ms latency, increase your audio buffer size. A buffer of 256 or 512 samples gives the CPU enough breathing room to process chunks of audio efficiently before returning to other tasks.

## Summary

When dealing with 100% CPU usage, don't blindly optimize your math. Profile your app and find out *which thread* is burning the cycles:
* **Background Thread:** Look for `while` loops that don't sleep. Use `juce::WaitableEvent`.
* **UI Thread (Message Thread):** Look for timers firing faster than 60Hz or massive `repaint()` regions.
* **Audio Thread (during silence):** Look for denormals in feedback loops (delays/reverbs/IIR filters).
* **Audio Thread (always):** Check your buffer size.
