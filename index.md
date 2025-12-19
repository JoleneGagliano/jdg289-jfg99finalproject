---
title: DJ Mixing Board Final Project
---

# DJ Mixing Board Final Project

## Project Introduction

**One-sentence sound bite:**  
A real-time embedded DJ mixing board that allows a user to mix two audio sources with adjustable volume and filtering using physical controls.

**Summary (what we did and why):**  
This project implements a real-time embedded DJ mixing board on the RP2040 microcontroller that mixes two audio channels with user-controlled gain and filtering while also supporting triggered playback of stored audio samples. The system processes audio in real time using ADC inputs, digital filtering, and DAC output, demonstrating core embedded systems concepts including concurrency, interrupt-driven timing, and hardware–software integration.

The purpose of this project was to explore real-time audio signal processing in an embedded environment. Audio mixing was chosen because it is highly interactive and immediately exposes timing or processing errors through audible artifacts. The project combines continuous analog inputs, digital control via potentiometers, and deterministic real-time execution, making it a strong application for studying real-time constraints and signal flow.

---

## High-Level Design

### Rationale and Sources of the Project Idea
The idea for this project was inspired by physical DJ controllers and audio mixers, which provide a familiar interface for manipulating sound using knobs and buttons. These systems map naturally to embedded design challenges such as strict timing requirements, concurrency between tasks, and user interaction through hardware controls. By implementing a DJ-style mixing board, the project emphasizes real-time performance while remaining intuitive to use and debug.

Additionally, the theme for this project was to do something that we really enjoyed and would be motivated to get working. This manifested itself into the final DJ mixing board being something we could play with and have fun with while testing. That is to say, we had a direct interest in the project we chose.

---

## Background Math

The mathematical foundation of the project includes linear gain scaling, first-order low-pass filtering, and amplitude limiting to approximate aggressive high-pass behavior. The low-pass filters are implemented using the standard recursive form:

\[
y[n] = y[n-1] + a\left(x[n] - y[n-1]\right)
\]

where the coefficient \(a\) is dynamically adjusted using a potentiometer input.

---

## Logical Structure

The system processes two independent audio channels using the RP2040’s ADC inputs on GPIO26 (ADC0) and GPIO28 (ADC2). Each channel has its own gain, low-pass filter, and high-pass cutoff controlled by potentiometers connected through an external ADS7830 ADC accessed over I²C. Audio samples are processed digitally and output through an SPI-connected dual-channel DAC.

At each timer interrupt, the system reads ADC values, applies filtering and gain, optionally overrides the signal with stored audio samples, and writes the result to the DAC. Because we are using mono audio, each interrupt outputs a value to both the left and right speakers. The interrupt toggles which song it is outputting for the interrupt, resulting in the “mixing” of the two audios.

A second timer runs at a much lower rate to update potentiometer values over I²C, decoupling slow control updates from the high-speed audio path.

---

## Hardware/Software Tradeoffs

### Hardware vs. Software Filters
We use both hardware and software filters.

Our hardware filtering includes a voltage biasing / DC-blocking filter that cleans up the input audio to contain only a clean AC input from the 3.5mm audio jack, while biasing the waveform around 1.65 V to keep our ADC inputs within 0–3.3 V.

**Filter schematic:**  

Our software filters allow adjustability and control over the effects we want to impose on our sampled audio. Although the inputs are analog, we get full control over how we interpret those values (0–3.3 V) to drive our digital low-pass and high-pass filters.

**Filter code snippet:**  
```c
// paste your filter code snippet here
