---
layout: post
title: "Dual Homebrew Setup for ARM (Apple) vs x86 (Intel) Compatibility"
author: Mike Rodriguez
date: 2025-01-11
tags: [homebrew, brew, mac, macos, osx, apple, arm, arm64, M1, M2, M3, M4, intel, x86, x86_64, development, programming, build, setup]
---

# Background

Modern macOS machines use M series chips (ie. they use "Apple silicon"). These chips are based on
the ARM architecture, which is typically referred to as `arm64`. Prior to this, Apple machines
typically used Intel processors that were based on the `x86_64` architecture. You can quickly check
which architecture your macOS device has via the `arch` command. This change has caused
compatibility problems over the last several years where some programs have not been ported to the
new architecture. Apple has provided a compatibility software solution known as Rosetta that is
[further described here](https://support.apple.com/en-us/102527). This is a well known solution to
the compatibility problem and apparently is reasonably performant, etc.

I prefer and have extensively used the popular [Homebrew package manager](https://brew.sh/) (aka.
`brew`) for macOS (and OSX before it). I install nearly all of my developer packages via this tool
when there is a supported "Homebrew Formulae" available for it. Based on open source project
documentation and discussions in forums, blogs, etc, this seems to be what many developers do on
macOS devices.

I will show here how to have a "dual" `brew` setup where you can easily toggle your terminal to use
`arm64` vs `x86_64` as the `arch` while resolving `brew` and its installed packages appropriately
for the given architecture.

# Motivating example

There are many possible scenarios for why you may want to have a "dual" installation of `brew`. This
is just to serve as a motivating example with a specific solution shown exercising the setup.

I recently had a scenario where I needed to install `ruby` via `rbenv` for use with gems that
required the `x86_64` architecture. However, I am doing this using an `arm64` macOS device. To do
this, I needed my `ruby` version to be also based on this same architecture. However, I already used
`brew` to install many packages, including `ruby`, based on their `arm64` versions. This is what is
native to my device and what I'd prefer where possible.

The only clear approach to install `ruby` and related packages for `x86_64` is to use `brew` for
`x86_64` as well. It seems the smoothest past forward for this is to use dual, isolated `brew`
installations for `arm64` and `x86_64`.

# Setting it up

Refer to this StackOverflow answer for ["How can I run two isolated installations of
Homebrew?"](https://stackoverflow.com/questions/64951024/how-can-i-run-two-isolated-installations-of-homebrew/64951025#64951025)
following steps 1-5 there as a precursor to the rest of my recommendations here. Here is a summary
of those steps detailed there that should be performed first:
1. Install `brew` natively on Apple Silicon (will install to `/opt/homebrew` by default). This
   should be the `arm64` version.
1. Install Intel-emulated `brew` (will install to `/usr/local` by default). This is the `x86_64`
   version.
   1. If you haven't yet installed Rosetta 2, you'll need to run `softwareupdate --install-rosetta`
      first.
1. Create an alias for Intel homebrew, eg. `brow` because "O"" for "old".
1. Add the `arm64` version of `brew` to your `PATH`.

I will assume zshell is used for the rest of this, but it shouldn't be too difficult to translate to
another shell environment as needed.
