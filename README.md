Fadecandy
=========

This is a **WORK IN PROGRESS**. Not complete, definitely not ready for prime-time yet.

Fadecandy is firmware for the [Teensy 3.0](http://www.pjrc.com/store/teensy3.html), a tiny and inexpensive ARM microcontroller board.

Fadecandy drives addressable LED strips with the WS2811 and WS2812 controllers. These LED strips are common and inexpensive, available from [many suppliers](http://www.aliexpress.com/item/5M-WS2811-LED-digital-strip-60leds-m-with-60pcs-WS2811-built-in-tthe-5050-smd-rgb/635563383.html?tracelog=back_to_detail_a) for around $0.25 per pixel.

This firmware is based on Stoffregen's excellent [OctoWS2811](http://www.pjrc.com/teensy/td_libs_OctoWS2811.html) library, which pumps out serial data for these LED strips entirely using DMA. This firmware builds on Paul's work by adding:

* A high performance USB protocol
* Zero copy architecture with triple-buffering
* Interpolation between keyframes
* Gamma and color correction with per-channel 256-entry lookup tables
* Temporal dithering

These features add up to give *very smooth* fades and high dynamic range. Ever notice that annoying stair-stepping effect when fading LEDs from off to dim? Fadecandy avoids that using a form of [delta-sigma modulation](http://en.wikipedia.org/wiki/Delta-sigma_modulation). It rapidly wiggles each pixel's value up or down by one 8-bit step, in order to achieve 16-bit resolution for fades.

Vitals
------

* 512 pixels supported per Teensy board (8 strings, 64 pixels per string)
* Constant hardware frame rate of 520 FPS, to support temporal dithering
* Full-speed (12 Mbps) USB
* 768-entry 16-bit color lookup table, for gamma correction and color balance

Prerequisites
-------------

* The recommended ARM toolchain, from <https://code.launchpad.net/gcc-arm-embedded>
* The Teensy Loader: <http://www.pjrc.com/teensy/loader.html>

Pin Assignment
--------------

The pin assignment is the same as the original [OctoWS2811](http://www.pjrc.com/teensy/td_libs_OctoWS2811.html) pinout:

Teensy 3.0 Pin | Function
-------------- | --------------
2              | Led strip #1
14             | Led strip #2
7              | Led strip #3
8              | Led strip #4
6              | Led strip #5
20             | Led strip #6
21             | Led strip #7
5              | Led strip #8
15 & 16        | Connect together

Remember that each strip may be up to 64 LEDs long. It's fine to have shorter strips or to leave some outputs unused. These outputs are 3.3V logic signals at 800 kilobits per second. It usually works to connect them directly to the 5V inputs of your WS2811 LED strips, but for the best signal integrity you should really use a level-shifting buffer to convert the 3.3V logic to 5V.

Color Processing
----------------

The Fadecandy firmware maintains a color lookup table with 256 8-bit entries for each of the three color channels. The input values to this LUT are the 8-bit colorspace as used in the framebuffer's 24-bit pixel values. The outputs are a 48-bit color which acts as the input for Fadecandy's dithering algorithm.

Why 48-bit color? In combination with our dithering algorithm, this gives a lot more color resolution, especially near the low end of the brightness range where stair-stepping and color shift can be most apparent.

Each pixel goes through the following processing steps in Fadecandy:

* 8 bit per channel framebuffer values are expanded to 16 bits per channel
* We interpolate smoothly from the old framebuffer values to the new framebuffer values
* This interpolated 16-bit value goes through the color LUT, which itself is linearly interpolated
* The final 16-bit value is fed into our temporal dithering algorithm, which results in an 8-bit color
* These 8-bit colors are converted to the format needed by OctoWS2811's DMA engine
* In hardware, the converted colors are streamed out to eight LED strings in parallel

USB Protocol
------------

To achieve the best CPU efficiency, Fadecandy uses a custom packet-oriented USB protocol rather than emulating a USB serial device. This simple USB protocol is easy to speak using cross-platform libraries like [libusb](http://www.libusb.org) and [PyUSB](http://pyusb.sourceforge.net/). Examples are included. If you use the included [Open Pixel Control](http://openpixelcontrol.org/) bridge, you need not worry about the USB protocol at all.

Attribute       | Value
--------------- | -----
Vendor ID       | 0x1d50
Product ID      | 0x607a
Manufacturer    | "scanlime"
Product         | "Fadecandy"
Serial          | Unique for each Teensy 3.0 board
Device Class    | Vendor-specific
Configurations  | 1
Endpoints       | 1
Endpoint 1      | Bulk OUT (Host to Device), 64-byte packets

The device has a single Bulk OUT endpoint which expects packets of up to 64 bytes. Multiple packets may be transmitted in one LibUSB "write" operation, as long as the buffer you provide is a multiple of 64 bytes in length.

Each packet begins with an 8-bit control byte, which is divided into three bit-fields:

Bits 7..6  | Bit 5       | Bits 4..0
---------- | ----------- | ------------
Type code  | 'Final' bit | Packet index

* The 'type' code indicates what kind of packet this is.
* The 'final' bit, if set, causes the most recent group of packets to take effect
* The packet index is used to sequence packets within a particular type code

The following packet types are recognized:

Type code | Meaning of 'final' bit          | Index range | Packet contents
--------- | ------------------------------- | ----------- | -------------------------------------
0         | Interpolate to new video frame  | 0 … 24      | Up to 21 pixels, 24-bit RGB
1         | Instantly apply new color LUT   | 0 … 24      | Up to 31 16-bit lookup table entries
2         |                                 |             | (reserved)
3         |                                 |             | (reserved)

In a type 0 packet, the USB packet contains up to 21 pixels of 24-bit RGB color data. The last packet (index 24) only needs to contain 8 valid pixels. Pixels 9-20 in these packets are ignored.

Byte Offset   | Description
------------- | ------------
0             | Control byte
1             | Pixel 0, Red
2             | Pixel 0, Green
3             | Pixel 0, Blue
4             | Pixel 1, Red
5             | Pixel 1, Green
6             | Pixel 1, Blue
…             | …
61            | Pixel 20, Red
62            | Pixel 20, Green
63            | Pixel 20, Blue

In a type 1 packet, the USB packet contains up to 31 lookup-table entries. The lookup table is structured as three arrays of 256 entries, starting with the entire red-channel LUT, then the green-channel LUT, then the blue-channel LUT. Each packet is structured as follows:

Byte Offset   | Description
------------- | ------------
0             | Control byte
1             | Reserved (0)
2             | LUT entry #0, low byte
3             | LUT entry #0, high byte
4             | LUT entry #1, low byte
5             | LUT entry #1, high byte
…             | …
62            | LUT entry #30, low byte
63            | LUT entry #30, high byte


Contact
-------

Micah Elizabeth Scott <<micah@scanlime.org>>

