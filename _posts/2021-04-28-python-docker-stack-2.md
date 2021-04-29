---
layout: post
title: "A Docker Stack for Personal and Team Projects in Python --- Part 2"
excerpt_separator: <!--excerpt-->
tags: [python, docker]
---

[Part 1 of this mini-series](https://zpz.github.io/blog/python-docker-stack-1/) gave an overview of my Docker stack for Python projects. In this post, I'll dive into the image-building part of the stack. This part is mostly specific to Python.
In fact, image-building has a lot to do with good practices regarding code structure of Python projects and workflows of Python package development and installation. I will use this opportunity to have some discussions on these matters.<!--excerpt-->

Since image building is in the context of a single repo, I'll focus on the [template repo](https://github.com/zpz/docker-project-template-py), which defines a complete, though minimal and useless, Python package. The main pieces of the code are

- Module code (code of the main package)
- Tests
- Package meta data and installation
- Dockerfile and image-building script

Let me state a few principles without elaboration:

1. The center piece of a Python project repo is a Python package. Note, *a* package, not *packages*.
2. Ideally, the package name and the repo name are the same; but they may be different, as is the case in this template repo.
3. Test code is as important as the "package" code itself.

With that, let's dive into some details.

## Project code structure

[Structure of code is key](https://docs.python-guide.org/writing/structure/). I [wrote about this before](https://zpz.github.io/blog/python-project-tips/), and the main points are

- Put the module code in `src/`.
- Put test code in `tests/` (that is, on the top level, parallel to the module code).

These recommendations have not changed. See [this opinion](https://hynek.me/articles/testing-packaging/) about `src/`.

The module code and tests are the center pieces of a Python project. Everything else is supportive. The template repo is laid out as follows:

```shell
$ tree .
.
├── build-docker
├── docker
│   └── Dockerfile
├── pyproject.toml
├── README.md
├── setup.cfg
├── src
│   └── example
│       ├── dummy.py
│       ├── __init__.py
│       └── py.typed
└── tests
    └── test_dummy.py
```

The repo (hence the parent directory) is named `docker-project-template-py`. This repo defines a Python package named `example`. The package source is located in `src/`, whereas test code is in `tests/`. The `Dockerfile` is in subdirectory `docker/`.
The other few files on the top level are concerned about Docker building (`build-docker`), package metadata and installation (`pyproject.toml`, `setup.cfg`), and other info (`README.md`). In a substantial project, there may be other top-level directories and files such as `bin/`, `doc/`, `LICENSE`, `CHANGELOG`, and so on.

## Meta data and the build/install toolchain

Meta data include package name, author, license, version, and that sort of things. They are pieces of info about the package, and they may be included in a built package (such as a tar ball of the source code for intended for distribution) or installed package.

I should point out that *this project is meant for production use*, rather than for providing a tool to be used by other projects. This is important, because in this context I will not **build** the package; rather, I only need to **install** it, and install it in a Docker image, that is.

I've settled on this recommendation:

- Use a `setup.cfg` to hold most meta data.
- Use a short (3-line) `pyproject.toml` to declare that this project uses `setuptools`.

My workflow to be described here does not need a `setup.py`.
If we use an equally short `setup.py` in lieu of `pyproject.toml`, things should work just as well. But since [`pyproject.toml` is more forward looking](https://snarky.ca/what-the-heck-is-pyproject-toml/), I've decided to use it.

Try to put all meta data in `setup.cfg` as long as it supports that.

We will use `python -m pip install .` to install the package `example`. When `setup.py` does not exist, it requires `pyproject.toml`.

### Where to put the package version

A single source of truth of the version is another subject of heated discussions; see [here](https://packaging.python.org/guides/single-sourcing-package-version/) and [here](https://stackoverflow.com/questions/458550/standard-way-to-embed-version-into-python-package) and [here](https://martin-thoma.com/python-package-versions/).

Here's how I do it (see [here](https://stackoverflow.com/questions/60430112/single-sourcing-package-version-for-setup-cfg-python-projects)):

1. Define the value in `src/example/__init__.py`:

   ```
   __version__ = '21.04.25'
   ```
2. Refer to it in `setup.cfg`:

   ```
   [metadata]
   name = example
   version = attr: example.__version__
   ```

In the Docker workflow, the tag of the built Docker image is far more important than this version of the package. If the developer does not diligently bump this `__version__` over time, it's largely OK.

## Dependency management and Dockerfile

Dependencies are usually listed in `setup.py` or a dedicated `requirements.txt`. In this workflow, however, my approach is:

- Let `Dockerfile` explicitly install everything needed, and don't mention the dependencies in `setup.py` and don't use a `requirements.txt`.

The main reason has to do with installing additional Linux libraries or compilation tools (`gcc`, `g++`, etc) if any dependency needs them. In those situations, it's most flexible to let `Dockerfile` take control of it all.

I've placed `Dockerfile` in directory `docker/` as opposed to leaving it on the root level. This is not crucial, but it highlights the fact that when this `Dockerfile` is used, the code will not be copied into the image (see subsequent sections), hence the directory `docker/` is self sufficient for the ["build context"](https://docs.docker.com/develop/develop-images/dockerfile_best-practices/#understand-build-context).


## Let the image be built

### The dev/prod image pair

### Building the dev image

### Building the prod image



That's all for this post. Please read the other parts:

1. [Part 1](https://zpz.github.io/blog/python-docker-stack-1/): overview.
2. Part 2: building images (this post).
3. Part 3: running containers.
4. Part 4: building images for a separately installable Python package. (In contrast, the images built in "Part 2" do not provide a separately installable package. In order to use the Python package therein, one needs to use these images as base image.)
