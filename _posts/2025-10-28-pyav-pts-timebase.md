### The Core Problem: Computers Need Integers

The fundamental thing to understand is that video files **do not store time in seconds**. Storing time as a floating-point number (like `10.534` seconds) is inefficient and can lead to rounding errors that accumulate over a long video, causing audio/video sync drift.

Instead, everything is based on a simple, ever-increasing integer counter. Think of it as a "tick" count.

---

### Step 1: The Counter - Presentation Timestamp (PTS)

Every single frame, whether it's a video frame or a chunk of audio, has a number attached to it called the **Presentation Timestamp (PTS)**.

*   **What it is:** Just an integer. A counter. A "tick" number.
*   **What it means:** It dictates *when* that frame should be shown to the viewer relative to other frames. A frame with `PTS=90000` comes after a frame with `PTS=0`.

That's it. On its own, a PTS value like `90000` is meaningless. It doesn't tell you if it's 1 second or 1 hour into the video. To make sense of it, you need a conversion rate.

### Step 2: The Conversion Rate - `time_base`

The **`time_base`** is the conversion rate that tells you how long one "tick" of the PTS counter lasts. It's the value of a single tick, measured in seconds.

*   **What it is:** A fraction (e.g., `1/90000`). It's a `Fraction` object in Python, not a float.
*   **What it means:** A `time_base` of `1/90000` means one PTS tick is equal to 1/90000th of a second.

The single most important formula you will use is:

`time_in_seconds = frame.pts * frame.time_base`

If a frame has `PTS=90000` and its `time_base` is `1/90000`, then:
`time_in_seconds = 90000 * (1/90000) = 1.0` second.

### Step 3: The "Gotcha" - Different Streams Have Different Clocks

A single media file (like an `.mp4`) is a container. Inside, it can have multiple independent streams: a video stream, an audio stream, a subtitle stream, etc.

**The video stream and the audio stream can, and often do, have different clocks.**

This means they have different `time_base` values.

*   A **video stream** might have a `time_base` of `1/90000`. This is a very high-resolution clock, needed to place video frames precisely. A 30 FPS video might have frames at PTS `0`, `3000`, `6000`, etc. (`3000 * 1/90000 = 1/30` seconds).
*   An **audio stream** might have a `time_base` that matches its sample rate, like `1/48000`.

This is why you have two different `time_base` concepts in PyAV:

1.  **`stream.time_base`**: This is the conversion rate for a *specific stream*. This is the one you care about most. When you want to seek, you need to know the `time_base` of the stream you are seeking on.
2.  **`frame.time_base`**: When you decode a frame, it comes from a specific stream, so it carries that stream's `time_base` with it. A video frame will have the video stream's `time_base`, and an audio frame will have the audio stream's `time_base`.

---

### The Execution Pipeline: A Practical Example

Let's trace exactly what our code needs to do.

**Our Goal:** Extract the segment from **10.0 seconds** to **15.0 seconds**.

**Step 1: Get the Right Clock for Navigation.**
Before we can jump to 10.0 seconds, we need to know which clock to use. We'll use the video stream's clock because seeking on video keyframes is the most reliable.

```python
# Open the file
container = av.open('my_video.mp4')
# Get the video stream
video_stream = container.streams.video[0]
# Get its specific time_base (its clock's conversion rate)
# Let's say it's Fraction(1, 90000)
video_time_base = video_stream.time_base
```

**Step 2: Translate Our Goal into the Video's Language (PTS).**
The `seek()` function doesn't understand "10.0 seconds". It understands PTS ticks. We need to convert our start time into the video stream's PTS counter.

We rearrange the formula: `pts = time_in_seconds / time_base`.

```python
start_time_sec = 10.0
# Calculate the target PTS tick
seek_pts = int(start_time_sec / video_time_base) # 10.0 / (1/90000) = 900000
```
So, "10.0 seconds" is equivalent to the PTS tick `900000` *on the video stream's clock*.

**Step 3: Tell PyAV to Jump.**
Now we give the `seek` command using the PTS value we just calculated.

```python
# Seek to the keyframe before PTS tick 900000 on the video stream
container.seek(seek_pts, stream=video_stream)
```

**Step 4: Decode and Translate Everything Back to Seconds.**
The container is now positioned correctly. We start decoding frames. The crucial thing here is that the loop will give us both video and audio frames, each with its own PTS and `time_base`. We must translate each one back to seconds to make sense of it.

```python
for frame in container.decode(video_stream, audio_stream):
    # This frame could be video or audio. We don't care.
    # We use ITS OWN time_base to convert ITS OWN PTS to seconds.
    
    current_time_sec = float(frame.pts * frame.time_base)
    
    # Let's say we get a video frame with PTS=899100 and time_base=1/90000
    # current_time_sec = 899100 * (1/90000) = 9.99 seconds
    
    # Let's say we get an audio frame with PTS=479520 and time_base=1/48000
    # current_time_sec = 479520 * (1/48000) = 9.99 seconds
```

**Step 5: Make Decisions in Our Language (Seconds).**
Now that every frame's timestamp has been converted to a simple float in seconds, our logic becomes easy and reliable.

```python
    if current_time_sec >= 15.0: # Our end_time
        break
    
    if current_time_sec >= 10.0: # Our start_time
        # This frame is inside our segment. Process it.
        if isinstance(frame, av.VideoFrame):
            # Do video stuff
        elif isinstance(frame, av.AudioFrame):
            # Do audio stuff
```

### The Rules to Live By

1.  **For Seeking (Python -> Video File):** Convert your time in **seconds** into **PTS** using the `time_base` of the **STREAM** you are seeking on. `pts = seconds / stream.time_base`.
2.  **For Analysis (Video File -> Python):** Convert a decoded **FRAME's** timestamp from **PTS** into **seconds** using the `time_base` of that specific **FRAME**. `seconds = frame.pts * frame.time_base`.
3.  **Keep Your Logic in Seconds:** Do all your `if/else` comparisons and time calculations in seconds. Only convert to/from PTS when you're directly interacting with PyAV's seek or frame objects.
