---
layout: ../../layouts/MarkdownPostLayout.astro
title: 'Terminal drawing with Swift'
author: 'Todd'
pubDate: 2023-06-25
---

## Introduction

Terminal based UIs have always held a certain fascination with me. They can be incredibly functional while also, at least from looks, being incredibly minimal. There is also something beutiful about a simple terminal UI. In a previous article I played around with crossterm in Rust and recently thought, huh, wonder what it would be like with Swift. This also gave me the change to play with some lower level things in Swift that so far, most of my tinkering has not lead me to. This got me to look at pointers and some unicode encoding.

Unlike with Rust where I used crossterm, I decided not to use a library that abstracts all the nitty gritty details of the terminal. Instead, I will be placing the terminal into raw mode and drawing without too many abstractions on top. The place to begin, is by looking at termios. Termios, for those who are unaware, well, I will just copy and past it from the [FreeBSD man pages](https://man.freebsd.org/cgi/man.cgi?query=termios&sektion=4).

  This describes a general terminal line discipline that is supported on tty asynchronous communication ports.

This can be thought of as the low level settings for the terminal. Not things like fonts or colors, but instead how the terminal actually functions at a deeper level. In case your wondering why I pulled the FreeBSD man page for this, lots of userland utilities on macOS come from the BSDs. You can see this when you run 'man termios' in iterm or the macOS terminal and go down to the bottom. It pretty much is the FreeBSD man page. Also, termios is a pretty generic structure, as in, most unix-like systems all have pretty much the same struct with same settings. So, this code should not only work on macOS, but also Linux and the BSDs.

Here is what I get after running my code in the terminal. Only this.

  ┏━━━━━━━━━━━━━━━━━━━┓
  ┃                   ┃
  ┃                   ┃
  ┃                   ┃
  ┃                   ┃
  ┃                   ┃
  ┃                   ┃
  ┃                   ┃
  ┃                   ┃
  ┃                   ┃
  ┗━━━━━━━━━━━━━━━━━━━┛

A simple box, but it takes a bit of code to get there, so lets jump in.

First things first, it is better to work off of the FreeBSD documentation or using the man page in macOS, which is essentially the FreeBSD man page. Online, the documentation apple gives is can be found [here](https://developer.apple.com/documentation/kernel/termios). It doesn't tell me too much other than the termios structure is rather accessible to me without any weird imports. Importing the Foundation should be good enough.

# Main method

Next, take a look at the main method, it is fairly straight forward, from there, I will jump into the methods thar are called.
```swift
  import Foundation

  func main() {
    let fd = STDIN_FILENO
    var tio: termios = termios()
    enableRawMode(fd: fd, orig_termios: &tio)
    clearScreen(fd: fd)
    let box = Box(x: 1, y: 1, width: 20, height: 10)
    box.draw()
    while true {
      var buf: UInt8 = 0
      let readResult = read(fd, &buf, 1)

      if readResult < 0 {
        fatalError()
      } else if readResult == 0 {
        break
      } else {
        assert(readResult == 1)
        print(buf)
      }

      if buf == Character("q").asciiValue! {
        break
      }
    }
    disableRawMode(fd: fd, tio: &tio)
  }

  main()
```

The main.swift file serves as the main entry point and is written a little like a script this way. So my main method needs to be called. If we don't want to have to directly invoke main, you need to put the main method in a class or structure and use an attribute of @main to do it. I didn't feel it necessary to this here, at least not yet. In the main method, I get the file descripter for STDIN_FILENO, which is the terminal for our purposes. Creating an instance of the termios structure and placing it into a mutable variable named tio. Then, enable raw mode (will get to this later). After that, clear out the screen, make the box and draw it. From there, loop on input, the important part here is that this will infinitely loop unless the user presses the letter q, there is some error handling also. When q is typed, break the loop and disable raw mode. Now, lets get to the fun bits.

## Enabling Raw Mode
```swift
  func enableRawMode(fd: Int32, orig_termios: UnsafeMutablePointer<termios>) {
    guard isatty(fd) != 0 else { fatalError() }
    // get the original terminal settings
    guard tcgetattr(fd, &orig_termios.pointee) >= 0 else  {
      fatalError("Issue initializing termina")
    }

    var tio = orig_termios.pointee
    tio.c_iflag &= ~UInt(IXON | ICRNL | ISTRIP | BRKINT)
    tio.c_oflag &= ~UInt(OPOST)
    tio.c_cflag &= ~UInt(CS8)
    tio.c_lflag &= ~UInt(ICANON | ECHO | ISIG | IEXTEN)
    
    guard tcsetattr(fd, TCSAFLUSH, &tio) >= 0 else {
      fatalError("Issue initializing terminal")
    }
    
    tcflush(fd, TCIOFLUSH)
  }
```

Enabling raw mode is one of the important functions at play here. We can consider when a terminal is initially launched, it is in 'cooked mode.' That is sort of the default. Raw mode entails disabling some processing and allowing us to pass input directly to the program running in the terminal rather allowing some default processing to happen. That is a quick birds eye view. These settings are changed by mutating the termios structure.

The arguments are an Int32 named fd (file descriptor) and a rather weird one, UnsafeMutablePointer of type termios named orig termios. File decsriptor is fairly straight forward, but the second one, not so much. Swift allows pointers and there are several kinds, and they are typed via generics. UnsafeMutablePointer is a generic that we can provide the unerlying type. The variable is named orig_termios since we are passing in the termios structure from main and this will be used to get and sort of stash the current settings the terminal operates on by default. This helps later when exiting raw mode and restoring back to where we were.

The first line calls a function in a guard statement to check if is atty. This function takes in a file descriptor and just checks if the file descripter points to a terminal. This is a guard so we don't try to pass in a filed descriptor for something like a simple text file named hello.txt. This should fail and throw the fatalError.

Following that is the tcgetattr function. This is how I get the original terminal settings. Or should I say, the termios structure that is tied to STDIN_FILENO tty. It works by passing in the file descriptor and a termios structure. I dereference the termios structurewith &orig termios.pointee. UnsafeMutablePointer is like a wrapper. To get to the value within, we must dereference it's pointee property within. Should this fall fail, a fatalError is thrown.

A new instance of a termios structure is created copying the data in the orig_termios. It is dereferenced via the UnsafeMutablePointer's pointee. Following that is to manipulate the data within the new instance of termios by setting the flags. Each of these modes are described in detail in the man pages. Quiet a few of them disable processing of things like hitting Ctrl + C. Some hvae to deal with echoing and more. The man page is an interesting read and I really recommend you giving it a [read](https://man.freebsd.org/cgi/man.cgi?query=termios&sektion=4).

Then, we need to do something with tio. We need the terminal to now use it for it's tty settings. This is done with [tcsetattr](https://man.freebsd.org/cgi/man.cgi?tcsetattr). This function takes our file descriptor for the terminal, an _action_, and the reference to the termios struct we want to put there. The action parameter determines when this change takes place. There are three options; TCSAFLUSH, TCSANOW, and TCSADRAIN. I opted for TCSAFLUSH which is described as

  The change occurs after all output written to fd has been transmitted to the terminal. Additionally, any input that has been received but not read is discarded.

Finally, calling flush on the file descriptor with [tcflush](https://man.freebsd.org/cgi/man.cgi?query=tcflush&apropos=0&sektion=3&manpath=FreeBSD+11-current&format=html). Once this is done, the new terminal settings will take affect. This also takes an action, which is described in the man page as

  Flush both data received but not read and data written but not transmitted.

## Diabling Raw Mode

Jumping ahead, now lets look at disabling raw mode.
```swift
  func disableRawMode(fd: Int32, tio: UnsafeMutablePointer<termios>) {
    guard tcsetattr(fd, TCSAFLUSH, &tio.pointee) >= 0 else {
      fatalError("Issue disabling raw mode")
    }
  }
```

This is needed to get back to the original settings when the program is being exited. Here the UnsafeMutablePointer for the original termios structure reappears. Inside the function, like when setting the termios structure to disable settings, the termios restructure will be set back to the original settings we saved earlier. If you've got a grip on enabling raw mode, disabling it is easy and simple.

## Box Structure

The box to draw is a simple swift structure. This is some of the code for it.
```swift
  struct Box {
    var x: Int16
    var y: Int16
    var width: Int16
    var height: Int16

    init(x: Int16, y: Int16, width: Int16, height: Int16) {
      self.x = x
      self.y = y
      self.width = width
      self.height = height
    }
  }
```

This is the base of the structure. For now, I left out the drawing code just to give a demonstration of the data I am working with. This structure has an x and y as a starting point. This serves as the upper left hand corner of the box. I guess, technically rectangle since it's sides can be variable length. Then the box has a height and a width. An intializer (constructor in non swift languages) is provided just to initialze the box with its values.

## Helper functions

Before drawing a box, it is good to talk about some helper functions.
```swift
  import Foundation

  func writeToTerminal(fd: Int32, message: String) {
    write(fd, message, message.utf8.count)
  }

  func jumpToPosition(fd: Int32, x: Int16, y: Int16) {
    writeToTerminal(fd: fd, message: "\u{1b}[\(x);\(y)H")
  }
```

These are the helper functions for drawing. Two simple functions are needed; writing to terminal and jumping to position. Jumping to position is where we tell the cursor to go on the screen, then from there, we can write linearly along that row. Jump position is an abstraction over writing to the terminal since to do anything, we must write to the terminal. Jump to terminal just formats what we write as the escape sequence to jump the cursor. So instead of writing physical text on the screen, this can be seen as a command to the terminal or a function.

To break down the commands a little, since they look a little weird. Some excellent documentation can be found [here](https://notes.burke.libbey.me/ansi-escape-codes/). For jumping, a focus is on the function in the article described as 'Cursor Position'. The command written to the terminal looks like
```sh
  \x1b[x;yH
```
Or using a point such as (1,1)
```sh
 \x1b[1;1H
```
However, swift, I found to be a little funky here. I couldn't simple pass in '\x1b' because I kept getting errors related to an invalid escape sequence. What I found usefule is to use the unicode escape sequence, which luckily evaluates out to the same hex code. the \x specifies that the input is hex and the actual escape code is #1B. So instead, using (1,1), for us it looks like
```sh
  \u1b[1;1H
```
Writing to the terminal is simple, pass the file descriptor, message and the length of the message by calling on the message's utf8 count property.


## Back to Main: ClearScreen

Now with that described, we can address the clearScreen method in main.
```swift
  func clearScreen(fd: Int32) {
    writeToTerminal(fd: fd, message: "\u{1b}[2J")
  }
```

I will leave it up to you to figure out what that command does using the documentation linked above. One hint.
```sh
  \x1b[2J
```

## Drawing the Box

Finally drawing the box, here is the complete code for the Box structure.
```swift
  import Foundation

  struct Box {
    var x: Int16
    var y: Int16
    var width: Int16
    var height: Int16
    init(x: Int16, y: Int16, width: Int16, height: Int16) {
      self.x = x
      self.y = y
      self.width = width
      self.height = height
    }
    func draw() {
      jumpToPosition(fd: STDIN_FILENO, x: self.x, y: self.y+1)

      for _ in x...x+width-1 {
        writeToTerminal(fd: STDIN_FILENO, message: "\u{2501}")
      } 

      jumpToPosition(fd: STDIN_FILENO, x: x + height, y: y)

      for _ in x...x+width-1 {
        writeToTerminal(fd: STDIN_FILENO, message: "\u{2501}")
      }

      jumpToPosition(fd: STDIN_FILENO, x: self.x, y: self.y + width)
      writeToTerminal(fd: STDIN_FILENO, message: "\u{2513}")
      jumpToPosition(fd: STDIN_FILENO, x: self.x, y: self.y)

      writeToTerminal(fd: STDIN_FILENO, message: "\u{250f}")
      for i in x+1...x+height-1 {
        jumpToPosition(fd: STDIN_FILENO, x: i, y: self.y)
        writeToTerminal(fd: STDIN_FILENO, message: "\u{2503}")
      }

      jumpToPosition(fd: STDIN_FILENO, x: self.x + height, y: self.y)
      writeToTerminal(fd: STDIN_FILENO, message: "\u{2517}")
      jumpToPosition(fd: STDIN_FILENO, x: self.x+height, y: self.y + width)
      writeToTerminal(fd: STDIN_FILENO, message: "\u{251b}")

      for i in x+1...x+height-1 {
        jumpToPosition(fd: STDIN_FILENO, x: i, y: self.y+width)
        writeToTerminal(fd: STDIN_FILENO, message: "\u{2503}")
      }
      
    }
  }
```

The addition is the draw function which will render the box to the terminal, taking advanatage of the helper functions to jump to position and write. The algorithm is simple, draw a side and move to another side, occasionally jumping to corners of the box and drawing the corner.

## Conclusion

That is the code to draw a box in the terminal in swift. Through it, the terminal was placed into raw mode, and drew the box. Input was polled on the q character, break from the polling and disable raw mode, returning the terminal back to it's original set up. 
