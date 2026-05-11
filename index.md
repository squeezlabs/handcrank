---
layout: default
title: CrankGPT — fully offline, human-powered local AI 
description: A local, voice assistant running on a hand-crank-powered single board computer.
---

![CrankGPT, a red metal box with a hand crank, voltmeter and OLED display](pics/IMG_2672.jpg)

CrankGPT is a fully offline and off-the-grid AI box. Our current demos are variations on voice assistants—turn the crank, say something, get a response—but we've generated images (small), made poetry (bad), and written code using the same setup. There's no battery or cloud. Just a hand crank, a little computer, and a small stack of speech and language models running locally. Provided the electronics are kept dry and at a reasonable temperature, there's no reason this thing won't still work in a thousand years.

As will be familiar to anyone who has ever undertaken a hardware project, it took about a week to build a proof of concept and many months of kernel optimizations, board revisions, code refactors, and CAD tweaks to get to a thing that works as we envisioned. This article walks through how we built it: the hardware, the local voice agent stack, and the engineering required to **make a conversation feel real on a device this small**.

## Motivations

1. For something to have "smarts" currently assumes a wall socket and a data center. CrankGPT is a small argument that neither has to be true.
2. Local models are private models. Why give away what we don't have to?
3. It offended our European small-practical-car sensibilities to see people around us throwing kilowatts and thousands of tokens at massive models to do things much smaller ones could do just as well for a fraction of the cost and energy.
4. Everyone is so busy making things bigger that we figured there are lots of smaller opportunities available for the taking. 


## Hardware

### Single Board Computer

We used a stock Raspberry Pi 5 with 8GB RAM and a cooling fan HAT. There are better performing SBDs for the same price (an Orange Pi with its faster DDR5 RAM is an even better fit for LLM inference as we'll discuss below), but it's hard to beat the Pi's accessibility and software ecosystem. The Pi runs speech recognition, the language model, and text-to-speech locally on CPU (no accelerators).

### Audio

We used the [KEYESTUDIO ReSpeaker 2-Mic Pi HAT](https://www.amazon.com/dp/B07H3T8SQY): an all-in-one audio I/O solution for Pi designed specifically for voice assistants. It includes a stereo MEMS mic array and various audio outputs (we used the older version with the WM8960 codec). It sits directly on the Pi's GPIO headers and has decent far-field mic performance, even within an enclosure.

### Power

We chose a cheap off-the-shelf switchable voltage [20W hand-crank generator](https://www.amazon.com/dp/B0F52VY4KF) marketed for emergency USB charging. The Pi normally draws around 1.5A, but when it's working hard (as it does when doing inference on the CPU), its current requirements can increase substantially, causing the generator voltage to sag below the Pi's required 4.8V or even, in the case of a momentary 5A spike, to trigger the generator's internal overcurrent protection and shut off the voltage output entirely, causing the Pi to brown out.

To ensure the Pi sees a steady voltage when the full inference stack kicks in (and to afford crankers a little rest), we built a custom capacitor board to smooth out the generator's output and act as a short-term (~20 second) power reservoir.

You can *feel* that load curve through the crank: when LLM inference and speech synthesis run together, the crank gets a lot harder to turn.

## Software

### Operating system

When you're cranking, every second counts, so the minute or so it takes Raspian to boot up becomes an eternity. [DietPi](https://dietpi.com/) is a minimalistic, stripped-down Debian-based image that prioritizes fast boot time over lots of immediately available default services. It shrank our startup time substantially, and turning off unneeded radio services (Bluetooth, Wi-Fi, etc.) reduced it even further: from power-on to a usable userspace in around 3 seconds.

### Voice agent

We wrote our own [edge voice agent](https://github.com/ktomanek/edge_voice_agent) optimized for RPI-class boards. Our motivation for building this from scratch rather than on top of existing frameworks (like e.g. Pipecat): we wanted to understand the system end to end and have as few dependencies as possible. The pipeline is the obvious one—Automatic Speech Recogntion (ASR) + Voice Activity Detection (VAD) → LLM → Text-to-Speech (TTS)—with every stage tuned for minimal latency on CPU.

### Speech recognition

[Moonshine](https://github.com/usefulsensors/moonshine) ASR turned out to be [by far the fastest option](https://github.com/ktomanek/captioning#results) for CPU-based ASR. It's slightly less robust in noisy environments (relevant in our scenario with a noisy crank) and on accented speech compared to Whisper-base-sized models or NVIDIA's FastConformer. But we optimized for low latency given our goal of a real-time voice agent. For endpointing, we use [Silero VAD](https://github.com/snakers4/silero-vad).

### Language model(s)

The LLM runs on [llama.cpp](https://github.com/ggerganov/llama.cpp). Our preferred models are small [Liquid AI LFM2](https://www.liquid.ai/blog/liquid-foundation-models-v2-our-second-series-of-generative-ai-models) variants (e.g. 350M or 1.2B), along with [Gemma 3](https://deepmind.google/models/gemma/gemma-3/) in its 1B form.

Raspberry Pi 5 performance measured using llama.cpp (`llama-bench` with pp512 and tg128, 4 threads each:

| model | quant | memory | prefill t/s | gen t/s | 
| ----- | ----- | -----  | -----  | ----- |
| lfm2.5 350M |  Q4_K_M | 354.48 MiB | 222.65 ± 1.09 | 48.86 ± 0.02 |
| lfm2.5 1.2B |  Q4_K_M | 762.49 MiB | 71.31 ± 0.04 | 15.01 ± 0.01 |
| gemma3 1B |  Q4_K_M | 762.49 MiB | 46.12 ± 0.01 | 14.31 ± 0.01 |


Token generation is the biggest bottleneck in autoregressive decoding and is most constrained by memory bandwidth (not raw compute). This is clearly visible when comparing the prefill and generation rates on a Raspberry Pi 5 (DDR4 RAM) versus an Orange Pi 5 Pro (DDR5 RAM):

| model | quant | memory | prefill t/s | gen t/s | gen speedup |
| ----- | ----- | -----  | -----  | ----- | ----- |
| lfm2.5 350M |  Q4_K_M | 354.48 MiB | 221.46 ± 0.27 | 73.03 ± 2.34 | **+49%** |
| lfm2.5 1.2B |  Q4_K_M | 762.49 MiB | 67.68 ± 0.99 | 23.79 ± 0.20 | **+58%** |
| gemma3 1B |  Q4_K_M | 762.49 MiB | 39.47 ± 0.30 | 18.43 ± 0.58 | **+29%** |

Generation rates on the Orange Pi 5 Pro are 29-58% higher, mainly due to the significantly higher memory bandwidth of DDR5. (Note: on the Orange Pi 5 Pro we restricted execution to the 4 performance cores for a fair comparison.) 

Most larger LLMs—even those marketed as `edge-optimized`—are way too slow on either platform to be useful in a real-time voice agent. Single-digit token generation rates (e.g. Qwen 3.5 2B at 7.8 tok/sec) lead to response times with latency too high for conversation that feels anywhere close to real time.

### Text-to-speech

There's a growing list of natural-sounding, CPU-runnable voice models, but most simply don't run in real time on a Raspberry Pi. [Kokoro](https://github.com/hexgrad/kokoro), [KittenML](https://github.com/KittenML/KittenTTS), [PocketTTS](https://huggingface.co/kyutai/pocket-tts) and [Piper](https://github.com/OHF-Voice/piper1-gpl) are the likely contenders for low-resource edge inference. Piper wins by a large margin on latency and generation speed. [Concretely](https://github.com/ktomanek/edge_tts_comparison#non-streaming), on a Raspberry Pi 5, Piper synthesizes our 20-word test utterance in about 0.50s, while Kokoro is about 9× slower. PocketTTS does support streaming, which significantly reduces time-to-first-byte, but [its real-time factor (RTF) is still above 1.0 on a Raspberry Pi causing audible stuttering](https://github.com/ktomanek/edge_tts_comparison#streaming). 

Piper's headroom is what lets it keep up with streaming LLM output in a real conversation. The others just can't.

We stream the LLM's output sentence-by-sentence into Piper. To avoid pauses during playback, we cap the maximum sentence length, and we cap it more aggressively for the *first* sentence. That gets speech started as fast as possible without forcing the model to pre-commit to a short answer overall. The user hears the first words quickly, and the model keeps generating in the background while playback catches up.

### Runtime

All components run on ONNX Runtime, PyTorch dependencies (lingering in some components while not technically required) were removed to save RAM and improve startup time.

## Putting it together


### Startup Time

It takes about 30 seconds from the moment you start cranking to the moment you're having a conversation with CrankGPT. Startup time includes:

- **~10–15s** — Pi 5 cold boot through full firmware sequence
- **~3s** — Linux boot to userspace (DietPi)
- **~10-15s** — Voice Agent startup (python imports, loading model weights)


Even with all the obvious optimizations—BOOT_DELAY=0, splash disabled, unused boot sources removed, fastest available SD card—the Pi 5's pre-Linux stage still costs us ~10–15 seconds. Unlike the Pi 4, the Pi 5 runs a much more PC-like firmware sequence (PMIC ramp, RP1 init, PCIe/USB enumeration via the EEPROM bootloader) before it ever loads a kernel, and that floor is hard to break through from userland. And unfortunately, Pi 5 doesn't have a sleep mode/DRAM preservation, so every time the voltage drops below its minimum requirements, you have to start from zero.

During voice agent startup, the slowest part are Python imports on first run. We tried the obvious fixes and none of them helped meaningfully. Precompiling bytecode (`compileall`) was a no-op—Python already caches `.pyc` files automatically, so there was nothing left to compile on a warm install. Lazy imports trimmed a few hundred milliseconds at best; the bulk of cold-start time isn't in our code, it's in `dlopen`-ing large shared libraries (ONNX Runtime in particular) and in hundreds of small random reads off the SD card as Python walks the import graph. Warming the page cache after boot helped only marginally because for the first invocation the page cache *is* cold by definition.

NVMe was the most surprising dead end. Faster random reads should have been a clear win, but on the Pi 5 the EEPROM bootloader has to enumerate PCIe and load the NVMe controller's firmware before it can boot, adding roughly 10 seconds to the pre-Linux stage. What we gained at runtime, we lost at boot, and then some. For our use case—cold start every session—an SD card ended up being faster end to end.

To reduce startup time further, dropping the Python layer and replacing the agent glue with a C (or Rust) version could probably save another ~5s of startup time. We leave this as an exercise to the reader.


### Latency Measurements

Our goal was to build a fully offline and off-the-grid voice agent that allows for smooth, real-time conversations without the multi-second latencies often seen in demonstrations of local voice agents. As described above, LLM selection (and its respective token generation rate) is the primary driver of the time-to-first-byte (TTFB) the user perceives. Below are some measurements from a typical conversation, averaging TTFB across all turns:

| LLM used              | TTFB  |
| --------------------- | ------- |
| Gemma3 1b             | ~2.9 sec |
| LFM2.5 1.2b           | ~1.5 sec|
| LFM2.5 350m           | ~0.8 sec|


### Practical Power Needs

CrankGPT's power draw really depends on the amount of AI inference running. The Pi is picky about its voltage, which needs to be between 4.8V and 5.3V, so the interesting variable is current. We've observed brief current spikes of up to 5A under maximum load. Here are a few common scenarios:

| Scenario                      | Voltage | Current | Power  |
| ---------------------         | ------- | ------- | ------ |
| Idle (just keep the Pi alive) | ~5 V    | ~0.8 A  | ~4 W   |
| ASR (Moonshine)       | ~5 V  | ~1.6 A  | ~8 W   |
| LLM + TTS inference   | ~5 V  | ~3 A    | ~15 W  |


