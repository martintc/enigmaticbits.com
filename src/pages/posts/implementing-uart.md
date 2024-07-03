---
layout: ../../layouts/MarkdownPostLayout.astro
title: 'Implementing UART'
author: 'Todd'
pubDate: 2022-09-27
---

# USART on the AtMega328P

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
- GNU Screen

The software above is just the software that I am using. It works on MacOS and Linux. However, other tools exist.

## Goal

In this session, the goal is to implement the Universal Synchronous/Asyncrhonous Receiver Transmitter in just a mode to transmit data. This will result in being able to print text to a serial console that we can access over USB.

At the end of this article, I will have done a walkthrough of code that can operate the USART in transmit mode to print a message, blink and LED and delay between those action.

## GNU Screen

One tool, or a tool with similar functionality, that will be needed if GNU Screen. I can not possibly cover all tools that offer this same functionality, so I will only cover the one I use, GNU Screen. If you are not familiar with GNU Screen, it can create terminal like sessions that can be attached and detatched from. I often use it as a way to have a persistent session on a server I work on over SSH between session. In this article and in the future, we will use that as our serial console to view the text display from our microcontroller.

To be able to spawn a GNU Screen session to our arduino we will need to run the follow:

```sh
sudo screen /dev/tty.usbserial-140 57600
```

In the command we call `screen`, then we define the device we want to create a session to (`/dev/tty.usbserial-140`) and finally specify the baud rate (`57600`).

To gracefully exit from this session, my recommend method is to hold the `control key` and press `a`, then release those button and press `k`. A prompt will ask if you want to kill the session, hit `y` to do so.

NOTE: In order to flash to the AtMega 328P, we can not have any serial connections to the device, otherwise it will fail.

## Makefile

```sh
CC = avr-gcc
CC_FLAGS = -Os -DF_CPU16000000UL -mmcu=atmega328p
LINK_FLAGS = -mmcu=atmega328p
OBJ_CPY = avr-objcopy
OBJ_CPY_FLAGS = -O ihex -R .eeprom
AVR_DUDE = avrdude
AVR_DUDE_FLAGS = -F -V -c arduino -p ATMEGA328P -P

DEV = /dev/
USB = tty.usbserial-1140

SCREEN = screen

BAUD = 57600


all: compile link cpy flash screen

compile:
    ${CC} ${CC_FLAGS} -c -o led.o led.c
    ${CC} ${CC_FLAGS} -c -o usart.o usart.c
    ${CC} ${CC_FLAGS} -c -o main.o main.c

link:
    ${CC} ${LINK_FLAGS} *.o -o main

cpy:
    ${OBJ_CPY} ${OBJ_CPY_FLAGS} main main.hex

flash:
    # 57600
    sudo avrdude -F -V -c arduino -p ATMEGA328 -P ${DEV}${USB} -b ${BAUD} -U flash:w:main.hex

screen:
    ${SCREEN} ${DEV}${USB} ${BAUD}
```

A new rule is added for screen, this is mostly to add speed for testing. If you are not using GNU screen, this rule will need to be removed. The `usb` section also need to be adjusted to what the device appears as on your computer.

## main.c
```c
#include "led.h"
#include "usart.h"

void delay() {
  for (volatile int i = 0; i < 1000; i++) {
    for (volatile int j = 0; j < 1000; j++) {
    }
  }
}

int main () {

  /* initialize usart */
  usart_init();
  /* set message */
  usart_set_message("Hello World");
  /* initialize LED */
  init_led();

  while(1) {
    led_toggle();
    usart_print();
    delay();
  }

  return 0;
}
```

Above is our main.c where we include our relevant headers and implement the same delay function. The only change to our delay function was that I made the inner loop count to 64 to create a longer delay and I changed that inner variable `j` to an int since a long was not necessary.

Down in the main function itself, we start off by initializing the USART, setting our message we want to print and initializing the LED. Then, we hit our cyclic executive, or essentially our basic process scheduler for the program. We will toggle the LED, print the message we set and wait. When the wait is over, we go back to toggling the LED, printing the message and waiting again. We do this cycle indefinitetly.

## led.c / led.h

The LED class I will not be covering here since it was covered in the previous article, however, the source is down below.

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

## usart.h

```c
#ifndef USART_H
#define USART_H

void usart_set_message(char* message);

void usart_init();

void usart_print();

#endif
```

In `usart.h`, we simply define our public methods for our USART driver class. We want to be able to set our message, initialize our hardware and finally trigger the print functionality we build.

### usart.c
```c
/* Transmit/Receive buffer */
#define UDR0 ((*(volatile unsigned char*)0xC6))
// Can only write as transmit buffer when UDRE0 flag set

/* USART Control and Status Register 0 A */
#define UCSR0A ((*(volatile unsigned char*)0xC0))
// bit 7 - USART Recv Complete [RXC0]
// bit 6 - USART transmit complete [TXC0]
// bit 5 - Data Register Empty [UDRE0]
// bit 4 - Frame error [FE0]
// bit 3 - Data OverRun [DOR0]
// bit 2 - USART Prity Error [UPE0]
// bit 1 - Double USART TX speed [U2X0]
// bit 0 - Multi-processor Comms [MPCM0]

#define UDRE 5

/* USART Control and Status Register 0 B */
#define UCSR0B ((*(volatile unsigned char*)0xC1))
// bit 7 - RX Complete Interrupt Enable 0 [RXCIE0]
// bit 6 - TX Complete Interrupt Enable 0 [TXCIE0]
// bit 5 - USART Data Register Empty IR enable 0 [UDRIE0]
// bit 4 - Receiver Enable 0 [RXEN0]
// bit 3 - Transmitter Enable 0 [TXEN0]
// bit 2 - Char size 0 [UCSZ02]
// bit 1 - Receive Data Bit 8 0 [RXB80]
// bit 0 - Transmit Data bit 8 0 [TCB80]

#define TXEN0 3

/* USART Control and Status Register 0 C */
#define UCSR0C ((*(volatile unsigned char*)0xC2))
// Bit 7/6 - USART Mode Select 0 n [UMSEL0n]
//    00 for asynch
// bit 5/4 USART Partiy mode 0 n [UPM0n]
//    00 for disabled
// bit 3 - USART Stop bit Select 0 [USBS0]
// bit 2 - USART character size/data order [UCSZ01/UDORD0]
// bit 1 - USART character size/clock phase [UCSZ00/UCPHA0]
// bit 0 - Clock Polarity [UCPOL0]

/* USART Baud Rate 0 Register Low */
#define UBRR0L ((*(volatile unsigned char*)0xC4))
/* USART Baud Rate 0 Register High */
#define UBRR0H ((*(volatile unsigned char*)0xC5))

/*
 * Baud rate calculation
 * (cpu_frequency / (desired_baudrate * 16UL)) - 1
 */
#define F_CPU 16000000UL
#define USART_BAUD_RATE 57600

#define BAUD_PRESCALAR (((F_CPU / (USART_BAUD_RATE * 16UL))) - 1)


// NOTE: UBBR is a 12 bit register using 8 bits low and 4 bits high

// file scope variable that acts as a 'private data member'
static char* message;
// can only be changed via 'public method' usart_set_message()

// Sets the private member data message
void usart_set_message(char* msg) {
  message = msg;
}

// initializes the UART for use
void usart_init() {
  // reset USDR0 to empty, emptying RX/TX buffer
  // set baud rate
  UBRR0H = BAUD_PRESCALAR >> 8;
  UBRR0L = BAUD_PRESCALAR;
  // set frame format
  UCSR0C = (0b11<<1); // set to 8 bit format
}

// carriage return line feed
void usart_crlf(){
  while ((UCSR0A & (1<<UDRE))== 0);
  UDR0 = '\r';
  while ((UCSR0A & (1<<UDRE))== 0);
  UDR0 = '\n';
}

// print the message to the uart
void usart_print() {
  // clear transmit buffer
  UDR0 = 0x00;
  // enable transmit mode
  UCSR0B = (1<<TXEN0);
  for (int idx = 0; message[idx] != 0; idx++) {
    while ((UCSR0A & (1<<UDRE))== 0);
    UDR0 = (char) message[idx];
  }
  usart_crlf();
  // disable transmit mode
  UCSR0B = (0 << TXEN0);
}
```

Our USART is a lot more involved than our LED. We are spread across using multiple registers and appear to have a lot going on. I tried to comment to illistrate various aspects. At the start of the file, we define our registers. I added comments to cover the bits of significant registers that we are using and provided their name and in some instances a short description. Most of the bits in these registers, we will not have to use to accomplish our goals.

So, looking at our datasheet for the AtMega 328P, we need to look at the section for the USART. First register to look at is `UDR0`. This register is our buffer. When we receive (which is not covered in this tutorial), the recieved information is placed in the buffer. That buffer is also the same as the transmit buffer, so it can only work one way at a time. When in transmit, we place an 8 bit value we want in that register to be transmitted to our USART interface. Important to notice, it is an 8-bit register (or 1 byte or the size of 1 char).

Next register to look at is `UCSR0A`. This register, each bit represent something to the hardware, our concern for this is bit 5. This register will tell us when the `UDR0` register has been cleared after transmitting. When the bit in `UDRE0` is 0, our `UDR0` register is empty. When we transmit over the USART, when a value has been transmitted, it will zero out our `UDR0` register.

In register `UCSR0B`, the bit we are concerned with is the `TXEN0` bit. In order to set the USART to transmit mode, that bit needs to be set to 1.

For register `UCSR0C`, we are mostly concerned with bits 1 and 2. Here we are setting out bit size. We plan to transmit 8 bits. so we need to write a `1` to both of these locations. In the datasheet, there is a table that shows different configurations for this.

Lastly for registers, `UBBR0` which I have as two seperate registers denoted as ending in `L` and `H` for their high and low portions. This register is a 12 bit register that sets the baud rate for the USART to operate at. The layout is the full 8 bits located in the lower portion of the register and the low 4-bits of the hight register. The high 4 bits of the high register are ignored by the microcontroller.

Following the registers, I have some pre-compiler definitions used to calculate the BAUD prescalar. Going back to register `UBBR0`, we often call these prescalars. We can think of them as a nob that goes to different set points. We have our microcontroller clock rate (16 MHz) and our desired Baud rate, using the calculation in `BAUD_PRESCALAR`, we can calculate the proper prescaler we need to set register `UBBR0`.

The first non-pre-compiler definition is defining a pointer to a character array called `message`. This is file-scope and is only visibile to the methods in `usart.c`. Main can not directly manipulate this variable. It can only do so by the method `usart_set_message`. This creates a sort of private data member that can be changed through a public method giving us an OOP like abstraction of encapsulation.

Next, we have the `usart_init` method that initializes the settings for our usart by manipulating most of the bits I touched on above covering the registers we are using.

In the `usart_print` method, we start off by zeroing out our `UDR0` register, we do not want to assuming it is zeroed. Then, we enable transmit mode by manipulating rh `TXEN0` bit. Next, we enter a for loop for the length of the message. During each iteration we check to see if `UDR0` is empty. If not, we loop and wait. Once it is, we load the next character into the `UDR0` register.  Once the full contents of the message have been sent out to the USART, we call on the `usart_crlf` which simply transmits our carriage return feed line to go to the next line of our USART interface then we disable transmit mode.

## Terminal output
```sh
USART-Intro % make
avr-gcc -Os -DF_CPU16000000UL -mmcu=atmega328p -c -o led.o led.c
avr-gcc -Os -DF_CPU16000000UL -mmcu=atmega328p -c -o usart.o usart.c
avr-gcc -Os -DF_CPU16000000UL -mmcu=atmega328p -c -o main.o main.c
avr-gcc -mmcu=atmega328p *.o -o main
avr-objcopy -O ihex -R .eeprom main main.hex
# 57600
sudo avrdude -F -V -c arduino -p ATMEGA328 -P /dev/tty.usbserial-1140 -b 57600 -U flash:w:main.hex
Hello World

avrdude: AVR device initialized and ready to accept instructions

Reading | ################################################## | 100% 0.00s

avrdude: Device signature = 0x1e950f (probably m328p)
avrdude: Expected signature for ATmega328 is 1E 95 14
avrdude: NOTE: "flash" memory has been specified, an erase cycle will be performed
         To disable this feature, specify the -D option.
avrdude: erasing chip
avrdude: reading input file "main.hex"
avrdude: input file main.hex auto detected as Intel Hex
avrdude: writing flash (418 bytes):

Writing | ################################################## | 100% 0.16s

avrdude: 418 bytes of flash written

avrdude: safemode: Fuses OK (E:00, H:00, L:00)

avrdude done.  Thank you.

screen /dev/tty.usbserial-1140 57600
[screen is terminating]
```
