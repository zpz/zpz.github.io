---
layout: post
title: "Setting up Mac for Software Development"
---

With a brand new Mac laptop, I usually do the following set-up in prep for programming work.

1. Customize `Finder`.

   - In the "Sidebar" on the left, remove some items in the "Favorites" section that are not very useful.
   - Add user's home directory to the "Favorites" section. This is done in "Finder -> Preferences -> Sidebar".
   - You may also want to add the "Path" tool to the toolbar. Do this via "View -> Customize Toolbar...".

2. Customize `Terminal`.

   - Add `Terminal` (in "Applications/Utilities") to the "Dock".
   - In "Terminal -> Preferences -> Profiles", choose a profile, set font, background color and opacity, and window size.
   - In the "Shell" tab of "Preferences", for "When the shell exits:", choose "Close if the shell exited cleanly".

3. Let windows always show vertical scroll bar on the right.

   "Apple -> System Preferences -> General", choose "Always" for "Show scroll bars:".

4. Install `Homebrew`.

   Search for "mac homebrew", then follow instructions.

5. `brew install python3`.

6. `brew install git`.

   - Then customize `git`, e.g. download [this 'gitconfig'](https://github.com/zpz/linux/blob/master/git/gitconfig) and make it `~/.gitconfig`.

7. Install `Docker`.

   Go to [docker.io](docker.io) and follow instructions.

8. Customize `bash` environment.

   Download [`bashrc`](https://github.com/zpz/docker/blob/master/dotfiles/bash/bashrc) and
   [`bash_profile`](https://github.com/zpz/docker/blob/master/dotfiles/bash/bash_profile),
   make them `~/.bashrc` and `~/.bash_profile`, respectively. Adapt as needed.

9. Install and customize `Neovim`.

   Follow instructions [here](https://github.com/zpz/docker/tree/master/dotfiles/nvim).

10. Install favorite IDE's, such as `PyCharm` or `VS Code` or `IntelliJ IDEA`.

11. Create a directory tree for work. I prefer to use the following fixed directory structure
    to host code work,

    ```
    cd ~
    mkdir -p work
    cd work
    mkdir -p bin config data log src tmp
    ```

    and remember to add `~/work/bin` to `PATH` in `~/.bashrc`.


