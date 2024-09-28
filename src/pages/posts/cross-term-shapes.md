---
layout: ../../layouts/post.astro
title: 'Crossterm Shapes'
author: 'Todd'
pubDate: 2022-09-27
---

# Intro

Lately I have been getting back into writing more rust code. One thing that has my interest in Rust is getting back into playing with terminal user interfaces and drawing graphics in a terminal. As a dip back into this after not playing with it in many months, I have written a little program to draw some basic shapes in a terminal instance. I think eventually it would be fun to play around with building on top of making some basic shapes into creating a my own terminal user interface library. However, that can be a bit of a lofty goal. First, start with some basics.

# What we will be using:

- [Crossterm](https://crates.io/crates/crossterm)
- [Rust](https://www.rust-lang.org/)
- [Code used in article](https://github.com/martintc/ArticleCode/tree/main/crossterm-shapes)

# Crossterm

Crossterm is a multiplatform terminal manipulation library and makes it easy to do some interesting things in the terminal. It provides some nice abstractions so that we don't have to get too deep in the weeds on terminals, which I have done before in C. That can be fun, but not necessarily when you are just looking to play around. But to give a taste of doing this in a more low level way, you often have to use a series of codes to accomplish tasks such as changing the color of text in a terminal or moving around. These are usually ansi escape codes. And it might look a little something like this:

```sh
	\u1b[31mtest\033[ming\n
```

For information on something such as color code, I would refer you to take a look at [this](http://jafrog.com/2013/11/23/colors-in-terminal.html).

Luckily, crossterm provides some nice easy ways of dealing with this without all of that.

# Requirements

The code shown today will be able to accomplish a few things:

- Take control of the terminal it is being ran in
- Enable raw mode
- Process key events
- Move the cursor to a point
- Print a character at a given point
- Print out a square
- Print out a triangle

# Entry point

```rust
  fn main() -> Result<()> {
    let mut stdout = stdout();
    stdout.execute(terminal::Clear(terminal::ClearType::All))?;

    let (x, y) = crossterm::terminal::size()?;
    
    enable_raw_mode()?;

    
    loop {
        match process_input_events() {
            Some(KeyBinding::Quit) => {
		stdout.execute(terminal::Clear(terminal::ClearType::All))?;
		break;
	    },
	    Some(KeyBinding::Clear) => {
		stdout.execute(terminal::Clear(terminal::ClearType::All))?;
	    },
	    Some(KeyBinding::Square) => {
		print_square(&mut stdout, x, y)?;
	    },
	    Some(KeyBinding::Triangle) => {
		print_triangle(&mut stdout, x, y)?;
	    },
            Some(KeyBinding::None) => {},
            None => {}
        }
    }

    Ok(())
  }
```

First to do is to grab a struct of `Stdout.` According to the [documentation](https://doc.rust-lang.org/std/io/struct.Stdout.html), this is a handle on the global standard output stream of the current running process. In the case of our program, it is the terminal. We do this in the first like but evoking the `stdout()` function and setting it equal to a mutable `stdout` variable. The next thing we may want to do is to clear the terminal of pesky text we don't want. Crossterm gives us a nice function that can be used on stdout called [execute](https://docs.rs/crossterm/0.25.0/crossterm/trait.ExecutableCommand.html#tymethod.execute). This allows for executing commands on the stdout. We can pass a command provided in `terminal` called `Clear`. [Clear](https://docs.rs/crossterm/0.25.0/crossterm/terminal/struct.Clear.html) allows the terminal buffer to cleared, however it can take [enums that communicate how to clear it](https://docs.rs/crossterm/0.25.0/crossterm/terminal/enum.ClearType.html). In this case, we want to clear everything, so we can use `ClearType::All.` However, there are many different ways to clear the terminal. We can clear from cursor down, cursor up, only the current line and a few more options. 

Next thing we may want are some bounds on our terminal. We can get this using [terminal::size](https://docs.rs/crossterm/latest/crossterm/terminal/fn.size.html). This will return a tuple that we can unpack into their respective x and y coordinates. It should also be noted that main returns a `Result` and the `?` operator pretty much says if an error happens to bubble this up/bail out of the program. 

Then, want to enter into [raw mode](https://docs.rs/crossterm/latest/crossterm/terminal/index.html#raw-mode). Reading the documentation, raw mode will set the following modes:

- Input will not be forwarded to the screen
- Input will not be processed on pressing enter
- Input will not be line buffered
- Special keybindings like Ctl + C will not be processed (we will define our own quit keybinding)
- New line characters are not processed, so if we write, we must used `write!`. 

Following that is our main loop where we will process some inputs and react based on that. At the end, we return `Ok(())` if we are exiting successfully.


# Keybindings

```rust
  enum KeyBinding {
    None,
    Clear,
    Quit,
    Square,
    Triangle,
  }
```

The enum `KeyBinding` is a set of conditions we intend to handle. When the input process stage happens, it will try to gather input. Depending on this input, input processing will return an enum for the operation to conduct. The operations are fairly straigh forward. `None` is returned if there was no input to process. `Clear` will clear the terminal screen. `Quit` is how from the terminal we communicate that we are quitting. `Square` and `Trianglel` to draw those respective shapes on the terminal.

# Processing Input

```rust
  fn process_input_events() -> Option<KeyBinding> {
    if let Event::Key(event) = event::read().expect("Reading key event failed") {
        match event {
        KeyEvent {
            code: KeyCode::Char('q'),
            modifiers: event::KeyModifiers::CONTROL,
            ..
	    } => Some(KeyBinding::Quit),
	    KeyEvent {
			code: KeyCode::Char('s'),
			..
	    } => Some(KeyBinding::Square),
	    KeyEvent {
			code: KeyCode::Char('t'),
			..
	    } => Some(KeyBinding::Triangle),
		KeyEvent {
			code: KeyCode::Char('c'),
			..
	    } => Some(KeyBinding::Clear),
		_ => Some(KeyBinding::None),
        }
    } else {
        return None;
    }
  }
```

Here is the meat of input processing. Using [Event](https://docs.rs/crossterm/latest/crossterm/event/enum.Event.html) and [event](https://docs.rs/crossterm/latest/crossterm/event/index.html). `Event` is an enum with `Key` being the enum for key events. `Key` contains a [KeyEvent](https://docs.rs/crossterm/latest/crossterm/event/struct.KeyEvent.html). To get this, `event::read()` is called to read any events in the queue. Afterwards, matching on the type of `KeyEvent`. `KeyEvent` is a struct with several data members, a `KeyCode`, `KeyModifier`, `KeyEventKind`, and `KeyEventState`. We only need two here. So, match on the KeyEvent with the specific *codes* (of type KeyCode) and *modifiers* (of type KeyModifiers). The assignments are as follows:

```
     | KeyEvent | Key binding |
     |:---------|:------------|
     | Quit     | Ctrl + q    |
     | Square   | s           |
     | Triangle | t           |
     | Clear    | c           |
```

In all other events, return a `None` Keybinding. 

# Moving and printing points

For this program, points will be printed using an asterik (*). But, before printing shapes, the cursor must be moved. This is handled in the following function.

```rust
  fn move_to_position(mut stdout: &Stdout, x: u16, y: u16) -> Result<()> {
    stdout.execute(crossterm::cursor::MoveToRow(y))?;
    stdout.execute(crossterm::cursor::MoveToColumn(x))?;
    Ok(())
  }
```

Calling execute on `stdout` using the [crossterm::cursor functions](https://docs.rs/crossterm/latest/crossterm/cursor/index.html). This will allow a y and x to be passed in and the cursor will move to that location.

The following function will be called to print a point.

```rust
  fn print_point(mut stdout: &Stdout) ->  Result<()> {
    stdout.execute(crossterm::style::Print("*".blue()))?;
    Ok(())
  }
```

[Print](https://docs.rs/crossterm/latest/crossterm/style/struct.Print.html) is located in `crossterm::style` and allows for some simple and useful functionality. It will take an argument that implements `Display` and functions can be called on that to add some extra features like `.blue()` will take whatever is being printed and print it out as blue. 

# Drawing shapes

```rust
  fn print_square(mut stdout: &Stdout, x: u16, y: u16) -> Result<()> {
    stdout.execute(terminal::Clear(terminal::ClearType::All))?;

    let x_0 = x / 4;
    let x_1 = x_0 * 3;
    let y_0 = y / 4;
    let y_1 = y_0 * 3;

    	for j in x_0..x_1 {
	    for k in y_0..y_1 {
	    	move_to_position(&stdout, j, y_0)?;
	    	print_point(&stdout)?;
	    	move_to_position(&stdout, j, y_1)?;
	    	print_point(&stdout)?;
	    	move_to_position(&stdout, x_0, k)?;
	    	print_point(&stdout)?;
	    	move_to_position(&stdout, x_1, k)?;
	    	print_point(&stdout)?;
		}
    	}
    
    Ok(())
  }

  fn print_triangle(mut stdout: &Stdout, x: u16, y: u16) -> Result<()> {
    stdout.execute(terminal::Clear(terminal::ClearType::All))?;

    // let x_0 = x / 3;
    // let x_1 = x_0 * 2;
    // let y_0 = y / 3;
    // let y_1 = y_0 * 2;

    // for j in x_0..x_1 {
    // 	for k in y_0..y_1 {
    // 	    move_to_position(&stdout, j, y_1)?;
    // 	    print_point(&stdout)?;
    // 	    move_to_position(&stdout, x_0, k)?;
    // 	    print_point(&stdout)?;
    // 	}
    // }

    let mut array = vec![vec![0; x.into()]; y.into()];

    array[10][10] = 1;
    array[11][10] = 1;
    array[12][10] = 1;
    array[13][10] = 1;
    array[14][10] = 1;
    array[11][11] = 1;
    array[13][11] = 1;
    array[12][12] = 1;

    for (i, row) in array.iter().enumerate() {
	for (j, _col) in row.iter().enumerate() {
	    if array[i][j] == 1 {
		move_to_position(&stdout, i as u16, j as u16)?;
		print_point(&stdout)?;
	    }
	}
    }
    
    Ok(())
  }
```

These functions will print the shapes respectively. I demonstrated two different ways. The square is a little algorithm that traces out a square based on the bounds provided with the sides computed. The triangle is a little more manual and demonstates how a 2-D array (vectors in Rust's specific case) can be used to draw to the screen. If one wanted to maybe make some more complicated drawing, some functionality could be built in to read in a text file with a shape drawn in it and use that as a base for what to draw.
