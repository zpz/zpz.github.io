---
layout: post
title: "A Docker Stack for Personal and Team Projects in Python --- Part 1"
excerpt_separator: <!--excerpt-->
tags: [python, docker]
---

I started using Docker in early 2016. After learning it for a month or two, I have never done code development outside of Docker again. My Docker workflow has evolved over time. The main drivers have been designing for teamwork. So far I have had three major design rounds, which happened in early 2018 (for one team), early 2019 (for another team), and late 2020 (major simplifying overhaul). By now, I feel the stack has reached a relatively stable and good stage (as I [have felt previously](https://zpz.github.io/blog/poor-mans-CD-system-using-Docker/)!), so I decided to write down the main ideas of it.<!--excerpt--> The stack for my personal experiments are available on Github, which will be referred to in the post. The principles are the same as those for a team stack.

To start off, what are the design goals for a Docker workflow? In hindsight, I think these two are the most important:

- It must be **very** easy to use.
- It must enforce or at least encourage standard, good practices, as opposed to homegrown, nonstandard ways that "also work" for the immediate task.

What is "easy to use"? These characteristics come to mind:

- It must automate everything that can be automated. Moreover, user is better off not seeing the things that are automated away. For example, user should not need to copy a 100-line YAML config file just to customize two lines of it.
- It must be fairly stable. You can not announce every few weeks, "Hi team, a new version of our Docker script is pushed. Please upgrade it on every machine and project where it is used." 

  However, it is nearly impossible to be absolutely bug free, and downright impossible to foresee every future need. We do need to make fixes and small changes from time to time. The design must allow some evolution, yet does not require user to upgrade in order to get the new functionalities. What? Well, it's like a service with a stable API. 
- The user interface must be minimal, that is, the user needs to remember only one or two commands with one or two options.

How can we achieve such automation and simplicity? As the stack evolved, it ended up with two techniques underpinning most of the solutions:

1. Give Docker images auto-generated, date-time based, sortable versions (or "tags" in Docker nomenclature). In most cases (or default cases), we would want to use the latest version of an image. With this versioning scheme, the latest version is easily found by code.
2. Use a utility image to host scripts, but run the scripts outside of their hosting images. Suppse a Bash script is contained in a string `${script}`, then we can run it this way

   ```
   $ bash -c "${script}" -- [args]
   ```

   where `[args]` are any arguments the script takes. It follows immediately that if image `abc:20210421` has a script `script.sh` in `/usr/local/bin`, we can use the script outside of Docker as follows:

   ```
   $ cmd="$(docker run --rm abc:20210421 cat /usr/local/bin/script.sh)"
   $ bash -c "${cmd}" -- [args]
   ```

   This is a critical little trick that makes the stack stable to its user, yet extensible in its behavior. We'll see how it works soon.

On the high level, the Docker stack consists of these main components:

1. Some requirements and assumptions about the directory layout on the host machine and inside the project repo. These requirements are easy to satisfy and are not restrictive in terms of capabilities.
2. A Docker image called [`tiny`](https://github.com/zpz/docker-tiny). This image contains a few commands that are expected to be very stable, such that other utilities refer to this image with a hard-coded tag (w/o worrying about updating it often), and use the commands contained in it. In particular, this image contains a command that finds the latest tag of a specified Docker image. Thanks to the sortable image versions, this script is stable.
3. A Docker image called [`mini`](https://github.com/zpz/docker-mini). This image contains additional commands for building project images and running project containers. The command internals may change from time to time, but their names should not change, and their user interfaces should be stable. Now, `tiny` and `mini` form something like a cache hierarchy. User code will use the very stable `tiny` to find, dynamically, the latest version of `mini`, and copy commands out of `mini` to use. ( I came to consciously, extensively use this pattern only in the late 2020 iteration.)
4. Some [base images](https://github.com/zpz/docker) for all projects to build on.
5. A [project template](https://github.com/zpz/docker-project-template-py) for new projects to copy and start from. This repo maintains the single source of truth for things like the code structure in a project repo, location of the Dockerfile, the build script, etc.

Next, I'll describe some details of `tiny` and the base images.
By the way, since `tiny` and `mini` are used in many places, and a newer version of `mini` may be downloaded anytime automatically, these images should be as small as possible. Noticing that both images are shell-script only, and do not even require Bash shell, they are built on the very smallest base image, `busybox`. In fact, both images are below 1.5 Mb in size.

The image [`tiny`](https://github.com/zpz/docker-tiny)
contains two types of commands. The first type of commands generate version strings to be used by image building scripts. Specifically, there are two commands for two variants of the versioning scheme:

```shell
$ docker run --rm zppz/tiny:21.01.02 make-date-version
21.04.25

$ docker run --rm zppz/tiny:21.01.02 make-datetime-version
20210425-233226
```

The date-based versions are recommended for utility libraries that are not released so often. The datetime-based versions are recommended for "product projects", which may have frequent builds.

Note that I used the exact `tiny` image `zppz/tiny:21.01.02`. User scripts can do the same thing because the image is very stable.

The second type of commands concern finding the latest tag of a specified Docker image. Currently, there's only one command of this type, namely `/usr/tools/find-image`. This command is designed to run outside of Docker. For example,

```shell
$ cmd="$(docker run --rm zppz/tiny:21.01.02 cat /usr/tools/find-image)"

$ bash -c "${cmd}" -- zppz/mini
zppz/mini:21.04.25
```

This is also how it is intended to be used in user scripts.

The [base images](https://github.com/zpz/docker) contain some basic and common stuff so that project repos have a common baseline, and don't need to repeat the same setup. As of now, the images contain

- A carefully chosen base, currently `ubuntu:20.10`. Some considerations in this choice include the distro' default Python version (3.8), compatibility with `cuda`, compatibility with the team's build environment---is it in the Debian lineage or CentOS lineage?---etc. (For a team, `ubuntu:20.04` might be preferable as it is a "Long Term Support" version.)
- Non-root user account `docker-user`, in group `docker-user`, with home directory `/home/docker-user`. It is the intention that downstream images always run as this user.
- Very basic Linux packages such as `curl`, `unzip`, etc. Note, the base images should strike a balance between light weight and usefulness. For example, I do not recommend having `vim` and `git` in there, because it is not recommened to *develop code* within a container (it is rather for executing code).
- Python 3.8.
- A few Python packages related to testing (`pytest`) and debugging (`pudb`).
- A *better Python REPL* called `ptpython`.
- Jupyter notebook package.
- Nice configuration for the things installed, such as informative Bash prompt, `ls` coloring, Jupyter behavior, integration between `pudb` and `pytest`, etc.

That's all for this post. Please read the subsequent parts:

1. Part 1: overview (this article).
2. Part 2: building images.
3. Part 3: running containers.
4. Part 4: building images for a separately installable Python package. (In contrast, the images built in "Part 2" do not provide a separately installable package. In order to use the Python package therein, one needs to use these images as base image.)
