Title: Making our own spectrogram

URL Source: https://fasterthanli.me/articles/making-our-own-spectrogram

Published Time: 2025-09-15T07:30:00Z

Markdown Content:
### Bonus content

This article comes with a bonus code repository.

Please log in to access it:

A couple months ago [I made a loudness meter](https://fasterthanli.me/articles/the-science-of-loudness) and went way too in-depth into how humans have measured loudness over time.

![Image 1: A screenshot of the fasterthanlime audio meter, with RMS, sample peak, true peak, and various loudness metrics.
](https://cdn.fasterthanli.me/content/articles/making-our-own-spectrogram/fasterthanlime-audio-meter@2x~d5e6b54e3ade21f5.jxl)

Today we’re looking at a spectrogram visualization I made, which is a lot more entertaining!

![Image 2](https://cdn.fasterthanli.me/content/articles/making-our-own-spectrogram/spectrogram-sonata-op25-3@2x~c22af608785d58b2.jxl)

We’re going to talk about how to extract frequencies from sound waves, but also how my spectrogram app is assembled from different Rust crates, how it handles audio and graphics threads, how it draws the spectrogram etc.

For now, let’s start at the beginning, with Fourier transforms.

[The humble sine wave --------------------](https://fasterthanli.me/articles/making-our-own-spectrogram#the-humble-sine-wave)
I’d always heard that Fourier transforms allowed translating from the time domain to the frequency domain, and vice versa.

But it was all a bit abstract until I saw it. Just like the whole “sound is a wave” thing didn’t really click for me, until I played around with the parameters of a sine wave:

f(t)=A⋅sin⁡(2 π⋅f⋅t+ϕ)+C

A=0.500(Amplitude)

f=222 Hz(Frequency)

C=0.00(DC Offset)

ϕ=0.00 rad(Phase)

Amplitude (A) scales the wave vertically; more amplitude makes for a louder sound, less amplitude makes it quieter.

Frequency (f) scales it horizontally, over time: a higher frequency makes for a higher pitch, and a lower frequency, you guessed it, makes it deeper.

The DC offset (C) moves the wave vertically and doesn’t make any audible difference, except if it gets close enough to 1 or -1, then the wave is clipped and we hear a kind of electric sound.

Finally, the phase (ϕ) moves the wave horizontally, which only starts to matter when we’re mixing several waves together!

In stereo audio, we have two different signals, left and right, and they can interfere with each other, to the point of cancelling out!

ϕ R=1.57 rad(Interchannel phase difference)

Playing with this slider should result in perceived volume changes if you’re listening on something like a bluetooth speaker, whereas if you’re using headphones, it’ll sound weird, “dephased”.

It’s really hard to describe, so get a pair of headphones and try it out!

[Approximating a square wave ---------------------------](https://fasterthanli.me/articles/making-our-own-spectrogram#approximating-a-square-wave)
Sine waves and cosine waves (same idea, just phase-shifted by π/2) are not the only waveforms — another famous one is the square wave!

But what’s fascinating is that if we add up enough cosine waves, we can actually approximate a square wave:

One harmonic

f(x)=0+∑n=1 1 A n cos⁡(2 π f n x+ϕ n)where:

A 1=1.273,f 1=4,ϕ 1=1.583

One cosine gives us this very soft sound, but as we keep adding harmonics, the shape of the reconstructed wave, aka the sum (∑) of all of our cosines, looks more and more like a square wave. And it sounds a lot sharper, and louder, too!

The Fourier transform is the operation that gives us the amplitude and phase of cosines of different frequencies that we can add together in order to reproduce the input signal.

[A real-world sample -------------------](https://fasterthanli.me/articles/making-our-own-spectrogram#a-real-world-sample)
And it’s not limited to synthetic signals like our square wave here!

This recording of me making a funny noise with my mouth can also be reproduced by a pile of cosines:

One harmonic

#### Frequency Spectrum (FFT)

Gray bars show all frequencies, red bars show the 1 harmonic used for reconstruction

In fact, we don’t even need to use the whole output of the Fourier transform — just using 64 cosines gives us a pretty good-sounding reconstruction of that “pop”.

[Chunking and windowing ----------------------](https://fasterthanli.me/articles/making-our-own-spectrogram#chunking-and-windowing)
The operation I used for the preceding demos is the FFT, the _fast_ fourier transform. You can find implementations of it for most programming languages, and you always have to pick an input size, which is usually a power of two.

We’ve picked an input size of 1024 for all our demos so far. If we want to process an entire song, we’re going to have to divide the input into chunks:

8 harmonics

That reconstruction doesn’t sound correct though. Even with the maximum number of harmonics, there are discontinuities at the boundary between chunks.

See how the green line spikes? That’s because when analyzing a chunk, We implicitly assume that it is cyclical, whereas actually it has a chunk before it and a chunk after it, with a different signal.

The fix is to use something like the Hann windowing function, so that we analyze overlapping chunks.

Drag the window around to “sample” different parts of the input signal and see how it tapers off at the edge of the window.

The Hann window has a very nice property: it’s designed so that two adjacent, half-shifted Hanns sum to a constant 1:

Which means if we sample with Hann windows and we reconstruct by doing overlap-add, which is adding together two overlapping windows, then we get the correct result and we don’t get any of that spectral leakage at the boundary between chunks that we had before.

The reconstruction is now correct:

8 harmonics

50% overlap

…except at the beginning and the end, where there’s only one Hann window, and we get nasty artifacts.

We _could_ solve that by padding the start and end with silence: that’s something done by real-world audio codecs like MP3!

![Image 3: Amos](https://cdn.fasterthanli.me/content/img/reimena/amos-neutral~55a3477398fe0cb1.jxl)

That’s why it took a while to figure out gapless playback! Audio encoders didn’t record how many samples to skip at the beginning and at the end in the file — they eventually started putting it [in metadata headers](https://wiki.hydrogenaudio.org/index.php?title=MP3).

[The Gabor limit ---------------](https://fasterthanli.me/articles/making-our-own-spectrogram#the-gabor-limit)
When designing a spectrogram, there is a compromise to be made between time resolution and frequency resolution.

The size of the FFT is not just about how much computation we can afford to do. The time complexity of an FFT is O(N log⁡N), so we can push it a lot higher than 1024.

But the bigger the FFT size, the less time resolution we have:

FFT input size

Small FFT Large FFT

This is something annoying, but fundamental, called the [Heisenberg-Gabor limit](https://en.wikipedia.org/wiki/Uncertainty_principle), and which says that “one cannot simultaneously sharply localize a signal f in both the time domain and frequency domain.”

And in a spectrogram, we care about both, since we’re plotting frequency over time.

![Image 4](https://cdn.fasterthanli.me/content/articles/making-our-own-spectrogram/spectrogram-preview-plotted-over-time@2x~11944fecd580d1f7.jxl)

In other words, we have to pick between things being blurry on the horizontal axis or on the vertical axis.

In my program, I chose a window of size 4096, which is offset by 128 samples every time — this means there’s 97% overlap between two windows:

That overlap means some smearing is happening:

Your browser does not support the video tag.

But it’s a compromise! With the same FFT size, zero overlap would make for a very slow-moving visualization where everything is compressed horizontally:

Your browser does not support the video tag.

We can compensate this by making the FFT smaller, but that makes us lose a _lot_ of frequency resolution:

Your browser does not support the video tag.

[Interpolation -------------](https://fasterthanli.me/articles/making-our-own-spectrogram#interpolation)
It doesn’t look _that_ bad, because there’s interpolation going on: the computed frequency bin index is a floating point number:


```
fn frequency_to_bin_index(freq: f32, sample_rate: f32, fft_bins: usize) -> f32 {
    // Convert frequency to FFT bin index
    // With Nyquist dropped, we have N/2 bins covering 0 to Nyquist
    let nyquist = sample_rate / 2.0;
    let freq_per_bin = nyquist / fft_bins as f32;
    freq / freq_per_bin
}
```

So when drawing a pixel, if we get bin index 3.5, we mix the value of bin 3 and bin 4 half each:


```
for y in 0..height {
    // Map y-coordinate to frequency
    let normalized_y = (height - y) as f32 / height as f32; // 0 at bottom, 1 at top
    let freq = normalized_to_frequency(normalized_y, sample_rate);
    // ...then to a (non-integer) frequency bin
    let freq_index = frequency_to_bin_index(freq, sample_rate, fft_bins);

    let lower_index = freq_index.floor() as usize;
    let upper_index = (freq_index.ceil() as usize).min(model.len().saturating_sub(1));
    let interpolation_factor = freq_index.fract();

    // deal with edge cases (literally)
    let value = if lower_index >= model.len() {
        0.0
    } else if lower_index == upper_index {
        model[lower_index]
    } else {
        let lower_value = model[lower_index];
        let upper_value = model[upper_index];
        // linear interpolation is happening here 👇
        lower_value * (1.0 - interpolation_factor) + upper_value * interpolation_factor
    };

    // map [-80..0] dB to [0..1]
    let normalized = db_to_normalized(value);
    // see below for colormap explanation
    let rgb = colormap(normalized);
    frame_buffer[y * width + x] = Color32::from_rgb(rgb.0, rgb.1, rgb.2);
}
```

If we disable interpolation, we can see the boundary between frequency bins:

Your browser does not support the video tag.

[Color mapping -------------](https://fasterthanli.me/articles/making-our-own-spectrogram#color-mapping)
Another design choice I had to make was the color map. Mapping 0-1 to black-white is trivial, but hard to read: can you follow the melody here?

Your browser does not support the video tag.

And isn’t easier to follow it there?

Your browser does not support the video tag.

On my first attempt, I built the color map via trial and error, trying to get it to look like other tools I was familiar with.

But a patron reached out to let me know about the perceptually uniform color map collection “colorcet”.

![Image 5: A collection of colorcet gradients](https://cdn.fasterthanli.me/content/articles/making-our-own-spectrogram/colorcet-samples@2x~deb3daf9c024db63.jxl)

It turns out there’s a Rust crate for it, and for color gradients in general:


```
spectrogram on  main [!] via 🦀 v1.90.0
❯ cargo add colorcet colorgrad
    Updating crates.io index
      Adding colorcet v0.2.1 to dependencies
      Adding colorgrad v0.7.2 to dependencies
             Features:
             + named-colors
             + preset
             - ggr
             - lab
    Updating crates.io index
    Blocking waiting for file lock on package cache
     Locking 14 packages to latest Rust 1.90.0 compatible versions
      Adding colorcet v0.2.1
      Adding colorgrad v0.7.2
      Adding csscolorparser v0.7.2
      Adding phf v0.11.3
      Adding phf v0.13.1
      Adding phf_generator v0.11.3
      Adding phf_generator v0.13.1
      Adding phf_macros v0.11.3
      Adding phf_macros v0.13.1
      Adding phf_shared v0.11.3
      Adding phf_shared v0.13.1
      Adding rand v0.8.5
      Adding rand_core v0.6.4
      Adding siphasher v1.0.1
```

```
use std::sync::LazyLock;

static COLORMAP_GRADIENT: LazyLock<LinearGradient> = LazyLock::new(|| {
    let cmap: ColorMap = "fire".parse().unwrap();
    cmap.try_into().unwrap()
});

/// Return the color to use for a given (normalized) decibel level
fn colormap(value: f32) -> Rgb {
    let [r, g, b, _a] = COLORMAP_GRADIENT.at(value).to_rgba8();
    Rgb(r, g, b)
}
```
[Frequency mapping -----------------](https://fasterthanli.me/articles/making-our-own-spectrogram#frequency-mapping)
And then we arrive to the vertical axis. If we naively plot 20 Hz to 20 Khz linearly, we get something like this:

Your browser does not support the video tag.

Which looks nice and perfectly showcases harmonics — spaced evenly — but doesn’t really match the way humans perceive pitch, and gives way too much real estate to the 10Khz-20KHz range, where nothing really interesting happens, while squishing together a bunch of lower frequencies where a LOT of things happens.

Our fix is a two-parter: first, use a logarithmic scale instead of a linear one:

Your browser does not support the video tag.

Better, but it’s still giving too much space to very low and very high frequencies: just like the color map, we can make our own scale here:


```
fn get_frequency_breakpoints(max_freq: f32) -> [(f32, f32); 8] {
    [
        (0.0, 20.0),     // 0% -> 20 Hz
        (0.05, 100.0),   // 5% -> 100 Hz
        (0.15, 500.0),   // 15% -> 500 Hz
        (0.3, 1000.0),   // 30% -> 1 kHz
        (0.45, 2000.0),  // 45% -> 2 kHz
        (0.7, 5000.0),   // 70% -> 5 kHz
        (0.9, 10000.0),  // 90% -> 10 kHz
        (1.0, max_freq), // 100% -> just below Nyquist
    ]
}
```

And do log interpolation between those:


```
// Non-linear frequency mapping functions using piecewise scaling
fn normalized_to_frequency(normalized_y: f32, sample_rate: f32) -> f32 {
    let max_freq = sample_rate * 0.499;
    let breakpoints = get_frequency_breakpoints(max_freq);

    // Find which segment we're in
    for i in 0..breakpoints.len() - 1 {
        let (y1, f1) = breakpoints[i];
        let (y2, f2) = breakpoints[i + 1];

        if normalized_y >= y1 && normalized_y <= y2 {
            // Linear interpolation within this segment
            let t = (normalized_y - y1) / (y2 - y1);

            // Use log interpolation for frequency
            let log_f1 = f1.ln();
            let log_f2 = f2.ln();
            let log_freq = log_f1 + t * (log_f2 - log_f1);
            return log_freq.exp();
        }
    }

    breakpoints[breakpoints.len() - 1].1
}
```

This gives, in my opinion, a pretty good compromise:

Your browser does not support the video tag.

I honestly didn’t realize there were so many design choices to make when making a spectrogram! It’s a tool, and like any tool, it has to be adjusted to be useful to whoever uses it.

[Audio input -----------](https://fasterthanli.me/articles/making-our-own-spectrogram#audio-input)
Let’s now talk about some of the technical aspects. Last time I made a program that processed audio, in [The science of loudness](https://fasterthanli.me/articles/the-science-of-loudness), I had it load audio files from disk.

This time I wanted to be able to plot any sound that played on my computer: I own a copy of [RogueAmoeba’s Loopback](https://rogueamoeba.com/loopback/), which makes it particularly easy:

![Image 6: RogueAmoeba's Loopback utility showing a virtual audio device named Loopback Audio, monitored by my MacBook Pro Speakers.
](https://cdn.fasterthanli.me/content/articles/making-our-own-spectrogram/loopback@2x~6d14436413d20bb0.jxl)

Audio applications, like Apple’s Music app, or even web browsers, simply output to a virtual audio device instead of my MacBook Pro’s speakers. That virtual device shows up as an input, just like a microphone would:

![Image 7: List of macOS input devices showing MacBook Pro Microphone, dusk Microphone (my phone), and Loopback Audio.
](https://cdn.fasterthanli.me/content/articles/making-our-own-spectrogram/macos-input-devices@2x~b549e8cf8c1ecb08.jxl)

Which means we “just” have to capture audio from a chosen input device.

The [cpal](https://lib.rs/crates/cpal) crate makes this relatively easy. After picking an input device, we can create a stream by passing a callback, like so:


```
device.build_input_stream(
    &config.config(),
    move |data: &[f32], _: &_| {
        // do something with data
    },
    err_fn,
    None,
)?
```

That’s for 32-bit floating samples: if you want to support other sample formats, you have to match on the config — not all details are hidden from you:


```
let stream = match config.sample_format() {
    cpal::SampleFormat::F32 => {
        device.build_input_stream(
            &config.config(),
            move |data: &[f32], _: &_| {
                // do something with f32 samples
            },
            err_fn,
            None,
        )?
    }
    cpal::SampleFormat::I16 => {
        device.build_input_stream(
            &config.config(),
            move |data: &[i16], _: &_| {
                // do something with i16 samples
            },
            err_fn,
            None,
        )?
    }
    sample_format => {
        return Err(format!("Unsupported sample format: {sample_format:?}").into());
    }
};
```

It’s important to note that we don’t control when this callback is called. Once we “play” the stream, and until we drop it, it’s being called by the audio system, regularly, whenever there’s a bufferful of samples available.

[Drawing the UI --------------](https://fasterthanli.me/articles/making-our-own-spectrogram#drawing-the-ui)
Similarly, on the user interface side, we’re interacting with a system that runs at its own pace.

With [egui](https://lib.rs/crates/egui) and [eframe](https://lib.rs/crates/eframe), we roughly do this:


```
eframe::run_simple_native(
    "spectrogram",
    NativeOptions {
        viewport: ViewportBuilder::default().with_inner_size(vec2(1100.0, 600.0)),
        ..Default::default()
    },
    move |ctx, _frame| {
        // do something with ctx
    }
).unwrap();
```

…where the closure is called whenever eframe decides it’s time to draw again. We’re not in control here either!

So we have two separate event loops which we don’t control — how do we communicate between those? The short answer is: channels.

At the beginning of main, we create a bounded channel of capacity 10 (just so it doesn’t fill up all available memory):


```
// Create channel for passing audio buffers
    const CHANNEL_CAPACITY: usize = 10;
    let (sender, receiver) = bounded::<(Vec<f32>, f32)>(CHANNEL_CAPACITY);
```

In our audio callback, we downmix all audio channels to mono by averaging them, collect mono samples into a buffer, and as soon as it contains enough samples, we send it through the Rust channel (let’s not mix up audio channels and Rust channels):


```
// Convert to mono and collect samples
for chunk in data.chunks(channels) {
    let mono_sample = chunk.iter().sum::<f32>() / channels as f32;
    sample_buffer.push(mono_sample);

    // Send buffer when we have STEP_SIZE samples
    if sample_buffer.len() == STEP_SIZE {
        if sender
            .try_send((sample_buffer.clone(), sample_rate))
            .is_err()
        {
            // Channel full, skip this buffer
        }
        sample_buffer.clear();
    }
}
```

Another thread receives those samples and copies them to its own buffer:


```
fn process_audio(receiver: Receiver<(Vec<f32>, f32)>, model: Model) {
    let mut window = vec![0.0f32; WINDOW_SIZE];
    let mut windowed_samples = vec![0.0f32; WINDOW_SIZE];
    let mut window_pos = 0;
    let hann_window = compute_hann_window(WINDOW_SIZE);
    let power_gain: f32 = hann_window.iter().map(|&w| w * w).sum::<f32>() / WINDOW_SIZE as f32;

    while let Ok((chunk, sample_rate)) = receiver.recv() {
        // Copy the chunk into our window
        for sample in chunk {
            window[window_pos] = sample;
            window_pos += 1;

            if window_pos == WINDOW_SIZE {
                 // ✂️ cut for clarity
            }
        }
    }
}
```

And when _that_ buffer is full, first we apply the Hann window, using coefficients we’ve pre-calculated outside the loop, then we perform the fast fourier transform on that:


```
for (i, sample) in window.iter().enumerate() {
    windowed_samples[i] = sample * hann_window[i];
}

let fft = windowed_samples[..].real_fft();
```

The result of that operation is an array of complex numbers, which we’ll manipulate a bit.


```
let mut fft_magnitudes: Vec<f32> = Vec::with_capacity(fft.len() - 1);
for i in 0..(fft.len() - 1) {
    // ✂️ cut for clarity
}
```

First, we’re only interested in the magnitude, so we’ll take the norm of the complex number:


```
let magnitude = fft[i].norm();
```

You can imagine complex numbers as two-dimensional vectors: with x as the real component and y as the imaginary component. The norm gives us the length of that vector.

Then, we don’t actually want to plot amplitudes, we want to plot powers, so we’ll need to square it:


```
let power = magnitude * magnitude;
```

To get numbers between 0 and 1, we need to normalize the output of the FFT, which depends on the number of frequency bins:


```
let normalized_power = power / (WINDOW_SIZE as f32 * WINDOW_SIZE as f32);
```

Next up, we need to compensate for the Hann window tapering off at the edge: it reduces the overall power by a factor we’ve calculated ahead of time:


```
let corrected = normalized_power / power_gain;
```

We’re dividing by a value that’s smaller than 1 (approx 0.375), so the result is actually larger.

We’re not done yet! For every bin except the first, which is just the DC offset, we need to double the value:


```
// Double power for one-sided spectrum (except DC bin)
let final_power = if i == 0 {
    corrected // DC bin: don't double
} else {
    2.0 * corrected // All other bins: double
};
```

And finally, we convert that power to decibels, using a 10x factor:


```
// Convert to dB (10*log10 for power)
let db_value = 10.0 * (final_power.max(1e-20)).log10();
fft_magnitudes.push(db_value);
```

That `fft_magnitudes` array is what we’ll be drawing from the UI thread: we need to send it over somehow!

We could use channels here too, but we’re already drawing at a regular rate: in the eframe callback, the very first thing we do is request a repaint.


```
eframe::run_simple_native(
    "spectrogram",
    Default::default(),
    move |ctx, _frame| {
        ctx.request_repaint(); // 👈

        // drawing happens here
    }
).unwrap();
```

If we didn’t, to save energy, our app would only repaint when someone presses the keyboard, clicks the mouse, etc.

So, we have that infinite loop running somewhere close to 60Hz (hopefully), and from there, we can check a mutex:


```
type Model = Arc<Mutex<ModelInner>>;

#[derive(Default)]
struct ModelInner {
    ffts: VecDeque<Vec<f32>>,
    sample_rate: f32,
}
```

Those are the only things shared between the audio processing thread and the UI thread: we need the sample rate because consumer audio hardware can’t pick between 44.1Khz and 48Khz, and the VecDeque of ffts is what we just calculated: the audio processing thread pushes new ones to the back:


```
// Update model
let mut guard = model.lock().unwrap();
guard.ffts.push_back(fft_magnitudes);

// Shift window by STEP_SIZE
window.copy_within(STEP_SIZE.., 0);
window_pos = WINDOW_SIZE - STEP_SIZE;

// Zero-fill the end
window.iter_mut().skip(window_pos).for_each(|x| *x = 0.0);
```

And when drawing the UI, we pop some from the front!


```
fn redraw(frame_buffer: &mut [Color32], width: usize, height: usize, state: &mut State) {
    let mut guard = state.model.lock().unwrap();
    let sample_rate = guard.sample_rate;

    // here we pop! 👇
    while let Some(model) = guard.ffts.pop_front() {
        let x = state.index;

        state.index += 1;
        if state.index == width {
            state.index = 0;
        }

        // ✂️ cut: draw column `x`
    }
}
```

In summary, our data flow looks a little bit like this:

![Image 8](https://cdn.fasterthanli.me/content/articles/making-our-own-spectrogram/data-flow~a8fad2ae902f7f8f.svg)

We want the cpal callback to be as quick as possible, which is why we send samples down a channel for the FFT to be done in another thread. That fft thread does computations on its own time and places the result back in a VecDeque when they’re ready. And the UI thread pops them from there when it has a chance to draw something new.

However, I’m not entirely happy with this design: can you think of a situation in which this design would be dangerous, or at least non-optimal?

Click for the answer
> The `VecDeque` is not bounded. If the audio callback keeps firing but drawing is blocked somehow (maybe the window is minimized?), the fft thread will keep pushing to it, and we could eventually run out of memory.
> 
> 
> A simple fix would be to stop pushing if the `VecDeque` is larger than a certain size.

[Updating the texture --------------------](https://fasterthanli.me/articles/making-our-own-spectrogram#updating-the-texture)
There is one last technical bit I want to cover: most of the UI (the input picker, the color map, the frequency map, etc.) is drawn from scratch on every frame. That’s just what immediate-mode GUIs, like egui, do.

But I didn’t want to do that for the spectrogram itself because… we’re drawing individual pixels, one column at a time. Epaint lets us draw rectangles, but, uhh, that seems suboptimal.

It would be much easier if we could just create some sort of image, or texture, or pixel buffer, that we could update whenever a new column comes in!

Well, egui supports this, although there’s a lot more copying than I’d like. Here’s what I ended up with.

Outside of the eframe loop, we prepare some things:


```
let mut texture: Option<egui::TextureHandle> = None;
let size = [3700, 2048];
let mut img = egui::ColorImage::filled(size, Color32::BLACK);
let mut state = State {
    model,
    index: 0,
    buffer: None,
};
```

A texture handle, which is initially `None`, a dummy image, and a state object that will be borrowed mutably by the paint callback.

The `State` struct keeps track of the model, which we share with the FFT thread:


```
struct State {
    model: Model,
    index: usize,
}
```

Inside the paint callback, we allocate a texture handle (if we don’t have one yet):


```
let texture = texture.get_or_insert_with(|| {
    ctx.load_texture(
        "spectrogram",
        egui::ColorImage::filled([1, 1], Color32::WHITE),
        Default::default(),
    )
});
```

Then draw onto an image and update the texture with it:


```
redraw(&mut img.pixels[..], size[0], size[1], &mut state);
texture.set(img.clone(), Default::default());
```

Finally, deep in egui drawing, we get to call `ui.image()` with our texture:


```
// ✂️ cut: more UI code

ui.horizontal(|ui| {
    egui::Frame::default()
        .stroke(Stroke::new(1.0, Color32::DARK_GRAY))
        .show(ui, |ui| {
            ui.image((texture.id(), texture.size_vec2() * 0.25));
        });
})

// ✂️ cut: even more UI code
```

There’s a smarter move here, since we’re only drawing one column at a time, which is to use [set_partial](https://docs.rs/egui/0.32.1/egui/struct.TextureHandle.html#method.set_partial) to only change _part_ of the texture, but uhhh… computer fast enough.

You may have noticed the `0.25` factor in the texture size: I added this because my computer display is high-DPI: that’s why I made the texture so dang large (3700x2048), we’re effectively drawing at 4x.

If it runs poorly on your computer, you can probably turn down the size and turn up the scale factor. For me, it runs in real-time, but it is a CPU hog:

![Image 9: spectrogram is using 52% CPU, and WindowServer is using 45% CPU.
](https://cdn.fasterthanli.me/content/articles/making-our-own-spectrogram/cpu-hog@2x~5182cca9a4774a79.jxl)

[Profiling my program --------------------](https://fasterthanli.me/articles/making-our-own-spectrogram#profiling-my-program)
Speaking of… _why_ is it a CPU hog? What is it spending all its time doing?

It’s time to whip out Apple’s Instruments — I’m lucky to have a recent enough mac that I can use [Processor Trace](https://developer.apple.com/documentation/xcode/analyzing-cpu-usage-with-processor-trace) on it.

All I need is to enable some debug information for my release build:


```
# in `Cargo.toml`

# ✂️ cut: package info, dependencies, etc.

[profile.release]
debug = 1
split-debuginfo = "packed"
```

The `split-debuginfo` setting is needed to generate `.dSYM` bundles we’ll need later.

And then, after `cargo build --release`, to add some entitlement to the compiled binary.

The `hello.entitlements` file contains:

󰗀
```
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
	<key>com.apple.security.get-task-allow</key>
	<true/>
</dict>
</plist>
```

And it’s applied with:


```
$ codesign -s - -f --entitlements ~/hello.entitlements target/release/spectrogram
```

After that, I was able to start a new processor trace profile:

![Image 10: The Instruments app's template picker, with processor trace highlighted.
](https://cdn.fasterthanli.me/content/articles/making-our-own-spectrogram/instruments-processor-trae@2x~9af2d87e00bf8927.jxl)

I recorded only for 1 second, which was already extremely heavy: to be honest, regular sampling would’ve been more than enough for this, but it’s fun to be able to say for sure that we spend 25% of CPU cycles on cloning `ColorImage`, here:


```
texture.set(img.clone(), Default::default());
```

![Image 11: A screenshot of Instruments showing that we spend 25% of our time in ColorImage::clone
](https://cdn.fasterthanli.me/content/articles/making-our-own-spectrogram/colorimage-25percent@2x~85c806391a4f88ec.jxl)

Or that we spend 40% of our time, overall, uploading textures from the CPU to the GPU:

![Image 12: The same screenshot with glTexImage2D highlighted
](https://cdn.fasterthanli.me/content/articles/making-our-own-spectrogram/glteximage2d@2x~48c88658e1cba77d.jxl)

…via glTexImage2D, an OpenGL function call, which, on macOS, is implemented by `AppleMetalOpenGLRenderer`.

We can find out man other things: that egui runs tessellation on the CPU, which is unsurprising.

![Image 13: Stack frames related to tessellation: here, mostly rectangles.
](https://cdn.fasterthanli.me/content/articles/making-our-own-spectrogram/tessellation@2x~38a9e015aaa82f71.jxl)

Tessellation is, roughly, transforming shapes into polygons, so they can be rendered by the GPU.

And that’s just the main thread! Here’s our audio processing thread, spending 27% of its time performing the FFT:

![Image 14: Stack traces relative to performing the FFT
](https://cdn.fasterthanli.me/content/articles/making-our-own-spectrogram/processings-thread-fft@2x~7ff40ea4da9225fb.jxl)

I see a few threads related to CoreAnimation, and.. a `com.apple.audio.IOThread.client`!

![Image 15: Stack traces related to our audio input callback, with Vec::clone highlighted
](https://cdn.fasterthanli.me/content/articles/making-our-own-spectrogram/vec-clone-in-audio@2x~bbda0a63e5204fbb.jxl)

Spending about 7% cloning its sample buffer, and another 7% in try_send! Our channel business is not free.

[Having fun ----------](https://fasterthanli.me/articles/making-our-own-spectrogram#having-fun)
So, yeah, there are optimizations opportunities. But I didn’t build this spectrogram to be the fastest or most optimized.

I built it to have fun.

So let’s have fun! So many songs are really nice to watch as a spectrogram.

Aretha Franklin’s This Bitter Earth shows wonderful use of vibrato in her voice:

Your browser does not support the video tag.

When electronic music makes uses of frequency sweeps, it’s immediately visible on the spectrogram, like in in the intro of Icarus by Madeon:

Your browser does not support the video tag.

Chiptune is always entertaining to look at — here’s [Discovery by Meganeko](https://meganeko.bandcamp.com/track/discovery):

Your browser does not support the video tag.

Pop songs often have very tight, very clean mixes and make intentional use of negative space in the frequency realm: Sub Urban’s Cradles starts out with just middle frequencies and suddenly expands as the drums and vocals come in:

Your browser does not support the video tag.

Compare and contrast with this 1954 recording of Line Renaud singing “Je ne sais pas”, which contains background noise in the low frequencies long before the bass enters:

Your browser does not support the video tag.

Radiohead’s Nude has a lot of things going on and is almost over-produced (and I mean that in a good way), and the spectrogram reflects that fully:

Your browser does not support the video tag.

The first few seconds 