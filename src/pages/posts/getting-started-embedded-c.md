---
layout: ../../layouts/post.astro
title: 'Getting Started: Embedded C'
author: 'Todd'
pubDate: 2022-09-27
---

# Introduction to Embedded C with the AtMega328P

# Goal

By the end of this tutorial, you will be able to compile a binary written for the avr instruction set targeting the AtMega328P microcontroller on an Arduino Nano or Arduino Uno R3 form factor.

## Needed Documentation
- [Arduino Nano PCB design](https://content.arduino.cc/assets/NanoV3.3_sch.pdf?_gl=1*rft1wh*_ga*NTYyODUzODE1LjE2MzMxNjQ3NDg.*_ga_NEXN8H46L5*MTYzODg4NTI3Ny43LjEuMTYzODg4NTcwOC4w)
- [Atmega328P datasheet](https://ww1.microchip.com/downloads/en/DeviceDoc/Atmel-7810-Automotive-Microcontrollers-ATmega328P_Datasheet.pdf)

## Materials used
- [Arduino Nano](https://store.arduino.cc/products/arduino-nano)
- [Arduino Uno R3](https://store.arduino.cc/products/arduino-uno-rev3/)
- Relevant cable
- Computer

## Software Used
- make (for easy of compiling and building)
- avr-gcc (our compiler for avr ISA)
- binutils-avr
- avr-libc
- avrdude (used to flash board)

The software above is just the software that I am using. It works on MacOS and Linux. However, other tools exist.

## Introduction

Embedded programming has become a wildly popular topic with people using microcontrollers to control christmas lights or some home automation. Embedded also has an every growing presence in industry as internet of things and industrial internet of things are becoming more wanted to automate tasks and optimize processes. This is an introduction tutorial for doing embedded development using the C programming language. Unlike many other embedded tutorials, however, this one will be bare metal. We are going to ignore the useful tools that arduino has made for programming their boards and we are also going to ignore the avr c libraries. All we are going to have are the bare bones of the C programming language.

The Arduino platform for hardware is choosen for a few reasons. First being, they are cheap. Secondly, they are accessible. Thirdly, they make it easy to flash and work with. That is the purpose of a development board, to make it fast and easy to prototype.

## Motivations

For those not familiar with embedded, these are prized skills. Just as an example to give motivation to the topic, the chemical manufacturing plant I work at, they are designing solutions to products and the production line by using the Atmega328P microcontroller and even are using an Arduino Uno R3 for prototyping!

## Bare metal

As mentioned, we are not using Arduino's development tools or the avr standard c library. For those not familiar with C, C is a small programming language that provides the basics to build cathedrals out of. Now let us take a moment, to talk about what this term 'basics' mean. The C standard libraries are nnot inherent to C. These libraries are actually often implemented by the operating system and provided by the operating system as core utilities to build with and upon. An example is 'stdio.' Stdio is hardware dependent (the implementation varies depending on the processor) and is provided by the operating system you are using. In fact, all of the standard headers are. So, we are going to be working without them. Basic functionality will be written by us. We literally are just using the core of the C programming language.

### The LED

To make things simple, we are just going to use the onboard led. Looking closely at the board, there is a printed 'L.' Next to this L, is the LED that we are going to be turning on.

Looking at the Arduino Nano PCB design, there is a block with a '328P-MUR' label. This is the AtMega328P. Looking at the block and what is shows as being connected to, we can see a single yellow LED. Tracing this ling, we see is connects to the AtMega328P. The description is 'PB5(SCK)'. This is important, as this tells us our hardware register that we will need to manipulate for this.

## Hardware registers

Hardware registers are how interactions are handled in the microcontroller. Some of these hardware registers are completely internal, and some are used to manipulate things external to the microcontroller. An example of an internal use is the built-in temperature sensor within the AtMega328P or the timers built into the chip. For the external, our LED is a great example. While the LED is built onto the board, it is not apart of the AtMega328P package, it is an external device.

In order to interact with with these hardware registers, they have special memory locations. These memory locations also have a specific size. Most hardware registers on this microcontroller are 8-bits wide, however there are some special purpose registers that are larger. For now, we are only concerned with the 8-bit registers since that is what we will be using to manipulate the LED.

To manipulate these registers, we set them equal to values. We will do all of this by creating pointers directly to that memory location. Or, we can also use logical operators, however for this, we will not be using logical operators.

## DDRB, PORTB, and PINB

Now it is time to crack out the handy datasheet as this will tell us a few things. First, it will tell us what DDRB, PORTB, and PINB and how to use them. Going to section 18.2.1, we are given a basic definition. First we should mention that there are '4' ports on our microcontroller, they are A, B, C, and D. Going back to above where we saw 'PB5,' That is telling us that the LED is connected the Port B. The 5 represents which specific pin as each port has multiple pins. So in this case, our LED is connected to Port B, pin 5. So hence the use of DDRB, PORTB and PINB.

DDRB is going to allow us to set the designated pin to input or output mode. In order to blink the LED, we are going to need to set pin 5 to output.

PORTB has several features, but our main focus is going to that it is the data register. If we write 1 to PORTB, bit 5, while DDRB pin 5 is in output mode, it will output a voltage to that pin. When it is zero, it will turn off the voltage. There are other features of the PORTB register, however that is outside the scope of this tutorial.

PINB is a register we are going to use to toggle the value of PORTB. Writing a 1 to PINB, bit 5, it will toggle the value in the data register between 1 and 0.

All of these registers can be read about in our datasheet for the AtMega328P. A fast way to get to them is to go towards the end of the document and look at the "Register summary." This will tell us where information is about our specific registers. It will give us the location in memory they are located at and the breakdown of the bits. Some hardware registers have a different purpose for each bit, some do not. In the case of DDRB/PORTB/PINB, each bit represents a pin that can be controlled through port B. There are a total of 8 possible pins to control.

NOTE: Not so much for this tutorial, but it is important to pay attention to the information about bits within a register. Some bits are manipulated by other hardware registers.

## Our makefile

To make building simple, here is a Makefile used for building the code and even flashing it.

```sh
   CC = avr-gcc
   CC_FLAGS = -Os -DF_CPU16000000UL -mmcu=atmega328p
   LINK_FLAGS = -mmcu=atmega328p
   OBJ_CPY = avr-objcopy
   OBJ_CPY_FLAGS = -O ihex -R .eeprom
   AVR_DUDE = avrdude
   AVR_DUDE_FLAGS = -F -V -c arduino -p ATMEGA328P -P


   all: compile link cpy flash

   compile:
	${CC} ${CC_FLAGS} -c -o hello.o hello.c

   link:
	${CC} ${LINK_FLAGS} hello.o -o hello

   cpy:
	${OBJ_CPY} ${OBJ_CPY_FLAGS} hello hello.hex

   flash:
	sudo avrdude -F -V -c arduino -p ATMEGA328 -P /dev/tty.usbserial-1140 -b 57600 -U flash:w:hello.hex
```

Not going to spend a lot of time on this. What I will touch on is 'cpy' and 'flash.' In copy, we are using a program to take the resulting binary after compiling and linking to turn that binary into a hex format. This is what we will be uploading or flashing to the arduino. Next is 'flash.' This is using avr dude to flash it to our device. '57600' is the baud rate and this changes depening on the device. For the nano I had to use 57600, while with my Uno R3, I can use 9600.

### Finding my device

You've got the arduino hooked up to your laptop from a USB port, how do you know where to flash it to? On Linux, a simple dmesg and reading the most recent entries should do it. For MacOS, I just list all devices in `/dev` and the one with the `tty.usbserial` or `tty.usbmodem` is usually the one. For Windows, your on your own.

## Turning on the LED

Before we get to blinking, we need to first write some code to turn it on.

```c
    #define PINB ((*(volatile unsigned char*)0x23))
    #define DDRB ((*(volatile unsigned char*)0x24))
    #define PORTB ((*(volatile unsigned char*)0x25))

    int main() {

   	   DDRB = 0b00100000;
	   PORTB = 0b00100000;

	   while(1);

	   return 0;
    }
```

Pretty simple. At the top, we define precompile macros for convience. However, notice the value they are being set to.

```c
       ((*(volatile unsigned char*)0x23))
```

We are creating a pointer to an address in memory. Specially as noted above, to address 0x23, which is out PINB register. We are using `volatile unsigned char`. All these registers are 8 bits in length, which is the same size as a char on the AtMega328P. Volatile is there to force the compiler to tell the program to always look at that address. The reason why this is important is that some compiler optimizations may make assumptions and create shortcuts. Here, we do not want to create shortcuts. Most of the time, the compiler makes the best choice, but in embedded, sometimes the programmer does know better than the compiler.

In our main function, I am setting DDRB to a value. I used binary format to make it easier to visualize, however, it is common to see hexidecimal being used. Since we want pin5, we need to set the 5th bit. We need to count the bits as zero indexed. So while this is an 8-bit register, we reference those bits as DDRB[0:7] (bits 0 to 7).

PORTB, we are just telling it to put a voltage to that pin.

The inifinite while loop at the end is necessary. If we don't have it, the program will end and the device will jump to the reset vector and start the program over. For this case, it would happen so fast you probably would not see the LED turn off, but it is a good habit since most embedded systems we want to run theoretically forever.

## Running it

With the arduino hooked up. If you can run make files and you have changed `/dev/tty.usbserial-1140` to the proper location your operating system has it at, you should just be able to type `make` at a terminal and it will compile and flash it to the device. Below is an example of my output as reference.

```sh
     HelloWorld % make flash
     sudo avrdude -F -V -c arduino -p ATMEGA328 -P /dev/tty.usbserial-1140 -b 57600 -U flash:w:hello.hex

     avrdude: AVR device initialized and ready to accept instructions

     Reading | ################################################## | 100% 0.00s

     avrdude: Device signature = 0x1e950f (probably m328p)
     avrdude: Expected signature for ATmega328 is 1E 95 14
     avrdude: NOTE: "flash" memory has been specified, an erase cycle will be performed
	To disable this feature, specify the -D option.
     avrdude: erasing chip
     avrdude: reading input file "hello.hex"
     avrdude: input file hello.hex auto detected as Intel Hex
     avrdude: writing flash (140 bytes):

     Writing | ################################################## | 100% 0.08s

     avrdude: 140 bytes of flash written

     avrdude: safemode: Fuses OK (E:00, H:00, L:00)

     avrdude done.  Thank you.
```