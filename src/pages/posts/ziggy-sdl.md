---
layout: ../../layouts/post.astro
title: 'Getting ziggy with SDL'
author: 'Todd'
pubDate: 2024-12-12
---

My daughter has recently inspired me to play around with an idea. She had gotten a tomagotchi and thought the thing was pretty cool. Looking at it, I thought, "huh, I can probably do something like that." Now, I've never played with one, but they seem pretty simple. Doing hardware and having a reason for doing some embedding programming, more or less an actual project, would be pretty fun also. However, before I juggle both, I just want to focus some on the software side of things. Also, I have been giving Zig the eye again, but I needed some decent and fun thing to do with Zig.

So, I have decided to try maybe proto-tpying a little bit of a tomagotchi clone in Zig. I've got a programming language, now I need some other bits. Obviously, I need to render something, so in walks SDL. Interoping Zig with C code is pretty simple. It is one of the 'promises' of Zig. Why not give it a shot? I will go over some basic on getting started, but not too much indepth. Just setting Zig up to work with SDL and just a few basic things while also talking a little bit about the Zig language. Nothing too complicated or indepth.

# Zig <3 C

I've tried experimenting with using SDL with Swift and its a bit of a pain. The libraries which provide wrappers and setting them up takes a bit to read through the documentation on SwiftPM. Rust isn't too bad, but doing things with bindgen is not exactly easy or low effort. Zig though promises it to be pretty smooth.

First things first, need to have SDL on the machine. Another reason why I've had some issues with Swift is that since SDL3 has been around, I've been wanting to use the new hotness with SDL. Why build stuff on SDL2 when SDL3 is available. Granted, I was using it before they started releasing binaries, so I've been compiling it myself. Which SDL provides some good documentation for. Zig makes this easy. It really is just a line or two in the `zig.build` file and the compiler and build system handles the rest.

```zig
exe.linkSystemLibrary("SDL3");
exe.linkSystemLibrary("c");
```

That is really all there is to it for incorporating it in the build. 

Much like Swift with SwiftPM, one thing that I like about Zig is that the build system is essentially defined in Zig itself. Well, with Swift, it might be more fair to say that defining dependencies and how those link and build are defined in Swift.

# Setting up a Game struct and the main file.

A lot of examples of Zig with SDL that I have come across have done all the important setup in the main function of `main.zig`. I didn't want to do this, I wanted to to be somewhat separate. Here is what my `zig.main` looks like.

```zig
const game = @import("game.zig");

pub fn main() !void {
    var g = game.Game.init();
    try g.initialize();
    defer g.destroy();
    try g.run();
}
```

All of the 'game' logic resides in the game struct and it's methods/functions. The first line is importing the `game.zig` file which contains the struct and it's associated logic. The main function returns void, but can also return an error. This is almost like `fn main() -> Result<()>` in Rust. What is does is simple, initialize a `Game` object, then we try to initialize it. The line after that I will get to in a second, but it is interesting, and then we just try to call the run method on the object.

The `defer g.destroy()` is interesting. It is used to ensure that resources are cleaned up with no longer needed. I am not fully aware of *all* of the details around it, but it works a little bit like the destructor on a smart pointer in C++ or the drop trait in Rust. Not exact, but so far, I have been able to get by with thinking about it like that.
