---
layout: post
title: "From Emacs to Neovim"
author: Mike Rodriguez
date: 2024-05-17
tags: [emacs, neovim, spacemacs, vim, vi, editor, development, programming ]
---

# Background

I used emacs for around 15 years using the standard style key bindings. I’ve primarily used it for Clojure/ClojureScript development. However, I have also used it for C, Ruby, Python, Common Lisp, and various data formats (eg. JSON/XML) from occasionally.

About 6 years ago I started to use the [spacemacs distribution](https://www.spacemacs.org) after seeing several of my coworkers using it. I thought that this would be useful due to the general knowledge sharing of common configuration advice and best practices. I found the the defaults were nice when getting started with spacemacs to replace my existing emacs setup. What I had previously was what I had configured in an ad hoc fashion over a few years with no particular setup framework in mind. Notably, I did not use `evil-mode` bindings at all. The helm search buffer key sequences were the closest I had to something like vim bindings.

Recently, I have started work with more languages such as TypeScript/JavaScript and Golang and I have been curious if I could find better and/or more supported tooling available if I was willing to venture beyond the emacs editor. A natural fit for both seemed to be [vscode](https://code.visualstudio.com). I knew I wanted powerful editing capabilities I was familiar with though. I had tried some emacs key emulators in other IDEs before but did not find them to be sufficient. Without the elisp, the minibuffer, the "kill ring", among many other specifics to the emacs editor itself, I wasn't able to feel productive.

# The idea

Lately, I've seen and heard a lot more about [neovim](https://neovim.io) as an editor. It seems to have brought a new surge in popularity vim. I have always been "vim-curious". I knew a little bit about how it philosophically differed from emacs and I was intrigued.

Since I was using spacemacs, I thought this may be an interesting opportunity to explore vim via `evil-mode`. My plan was to ease into it by using spacemacs ["hybrid editing style"](https://develop.spacemacs.org/doc/DOCUMENTATION.html#hybrid) which uses `evil-mode` for all vim modes (called "states" in spacemacs) other than "insert". This was a nice escape hatch for me to not make me to be a bit more productive as I learned. Along with this, I did neovim tutorials and read docs to get started learning more of the vim side. I gradually applied these ideas/concepts more to my hybrid editing style spacemacs setup.

# Learning process

It was hard for a week of dedicated focus on learning vim keys while doing my day to day work. By about 3 weeks, I felt much more proficient and efficient. I only had some lingering inefficient movement annoyances I knew I’d need to investigate and improve over time. Basically, I just wait until the annoyance becomes frequent enough to be [worth the investment to address](https://xkcd.com/1205).

From there, I started using vscode for TypeScript with the [vscode-neovim](https://github.com/vscode-neovim/vscode-neovim) extension. I was impressed by how well this extension blended well within vscode. I was also able to find a lot of good resources available when troubleshooting and learning about the extension.

# Outcome

I still use spacemacs with hybrid mode bindings while doing Clojure/ClojureScript and various other data files and languages I may lightly use. I still prefer "insert" mode (aka. spacemacs "insert state") to stay emacs traditional bindings. It seems to me that it is a "better" insert mode experience to vim within emacs. However, I now find myself using a lot less insert mode than I used to.

When I switch to vscode with the vscode-neovim extension, I can quickly adapt to that IDE environment. I can even toggle back and forth throughout the day without a struggle. Overall, the switch to vim bindings seems to be a worthwhile investment. My general thought is that you can get more “range”/"reach" out of vim bindings due to the extensions available in popular IDEs and other text editors (including in the terminal which is "the default" for neovim). To me, the editing efficiencies are just as powerful and efficient when compared to my traditional emacs experiences.
