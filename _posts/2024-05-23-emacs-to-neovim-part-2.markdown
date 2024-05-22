---
layout: post
title: "From Emacs to Neovim (Part 2)"
author: Mike Rodriguez
date: 2024-05-22
tags: [emacs, neovim, spacemacs, vim, vscode, vscode-neovim, vi, editor, development, programming]
---

# Background

This is a continuation of my post ["From Emacs to Neovim"]({% post_url 2024-05-20-emacs-to-neovim %}). In this post, I am going to share some concrete configuration examples and concepts that have helped me adapting to vim configuration and bindings coming from an `emacs` background.

Note that my current, primary tooling focus is:
* [spacemacs distribution](https://www.spacemacs.org) of `emacs` primarily for Clojure/ClojureScript development
* [neovim distribution](https://neovim.io) of vim for simple terminal file editing
* [vscode-neovim](https://github.com/vscode-neovim/vscode-neovim) for TypeScript/JavaScript development

# Neovim configuration

My public dotfiles can be found @ <https://github.com/mrrodriguez/nvim-dotfiles>

So far, I have not worked much to customize `neovim` for a "standalone IDE" type of experience. Rather I have set it up for basic terminal file editing and then utilize this same configuration via the `vscode-neovim` extension for `vscode`, which I will describe more below in its own section.

I have chosen to setup the `neovim` configuration in a way where I can mix both vimscript configuration along with lua configuration. I found this to be convenient in that some docs and examples are in one or the other and I haven't spent a lot of time learning either one yet to know how to translate between the two.

I am particularly a fan of `nvim-surround` to help supplement for some of the utility I am used to from using [paredit](https://paredit.org) I have heavily relied upon in `emacs`. It is not entirely meant to serve the same purpose, but it does cover the most significant use-case I have had for `paredit` which is "slurping" and "barfing" (this is `paredit` terminology) surrounding delimiters around chunks of text/words.

#### Neovim references

I have found these sources to be valuable in getting started with this configuration:
* [This is a good post](https://builtin.com/software-engineering-perspectives/neovim-configuration) on starting a new neovim configuration.
* These are two good posts concerning how to mix vimscript and lua configuration files:
  * ["Is it possible to use init.vim and init.lua together?"](https://www.reddit.com/r/neovim/comments/zfimqo/is_it_possible_to_use_initvim_and_initlua_together)
  * ["How to handle init.lua examples if you use init.vim?"](https://www.reddit.com/r/neovim/comments/1913nyw/how_to_handle_initlua_examples_if_you_use_initvim)
* [This is the neovim Lua guide](https://neovim.io/doc/user/lua-guide.html).
* [This was a useful post](https://github.com/vscode-neovim/vscode-neovim/issues/819#issuecomment-1035983972) describing the difference from using `runtime` vs `source` in a Lua script.

I haven't worked with many plugins yet, but I have followed the popular recommendations to manage them with [lazy.nvim](https://github.com/folke/lazy.nvim). In my configuration files it can be seen how to use this to install [nvim-surround](https://github.com/kylechui/nvim-surround).

# Spacemacs configuration

My public dotfiles can be found @ <https://github.com/mrrodriguez/emacs-dotfiles>

#### Spacemacs references

# VSCode configuration

#### VSCode references
