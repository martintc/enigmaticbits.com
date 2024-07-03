---
layout: ../../layouts/MarkdownPostLayout.astro
title: 'Embedded Hello World'
pubDate: 2022-09-27
author: 'Todd'
---

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

## Goal

In this session of the series, the goal is to accomplish the 'Hello World' of embedded programming. Along the way we will discuss the cyclic executive, poor delay timers, and touch some on driver design.

## Hello World

In embedded development, the iconic hello world is a little different. With most programming environments, we expect a 'hello world' to just print out a line of text to the standard input/output. In embedded, it is blinking an LED. The reason is because in order to print a line of text, a series of drivers need to be written such as a UART, interrupts, timers, and more.

## Cyclic executive

One main component on this program will be exploring the idea of the cyclic executive. In the first tutorial, we had just a while loop that looked like so:
```c
    while(1) {}
```

This cause an infinite loop so that the return statement was never hit and the program's life did not end. If we let the program run to completion, the Atmega device would hit the reset vector and essentially start over. This is not preferable in embedded development since the idea is we can create a device and it just does it's job indefinitely. We can not expect users to hook it up and tamper with the code.

Now, we will explore another purpose for the loop. We will call this the cyclic executive, another way to look at it is a process scheduler, however that would become more apparent with more going on, such as getting temperature readings, updating internal logs in the EEPROM, and communicating the temperature over ethernet or a UART. However, what we are doing is defining recurring processes to run, or that is one approach to it. The cyclic executive is just the loop of what we want the micro-controller to do forever.

## Blinking the LED

In order to blink the LED, we will need a way to tell the LED to turn off and on. If we remember from the last article, we may remember that there is a hardware register to handle the toggling. This is PINB (0x23).  Writing a 1 to the 5th bit position of this register will toggle our LED on and off. We can also designate the appropriate pin as an output using DDRB (0x24).

Last, we need some time sequence. The Atmega328P has a clock cycle of 16 Mhz, meaning it will experience 16,000,000 clock cycles in a second. Theoretically, it will be off for 8 million cycles and on for 8 million cycles in a second, way to fast to notice. If we don't implement some sort of delay, it will appear as if the LED comes on, but never turns off. So, we need a delay timer.

The Atmega 328P comes with some built in timers, however, we are going to simplify this by using a software delay. Software delays are rather unreliable, but will show the important later as to why hardware timers are so much more convient. Using a software delay, we are not able to garuntee that a certain trigger will happen in a certain time frame. Also, when we go to toggle our LED, this is a time gap that happens to shift the accuracy of our delay.

## Device driver design

The code implements a simplified device driver design. In future articles, we will explore this a little more indepth, but this is just a simplified design that will introduce the reader to working with more than a single file.

## The code

### Makefile
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
	${CC} ${CC_FLAGS} -c -o led.o led.c
    	${CC} ${CC_FLAGS} -c -o main.o main.c

    link:
	${CC} ${LINK_FLAGS} *.o -o led

    cpy:
        ${OBJ_CPY} ${OBJ_CPY_FLAGS} led led.hex

    flash:
        sudo avrdude -F -V -c arduino -p ATMEGA328 -P /dev/tty.usbserial-140 -b 57600 -U flash:w:led.hex
```

### main.c
```c
    #include "led.h"

    void delay() {
      for (volatile int i = 0; i < 1000; i++) {
      	  for (volatile long j = 0; j < 32; j++) {
    	  }
      }
    }

    int main () {

        /* initialize LED driver */
        init_led();

  	    while(1) {
            led_toggle();
            delay();
        }

        return 0;
    }
```
 

### led.h

```c
    #ifndef LED_H
    #define LED_H

    void init_led();

    void led_toggle();

    #endif
```

### led.c
```c
    #define DDRB ((*(volatile unsigned char*)0x24))
    #define PORTB ((*(volatile unsigned char*)0x25))
    #define PINB ((*(volatile unsigned char*)0x23))

    void init_led() {
    	 DDRB = 0x20;
    }

    void led_toggle() {
    	 PINB = 0x20;
    }
```

## Terminal output from running make
```sh
   Device-Driver-Design % make
   avr-gcc -Os -DF_CPU16000000UL -mmcu=atmega328p -c -o led.o led.c
   avr-gcc -Os -DF_CPU16000000UL -mmcu=atmega328p -c -o main.o main.c
   avr-gcc -mmcu=atmega328p *.o -o led
   avr-objcopy -O ihex -R .eeprom led led.hex
   # 57600
   sudo avrdude -F -V -c arduino -p ATMEGA328 -P /dev/tty.usbserial-140 -b 57600 -U flash:w:led.hex

   avrdude: AVR device initialized and ready to accept instructions

   Reading | ################################################## | 100% 0.00s

   avrdude: Device signature = 0x1e950f (probably m328p)
   avrdude: Expected signature for ATmega328 is 1E 95 14
   avrdude: NOTE: "flash" memory has been specified, an erase cycle will be performed
         To disable this feature, specify the -D option.
   avrdude: erasing chip
   avrdude: reading input file "led.hex"
   avrdude: input file led.hex auto detected as Intel Hex
   avrdude: writing flash (264 bytes):

   Writing | ################################################## | 100% 0.12s

   avrdude: 264 bytes of flash written

   avrdude: safemode: Fuses OK (E:00, H:00, L:00)

   avrdude done.  Thank you.
```

## main.c walk through

First thing done in `main.c` is to include our `led.h` file. This will let main know of the methods that are external to `main.c`.

Next, there is the delay method. This is just a method that employs a nested for-loop where it increments variables `i` and `j`. Both of those variables are defined as `volatile`. This is because if they are not implemented as `volatile`, the compiler may optimize them away, remember in the first article, sometimes the compiler will make decisions we don't one. In this case, the compiler will assume the output of those for-loops and optimize them out. We can prevent this by declaring those two variables as `volatile`. This tell the compiler that we need forced checking by the processor on those values.

The main function is fairly straight forward. First, initializing the led, which we will see how it is done in `led.c`. The while loop occurs, every iteration of the while loop, we are causing a delay and then toggling the LED.

## led.h walk through

In this header file, we are just defining methods that we want external files to know about. In this case, we only want `main.c` to know about `init_led` and `led_toggle`. Method that are implemented in `led.c`, but not declared in `led.h` that is included in `main.c`, can not be called on by `main.c`.  

## Taking a look at OOP design

In the led.h walk through, it was just mentioned that `main.c` will only know about methods that are declared in `led.h`. If in `led.c` we have a method called `increment()` that just incremented a value, since the method signature is not in `led.h`, the code in `main.c` does not know about this. This can create a way to implement private methods. All methods declared in the header file are public methods. In this article, we will not implement private methods in C, but I want to mention this to show a little bit about device driver design. We will be exploring this more in later articles in this series.

## led.c

Much like in the last article, we start off by defining our hardware registers. I defined PORTB, however, we do not directly use the register. Instead, we will toggle the value in PORTB by writing a '1' to PINB.

Next we initialize the led device driver. This is done just by writing a '1' to bit 5 in the hardware register DDRB. Instead of binary, we used the hexadecimal representation. During the initialization phase, we are just concerned with setting up the hardware, which in this case it turning the appropriate pin in Port B as an output.

Lastly, we toggle the LED by writing a '1' to the 5th bit of the PINB register. 
