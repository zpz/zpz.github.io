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

5. `brew install` `python3` and `git` (if not already installed).

6. Customize `git`.

   Follow instructions [here](https://github.com/zpz/linux/tree/master/git).


7. Customize `bash` environment.

   Follow instructions [here](https://github.com/zpz/linux/tree/master/bash).


8. Install favorite IDE's, such as `VS Code` or `PyCharm` or `IntelliJ IDEA`.

9. Create a directory tree for code work. I prefer to use the following fixed directory structure

    ```sh
    cd ~
    mkdir -p work
    cd work
    mkdir -p bin config data log src tmp
    ```

    and remember to add `~/work/bin` to `PATH` in `~/.bashrc`.

10. Install and customize `Neovim` (if you are a `vim` user).

   Follow instructions [here](https://github.com/zpz/linux/tree/master/neovim).

11. Install `Docker` (if not already installed).

   Go to [docker.io](https://www.docker.io) and follow instructions.
