---
layout: ../../layouts/MarkdownPostLayout.astro
title: 'Ravenports: making a dasm port'
author: 'Todd'
pubDate: 2023-01-24
---

My first [article](https://martintc.tech/post/ravenports-dwm/) for making a port was a little complex. In this post, I will cover some of the details for making a simpler port for Ravenports. To view all the files, you can reference the [pull request](https://github.com/Ravenports/ravensource/pull/248/files) that was approved and pulled into Ravensource.

First, covering some of the workflow for creating a new port in Ravenports. For myself, since I am being mentored, I am pulling directly against the [ravensource repository](https://github.com/Ravenports/ravensource). However, there is the [custom source](https://github.com/Ravenports/customsource) repository which is meant for new contributors and for WIP (Work In Progress) ports. Over time, often many times a week, a "release" is rather cut from ravensource and the ports tree is generated. This ports tree is the [ravenports](https://github.com/Ravenports/ravenports) repository. Then from here, a more official release is cut. From my understanding, this is when all the packages in the ports tree are compiled for all targeted architectures and will be made available through the binary package manager, ravensw.

Next, always starting with the specification.
```sh
  DEF[PORTVERSION]=	2.20.14.1
  # ----------------------------------------------------------------------------

  NAMEBASE=		dasm
  VERSION=		${PORTVERSION}
  KEYWORDS=		devel lang
  VARIANTS=		standard
  SDESC[standard]=	Versatile macro assembler for 8bit microprocessors
  HOMEPAGE=		https://dasm-assembler.github.io
  CONTACT=		Todd_Martin[warfox@sdf.org]

  DOWNLOAD_GROUPS=	main
  SITES[main]=		GITHUB/dasm-assembler:dasm:${PORTVERSION}
  DISTFILE[1]=		generated:main

  SPKGS[standard]=	single

  OPTIONS_AVAILABLE=	none
  OPTIONS_STANDARD=	none

  LICENSE=		GPLv2:single
  LICENSE_FILE=		GPLv2:{{WRKSRC}}/LICENSE
  LICENSE_TERMS=		single:{{WRKDIR}}/TERMS
  LICENSE_SOURCE=		TERMS:{{WRKSRC}}/src/main.c
  LICENSE_AWK=		TERMS:"^$$"
  LICENSE_SCHEME=	solo

  FPC_EQUIVALENT=	devel/dasm

  USES=			gmake

  post-stage:		${STRIP_CMD} ${STAGEDIR}${PREFIX}/bin/*
```
Here the version number is set, so is the namebase along with other basic information on the port. The interesting bit is with the `DOWNLOAD_GROUPS`. Here, the source is being pulled form github. As of this writing, [chapter 11](https://github.com/Ravenports/Ravenports/wiki/Chapter-11) of the wiki in the ravenports repository covers the basic usage of pulling sources from a github repository. Although, [many](https://github.com/Ravenports/Ravenports/wiki/Sites) are also covered such as OpenBSD mirrors, GNU's mirrors, and SourceForge are commons ones that can be found being used when going through specification files.

Only a single package of package type single is defined with no options, however the license sections gets a little interesting. DASM is licensed with GPLv2. What is required here is a license file and a terms file. The license file is rather simple, most repositories have a LICENSE or License.txt file which should contain this as is the case in DASM. However, the terms are a little tricky. I am still not 100% on the details of this, but most of the time, the terms can be found in a source file as usually the header. Which what I have done here is use awk to go and grab those license terms. For DASM, these are the license terms.
```sh
  /*
    the DASM macro assembler (aka small systems cross assembler)
    Copyright (c) 1988-2002 by Matthew Dillon.
    Copyright (c) 1995 by Olaf "Rhialto" Seibert.
    Copyright (c) 2003-2008 by Andrew Davie.
    Copyright (c) 2008 by Peter H. Froehlich.
    This program is free software; you can redistribute it and/or modify
    it under the terms of the GNU General Public License as published by
    the Free Software Foundation; either version 2 of the License, or
    (at your option) any later version.
    This program is distributed in the hope that it will be useful,
    but WITHOUT ANY WARRANTY; without even the implied warranty of
    MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
    GNU General Public License for more details.
    You should have received a copy of the GNU General Public License along
    with this program; if not, write to the Free Software Foundation, Inc.,
    51 Franklin Street, Fifth Floor, Boston, MA 02110-1301 USA.
  */

  /*
   *  MAIN.C
   *  DASM   sourcefile
   *  NOTE: must handle mnemonic extensions and expression decode/compare.
   */
```
Next, specifying that gmake is used to build this port. Ravenadm will automatically call the generic
```sh
      gmake -f Makefile -j4 all
```
Where the 'j' option is dependant on `Max. jobs per builder` specified in the ravenadm configuration tool.

Then to finish off, the last bit of the specification is stripping the binaries. At a surface level, some binaries and libraries are compiled with everything for debugging, this is the `-g` switch when compiling in most C compilers. Stripping will strip this out of the binaries or libraries since that is not needed for someone to just use the produced binary. This occurs in the `post-stage` and ravenports gives us some nice short hands. `${STRIP_CMD}` will be the strip command as the name implies. `${STAGEDIR}${PREFIX}` will take us to where ravenports stages these files for packaging. I do use an asterick for all files in the `bin` directory since DASM does compile and make two binaries; dasm and ftohex.

Outputs that end up in the final package can be found in `manifests/plist.single`. This is a list of all files that will be in the package generated. As it can be seen, there are listing for both binaries this package makes when compiling. Ravenports also lists the path starting after `${STAGEDIR}${PREFIX}` since it knows to fill what is needed later on.

This port did require a single patch to the makefile. This actually was not necessary as it could have been handled in the specification during on of the pre- or post- stages. However, I instead implemented the install block in the Makefile.

This is a pretty good sample of a simpler port than the dwm port that was covered in a previous article.

