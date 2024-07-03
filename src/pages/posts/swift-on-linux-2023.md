---
layout: ../../layouts/MarkdownPostLayout.astro
title: 'Swift on Linux (2023)'
author: 'Todd'
pubDate: 2023-05-23
---

# Introduction

As seen from previous posts, I have taken to dabbling with Swift on my M1 MacBook Air running MacOS. Now, it is time to take a look at how swift does on Linux. I believe Swift is an interesting language that can bring a rather unique value proposition. As someone who has written Rust for open sourced projects and professionally (for a paycheck) C#, Swift strikes a really interesting balance. In Rust, I sometimes end up at a point where I am tackling a problem and I am thinking to myself, "if only I had X from C# to do this." And likewise, in C# (especailly when it comes to errors and enums), I hit a situation where I look at something and go, "if I only I had Y from Rust." Swift, seems to be able to strike that balance some. It has the features of C# I want while also have quiet a few (not all) of the features of Rust I want. So I tend to describe it as, it is a perfect middle ground (or at least the best middel ground present) between C# and Rust.

# Creating an initial project on Linux

My main linux machine right now is running Fedora 37. Swift has been in the main repositories of Fedora for awhile. So getting Swift 5.8 was as simple as invoking the package manager.
```sh
  sudo dnf install swift
```
Now, with the basic tool chain installed, it is time to get to work. I create a work space, which was just a folder called "Hello." Navigating inside of that folder, time to invoke swift to create the project essentially.
```sh
  swift package init --type executable
```
This is just to create a simple executable package. Which gives me a little folder structure and a `main.swift` file.
```swift
  // The Swift Programming Language
  // https://docs.swift.org/swift-book

  print("Hello, world!")
```
Just a simple hello world. First, to see if it actually works.
```sh
  swift build
```
Okay, it compiles fine apparently. Now, how about running?
```sh
  swift run
```
Look like it can at least run a hello world.
```sh
  ╭─todd@fedora ~/Learning/Swift/Hello 
  ╰─$ swift run
  warning: 'hello': warning: direct reference to protected function `$sSJ12isWhitespaceSbvg' in `/usr/libexec/swift/5.8/bin/../lib/swift/linux/libswiftCore.so' may break pointer equality
  Building for debugging...
  Build complete! (0.08s)
  Hello, world!
```
Looks like we have lift off.

# Package.swift

Creating the project also gives us a `Package.swift` file. Which, at least shown what we are given, defines some simple things about the package. This is also, I believe, how 3rd party packages can be pulled in. From my understanding, you can do something like define dependencies by linking to their git repos and even specify a release from the repo and during the build phase, it will pull in the code and compile it with the application. Like with all good Linux applications, we need a command line interface, so I will use this as an opportunity to try this out.

There is a library, apparently from a google employee, that help provides a nice way to help build a command line driven program called [CommandlineKit](https://github.com/objecthub/swift-commandlinekit). Unfortunatley, the github repo also appears to make mention of Swift 5.5, but lets hope versioning is on my side here. The current version Fedora has is 5.8 (5.9 should be out with WWDC just happening). So, lets hope it is compatible.
```swift
  // swift-tools-version: 5.8
  // The swift-tools-version declares the minimum version of Swift required to build this package.

  import PackageDescription

  let package = Package(
    name: "Hello",
    dependencies: [
        .package(url: "https://github.com/jordanbaird/Prism", from: "0.1.2"),
    ],
    targets: [
        // Targets are the basic building blocks of a package, defining a module or a test suite.
        // Targets can depend on other targets in this package and products from dependencies.
        .executableTarget(
            name: "Hello",
            dependencies: ["Prism"],
            path: "Sources"),
    ]
  )
```
Alright, so far, I am impressed. When I first looked at this file, I didn't pay much attention to it, but adding a dependency made me actually look at it and see something here. This file defines the package, duh. However, the package is just defined like a *struct* in Swift. I like this a lot. When the build system is essentially written in the language itself. This is something zig does with *zig.build*. Rust does have a *build.rs* but you still have a *Cargo.toml* file, which is good, but its not optimal. C# has solution and csproj files which are XML/XAML/XML-like (YUK!).

I think coloring can look pretty good for a command line utility, so Prism is a nice package that looks like it can give some coloring.
```swift
  // The Swift Programming Language
  // https://docs.swift.org/swift-book

  import Prism

  let text = Prism(spacing: .spaces) {
    ForegroundColor(.green, "This text's color is green")
  }

  print(text)
```
It does indeed print the text as green.

Now, lets get some text from the command line and run it.
```swift
  // The Swift Programming Language
  // https://docs.swift.org/swift-book

  import Prism
  import Foundation

  if CommandLine.arguments.isEmpty {
    print(Prism(spacing: .spaces) {
        ForegroundColor(.red, "No input into program")
    })
  }

  for argument in CommandLine.arguments {
    print(Prism(spacing: .spaces) {
        ForegroundColor(.green, "Arg: \(argument)")
    })
  }
```

```sh
  ╭─todd@fedora ~/Learning/Swift/Hello 
  ╰─$ ./.build/x86_64-unknown-linux-gnu/debug/Hello hello
  Arg: ./.build/x86_64-unknown-linux-gnu/debug/Hello
  Arg: hello
```

So, interestingly enough, there will always be atleast one argument. We also need to ensure we run the program from the debug folder. Using *swift run* and trying to pass in arguments doesnt work.

To tame, this a little more, there is a nice library that is available called *Swift-CommandLineKit*. This also introduces in how to do the *package.swift* file to entertain more than a single dependency.
```swift
  // swift-tools-version: 5.8
  // The swift-tools-version declares the minimum version of Swift required to build this package.

  import PackageDescription

  let package = Package(
    name: "Hello",
    products: [
        .executable(name: "Hello", targets: ["Hello"])
    ],
    dependencies: [
      .package(url: "https://github.com/jordanbaird/Prism", from: "0.1.2"),
      .package(url: "https://github.com/objecthub/swift-commandlinekit", from: "0.3.5")
    ],
    targets: [
        // Targets are the basic building blocks of a package, defining a module or a test suite.
        // Targets can depend on other targets in this package and products from dependencies.
        .executableTarget(
            name: "Hello",
            dependencies: [
              .product(name: "Prism", package: "Prism"),
              .product(name: "CommandLineKit", package: "swift-commandlinekit")
            ],
            path: "Sources"),
    ]
  )
```

Now that is sorted, how about something a little more proper?
```swift
  // The Swift Programming Language
  // https://docs.swift.org/swift-book

  import CommandLineKit
  import Foundation
  import Prism

  var flags = Flags()

  let help = flags.option("h", "help", description: "Show description of usage.")

  try flags.parse()

  if help.wasSet {
    print(Prism(spacing: .spaces) {
              ForegroundColor(.red, "hello")
    })
  }
```

Look like it works so far. However, I want to print the actual flag description. I will change out the string that is printed from *ForegroundColor*.
```swift
  print(Prism(spacing: .spaces) {
    ForegroundColor(.red, "\(flags.usageDescription())")
  })
```

Here is the output
```sh
  ╭─todd@fedora ~/Learning/Swift/Hello 
  ╰─$ ./.build/x86_64-unknown-linux-gnu/debug/Hello -h
  USAGE: Hello [<option> ...] [--] [<arg> ...]
  OPTIONS:
    -h, --help
      Show description of usage.
```

Alright, looks like I am getting somewhere. Now to do something a little useful. One thing I am excited to try out eventually is [Vapor](https://vapor.codes/), however before going there, it would be a good idea to make sure any networking at all works as it should. I assume it does, but Swift is such an Apple focused language, there is a chance that perhaps things don't work exactly as they should. So, my plan is to add a REST API call to a free HTTP REST API for a [deck of card](https://deckofcardsapi.com/). There will be a new flag which will be called 'new' that will request a new deck from the API. I am not going to worry about actually deserializing the returned JSON, just make the call and ensure that I get back a 200 OK response.

The following is the endpoint to hit.
```sh
  https://deckofcardsapi.com/api/deck/new/shuffle/?deck_count=1
```
Here is the code
```swift
  // The Swift Programming Language
  // https://docs.swift.org/swift-book

  import CommandLineKit
  import Foundation
  import FoundationNetworking
  import Prism

  var flags = Flags()

  let help = flags.option("h", "help", description: "Show description of usage.")

  let newDeck = flags.option("n", "new", description: "Request a new deck.")

  try flags.parse()

  if help.wasSet {
    print(Prism(spacing: .spaces) {
              ForegroundColor(.yellow, "\(flags.usageDescription())")
          })
    exit(0);
  }

  if newDeck.wasSet {
    let semaphore = DispatchSemaphore(value: 0)
    
    guard let url = URL(string: "https://deckofcardsapi.com/api/deck/new/shuffle/?deck_count=1") else {
        print(Prism(spacing: .spaces) {
                  ForegroundColor(.red, "Error making url")
              })
        exit(0)
    }

    let task = URLSession.shared.dataTask(with: url) {
        (data, response, error) in
        guard let data = data, error == nil else {
            fatalError(error!.localizedDescription)
        }
        print("Error: \(String(describing: response))")
        print("Fetched data size: \(data)")
        semaphore.signal()
    }

    task.resume()
    semaphore.wait()
  }
```

Lets do a diff to see the difference.
```sh
  diff main.swift.old main.swift > diff
```
Now the contents of the diff
```sh
  --- main.swift.old	2023-06-07 21:34:11.054633442 -0700
  +++ main.swift	2023-06-07 21:29:47.711075885 -0700
  @@ -3,16 +3,45 @@
  import CommandLineKit
  import Foundation
  +import FoundationNetworking
  import Prism
  var flags = Flags()
  let help = flags.option("h", "help", description: "Show description of usage.")
  +let newDeck = flags.option("n", "new", description: "Request a new deck.")
  +
  try flags.parse()
  if help.wasSet {
       print(Prism(spacing: .spaces) {
  -              ForegroundColor(.red, "\(flags.usageDescription)")
  -    })
  -}
  \ No newline at end of file
  +              ForegroundColor(.yellow, "\(flags.usageDescription())")
  +          })
  +    exit(0);
  +}
  +
  +if newDeck.wasSet {
  +    let semaphore = DispatchSemaphore(value: 0)
  +    
  +    guard let url = URL(string: "https://deckofcardsapi.com/api/deck/new/shuffle/?deck_count=1") else {
  +        print(Prism(spacing: .spaces) {
  +                  ForegroundColor(.red, "Error making url")
  +              })
  +        exit(0)
  +    }
  +
  +    let task = URLSession.shared.dataTask(with: url) {
  +        (data, response, error) in
  +        guard let data = data, error == nil else {
  +            fatalError(error!.localizedDescription)
  +        }
  +
  +        print("Response: \(String(describing: response))")
  +        print("Fetched data size: \(data)")
  +        semaphore.signal()
  +    }
  +
  +    task.resume()
  +    semaphore.wait()
  +}
```

The first change is a little odd, or atleast, I have not seen it in swift projects on macOS. From what I have seen, you only need to import *FoundationNetworking* on linux and potentially other non-macOS operating systems such as Windows?

A new flag option is added for requesting a new deck.

The next biggest change is down towards the bottom where the program handles if an option is set for *newDeck*. A semaphore is needed to make the program wait for the task to finish as seen in the lower portion of the program. Otherwise, creat a URL object from a string representation of the URL, using a guard let for if for some reason that fails. Followed by creating a *URLSession* with a *dataTask* using the url. A closure takes in *data*, *response*, and *error*. If *error* **is not** nil, then perform a *fatalError*.  Otherwise, print the response and the data. The data, with how it is being printed, will only print the length of the data. Signal the semaphore so that way we can resume and finish out the program,
```sh
  ╭─todd@fedora ~/Learning/Swift/Hello 
  ╰─$ ./.build/x86_64-unknown-linux-gnu/debug/Hello -n
  Response: Optional(<HTTPURLResponse 0x00007f8f2c12f6d0> { URL: https://deckofcardsapi.com/api/deck/new/shuffle/?deck_count=1 }{ status: 200, headers {
     "Access-Control-Allow-Origin" = *;
     "Alt-Svc" = h3=":443"; ma=86400;
     "Cf-Cache-Status" = DYNAMIC;
     "Cf-Ray" = 7d3e7e4f3ec5eb73-SEA;
     "Content-Encoding" = br;
     "Content-Type" = "application/json";
     "Date" = Thu, 08 Jun 2023 04:46:50 GMT;
     "Nel" = {"success_fraction":0,"report_to":"cf-nel","max_age":604800};
     "Referrer-Policy" = same-origin;
     "Report-To" = {"endpoints":[{"url":"https:\/\/a.nel.cloudflare.com\/report\/v3?s=29uaD8qnUgUPvB1r5yx0zVHRvkrKn%2BxmSIEJZre%2FcZmGzVhlcshKWNymfLhVxSdRi4pvg4pmMuMIHMbenKYeaYTU0Wo%2F66xNGTCCzFbaRa4pAOizwibFnSJPYw5%2FeivxrkKlYhY9hTJt00s1E5%2BlFZg%3D"}],"group":"cf-nel","max_age":604800};
     "Server" = cloudflare;
     "Vary" = Origin;
     "x-content-type-options" = nosniff;
     "x-frame-options" = DENY;
  } })
```
Fetched data size: 79 bytes


