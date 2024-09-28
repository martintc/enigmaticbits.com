---
layout: ../../layouts/post.astro
title: 'SwiftDL3 - Wrapping SDL3 for Swift Interop'
pubDate: 2024-07-16
author: 'Todd'
---

Over the last 6 months, I have been dabbling in a bit of graphics programming and game development. I started working on a framework called Flatland when I realized that I was perhaps a little over my head. I plan to write about that later. But there were definitely some areas I was weak in. The motivation of Flatland is a sort of framework/engine to make 2D games that works with SwiftUI and Apple Metal as the graphics API. While I can render graphics, doing so well is the difficult part where I found that I couldn't really figure out a good way to organize it. 

So, I decided to take a bit of a step back and focus on perhaps a simpler engine. One that is more like a Wolfenstien-esque style engine. So 2D that fakes 3D via ray tracing. Plus, there are tons of reference material and posts to read about why various design choices are made regarding a Wolfenstein style engine. And of course, I want to do it in Swift.

Initially my plan was to stick with UIKit or SwiftUI, but then I figured, well, why not just use SDL? Especially since SDL3 is sorta out right now. Well, looking at the repository, there are not official tags, but the 'main' branch is essentailly SDL3. Plus, it will simplify some things since I am not well versed in UIKit. Plus, SDL is also more cross platform. Perhaps getting back into dabbling with SDL, I can stick with SDL when I get back to Flatland to render using Metal. Or use a software renderer as a stepping stone for the design of the rendering engine with an Apple Metal backend. TBut the cross-platformness of SDL would be great because I would also be able to use this on Linux. And not going to lie, I am secrely hoping I wake up one day to find that there is some upstream support for FreeBSD/NetBSD for Swift. 

However, this presents a little bit of a problem. The current libraries available via Swift Package Manager (SPM) are a mix of either out of date, or still only target SDL2. So, guess what? I guess I need to make my own package to wrap SDL3. 

So first things first, how do we do this? Well, Swift does provide a [guide](https://www.swift.org/documentation/articles/wrapping-c-cpp-library-in-swift.html), so lets follow along and see how it goes.

I create my folder.

```sh
mkdir SwiftDL3
```

Then time to navigate into it and initialize a new library package.

```sh
cd SwiftDL3
swift package init --type library --name SwiftDL3
```

Now, I have the skeleton for the project. A few more things to do before really digging in.

Unfortunately, it doesn't appear to initialize a git repositority for us and that will be important with as per the instructions, we need to use a git submodule. Or atleast, that is route I plan to go.

```sh
git init
```

Also I don't quiet like the `SwiftDL2.swift` file, so I want to get rid of that.

```
rm Sources/SwiftDL3/SwiftDL3.swift
```

Now, nagivate into the folder for the library and add our git submodule

```sh
cd Sources/SwiftDL3
git submodule add https://github.com/libsdl-org/SDL.git
```

Once that is done cloning, verify the `.gitmodules` file.

```sh
cat ../../.gitmodules
[submodule "Sources/SwiftDL3/SDL"]
	path = Sources/SwiftDL3/SDL
	url = https://github.com/libsdl-org/SDL.git
```

Looks good so far.

```sh
cd ../../
```

Let take a look at the current `Package.Swift` file.

```swift
// swift-tools-version: 5.10
// The swift-tools-version declares the minimum version of Swift required to build this package.

import PackageDescription

let package = Package(
    name: "SwiftDL3",
    products: [
        // Products define the executables and libraries a package produces, making them visible to other packages.
        .library(
            name: "SwiftDL3",
            targets: ["SwiftDL3"]),
    ],
    targets: [
        // Targets are the basic building blocks of a package, defining a module or a test suite.
        // Targets can depend on other targets in this package and products from dependencies.
        .target(
            name: "SwiftDL3"),
        .testTarget(
            name: "SwiftDL3Tests",
            dependencies: ["SwiftDL3"]),
    ]
)
```

Let try just running this.

```sh
swift build
```

Looks like I already hit an error.

```sh
error: 'swiftdl3': manifest property 'defaultLocalization' not set; it is required in the presence of localized resources
```

Doing a quick google search, I found a [page](https://stackoverflow.com/questions/67871929/swift-package-manifest-property-defaultlocalization-not-set) that provided a problem. Lets just add this to the `Package.swift`.

```swift
  defaultLocalization: "en",
```

Building now results in a different error.

```sh
warning: 'swiftdl3': 'SwiftDL3' was identified as an executable target given the presence of a 'main.swift' file. Starting with tools version 5.4.0 executable targets should be declared as 'executableTarget()'
error: 'swiftdl3': library product 'SwiftDL3' should not contain executable targets (it has 'SwiftDL3')
```

Seems a little odd that the simpiler is thinking it is an executable target and not a library. Especially since there is not a `main.swift` file.

Making some edits to the package file. It looks like I have some progress.

```swift
// swift-tools-version: 5.10
// The swift-tools-version declares the minimum version of Swift required to build this package.

import PackageDescription

let package = Package(
  name: "SwiftDL3",
  defaultLocalization: "en",
    products: [
        // Products define the executables and libraries a package produces, making them visible to other packages.
        .library(
            name: "SwiftDL3",
            targets: ["SwiftDL3"]),
    ],
    targets: [
        .target(
          name: "SwiftDL3",
          dependencies: [],
          exclude: [],
          sources: [],
          cSettings: []
        ),
    ]
)
```

Now, I get the following error when building.

```sh
error: 'swiftdl3': public headers ("include") directory path for 'SwiftDL3' is invalid or not contained in the target
```

After reading some documentation and playing around, it looks like I've got something compiling. Lets take a look at the files.

```swift
// swift-tools-version: 5.10
// The swift-tools-version declares the minimum version of Swift required to build this package.

import PackageDescription

let package = Package(
  name: "SwiftDL3",
  defaultLocalization: "en",
    products: [
        // Products define the executables and libraries a package produces, making them visible to other packages.
        .library(
            name: "SwiftDL3",
            targets: ["SwiftDL3"]),
    ],
    targets: [
        .target(
          name: "SwiftDL3",
          dependencies: [],
          exclude: [],
          sources: [],
          publicHeadersPath: "SDL/include/SDL3",
          cSettings: []
        ),
    ]
)
```

I also modified the `.gitmodules` file to the following.

```sh
[submodule "SDL"]
	path = Sources/SwiftDL3/SDL
	url = https://github.com/libsdl-org/SDL.git

```

Now for the moment of truth. To see if it can be using in an actual executable.

Naviagting out of the directory for this, I will create a new folder and project within it. Then I plan to map SwiftDL3 as a dependency on my local file system.

```sh
cd ../
mkdir SwiftDL3Test
cd SwiftDL3Test
swift package init --type executable --name SwiftDL3Test
```

