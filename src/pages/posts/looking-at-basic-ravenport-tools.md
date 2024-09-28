---
layout: ../../layouts/post.astro
title: 'Looking at basic Ravenports tools'
author: 'Todd'
pubDate: 2023-01-26
---

I have done a couple of ravenport articles about making ports, but what about the actual personal workflow for developing a port? That is something I have not tackled yet and I will do so here. My plan is to give a basic outline of my process right now and some tooling in ravenports to help with this.

Starting with my own process. I am right now, doing most of my ports development, using a virtual machine in QEMU with NetBSD 9.3. As of right now, the installation for it is pretty basic, mostly defaults. If you want to have a working Xorg set up to use a window manager or desktop environment in the VM, I would recommend selecting the full installation option with all sets, it will come with xorg/x-11 that NetBSD ships with the image. Right now the xorg/x-11 for NetBSD provided by Ravenports has some issues. However, we can install a window manager from ravenports and run it with the full X11 sets provided by the installation media. I have bounced back and forth between using ctwm that comes with the NetBSD provided X11 sets and openbox that I get as a binary from ravenports. I do recommend having a windows manager even though ports development is a lot of text editing and compiling. I try to have one terminal for editing my specifications and other file and another terminal window to pull up logs of my builds. This way when ravenadm fails a build, in another terminal and I can use the up arrow to recall the last command I used to read through the logs. Ports development leads to a lot of that. Doing a few things, building, waiting, and finally reading logs to troubleshoot and restart the process.

Another crucial part to my process, but also immensely helpful that ravenports provides is the idea of "unkindness." Unkindness is like a supplementary source for build specifications that an end user can do. When set appropriately, when `ravenadm build <port_name>` or `ravenadm test <port_name>` is ran, ravenadm will look at the currently set unkindness and build the final sheets on the fly. This allows keeping WIP ports seperate from the actual ports tree or copy of the ports tree. It also reduces some of the commands. One of the command is reduces it the need to call `ravenadm dev buildsheet . save` The buildsheet will be made and test/build launched using unkindness. In order to set unkindness, need to access the ravenadm configuration.
```sh
  sudo ravenadm configure
```
This will pull up a menu. Looking at option `E`, it allows for setting a custom ports directory. This is what I have been calling unkindness. Selecting that option and then entering in the full path to the directory that serves as a custom ports directory. I manage this using git and keeping it all in a respository in [github](https://github.com/martintc/unkindness).

There are some other options in this configure menu that are of use to play with. The conspiracy directory is where all the buildsheets that ravenadm reference actually live. This probably should not be messed with in most cases. Most of the top half of options besides setting a custom ports directory should not be messed with. There is an option to enable a compiler cache directory. Most interesting is being able to set our number of concurrent builders and max jobs per builder. Ravenadm should pick good defaults here, but sometimes it is worth the effort to tweak. I have an old Macbook 2007 with a Core 2 Duo processor that I usually set both of those values to 1 when I am building ports on it. There is an option to fetch prebuilt packages which can be nice to cut down package building time and lessen a load on a machine. I do this on that same 2007 Macbook running NetBSD 9.3. The display when building can also be turned off if your not into the pretty ncurses display that shows the status of building.

When a new conspiracy collection from the Ravenports team has been generated. In order to upgrade your local snapshot of the conspiracy collection (ports tree), there is a command for that.
```sh
  sudo ravenadm update-ports
```

In order to see if a newer version is available,
```sh
  sudo ravenadm check-ports
```
If you want to build the entire ports collection
```sh
  sudo ravenadm build-everything
```
If your want to produce your own mirror of the entire ports collection
```sh
  sudo ravenadm generate-repository
```
To run test across all ports
```sh
  sudo ravenadm test-everything
```
A really important command to remember when trying to either find a port's location, if it exists or where to place a port when bringin in a new port
```sh
  ravenadm locate <port_name>
```
It will say if it exists or not. The path provided also will show where it lives or where it should live.

