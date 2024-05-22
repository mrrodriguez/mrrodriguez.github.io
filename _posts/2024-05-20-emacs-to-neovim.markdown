---
layout: post
title: "From Emacs to Neovim"
author: Mike Rodriguez
date: 2024-05-20
tags: [emacs, neovim, spacemacs, vim, vscode, vscode-neovim, vi, editor, development, programming]
---

### Update: 2024-05-22

I continue this discussion into a "Part 2" post @ ["From Emacs to Neovim (Part 2)"]({% post_url 2024-05-20-emacs-to-neovim %}) where some concrete configuration examples are provided.

# Background

I have been using emacs for around 15 years with a "traditional" configuration and key bindings setup. I’ve primarily used it for Clojure/ClojureScript development. Occasionally, I have also used it for XML, HTML, JSON, CSS, C/C++, Ruby, Python, JavaScript, and Common Lisp.

About 6 years ago I started to use the [spacemacs distribution](https://www.spacemacs.org). This was done after seeing several of my coworkers using it. I thought that this would be useful due to the general knowledge sharing of common configuration advice and best practices. The defaults worked well when getting started with spacemacs to replace my existing emacs setup. My previous setup was something I had configured in an ad hoc fashion over several years with no particular setup framework in mind. Notably, I did not use `evil-mode` bindings at all. The `helm` search buffer and `magit` key sequences were the closest I had to something like vim bindings (eg. mnemonic key sequences in a "non-insert" mode).

I recently have been working more with languages such as TypeScript/JavaScript and Golang. This has made me curious if I could find better and/or more supported tooling available if I was willing to venture beyond the emacs editor. A natural fit for both seemed to be [vscode](https://code.visualstudio.com). I knew I wanted powerful editing capabilities like I was familiar with already via emacs. I tried some emacs key emulators/extensions in other IDEs before. I did not find them to be sufficient though. Without the elisp, the minibuffer, the "kill ring", among many other specifics to the emacs editor, the integration didn't seem powerful or similar enough to real emacs.

# The idea

Lately, I've seen and heard a lot more about [neovim](https://neovim.io) as an editor. It seems to have brought a new surge in popularity vim. I have been curoius about vim for some time. I knew a little bit about how it "philosophically" differed from emacs and was intrigued.

Since I was using spacemacs, I thought this may be an interesting opportunity to explore vim via `evil-mode`. My plan was to ease into it by using spacemacs ["hybrid editing style"](https://develop.spacemacs.org/doc/DOCUMENTATION.html#hybrid) which uses `evil-mode` for all vim modes (called "states" in spacemacs) other than "insert". This was really convenient for me to be (somewhat) efficient and productive as I learned. Along with this, I did neovim tutorials and read docs to get started learning more of the vim side. I gradually applied these ideas/concepts more to my hybrid editing style spacemacs setup.

# Learning process

It was difficult for a week of dedicated focus on learning vim keys while doing my day to day work. This definitely made me slower for a short period. By about 3 weeks, I felt much more proficient and efficient. After that, I only had some lingering inefficient movement annoyances I knew I’d need to investigate and improve over time. Basically, the thought is to wait until the annoyance/inefficiency becomes frequent enough to be [worth the time investment to address](https://xkcd.com/1205).

From there, I started using vscode for TypeScript with the [vscode-neovim](https://github.com/vscode-neovim/vscode-neovim) extension. I was impressed by how well this extension blended well within vscode. I was also able to find a lot of good resources available when troubleshooting and learning about the extension. The ability to use my actual neovim configuration files from within vscode in non-trivial ways was great too.

# Outcome

I still use spacemacs with hybrid mode bindings while doing Clojure/ClojureScript and various other data files and languages I may lightly use. I still prefer "insert" mode (aka. spacemacs "insert state") to be done with emacs traditional bindings. To me, it is a "better" insert mode experience compared to vim bindings within emacs. However, I now find myself using a lot less insert mode than I used to.

When I switch to vscode with the vscode-neovim extension, I can quickly adapt to that IDE environment. I can even toggle back and forth throughout the day without a struggle. Overall, the switch to vim bindings seems to be a worthwhile investment. I also use neovim now when I want to quickly open/edit text files from the terminal. My general thought is that you can get more “range”/"reach" out of vim bindings due to the extensions available in popular IDEs and other text editors (including in the terminal which is "the default" for neovim). To me, the editing efficiencies are just as powerful and efficient when compared to my traditional emacs experiences.

I will post a follow-up to this post to discuss some helpful resources and/or plugins/extensions/configuration that helped me transition from my traditional emacs setup concepts to vim concepts (via neovim specifically).
