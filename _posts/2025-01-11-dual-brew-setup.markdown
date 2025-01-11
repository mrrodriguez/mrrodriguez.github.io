---
layout: post
title: "Dual Homebrew for Apple/Intel Compatibility"
author: Mike Rodriguez
date: 2025-01-11
tags: [homebrew, brew, mac, macos, osx, apple, silicon, arm, arm64, M1, M2, M3, M4, intel, x86, x86_64, development, programming, build, setup]
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

# Solution

## General environment setup

Refer to this StackOverflow answer for ["How can I run two isolated installations of
Homebrew?"](https://stackoverflow.com/questions/64951024/how-can-i-run-two-isolated-installations-of-homebrew/64951025#64951025)
following steps 1-5 there as a precursor to the rest of my recommendations here. Here is a summary
of those steps detailed there that should be performed first:
1. Install `brew` natively on Apple Silicon (will install to `/opt/homebrew` by default). This
   should be the `arm64` version.
1. Install Intel-emulated `brew` (will install to `/usr/local` by default). This is the `x86_64`
   version.
   - If you haven't yet installed Rosetta 2, you'll need to run `softwareupdate --install-rosetta`
     first.
1. Create an alias for Intel homebrew, eg. `brow` because "O"" for "old".
1. Add the `arm64` version of `brew` to your `PATH`.

I will assume zshell is used for the rest of this, but it shouldn't be too difficult to translate to
another shell environment as needed.

Below I will share snippets to add `.zshrc` to support toggling within a terminal between an
environment using `arm64` or `x86_64` version of `brew` packages (isolated from one another). I'll
provide a brief description to explain each part. This may sometimes be done as shell comments since
it's likely a good idea to leave those comments in the script to help remember what this is for.

I want to make it obvious when I'm in the terminal, which environment for `brew` I'm working in:
```sh
# This customizes the iTerm title bar to show useful info for in context of the
# tab showing in the terminal.
precmd() {
  local max_length=25  # Max total length for the tab title
  local prefix=".."    # Prefix to indicate truncation
  local prefix_length=${#prefix}
  local allowed_length=$((max_length - prefix_length))  # Remaining length for the path
  local truncated_path
  local current_path="$PWD"

  # Check if the path exceeds the max length
  if [[ ${#current_path} -gt $max_length ]]; then
    # Extract the last allowed_length characters from the path
    truncated_path="${prefix}${current_path: -$allowed_length}"
  else
    # If the path is short enough, use it as-is
    truncated_path="$current_path"
  fi

  # Set the iTerm2 tab and window titles, respectively.
  print -Pn "\e]1;$truncated_path ($(uname -m))\a"
  print -Pn "\e]2;%n@%m: $truncated_path ($(uname -m))\a"
}
```

```sh
current_arch() {
    echo "Current arch: $(uname -m)"
    echo "ARCH env var: ${ARCH:-NONE}"
}
```

I want to be able to easily toggle the environment my terminal us currently using:
```sh
# Toggles for using arm64 vs x86_64 homebrew.
switch_arch() {
    local new_arch=$1
    if [[ "$new_arch" != "arm64" && "$new_arch" != "x86_64" ]]; then
        echo "Invalid architecture. Use 'arm64' or 'x86_64'"
        return 1
    fi

    # Set the ARCH environment variable
    export ARCH=$new_arch

    # Only switch if we're not already on the desired architecture
    if [[ "$(uname -m)" != "$new_arch" ]]; then
        echo "Switching to new arch: ${ARCH}"
        # Use arch -x86_64 or arch -arm64 to execute subsequent commands
        if [[ "$new_arch" == "x86_64" ]]; then
            # Switch to x86_64 architecture
            exec arch -x86_64 /bin/zsh
        else
            # Switch to arm64 architecture
            exec arch -arm64 /bin/zsh
        fi
    fi
}

# Initial architecture setup - runs when shell starts
initial_arch_setup() {
    # If ARCH is set (like from VSCode), switch to that architecture
    if [[ -n "$ARCH" ]]; then
        switch_arch "$ARCH"
    fi
}

# Run the initial setup
initial_arch_setup
```

These are helpful aliases:
```sh
alias zarm='ARCH=arm64 initial_arch_setup'
alias zros='ARCH=x86_64 initial_arch_setup'
alias brow='arch -x86_64 /usr/local/Homebrew/bin/brew'
alias ib='PATH=/usr/local/bin'
```

Any time this `.zshrc` file is loaded, we want to make sure the environment is setup as expected,
eg. ensuring the correct `PATH` is used:
```sh
if [[ "$(uname -m)" == "arm64" ]]; then
    eval "$(/opt/homebrew/bin/brew shellenv)"
else
    alias brew='arch -x86_64 /usr/local/bin/brew'
    export PATH="/usr/local/bin:$PATH"
    eval "$(/usr/local/bin/brew shellenv)"
fi
```

## The ruby + rbenv setup

This should be applicable to other languages that have similar environmental setups to `ruby` via
`rbenv`, eg. `python`, `node`, etc.

```sh
# For Ruby

if [[ $(uname -m) == "arm64" ]]; then
    echo "Using rbenv with arm64"
    export RBENV_ROOT="${HOME}/.rbenv"

    if which rbenv > /dev/null; then eval "$(rbenv init -)"; fi
    rbenv global 3.2.0

else
    echo "Using brew/rbenv with x86_64"
    export RBENV_ROOT="${HOME}/.rbenv_x86"

    # Initialize rbenv for x86
    if which rbenv > /dev/null; then eval "$(rbenv init -)"; fi
    rbenv global 3.2.0
fi
```
