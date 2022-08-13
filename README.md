# STM32F4xx "Black Pill" USB to I2S DAC Audio Bridge

## Features

* USB Full Speed Class 1 Audio device, no special drivers needed
* Isochronous with endpoint feedback (3bytes, 10.14 format) to synchronize sampling frequency
* Bus powered
* Supports 24-bit audio streams with Fs = 44.1kHz, 48kHz or 96kHz
* USB Audio Volume (0dB to -96dB, 3dB steps) and Mute control 
* I2S master output with I2S Philips standard 24/32 data frame
* Uses inexpensive Aliexpress-sourced STM32F4xx (I saw several models on Taobao around 10+ RMB with 22x2 pins they all should work).
* Uses inexpensive ES9023 DAC bus or self powered (with power board).
* Optional power supply board and amplifier
* Build support (Makefile option) for STM32F401CCU6 and STM32F411CEU6 boards 
* Optional MCLK output generation on STM32F411

## Wish List

* 32bit support
* MCLK support with other F401 models with 64 or more pins
* UAC 2 if ?

16bit data will be padded to 24bit. However 44.1 and 48 kHz data will NOT be resampled if you setup correctly. On Windows if you allow exclusive access for the device and you select WSAPI as Foobar's output, then it will switch to corresponding Fs to the actual sample rate of your source file when you play it. But otherwise there will be resampling on PC side.

## Credits
* [har-in-air/STM32F411_USB_AUDIO_DAC](https://github.com/har-in-air/STM32F411_USB_AUDIO_DAC) where I originally forked from


## Software Development Environment
* Debian 11 in WSL on Windows 10 with GCC 10
* STM32 F4 library v1.26.1
* Makefile project. Edit makefile flags to
  * Select STM32F411 or STM32F401
  * Enable MCLK output generation (STM32F411 only)
  * Enable diagnostic printout on serial UART port 

## Hardware

* TAOBAO STM32F4 core board / mini board with 2x19 or 2x22 pins
	* I2S_2 peripheral interface generates WS, BCK, SDO and optionally MCK
	* external 7-segment LED indicate sampling frequency
	* On-board LED for diagnostic status
	* UART2 serial interface for diagnostic information
* ES9023 I2S DAC module
	* asynchronized mode with 50MHz MCK
```
F4xx    ES9023      Description
--------------------------------------------------------------------
GND     GND
B13     BCK         I2S_BCK (Bit Clock)
B15     SDI         I2S_SDI (Data Input)
B12     LRCK        I2S_WS (LR Clock)
GND     DIF         Format = I2S
A8      MUTE_B      Mute control
A6      -           I2S_MCK (not used)
--------------------------------------------------------------------
C13     on-board    Diagnostic
--------------------------------------------------------------------
A11     USB_D-      USB data
A12     USB_D+
A2      TX          Serial debug
A3      RX
PA0                 KEY button. Triggers endpoint  
                    feedback printout if enabled with  
                    DEBUG_FEEDBACK_ENDPOINT
--------------------------------------------------------------------
7-segment LED for frequence display
4 = 44.1kHz
8 = 48kHz
9 = 96kHz
F4xx    LED pin
--------------------------------------------------------------------
B0      a
B1      b
B3      c
B4      d
B5      e
B6      f
B7      g
--------------------------------------------------------------------
```    

<img src="docs/prototype.jpg" />

## Checking USB Audio device on Ubuntu 20.04

* Execute `lsusb` with and without the USB-Audio DAC plugged in, you should see the 
  new USB device
  
<img src="docs/lsusb.png" />
  
* Execute `aplay -L` and look for `PCM5102 DAC`

<img src="docs/aplay_output.png" />

* Run the `Sound` application without the USB-Audio DAC plugged in and check the
  Speaker/Headphone output options
* Plug in the USB-Audio DAC and check again, you should see at least one new option.
  Select this for playing sound output via the USB-Audio DAC
* Execute `cat /proc/asound/DAC/stream0` when a song is playing

<img src="docs/stream.png" />

## Optimizing Pulseaudio on Ubuntu 20.04 for USB-Audio DAC

* Edit `/etc/pulse/daemon.conf` as root
* Force re-sampling to 96kHz
* Resize to 24bits
* Use highest quality re-sampling algorithm
* Save file, log out and log in again for the changes to take effect

<img src="docs/pulseaudio_config.png" />

## Optimizing Windows 10 for USB-Audio DAC

* Use the Control Panel Sound playback device properties dialog

<img src="docs/win10_96kHz_24bit.png" />


## Endpoint Feedback mechanism

<img src="docs/feedback_endpoint_spec.png" />

Unfortunately, we do not have any means of measuring the actual Fs (accurate to 10.14 resolution)
generated by the PLLI2S peripheral on an SOF resolution interval of 1mS. So we calculate
a nominal Fs value by assuming the HSE crystal has 0ppm accuracy (no error), and use the PLLI2S N,R,
I2SDIV and ODD register values to compute the generated Fs value. For example when MCLK output
is disabled, the optimal register settings result in a value of 96.0144kHz.

<img src="docs/i2s_pll_settings.png" />

Since the USB host is asynchronous to the PLLI2S Fs clock generator, the incoming Fs rate of audio packets will be slightly different. We use
a circular buffer of audio packets to accommodate the difference in incoming and outgoing Fs. 

The USB driver writes incoming audio packets to this buffer while the I2S transmit DMA reads from this buffer. We start I2S playback when the buffer is half full, and then try to maintain this position, i.e. the difference between the write pointer and read pointer should optimally be half of the buffer size.

Any change to this pointer distance implies the USB host and I2S playback Fs values are not in sync.
To correct this, we implement a PID style feedback mechanism where we report an ideal Fs feedback frequency
based on the deviation from the nominal pointer distance. We want to avoid the write process overwriting the unread packets, and we also want to minimize the oscillation in Fs due to unnecessarily large corrections.

This is a debug log of changes in Fs due to the implemented mechanism. The first datum is the SOF frame counter, the second is the pointer distance in samples, the third is the feedback Fs. As you can see, the feedback is able to minimize changes in pointer distance AND oscillations in Fs frequency.

<img src="docs/endpoint_feedback.png" />
