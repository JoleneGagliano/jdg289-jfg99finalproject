---
title: DJ Mixing Board Final Project
---

# DJ Mixing Board – Final Project

## Project Introduction

This project is a real-time embedded DJ mixing board that allows a user to mix two audio sources with adjustable volume and filtering using physical controls.
 
 This project implements a real-time embedded DJ mixing board on the RP2040 microcontroller that mixes two audio channels with user-controlled gain, low-pass filtering, and high-pass behavior, while also supporting triggered playback of stored audio samples. The system processes audio in real time using ADC inputs, digital filtering, and DAC output, demonstrating core embedded systems concepts including concurrency, interrupt-driven timing, and hardware–software integration.
 The primary goal of the project was to explore real-time audio signal processing in an embedded environment. Audio mixing was chosen because it is inherently timing-sensitive: even small deviations in sampling rate or processing latency immediately manifest as audible artifacts such as distortion, dropouts, or aliasing. By combining continuous analog inputs, digitally sampled control signals from potentiometers, and deterministic interrupt-based execution, this project provides a strong platform for studying real-time constraints, signal flow, and performance tradeoffs in embedded systems.


## High level design

The idea for this project was inspired by physical DJ controllers and audio mixers, which provide a familiar interface for manipulating sound using knobs and buttons. These systems map naturally to embedded design challenges such as strict timing requirements, concurrency between tasks, and user interaction through hardware controls. By implementing a DJ-style mixing board, the project emphasizes real-time performance while remaining intuitive to use and debug. Additionally, a key motivation was choosing a project that we were personally interested in and excited to build. This direct interest helped drive the project forward and resulted in a final system that was both functional and enjoyable to use during testing.

The system processes two independent audio channels using the RP2040’s ADC inputs on GPIO26 (ADC0) and GPIO28 (ADC2). Each channel has its own adjustable gain, low-pass filter strength, and high-pass cutoff. These parameters are controlled by six potentiometers connected to an external ADS7830 ADC, which is accessed over an I²C bus. Digitally processed audio samples are sent to an external dual-channel SPI DAC, which generates the final analog output.

At each interrupt of the high-speed repeating timer, the system reads the current ADC values, applies gain and filtering operations, optionally overrides the signal with stored audio samples, and writes the processed value to the DAC. As we are using mono audio, each interrupt outputs a value to both the left and right speakers. The interrupt toggles which song it is outputting for the interrupt, resulting in the mixing of the two audio streams. A second timer runs at a much lower rate to update potentiometer values over I²C, decoupling slow control updates from the high-speed audio path.

## background math

The mathematical foundation of the project includes linear gain scaling, first-order low-pass filtering, and amplitude limiting to approximate aggressive high-pass behavior. The low-pass filters are implemented using the standard recursive form y[n] = y[n−1] + a(x[n] − y[n−1]) where the coefficient a is dynamically adjusted using a potentiometer input.

## hardware/software tradeoffs

This project uses both hardware and software filtering to balance signal integrity and flexibility. Our implementation of a hardware filter includes a voltage biasing dc blocking filter that cleans up the input audio to only contain a clean ac input from the 3.5mm audio jack, while biasing our waveform around 1.65V as to keep our ADC inputs within 0-3.3V.

<img width="841" height="294" alt="Screenshot 2025-12-19 at 7 56 26 PM" src="https://github.com/user-attachments/assets/0abe601d-4c20-4f68-842c-6aae6481cf13" />
*Figure 1: Protoboard Schematic*

Our software filters allow for adjustability and control over the effects we want to impose on our sampled audio. Though the inputs are analog, we get full control over how we interpret those values of 0-3.3V, to drive our digital low-pass and high-pass filters.

```c
adc_select_input(0);               // select ADC0 (GPIO26)
uint16_t raw0 = adc_read();        
DAC_output_0 = (uint16_t)(raw0 * level_a); // scale by level_a
float alpha_l = 0.02f + lp_a * 0.4f;  //low pass
float x = (float)raw0;
y_a = y_a + alpha_l * (x - y_a);
```
*Applying scaling and filters to output*

Initially, the project envisioned streaming audio from a computer over the RP2040’s USB interface, eliminating the need for discrete analog audio inputs. This idea was reflected in the original PCB design, which does not directly connect 3.5 mm audio jacks to the RP2040’s ADC pins. However, it quickly became clear that implementing real-time USB audio streaming would be significantly more complex and time-consuming than building the inputs out in hardware, which we did using a protoboard that connected to the broken-out ADC inputs.

<img width="600" alt="Screenshot 2025-12-19 at 7 45 27 PM" src="https://github.com/user-attachments/assets/844c2762-d327-435f-b462-343ac22ba728" />
*Figure 2: Protoboard*

## Intellectual property considerations

The project does not infringe on existing patents, trademarks, or copyrighted designs. Audio samples used during testing were sourced from open platforms and were used strictly for educational purposes. No monetization is involved, and the use of audio content falls under fair use guidelines for academic and non-commercial projects.

## Program/hardware design

The software architecture is structured around two repeating timers. The primary timer runs at approximately 45 kHz and executes the sample_and_output_cb function, which performs all real-time audio processing. This sampling rate was chosen based on the Nyquist sampling theorem, which states that a signal must be sampled at least twice its highest frequency component to avoid aliasing. Since human hearing extends up to approximately 20 kHz, a sampling rate above 40 kHz ensures accurate reconstruction of the audible spectrum.

Within each interrupt, the function reads both ADC channels, applies gain scaling and filtering, manages playback triggers, and writes the resulting value to the DAC. Gain is applied as a linear scaling factor derived from the potentiometer input, while low-pass filtering is implemented using a first-order low-pass filter. The filter coefficient alpha is dynamically adjusted based on a potentiometer value, giving the user real-time control over the effective cutoff frequency.

```c
float alpha = 0.02f + lp_a * 0.4f;
y_a = y_a + alpha * (x - y_a);
float out = y_a * level_a;
```
*Low pass filter*

Our high-pass filter functions as a frequency cutoff by not playing certain frequencies based on a potentiometer value. This resulted in a cool audio effect that we chose to keep.

To achieve mixing behavior without doubling the computational load, the system alternates between processing the two audio channels on successive interrupts. A toggle variable determines which channel is processed on each invocation, effectively interleaving samples from the two sources at the DAC. Because the interrupt rate is high, this interleaving produces a perceptually smooth mix without audible artifacts.

The processed sample is then transmitted to the external dual-channel DAC over SPI. SPI communication was chosen for its high throughput and deterministic timing compared to other serial protocols.

A second repeating timer, running at a much lower rate (every 10 ms), executes the read_pot_cb function. This callback reads six potentiometers via an external ADS7830 ADC over I²C. These potentiometers control gain, low-pass strength, and high-pass cutoff for both audio channels.

```c
if (i2c_write_blocking(I2C_CHAN, ADDRESS, &cmd_lp_a, 1, true) < 0)
    return true;
i2c_read_blocking(I2C_CHAN, ADDRESS, &pot_val_lp_a, 1, false);
lp_a = (float)pot_val_lp_a / 255.0f;
```

*Example pot reading*

The filters, particularly the high-pass behavior, were the most challenging aspect to implement correctly without complete audio loss. We also initially struggled with timing, as if we attempted to do too much logic in the read and write timer, then the we would not meet timing requirements for the DAC resulting in no sound playing. This is when we introduced a slower timer to handle the readings of the potentiometers.

hardware details. Could someone else build this based on what you have written?  
The hardware consists of an RP2040 microcontroller, two analog audio inputs, an external ADS7830 ADC for potentiometer readings, and a dual-channel SPI DAC for audio output. GPIO pins are clearly defined for SPI, I²C, ADC, and digital input, and the code documents all pin assignments and communication protocols. A custom PCB designed in Altium was used for the main system, while the analog filters and audio jacks were implemented on a protoboard.

The pcb was placed in the enclosure and the lid was placed over it. Knobs were then placed on the potentiometers for better grip. One computer was connected to the RP2040 and the debugger from which the board was powered and flashed. The two audio inputs were then connected to computers which independently stream songs into. The audio output was then connected to the speaker. An oscilloscope was also connected to the ground pin on the protoboard, as this helped reduce noise on the sounds.

<img width="600" alt="Screenshot 2025-12-19 at 7 59 55 PM" src="https://github.com/user-attachments/assets/40adf742-a9fc-491d-9367-c9d235c94af3" />

*Figure 3: Custom PCB and Protoboard Schematic*


<img width="600" alt="Screenshot 2025-12-19 at 8 04 27 PM" src="https://github.com/user-attachments/assets/b8ee368a-e1f2-4f15-8f74-5ca3c67940ee" />

*Figure 4: Custom PCB*


<img width="600"  alt="Screenshot 2025-12-19 at 8 05 02 PM" src="https://github.com/user-attachments/assets/958d3604-203c-4bf4-8377-29806b77ed12" />

*Figure 5:Integrated DJ Mixing Board*


<audio controls preload="none">
  <source src="StarshipsXOneMoreTime.m4a" type="audio/mpeg">
  Your browser does not support the audio element.
</audio>

*Mix 1: Starships X One More Time*

<audio controls preload="none">
  <source src="Cornell University.m4a" type="audio/mpeg">
  Your browser does not support the audio element.
</audio>

*Mix 2: Starships X One More Time*


Several features were considered but ultimately not implemented, including advanced playback control, reverb effects, and BPM adjustment. These features would require significantly more memory than is available on the RP2040, as well as the addition of external memory hardware. Given time and hardware constraints, these ideas were deferred in favor of ensuring stable real-time audio performance.

AI tools were used selectively to assist with debugging and troubleshooting during development. Lines in code generated in AI are commented with “ // AI Help “.

## Results of the design

<img width="600" alt="Screenshot 2025-12-19 at 8 00 58 PM" src="https://github.com/user-attachments/assets/81d3c4a9-0a48-4979-9e54-8f3eefca2557" />
*Figure 6:Pre-1.65V Bias Audio Input*

<img width="600" alt="Screenshot 2025-12-19 at 8 02 07 PM" src="https://github.com/user-attachments/assets/b46f080d-25e0-4fc9-a9dd-1e324bb5f479" />
*Figure 7: Post-1.65V Bias Audio Input*

The system executes fast enough to meet all real-time constraints. No audible hesitation, flicker, or dropouts were observed during operation. Alternating SPI writes between DAC channels proved sufficient to maintain consistent output timing and audio quality.

The output audio sounded clear and clean, with no noticeable distortion or degradation. In many cases, it was difficult to tell that the signal was being processed through an embedded system, indicating accurate sampling, filtering, and reconstruction.

The PCB is enclosed in a plastic housing, preventing users from touching exposed circuitry during operation. This enclosure improves both electrical safety and overall robustness.

The system is intuitive and easy to use. Multiple users were able to interact with the mixing board without instruction beyond a brief explanation of each knob’s function, demonstrating effective user-centered design.

## Conclusions

The project successfully met its goals by implementing a stable, real-time embedded DJ mixing board with interactive physical controls and high-quality audio output. The system performed as expected under real-time constraints and demonstrated effective hardware/software integration. In future iterations, adding external memory would enable advanced features such as BPM adjustment, pitch shifting, reverb, echo, and playback control, bringing the system closer to a commercial DJ controller.

The design conforms to applicable standards by using a custom PCB instead of a breadboard, improving reliability and professionalism. The addition of an enclosure further enhances safety and usability.
