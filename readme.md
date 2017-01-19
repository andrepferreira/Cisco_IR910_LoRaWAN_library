	 / _____)             _              | |    
	( (____  _____ ____ _| |_ _____  ____| |__  
	 \____ \| ___ |    (_   _) ___ |/ ___)  _ \ 
	 _____) ) ____| | | || |_| ____( (___| | | |
	(______/|_____)_|_|_| \__)_____)\____)_| |_|
	  (C)2013 Semtech-Cycleo

LoRa concentrator HAL user manual
============================

1. Introduction
---------------

The LoRa concentrator Hardware Abstraction Layer is a C library that allow you
to use a Semtech concentrator chip through a reduced number of high level C
functions to configure the hardware, send and receive packets.

The Semtech LoRa concentrator is a digital multi-channel multi-standard packet
radio used to send and receive packets wirelessly using LoRa or FSK modulations.

2. Components of the library
----------------------------

The library is composed of 5 modules:

* loragw_hal
* loragw_reg
* loragw_spi
* loragw_aux
* loragw_gps

The library also contains 4 test programs to demonstrate code use and check
functionality.

### 2.1. loragw_hal ###

This is the main module and contains the high level functions to configure and
use the LoRa concentrator:

* lgw_rxrf_setconf, to set the configuration of the radio channels
* lgw_rxif_setconf, to set the configuration of the IF+modem channels
* lgw_start, to apply the set configuration to the hardware and start it
* lgw_stop, to stop the hardware
* lgw_receive, to fetch packets if any was received
* lgw_send, to send a single packet (non-blocking, see warning in usage section)
* lgw_status, to check when a packet has effectively been sent

For an standard application, include only this module.
The use of this module is detailed on the usage section.

/!\ When sending a packet, there is a 1.5 ms delay for the analog circuitry to
start and be stable (TX_START_DELAY).

In 'timestamp' mode, this is transparent: the modem is started 1.5ms before the
user-set timestamp value is reached, the preamble of the packet start right when
the internal timestamp counter reach target value.

In 'immediate' mode, the packet is emitted as soon as possible: transferring the
packet (and its parameters) from the host to the concentrator takes some time,
then there is the TX_START_DELAY, then the packet is emitted.

In 'triggered' mode (aka PPS/GPS mode), the packet, typically a beacon, is 
emitted 1.5ms after a rising edge of the trigger signal. Because there is no
way to anticipate the triggering event and start the analog circuitry
beforehand, that delay must be taken into account in the protocol.

### 2.2. loragw_reg ###

This module is used to access to the LoRa concentrator registers by name instead
of by address:

* lgw_connect, to initialise and check the connection with the hardware
* lgw_disconnect, to disconnect the hardware
* lgw_soft_reset, to reset the whole hardware by resetting the register array
* lgw_reg_check, to check all registers vs. their default value and output the
result to a file
* lgw_reg_r, read a named register
* lgw_reg_w, write a named register
* lgw_reg_rb, read a name register in burst
* lgw_reg_wb, write a named register in burst

This module handles pagination, read-only registers protection, multi-byte
registers management, signed registers management, read-modify-write routines
for sub-byte registers and read/write burst fragmentation to respect SPI
maximum burst length constraints.

It make the code much easier to read and to debug.
Moreover, if registers are relocated between different hardware revisions but
keep the same function, the code written using register names can be reused "as
is".

If you need access to all the registers, include this module in your
application.

**/!\ Warning** please be sure to have a good understanding of the LoRa
concentrator inner working before accessing the internal registers directly.

### 2.3. loragw_spi ###

This module contains the functions to access the LoRa concentrator register
array through the SPI interface:

* lgw_spi_r to read one byte
* lgw_spi_w to write one byte
* lgw_spi_rb to read two bytes or more
* lgw_spi_wb to write two bytes or more

Please *do not* include that module directly into your application.

**/!\ Warning** Accessing the LoRa concentrator register array without the
checks and safety provided by the functions in loragw_reg is not recommended.

### 2.4. loragw_aux ###

This module contains a single host-dependant function wait_ms to pause for a
defined amount of milliseconds.

The procedure to start and configure the LoRa concentrator hardware contained in
the loragw_hal module requires to wait for several milliseconds at certain
steps, typically to allow for supply voltages or clocks to stabilize after been
switched on.

An accuracy of 1 ms or less is ideal.
If your system doesn't allow that level of accuracy, make sure that the actual
delay is *longer* that the time specified when the function is called (ie.
wait_ms(X) **MUST NOT** before X milliseconds under any circumstance).

If the minimum delays are not guaranteed during the configuration and start
procedure, the hardware might not work at nominal performance.
Most likely, it will not work at all.

### 2.5. loragw_gps ###

This module contains functions to synchronize the concentrator internal 
counter with an absolute time reference, in our case a GPS satellite receiver.

The internal concentrator counter is used to timestamp incoming packets and to 
triggers outgoing packets with a microsecond accuracy.
In some cases, it might be useful to be able to transform that internal 
timestamp (that is independent for each concentrator running in a typical 
networked system) into an absolute UTC time.

In a typical implementation a GPS specific thread will be called, doing the
following things after opening the serial port:

* blocking reads on the serial port (using system read() function)
* parse NMEA sentences (using lgw_parse_nmea)

And each time an RMC sentence has been received:

* get the concentrator timestamp (using lgw_get_trigcnt, mutex needed to 
  protect access to the concentrator)
* get the UTC time contained in the NMEA sentence (using lgw_gps_get)
* call the lgw_gps_sync function (use mutex to protect the time reference that 
  should be a global shared variable).

Then, in other threads, you can simply used that continuously adjusted time 
reference to convert internal timestamps to UTC time (using lgw_cnt2utc) or 
the other way around (using lgw_utc2cnt).

3. Software build process
--------------------------

### 3.1. Details of the software ###

The library is written following ANSI C conventions but using C99 explicit
length data type for all data exchanges with hardware and for parameters.

The loragw_aux module contains POSIX dependant functions for millisecond
accuracy pause.
For embedded platforms, the function could be rewritten using hardware timers.

### 3.2. Building options ###

All modules use a fprintf(stderr,...) function to display debug diagnostic
messages if the DEBUG_xxx is set to 1 in library.cfg

The other settings available in library.cfg are:

* CFG_SPI configures how the link between the host and the concentrator chip 
 is done.

* CFG_CHIP configures what the exact model of chip is, because there are small 
  differences in capabilities between the 'normal' SX1301 production chip, and 
  the FPGA-based version.

* CFG_RADIO configures what chips are used for radios. Only the SX125x are 
  supported for now, but other radios could be supported if drivers are added.

* CFG_BAND configures frequency band limits. If you plan to use you system in 
  a specific band, the library can be used to enforced minimum & maximum 
  frequencies for RX and TX, preventing illegal 'out of band' emissions even in 
  case of bug in your program.
  Other band-specific rules (eg. TX power, channel spacing, dwell time, hopping 
  rules) are *NOT* enforced by the libloragw library and must be enforced in 
  your program.
  To disable band-specific limits, use the 'full' setting that will allow all 
  frequencies supported by the radio.

* CFG_BRD configures board misc parameters and calibration values.
  The RSSI reported by the library when a packet is received, and the TX power 
  specified when is packet is sent, very significantly with the board and 
  components use around the radios (eg. external PA and LNA, filters, RF 
  switches, etc). A limited number of specific board designs have been 
  calibrated by Semtech, if you don't find the board you used, ask Semtech 
  which available setting will give the most accurate results. If you use 
  elements such as external filters, long cables and antennas with gain, you 
  will have to offset their effect in your application.

### 3.3. Building procedures ###

For cross-compilation set the CROSS_COMPILE variable in the Makefile with the
correct toolchain name.

The Makefile in the libloragw directory will parse the library.cfg file and 
generate a config.h C header file containing #define options.
Those options enables and disables sections of code in the loragw_xxx.h files 
and the *.c source files.

The library.cfg is also used directly to select the proper set of dynamic 
libraries to be linked with.

### 3.4. Dynamic libraries requirements ###

Depending on config, SPI module needs LibMPSSE to access the FTDI SPI-over-USB
bridge. Please read install_ftdi.txt for installation instructions.

The code was tested with version 1.3 of LibMPSSE:
http://libmpsse.googlecode.com/files/libmpsse-1.3.tar.gz
SHA1 Checksum: 	1b994a23b118f83144261e3e786c43df74a81cd5

### 3.5. Export ###

Once build, to use that library on another system, you need to export the
following files :

* libloragw/library.cfg  -> root configuration file
* libloragw/libloragw.a  -> static library, to be linked with a program
* libloragw/readme.md  -> required for license compliance
* libloragw/inc/config.h  -> C configuration flags, derived from library.cfg
* libloragw/inc/loragw_*.h  -> take only the ones you need (eg. _hal and _gps)

After statically linking the library to your application, only the license 
is required to be kept or copied inside your program documentation.

4. Hardware dependencies
------------------------

### 4.1. Hardware revision ###

The loragw_reg and loragw_hal are written for a specific version on the Semtech
hardware (IP and/or silicon revision).

This code has been written for:

* Semtech SX1301 chip (or FPGA equivalent)
* Semtech SX1257 or SX1255 I/Q transceivers

The library will not work if there is a mismatch between the hardware version 
and the library version. You can use the test program test_loragw_reg to check 
if the hardware registers match their software declaration.

### 4.2. SPI communication ###

loragw_spi contains 4 SPI functions (read, write, burst read, burst write) that
are platform-dependant.
The functions must be rewritten depending on the SPI bridge you use:

* SPI master matched to the Linux SPI device driver (provided)
* SPI over USB using FTDI components (provided)
* native SPI using a microcontroller peripheral (not provided)

Edit library.cfg to chose which SPI physical interface you want to use.

You can use the test program test_loragw_spi to check with a logic analyser
that the SPI communication is working

### 4.3. GPS receiver (or other GNSS system) ###

To use the GPS module of the library, the host must be connected to a GPS 
receiver via a serial link (or an equivalent receiver using a different 
satellite constellation).
The serial link must appear as a "tty" device in the /dev/ directory, and the 
user launching the program must have the proper system rights to read and 
write on that device.
Use `chmod a+rw` to allow all users to access that specific tty device, or use
sudo to run all your programs (eg. `sudo ./test_loragw_gps`).

In the current revision, the library only reads data from the serial port, 
expecting to receive NMEA frames that are generally sent by GPS receivers as 
soon as they are powered up.

The GPS receiver **MUST** send RMC NMEA sentences (starting with "$G<any 
character>RMC") shortly after sending a PPS pulse on to allow internal 
concentrator timestamps to be converted to absolute UTC time.
If the GPS receiver sends a GGA sentence, the gateway 3D position will also be 
available.

The PPS pulse must be sent to the pin 22 of connector CONN400 on the Semtech 
FPGA-based nano-concentrator board. Ground is available on pins 2 and 12 of 
the same connector.
The pin is loaded by an FPGA internal pull-down, and the signal level coming 
in the FPGA must be 3.3V.
Timing is captured on the rising edge of the PPS signal.

5. Usage
--------

### 5.1. Setting the software environment ###

For a typical application you need to:

* include loragw_hal.h in your program source
* link to the libloragw.a static library during compilation
* link to the librt library due to loragw_aux dependencies (timing functions)
* link to the libmpsse library if you use a FTDI SPI-over-USB bridge

For an application that will also access the concentrator configuration 
registers directly (eg. for advanced configuration) you also need to:

* include loragw_reg.h in your program source

### 5.2. Using the software API ###

To use the HAL in your application, you must follow some basic rules:

* configure the radios path and IF+modem path before starting the radio
* the configuration is only transferred to hardware when you call the *start*
  function
* you cannot receive packets until one (or +) radio is enabled AND one (or +)
  IF+modem part is enabled AND the concentrator is started
* you cannot send packets until one (or +) radio is enabled AND the concentrator
  is started
* you must stop the concentrator before changing the configuration

A typical application flow for using the HAL is the following:

	<configure the radios and IF+modems>
	<start the LoRa concentrator>
	loop {
		<fetch packets that were received by the concentrator>
		<process, store and/or forward received packets>
		<send packets through the concentrator>
	}
	<stop the concentrator>

**/!\ Warning** The lgw_send function is non-blocking and returns while the
LoRa concentrator is still sending the packet, or even before the packet has
started to be transmitted if the packet is triggered on a future event.
While a packet is emitted, no packet can be received (limitation intrinsic to
most radio frequency systems).

Your application *must* take into account the time it takes to send a packet or 
check the status (using lgw_status) before attempting to send another packet.

Trying to send a packet while the previous packet has not finished being send
will result in the previous packet not being sent or being sent only partially
(resulting in a CRC error in the receiver).

### 5.3. Debugging mode ###

To debug your application, it might help to compile the loragw_hal function
with the debug messages activated (set DEBUG_HAL=1 in library.cfg).
It then send a lot of details, including detailed error messages to *stderr*.

6. License
-----------

Copyright (c) 2013, SEMTECH S.A.
All rights reserved.

Redistribution and use in source and binary forms, with or without
modification, are permitted provided that the following conditions are met:

* Redistributions of source code must retain the above copyright
  notice, this list of conditions and the following disclaimer.
* Redistributions in binary form must reproduce the above copyright
  notice, this list of conditions and the following disclaimer in the
  documentation and/or other materials provided with the distribution.
* Neither the name of the Semtech corporation nor the
  names of its contributors may be used to endorse or promote products
  derived from this software without specific prior written permission.

THIS SOFTWARE IS PROVIDED BY THE COPYRIGHT HOLDERS AND CONTRIBUTORS "AS IS" AND
ANY EXPRESS OR IMPLIED WARRANTIES, INCLUDING, BUT NOT LIMITED TO, THE IMPLIED
WARRANTIES OF MERCHANTABILITY AND FITNESS FOR A PARTICULAR PURPOSE ARE
DISCLAIMED. IN NO EVENT SHALL SEMTECH S.A. BE LIABLE FOR ANY
DIRECT, INDIRECT, INCIDENTAL, SPECIAL, EXEMPLARY, OR CONSEQUENTIAL DAMAGES
(INCLUDING, BUT NOT LIMITED TO, PROCUREMENT OF SUBSTITUTE GOODS OR SERVICES;
LOSS OF USE, DATA, OR PROFITS; OR BUSINESS INTERRUPTION) HOWEVER CAUSED AND
ON ANY THEORY OF LIABILITY, WHETHER IN CONTRACT, STRICT LIABILITY, OR TORT
(INCLUDING NEGLIGENCE OR OTHERWISE) ARISING IN ANY WAY OUT OF THE USE OF THIS
SOFTWARE, EVEN IF ADVISED OF THE POSSIBILITY OF SUCH DAMAGE.

*EOF*