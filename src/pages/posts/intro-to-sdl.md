---
layout: ../../layouts/post.astro
title: 'Intro to SDL'
author: 'Todd'
pubDate: 2022-08-25
---

So, Recently I have taken to exploring development with SDL2 and OpenGL. SDL2 was fairly easy to get start with. However, OpenGL appears to be a whole other beast. My plan is play around with graphics rendering using C, SDL2, and OpenGL. A little bit of a roadbump is exploring tutorials that are rather up to date.

On my fedora machine, the OpenGL version is 4.6, while it appears that most tutorials assume an older version of OpenGL version 3 or OpenGL version 2. Admittingly, I also have not messed aroung much with graphics programming. So this is an area I am for sure lacking in. Most of my experience is with using common, run of the mill frameworks such as Java swing, JavaFX, GTK, and Winforms. So for me, raw rendering graphics is a bit of a new field for me. Hopefully in this little post, I can post come content to point someone else looking into this in the right direction. As a warning, a lot of the OpenGL tutorials are also in C++. However, this is not so bad since it is fairly easy to 'convert' C++ into C code. 

To get started down this path to play with this, first step is to take a look at SDL. My ultimate goal is to mess around with the idea of creating a 2D platformer game using SDL and OpenGL.

# SDL2

First off the bat. It is helpful to pick up some library to couple with OpenGL such as SDL. Many others exists. There is, off the top of my head, GLFW and a few others. However, looking at SDL2, I liked how the documentation was laid out and how quick I found it to get up and running. SDL2 follows a basic format just to get something rendering. First, rendering SDL with the appropriate mode(s). Then creating a window and a renderer. This is atleast for working exclusively in SDL2 without coupling it with OpenGL.

In order to initailize SDL, we can call the following function.
```c
   SDL_Init(SDL_INIT_VIDEO);
```
The above will initialize SDL in video mode, which is the least we need to render graphics. It is also important to note that we will need to include the SDL library. This will entail ensuring that we have the library installed. On Fedora, the following is needed to get that.
```c
  sudo dnf isntall SDL2 SDL2-devel
```
This will place the libary in the appropriate place. Now we need a simple makefile.
```sh
  all:
    gcc -c main ./src*.c -lSDL2
```
Then to make sure that the headers are included. So far, we have this for our initial SDL2 program.
```c
  #include <stdio.h>
  #include <SDL2/SDL.h>

  int main(void) { 
      SDL_Init(SDL_INIT_VIDEO);
      return 0;
  }
```
Since we want to make sure that no errors have occured, we can see from the documentation that on `success`, the init function in SDL2 will return 0. In the event of an error, it will return a value less than zero. Also, a build in function can be taken advantage of to get details on the error. This is the `SDL_GetError` function that returns a `char*`.
```c
  #include <stdio.h>
  #include <SDL2/SDL.h>

  int main(void) {
    if (SDL_Init(SDL_INIT_VIDEO) < 0) {
      fprintf(stderr, "Error initializing SDL: %s\n.", SDL_GetError());
      return 0;
    }
    return 0;
  }
```
Next, creating a window. 
```c
  SDL_Window *window = SDL_CreateWindow(
    "A title",
    SDL_WINDOWPOS_UNDEFINED,
    SDL_WINDOWPOS_UNDEFINED,
    800,
    600,
    0
  );
```
The above code will create a window in SDL. It takes a variety of arguments. The first being some `char*` to represent the title of the window. The next two arguments are coordinated along the X and Y of a starting position for the window. Here, I don't really care where the window appears, so I pass in `SDL_WINDOWPOS_UNDEFINED`. The next two arguments are the width and height of the created window. Lastly, this argument is for setting various features. I don't really need any, so I just pass 0. However, when it comes to using OpenGL, a flag will need to be passed in there. Also, multiple flags can be passed by `ORing` them. 

Next is initializing the renderer.
```c
  SDL_Renderer *renderer = SDL_CreatRenderer(window);
```c
Creating the renderer is fairly easy. Just pass in the window object at initialization.

One thing we are going to want to go is some error handling so that way if the program crashes, we know where it took place. After initializing either the renderer or the window, we can add in the following code, replacing the component being checked.
```c
  if (window == NULL) {
     fprintf(strderr, "Error initializing SDL Window: %s\n", SDL_GetError());
     return 0;
  }
```
All in all, the source code looks like so.
```c
  #include <std.io.h>
  #include <SDL2/SDL.h>

  int main(void) {
    if (SDL_Init(SDL_INIT_VIDEO) < 0) {
      fprintf(stderr, "Error initializing SDL: %s\n", SDL_GetError());
      return 0;
    }

    SDL_Window *window = SDL_CreateWindow(
      "My Title",
      SDL_WINDOWPOS_UNDEFINED,
      SDL_WINDOWPOS_UNDEFINED,
      800,
      600,
      0
    );

    if (window == NULL) {
      fprintf(stderr, "Eror initializing SDL Window: %s\n", SDL_GetError());
      return 0;
    }

    SDL_Renderer  *renderer = SDl_CreateRenderer(window);
    if (renderer == NULL) {
      fprintf(stderr, "Error initializing SDL Renderer: %s\n", SDL_GetError());
      return 0;
    }

    return 0;
  }
```
Something else we will want to do is to make sure we clean out our resources.  Before the final `return 0`, we can use SDL's methods to clean up, by placing these lines.
```c
  SDL_DestroyRenderer(renderer);
  SDL_DestroyWindow(window);
  SDL_Quit();
```
