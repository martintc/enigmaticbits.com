---
layout: ../../layouts/MarkdownPostLayout.astro
title: 'Ravenports: Making a dwm port'
author: 'Todd'
pubDate: 2023-01-09
---

It has been a little while since I've played with Ravenports. To give a brief introduction, it is an alternative package management system developed by John Marino. It is focused on being a more modern FreeBSD ports tree or pkgsrc ports tree while also being cross platform much like pkgsrc. I was first introduced to it by a friend and I immediately liked some of the features. I even played a small role in porting over the ravensw tool (binary package management tool) to [netbsd](https://github.com/Ravenports/ravensw/commit/153cbc790866ed963f8b0c9e18c5486f88a70120) and [here](https://github.com/Ravenports/ravensw/commit/da89891c85ee062e476b3a245b831ad80cee80dc). 

After having been busy and not a chance to play with it, I am looking to get back into it. First, setting up a NetBSD VM since NetBSD is my preferred BSD. The instructions for getting Ravenports can be found on the the wiki for [Ravenports](https://github.com/Ravenports/Ravenports/wiki/quickstart-netbsd). John has made the process pretty easy and painless. Following those instructions will get Ravenports working on a NetBSD system. If using another system, instructions are also available.

To get back into the saddle, my plan is to try to write a new port for DWM. The choice for this is that DWM is fairly easy to build, being a suckless tool. I also favor it for a window manager when I am using one. If I start working more on Ravenports, I will probably want to stand up my NetBSD machine again using DWM as my window manager. Also, DWM will require a slight tweak for NetBSD which I have done before.

For a basic overview of making a new port, John provides [documentation](https://github.com/Ravenports/Ravenports/wiki/Ravenporters_Guide) giving details on the ports system along with a walk through of making a [port](https://github.com/Ravenports/Ravenports/wiki/Chapter-14). Which is the documentation I will be working off of.

First step is to select a good port name. This is fairly obvious, the port name should match the package.... So.... the name is dwm then.

Step two is to create an entry. First is to clone the ravensource repository which is where all specifications are located. A specification is like the recipe for the port on how Ravenports should build it. Ports are organized into buckets ranging from `0x00 to 0xFF.` To find the bucket the new port should be located, ravenadm provides a tool.
```sh
     ravenadm locate dwm
```
And it outputs

```sh
    Does not exist as /var/ravenports/conspiracy/bucket_F4/dwm
```

Yea, it isn't located there, but the file path tells me which bucket the port should exist in, which in this case is bucket `0xF4.` So, I can make my directory here.

Next, step three is a time saving step to create a template after navigating to the new directory made with `mkdir ~/ravensource/bucket_F4/dwm.`

```sh
      ravenadm dev template save
```

This will create a directory for `manifests,` `descriptions,` and a file called `specification.` 

For now, skipping step 4 since I am not familiar with this concept and may circle back around to it as I get further. At least not with the defining of it. I believe this has to do with the `SPKGS[]` that can be set.

As per step 5, since I am not finalizing a sub package yet, I will be skipping to step 7.

Now for the fun bits, filling out the specification sheet. Ravenports has a rule that a new port must come in at the latest version. If the latest version is 6.4, you can't introduce a new port at a version less than that. DWM's current latest version is 6.4, so that will be the `DEF[PORTVERSION]` value. Then name, `NAMEBASE` is dwm for the package. The `KEYWORDS` will be set to x11_wm as it is for the awesome window manager.

Variants. Ravenports has an idea of packages having variants. One example of this is git. There are two main different version of git in Ravenports. One is the full blown git with everything and there is a lite variant. Here, there will be a single variant that will be called standard in this case. Filling in a description, next setting the homepage for the project and in the first block, finishing it with contact information. Since I am introducing this port, I will put in my details.

In the next block, specify the download information. `SITES` specifies the main part of the URL to download with `DISTFILE` being the final bit. In my case
```sh
   SITES[main] = https://dl.suckless.org/dwm
   DISTFILE[main] = dwm-${PORTVERSION}.tar.gz:main|
```
A github repo can also be used, but I opted to get it from the suckless downloads.

The next piece I will fill in is the `FPC_EQUIVALENT` which is what the port would be on FreeBSD. Which is `x11-wm/dwm.` This can be found by searching for the package in the ports tree.

Important is to add licensing. DWM is MIT, so is important to list that and the abbreviations accepted by ravenports can be found in an article in the [wiki](https://github.com/Ravenports/Ravenports/wiki/Licensing).

I don't think I will run into build dependencies. It has been awhile, but I will call it quits on the specification file for now. If I need to add build dependencies, I will do so.

Moving to step eight, generating the distinfo file.
```sh
       ravenadm dev distinfo
```
Which that command will do for me. Ravenadm should also call out any issues.

And of course the first issue has appeared. My port version is commented out which I did not pay attention to. So, important note. The auto-generated template has the `DEF[PORTVERSION]` commented out. Easy fix, remove the `#` at the beginning of the line.

My next complaint is a space trapped between tabs. Ravenports is particular about formatting, but generally, it will guide you on how to fix it.

After a few correction about mis-typings, the `dev distinfo` is succeeding.

Since DWM uses gmake, it is also important to note my addition of the `USES` on the specification.
```sh
      USES= gmake
```
For those not familiar. There are two main different kinds of make. Gmake is the gnu make that one might be more familiar with as it is used a lot on linux systems. On the BSDs there is also BSDMake. On BSD systems, gmake is for the gnu make and the regular make command is for the BSD make. 

Step 9 is to incorporate the new port into the conspiracy directory with 
```sh
     ravenadm dev buildsheet . save
```
Then the index needs to be regenerated.
```sh
     ravenadm dev generate-index
```
Now, it is time for the magic. I know this will fail if I remember correctly. NetBSD will need a patch, but I will let the build process tell me what is wrong in case there is anything else I missed. Like perhaps I do need to define some build dependencies.

Ravenadm includes a test which is like the build command but also includes some extra tests.
```sh
	 ravenadm test dwm
```
The build failed of course, as expected. To see what happened, it generated a log in `/var/ravenports/primary/logs/logs` to look at.

Looking at the log, just as I expected. Hit an error on `drw.c` with the line
```sh
	#include <X11/Xlib.h>
```
One reason is that looking more at the logs, it is linking against `-L/usr/X11R6/lib` which does not exist on NetBSD. NetBSD instead has a path for `-L/usr/X11R7` instead. A patch will need to be applied. In general, I think John recommends to see if pkgsrc has a patch and using that if there is one instead of writing a patch.

However, this is the wrong thread to go down! As I found out from my friend over at the eerie linux blog, we need to context switch how we view the system. The solution normally would be replace the occurrences of `X11R6` with `X11R7`. In the case of Ravenports, our ports build inside of a chroot environment, or jail if FreeBSD. The chroot environment is also special, it is an environment specifically for Ravenports that tries to abstract away the details of the OS. Things in Ravenports should be as dependent on Ravenports components as much as possible, to the point where Ravenports ships its own NetBSD environment for it to reference itself. So I need to generate a patch taking this into consideration. What needs to be done is to replace every occurrence of `/usr/local` with `/raven`. Looking at `config.mk` for dwm, there is a `PREFIX` environment variable, this is what needs to be changed. The following is a portion of the patch, but not the full patch.
```sh
	 --- config.mk.orig	2022-10-04 19:38:18.000000000 +0200
	 +++ config.mk	2023-01-12 19:21:22.682277000 +0100
	 @@ -4,11 +4,11 @@
	 # Customize below to fit your system
 
	 # paths
	 -PREFIX = /usr/local
	 +PREFIX = __PREFIX__
	 MANPREFIX = ${PREFIX}/share/man
```
The purpose of using `__PREFIX__` was also a suggestion of my friend over at the [eerielinux](https://eerielinux.wordpress.com/). Instead of hard coding the patch, using PREFIX and later we will use sed to replace every occurrence of `__PREFIX__`. The reason for this is in the event, for some reason, an organization needs to use a path other than `/raven`. So now, that needs to be added to the specification. Down at the bottom of the specification, I added this bit.
```sh
    post-patch:
	${REINPLACE_CMD} 's!__PREFIX__!${PREFIX}!g' \
		${WRKSRC}/config.mk

`${REINPLACE_CMD}` is another way to call sed to edit in place. This will replace all occurrences of `__PREFIX__` with `${PREFIX}`, which in this case is `/raven` since it is not being overridden in the working folder for the file `config.mk`. 
```
Doing back through the test the port, there are more errors. Some of the errors that need to be tackled from here will have to do with freetype, X11 and pkgconfig. First tackling pkgconfig, which is simple. I need to add it to the `USES`.
```sh
      USES= gmake pkgconfig
```

This should take care of pkgconfig. Next, freetype, which is also simple. Freetype is a build dependency. Below the `FPC_EQUIVALENT` line and above `LICENSE` I add
```sh
     BUILD_DEPENDS= freetype:primary:standard
```

Originally, I had `freetype:complete:standard` but on advice from my friend, I don't need complete. Complete will pull docs and other things, in this case, I only need to get the code to compile and use to build. 

Next to tackle is X11. Looking at the [fluxbox specification](https://github.com/Ravenports/ravensource/blob/master/bucket_76/fluxbox/specification), I noticed a line with `XORG_COMPONENTS` So, I added that to my specification and just the pieces from X11 that I knew I needed to compile. 

Lastly, need to take a look at the full patch. 
```sh
	--- config.mk.orig	2022-10-04 19:38:18.000000000 +0200
	+++ config.mk	2023-01-12 19:21:22.682277000 +0100
	@@ -4,11 +4,11 @@
	# Customize below to fit your system

	# Paths
	-PREFIX = /usr/local
	+PREFIX = __PREFIX__
	MANPREFIX = ${PREFIX}/share/man

	-X11INC = /usr/X11R6/include
	-X11LIB = /usr/X11R6/lib
	+X11INC = __PREFIX__/include/X11
	+X11LIB = __PREFIX__/lib/X11

	# Xinerama, comment if you don't want it
	XINERAMALIBS  = -lXinerama
	@@ -16,7 +16,7 @@

	# freetype
	FREETYPELIBS = -lfontconfig -lXft
	-FREETYPEINC = /usr/include/freetype2
	+FREETYPEINC = __PREFIX__/include/freetype2
	# OpenBSD (uncomment)
	#FREETYPEINC = ${X11INC}/freetype2
	#MANPREFIX = ${PREFIX}/man
```

This is the full patch that was needed. One important thing to point out here are the modifications for `X11INC` and `X11LIB`. The paths need to be changed to reference the X11 components of Ravenports, which we get from the `XORG_COMPONENTS` is my understanding. So, I am not building against the normal `/usr/X11R7` since I am building in the Ravenports environment. 

Running the test, it will still fail, but that is because I am missing one more component. I don't have a manifest file. Luckily, we can rely on Ravenports generating this for free. I can also manually generate it, but, why not be lazy? The manifest for the build is located in `/var/ravenports/primary/manifest`. I copy that into my `manifest` folder and rename it to `plist.single`.

Running the test again........ SUCCESS! It builds. Now install it and test it.... IT WORKS! I now have a working port for dwm for Ravenports.

