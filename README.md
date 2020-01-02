# openMicroInverter ![ Fig. 0.](figures/oshw-logo-100-px.png  "oshw logo.")
### An open source hardware platform for experimenting with DC-to-AC conversion, power and energy metering and grid tie inverters.

### What is the openMicroInverter?
The *openMicroInverter*, or short *oμI*, is an **Arduino-UNO based** DC-to-AC power converter. The *oμI* platform is meant for doing experiments with power electronics and energy systems. The *oμI* is intended to work as:

1. DC-to-AC power inverter for off the grid applications,
2. AC power meter and energy metering device,
3. an inverter which phase-locks to the grid,
4. solar-inverter or inverter for energy storage systems and
5. a bi-directional power converter.

Why then using the 8-bit Arduino-UNO? Just because it is fun to squeeze the most out of it:-) Furthermore, this project shows the full capabilities of the [TrueRMS](https://github.com/MartinStokroos/TrueRMS) Arduino library from one of my other repositories. Applications i, ii and iii have been successfully realized so far...

This is work in progress...

### How does it work?
The converter topology of the *oμI* is a single-phase, single-step, transformer coupled power converter with a full-bridge inverter. This type of converter is not as efficient as the combination of a high frequency switching boost converter with DC-to-AC converter, but the *oμI* is simple to build and it is relatively safe to operate.

### The Hardware
There are two versions of the hardware. The first design is the development model and second design is the final design of the *oμI* platform. The development model has been used for the development of the software. Please refer to the schematic diagrams to understand further explanation.

**The openMicroInverter development hardware**
A rapid prototype of the inverter was realized for doing software development. The schematic of the development model is named *openMicroInverter_dev.pdf*.
A somewhat older H-bridge driver, the HIP4082, is used for the design. The HIP4082 is a medium voltage, medium frequency full-bridge driver, distributed nowadays by *Renesas*. The driver has a build in turn-on delay to create dead time required for switching between the top and the bottom FET's. With this feature it is possible to drive the chip directly from the PWM generators of the ATmega328P (which are less sophisticated than the timers of STM32 controllers...)

To be continued...

**The openMicroInverter hardware**
The *oμI* is designed as a single PCB module and is build with low-cost components.
 
This is work in progress...

### Software Description
Arduino-UNO Pin-out:

Pin | In/Out | Function
--- | ------ | --------
A0 | input | scaled ac mains voltage biased on 2.5V DC. Full scale represented voltage is about 700Vpp (peak-to-peak).
A1 | input | scaled ac inverter current biased on 2.5V DC. Full scale represented current is about 5App.
A2 | input | scaled ac inverter voltage biased on 2.5V DC. Full scale represented voltage is about 700Vpp.
A3 | input | scaled battery current biased on 2.5V DC. Full scale current is .. A.
A4 | input | scaled battery voltage. Full scale voltage is .. V.
A5 | input | (optional)
D2 | output | grid connect relay
D4 | output | debug output pin
D6 | input | external reference voltage input of the analog comparator (ZCD).
D8 | output | inverter enable output (a high=ON)
D7 | input | analog comparator input (AIN1) for zero-cross detection (ZCD) of the grid voltage. The analog comparator input is connected in parallel with A0.
D13 | output | Arduino LED (*PLL-locking* indicator)

**Software timing scheme**
The standard Arduino library functions like *analogRead, analogWrite* and *digitalWrite* are very time consuming and can not be used with the *oμI*. The software requirements are:

* The PWM frequency to steer the H-bridge must be chosen as high as possible.
* The ADC sampling frequency must be a multiple of 50Hz for stable RMS readings.
* If possible, the ADC should be synchronized to the timer used for the PWM generation to force the sampling to happen inbetween the switching transients.
* The PWM duty-cycle is limited by the hardware and can only be used up to about 90%. This is because of the charge pumps from the hi-side FET drivers of the HIP-4082 full-bridge driver. The dynamic range for control is limited in particular when using a 8-bit timer.
* TIM0 should not be used to keep the Arduino time and delay functions working. These functions might be used later in the program main loop.

Different schemes are currently being worked out to fit it all inside the Arduino. Schemes under investigation are:

Scheme | Description | Remarks
------ | ----------- | -------
*scheme 1* | TIM1 generates a PWM-frequency of 16MHz/512/4 = 7812.5Hz (128μs). The 50Hz reference waveform is generated by the DDS algorithm inside the TIM1 ISR. Each TIM1 interrupt the PWM is updated. The ADC-start is triggered by TIM1 and stays synced with TIM1. Six ADC-channels are time-multiplexed. A complete sequence of six channels takes 768us. The closesed whole number of samples that does fit in a 20ms mains cycle is 26.041. This causes a small ripple in the RMS readings. | With scheme, 1 the remaining available processing time in the TIM1 ISR is not enough for implementing power measurements.
*scheme 2* | TIM1 generates a PWM-frequency of 16MHz/(2*1333) = 6000Hz (166.67us). The TIM1 ISR function is empty and only needed to trigger the ADC. All DSP work is done inside the ADC ISR. The 50Hz reference waveform is generated by a DDS algorithm. Each ADC interrupt the PWM is updated and a single analog to digital conversion is processed. Six ADC-channels are time multiplexed. A complete sequence of six channels takes 1ms. Exactly 20 samples do fit in one mains cycle of 20ms. | So far the best scheme. Switching frequency is rather low and the duty-cycle ranges from about 12% to 88%. This is a little too limited and more than needed to keep the HIP-4082 hi-side chargepumps running...
*scheme 3* | TIM1 running at 6kHz triggers the ADC. The PWM is generated with timer2 at a higher frequency. The PWM is not in sync with the sampling proces and the wave amplitude resolution is only 8-bits. | To be confirmed...
*scheme 4* | ADC is free running and the PWM is generated by TIM1 or TIM2 for respectively 10- or 8-bit amplitude resolution in the output wave. The ADC frequency can not be a multiple of 50Hz and a ripple occurs in the RMS readings. | Scheme 4 doesn´t work. There is too much jitter in the sampling time period to be useful when the ADC is in free running mode.

**H-bridge switching modes**
* unipolar switching
* bi-polar switching
* hybrid switching

### The *PowerSys* Library
Refer to the readme file of the [PowerSys](/libraries/PowerSys/README.md) Library.

### Example Sketches
*Inverter1.ino* - Sketch to evaluate the different ways of gating the H-bridge to generate a sine wave output. This sketch works for the *openMicroInverter_dev* hardware. The inverter works in voltage-mode without output voltage control (open loop). The timing is according scheme 1, which gives stable readings of the measurements but the switching frequency is a little low and the duty-cycle range is limited between 12% and 88%

*Metering.ino* - This sketch is a power meter and energy metering example that works for the *openMicroInverter_dev* hardware.
This sketch uses a time base on interrupt basis and uses an ADC multiplexer running at 6kHz (120*50Hz). Each ADC-channel samples at 1kHz. This example works without inverter part (H-bridge + transformer and output filter). Re-wire the circuit such that A0 and A2 measure the AC-voltage and A1 the load current. The power is calculated from inputs A1 and A2.

*ZeroCrossingDetector* - This example demonstrates Zero Crossing Detection (ZCD) with the analog comparator of the ATMEGA328. The comparator interrupts on output toggle. The sign of the sine wave is determined from the ADC input. The output of the ZCD only toggles from high to low when the sign of the AC input is positive and vice verse.

![ZCD](figures/SCR_comparator.GIF  "ZCD in and output signals")

### Acknowledgements
A lot of time was saved in the development of this project by using the alternative Arduino-IDE *Sloeber*. Sloeber is a wonderful Arduino plugin for Eclipse. Thanks to Jantje and his contributors!
