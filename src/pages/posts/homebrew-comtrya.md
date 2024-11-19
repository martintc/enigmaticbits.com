---
layout: ../../layouts/post.astro
title: 'Homebrew: Adding Comtrya to Core'
author: 'Todd'
pubDate: 2024-11-19
---

# Homebrew: Adding Comtrya to Core

I use a different mix of systems. The server this site is hosted on is a Debian VPS. I have two machines. I use some local test servers in my home that are in the form of Mac Minis; one has FreeBSD installed while the other has NetBSD. A desktop that runs Arch linux and a Macbook Pro running macOS. However, I am often using my Macbook Pro.

Having done a little port work for Ravenports on the BSDs and running primarily macOS right now, I have always been interested in Homebrew and how that system works as far as packaging software. Homebrew is also cross platform, however, I have never used it on Linux. Although, it is hard to get deep into packaging and actually doing it without having something to package.

In comes [Comtrya](https://github.com/comtrya/comtrya). I have not written about it before, but I should write a few articles. Comtrya is an open source project that I have been a contributor of for awhile and that I now am the primary maintainer and do the releases of. I won’t go too much into the details of Comtrya, since that is not the main focus of this writing. 

What I will say is that Comtrya is distributed in a few ways. First, it is available in raw binaries under github [releases](https://github.com/comtrya/comtrya/releases/tag/v0.9.0). There is a script that can be curl’d and piped to bash that can pick the right binary and place it accordingly. I am not a huge fan of this method, but it is popular and for what Comtrya can do, I can see the reason why people would want it. Since it is a rust program, it can be gotten via cargo since it is published to [crates](https://crates.io/crates/comtrya). Installation can also occur through a small set of package managers. Primarily, before my work here, it was available in the AUR and in Ravenports. Even though I have introduced and maintained ports for Ravenports, I sadly am not the one to introduce Comtrya, or rather ironically. Instead a friend of mine does that, the same friend who introduced me to Comtrya to begin with. 

Since I mostly work off macOS now, I figured I should make it available via Homebrew. 

## Initial work - Making a tap

Before looking too deep into what is needed and the requirements to get it into homebrew core, which will be explained a little later, I decided to make a tap for Comtrya. A tap you can think of as like a repository for a package or packages. If your familiar with Ubuntu’s PPA, it is sort of like that. 

A tap also doesn’t have certain restrictions since it isn’t in homebrew-core. In a tap, it is okay to make a tap that just download the binary and places it. Which is the route I went for the tap, go with the easiest and make that available first.

Homebrew and the formula are primarily written in Ruby. Which is a little bit of coincidence since I recently have been using a little Ruby for a website with the Ruby on Rails framework. Packages are defined in Ruby and the recipe to make the package is called a formula. 
For someone to make use of a formula they just need to tell Homebrew to add the tap, which is like adding a PPA on Ubuntu for apt.

```shell
brew tap comtrya/comtrya
brew install comtrya
```

The formula has some details about the package and can perform some actions. The main action I focus is the install action. As mentioned, the route I went with was just grabbing the binary and installing it.

```ruby
# Documentation: https://docs.brew.sh/Formula-Cookbook
#            	https://rubydoc.brew.sh/Formula
# PLEASE REMOVE ALL GENERATED COMMENTS BEFORE SUBMITTING YOUR PULL REQUEST!
class Comtrya < Formula
  desc "Configuration And Dotfile Management"
  homepage "https://comtrya.dev"
  url "https://github.com/comtrya/comtrya/releases/download/v0.9.0/comtrya-aarch64-apple-darwin"
  sha256 "1ca79b45a9c6cd91654fefe47a22fcfdfee7d05bb792b69e39e35e9d396dad6e"
  license "MIT"
  version "0.9.0"

  def install
	bin.install "comtrya-aarch64-apple-darwin" => "comtrya"
  end
end
```

It is that simple. There are a few drawbacks here. It only pulls the aarch64 build for macOS. So this would not work on a x86_64 macOS machine or linux. So pretty much only apple silicon. But, this was just getting a little bit of experience before looking into the requirement for homebrew-core.

The repository for the tap is located on [github](https://github.com/comtrya/homebrew-comtrya) under Comtrya's organization.

## Getting into homebrew-core

Homebrew has what is called [“homebrew-core.”](https://github.com/Homebrew/homebrew-core) This is pretty much the official repository. The main requirement that comes into play for me to get Comtrya packaged is that, for a package to make it in, the formula must handle building the application from source. There are also some quality controls that are required also. 

To get an idea of how to do with rust, I took quiet a bit of inspiration from [ripgrep](https://github.com/Homebrew/homebrew-core/blob/master/Formula/r/ripgrep.rb), which is already in homebrew-core. Rust programs can be a bit of a beast to get into a ports tree or package manager depending on the philosophy. For instance, I believe Debian and FreeBSD’s ports tree want the package definition to be able to build itself, but also all of its dependencies with their package definitions to build the dependencies locally. Ravenports sort of does to, but a special short cut has been cut out for Rust programs and Go programs (with go mod). So taking Comtrya as an example, the last number I remember was somewhere in the ballpark of 387 dependencies. This includes transitive dependencies. Needing to bring in 387 recipes or package definitions isn’t fun, long and grueling work. Luckily, Homebrew does not require this. I don’t need to first package each dependency and make sure the versions are compatible, we can just trust the maker (in this case me) and cargo to do the right thing.

```ruby
class Comtrya < Formula
  desc "Configuration and dotfile management tool"
  homepage "https://comtrya.dev"
  url "https://github.com/comtrya/comtrya/archive/refs/tags/v0.9.0.tar.gz"
  sha256 "a5401004a92621057dab164db06ddf3ddb6a65f6cb2c7c4208a689decc404ad4"
  license "MIT"
  head "https://github.com/comtrya/comtrya.git", branch: "main"

  bottle do
	sha256 cellar: :any_skip_relocation, arm64_sequoia: "56bcd6407ac6f14a3ba79414c0651e52a2e262ce16b1148c70abc2ac1a121ead"
	sha256 cellar: :any_skip_relocation, arm64_sonoma:  "6686fa03d6d26a52beb6db4aa60d1dc51dfe34fda8e490d65c0ed96ee1adf883"
	sha256 cellar: :any_skip_relocation, arm64_ventura: "eaed784b320caf01dbf1130b956a34d5bf75d93c09555d9d5c3cdc799de13cad"
	sha256 cellar: :any_skip_relocation, sonoma:    	"846c88f71d195eb80fe7a33024cd985bcd023455a6d2e52582e66b6a40613adc"
	sha256 cellar: :any_skip_relocation, ventura:   	"ca956905d382263bc9ce55ddcda6f57612565322d33aa918621fe24e744f565e"
	sha256 cellar: :any_skip_relocation, x86_64_linux:  "635873d62afde5a0e0e100273ec87da7d4d6bc3ee74f59c894bf7e8d346b3b77"
  end

  depends_on "rust" => :build

  def install
	system "cargo", "install", *std_cargo_args(path: "./app")
  end

  test do
	assert_match "comtrya #{version}", shell_output("#{bin}/comtrya --version")

	resource "testmanifest" do
  	url "https://raw.githubusercontent.com/comtrya/comtrya/refs/heads/main/examples/onlyvariants/main.yaml"
  	sha256 "0715e12cbbb95c8d6c36bb02ae4b49f9fa479e2f28356b8c1f3b5adfb000b93f"
	end

	resource("testmanifest").stage do
  	system bin/"comtrya", "-d", "main.yaml", "apply"
	end
  end
end
```

Above is the formula that is now in homeware-core. As mentioned, we can just trust the maker and cargo to do the right thing. Defining rust as a build dependency pulls down all the rust tooling, including cargo, to build the package. Then building the package is as simple as invoking cargo to install.

One area I had an issue with that referencing ripgrep wasn’t much of a help with is how to build Comtrya for homebrew with it having workspaces and how the workspace are set up. The binary application is the app. There is also a lib that is built for the library crate and jsonschema crate. I did have to tell cargo to build specifically the app to successfully compile Comtrya with homebrew from source. From there, Homebrew's tooling takes care of most of it, including placing the binary in the appropriate place on each system. Since it is built from source, this package should technically work on any platform that homebrew supports.

As mentioned, there is some quality control that is required here. That is the test section. Each package that is being introduced into homebrew has to have at least one test. The general rule of thumb is that the test needs to flex how the application is used and should not just be a version check. 

In my test, I do run a version check. But I also pull down a resource, a test manifest and have Comtrya apply the manifest. The test passes so long as Comtrya doesn’t throw an error when running it. 

And of course, now Comtrya is [officially avaible](https://github.com/Homebrew/homebrew-core/pull/197214) in hombrew-core.

## Interacting with the maintainers of Homebrew

I found them to be helpful and pretty welcoming. There were some issues I had to fix. First was styling. Homebrew does have a tool that will help fix the styling similar to how cargo’s fmt works. Also a little bit of guidance of what is expected with tests. It was also handy to look at other recently closed pull requests and pending pull requests to see if there was anything I was missing.
