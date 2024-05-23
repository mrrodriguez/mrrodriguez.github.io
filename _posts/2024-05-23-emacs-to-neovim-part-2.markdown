---
layout: post
title: "From Emacs to Neovim (Part 2)"
author: Mike Rodriguez
date: 2024-05-22
tags: [emacs, neovim, spacemacs, vim, vscode, vscode-neovim, vi, editor, development, programming]
---

# Background

This is a continuation of my post @ ["From Emacs to Neovim"]({% post_url 2024-05-20-emacs-to-neovim %}). In this post, I am going to share some concrete configuration examples and concepts that have helped me adapting to `vim` configuration and bindings coming from an `emacs` background.

Note that my current, primary tooling focus is:
* [spacemacs distribution](https://www.spacemacs.org) of `emacs` primarily for Clojure/ClojureScript development
* [neovim distribution](https://neovim.io) of `vim` for simple terminal file editing
* [vscode-neovim](https://github.com/vscode-neovim/vscode-neovim) for TypeScript/JavaScript development

# Neovim configuration

My public dotfiles can be found @ <https://github.com/mrrodriguez/nvim-dotfiles>

So far, I have not worked much to customize `neovim` for a "standalone IDE" type of experience. Rather I have set it up for basic terminal file editing and then utilize this same configuration via the `vscode-neovim` extension for `vscode`, which I will describe more below in its own section.

I have chosen to setup the `neovim` configuration in a way where I can mix both vimscript configuration along with lua configuration. I liked this setup since I have found docs and examples are often in one or the other. I haven't spent a lot of time learning either one yet to know how to translate between the two.

I am particularly a fan of `nvim-surround` to help supplement for some of the utility I am used to from using [paredit](https://paredit.org) I have heavily relied upon in `emacs`. It is not entirely meant to serve the same purpose, but it does cover the most significant use-case I have had for `paredit` which is "slurping" and "barfing" (this is `paredit` terminology) surrounding delimiters around chunks of text/words.

#### Neovim references

I have found these sources to be valuable in getting started with this configuration:
* [This is a good post](https://builtin.com/software-engineering-perspectives/neovim-configuration) on starting a new `neovim` configuration.
* These are two good posts concerning how to mix vimscript and lua configuration files:
  * ["Is it possible to use init.vim and init.lua together?"](https://www.reddit.com/r/neovim/comments/zfimqo/is_it_possible_to_use_initvim_and_initlua_together)
  * ["How to handle init.lua examples if you use init.vim?"](https://www.reddit.com/r/neovim/comments/1913nyw/how_to_handle_initlua_examples_if_you_use_initvim)
* [This is the neovim Lua guide](https://neovim.io/doc/user/lua-guide.html).
* [This was a useful post](https://github.com/vscode-neovim/vscode-neovim/issues/819#issuecomment-1035983972) describing the difference from using `runtime` vs `source` in a Lua script.

I haven't worked with many plugins yet, but I have followed the popular recommendations to manage them with [lazy.nvim](https://github.com/folke/lazy.nvim). In my configuration files it can be seen how to use this to install [nvim-surround](https://github.com/kylechui/nvim-surround).

# Spacemacs configuration

My public dotfiles can be found @ <https://github.com/mrrodriguez/emacs-dotfiles>

`spacemacs` already comes with a lot of `vim` support via its use of [evil-mode](https://github.com/emacs-evil/evil). I haven't had to do too much configuration. There is also `spacemacs` specific features that typically involve the minibuffer and I use them the standard way. The `spacemacs` specific features are mostly expressed via a "leader key" which defaults to being the "space" key (aka. `SPC`). This works well with `dotspacemacs-editing-style` set to `hybrid` or `vim`.

I'll enumerate some quick configuration conveniences here that I added to `dotspacemacs/user-config`. Later on below, I'll enumerate the references that I used for some of these.

I found it convenient to have the editor change to "normal" mode (aka. `evil-normal-state` in `evil-mode`). This seems like a fairly common configuration people setup for `vim` style editors. This is done via:

```lisp
(add-hook 'after-save-hook #'evil-normal-state)
```

I also did not want to have to always use the "escape" key (aka. `ESC`) for returning to "normal" mode. This is very common in the `vim` community. There are many possible choices that are common and there are pros and cons to read about on them. I liked using `jk` which I set via:

```lisp
(setq-default evil-escape-key-sequence "jk")
```

Another thing I noticed that would help right early on was if I could go from "normal" mode to "insert" mode to insert a single character but then immediately be back in "normal" mode. I did this via:

```lisp
(evil-define-command my-evil-insert-char (count char)
  (interactive "<c><C>")
  (setq count (or count 1))
  (insert (make-string count char)))

(evil-define-command my-evil-append-char (count char)
  (interactive "<c><C>")
  (setq count (or count 1))
  (when (not (eolp))
    (forward-char))
  (insert (make-string count char)))

(define-key evil-normal-state-map (kbd "s") 'my-evil-insert-char)
(define-key evil-normal-state-map (kbd "S") 'my-evil-append-char)
```

#### Spacemacs references

I have found these sources to be valuable in updating my `spacemacs` configuration with a `vim` emphasis:

* ["Single character insert for Evil"](https://www.reddit.com/r/emacs/comments/7ogu7a/comment/ds9py2s/?utm_source=reddit&utm_medium=web2x&context=3)
* ["Spacemacs : insert single character in normal mode"](https://emacs.stackexchange.com/questions/32450/spacemacs-insert-single-character-in-normal-mode)

# VSCode configuration

There were certain helpful configuration changes to `vscode` that helped for the `vscode-neovim` integration for me. Later on below, I'll enumerate the references that I used for some of these.

To use my `neovim` configuration within `vscode-neovim` I added this to my User `settings.json`:

```json
"vscode-neovim.neovimInitVimPaths.darwin": "/Users/mikerod/.config/nvim/init.lua"
```

Using macOS I needed to change the setting to allow for holding a key to repeat it via:

```sh
defaults write com.microsoft.VSCode ApplePressAndHoldEnabled -bool false
```

for the same reason discussed above concerning `spacemacs` configuration, I wanted `jk` to return to "normal" mode the same way `ESC` does. To do this I added to the `vscode` User `settings.json` file:

```json
"vscode-neovim.compositeKeys": {
    "jk": {
      "command": "vscode-neovim.escape"
    }
  },
```

More options are supported here and it can be more complex. The docs cover this well (which I'll link below in references).

In the same way I wanted "save" to cause a return to "normal" mode in `spacemacs` that can be setup in `vscode` by adding this to the User `keybindings.json`:

```json
{
    "command": "runCommands",
    "key": "cmd+s",
    "when": "editorTextFocus && neovim.init && neovim.mode == 'insert'",
    "args": {
      "commands": ["workbench.action.files.save", "vscode-neovim.escape"]
    }
  }
```

#### VSCode references

I have found these sources to be valuable in updating my `vscode` configuration with a `vscode-neovim` emphasis:

* ["Integrate Neovim inside VSCode"](https://medium.com/@shaikzahid0713/integrate-neovim-inside-vscode-5662d8855f9d)
* ["Composite escape keys"](https://github.com/vscode-neovim/vscode-neovim/blob/02d13f0e119afbec8f68fe5add0f2c2a1072ec49/README.md#composite-escape-keys)
* ["In VSCode Neovim, press Ctrl+S to save file and switch to normal mode?"](https://stackoverflow.com/a/77769949/924604)
  * Note that there is a `"command": "runCommands"` now available so the `multi-command` extension is not needed. My configuration example above utilizes this instead.
  * See ["How can I run multiple commands with a single VS Code keybinding without an extension?"](https://stackoverflow.com/a/75808372/924604)
