---
layout: post
title: "A Docker Stack for Personal and Team Projects in Python --- Part 2"
excerpt_separator: <!--excerpt-->
tags: [python, docker]
---

[Part 1 of this mini-series](https://zpz.github.io/blog/python-docker-stack-1/) gave an overview of my Docker stack for Python projects. In this post, I'll dive into the image-building part of the stack. This part is mostly specific to Python.
In fact, image-building has a lot to do with good practices regarding code structure of Python projects and workflows of Python package development and installation. I will use this opportunity to have some discussions on these matters.<!--excerpt-->

Since image building is in the context of a single repo, I'll focus on the [template repo](https://github.com/zpz/docker-project-template-py), which defines a complete, though minimal and useless, Python package. First of all, this repo is laid out as follows:

```shell
$ tree .
.
├── build-docker
├── Dockerfile
├── README.md
├── setup.py
├── src
│   └── example
│       ├── dummy.py
│       ├── __init__.py
│       └── py.typed
│       └── _version.py
└── tests
    └── test_dummy.py

4 directories, 9 files
```

The repo (hence the parent directory) is named `docker-project-template-py`. This repo defines a Python package named `example`. The package source is located in `src/example`, whereas test code is in `tests/`. The other few files on the top level of the repo directory are concerned about Docker (`Dockerfile`, `build-docker`), package metadata and installation (`setup.py`), and other info (`README.md`). In a substantial project, there may be other top-level directories and files such as `bin/`, `doc/`, `LICENSE`, and so on. Nevertheless, the main topics are

- Package source code
- Tests
- `setup.py` and related
- Dockerfile and build script

Let me state a few principles without elaboration:

1. The center piece of a Python project repo is a Python package.
2. One repo should define a single Python package, not multiple packages.
3. Ideally, the package name and the repo name are the same; but they may be different, as is the case in this template repo.


That's all for this post. Please read the other parts:

1. [Part 1](https://zpz.github.io/blog/python-docker-stack-1/): overview.
2. Part 2: building images (this post).
3. Part 3: running containers.
4. Part 4: building images for a separately installable Python package. (In contrast, the images built in "Part 2" do not provide a separately installable package. In order to use the Python package therein, one needs to use these images as base image.)
