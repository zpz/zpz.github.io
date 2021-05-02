---
layout: post
title: "A Docker Stack for Personal and Team Projects in Python --- Part 2"
excerpt_separator: <!--excerpt-->
tags: [python, docker]
---

[Part 1 of this mini-series](https://zpz.github.io/blog/python-docker-stack-1/) gave an overview of my Docker stack for Python projects. In this post, I'll dive into the image-building part of the stack. This part is mostly specific to Python.
In fact, image-building has a lot to do with good practices regarding code structure of Python projects and workflows of Python package development and installation. I will use this opportunity to have some discussions on these matters.<!--excerpt-->

Since image building is in the context of a single repo, I'll use the [template repo](https://github.com/zpz/docker-project-template-py) as an example, which defines a complete, though minimal and useless, Python package. The main pieces of the code are

- Module code (code of the main package)
- Tests
- Code for package meta data and installation
- Dockerfile and image-building script

Let me state a few principles without elaboration:

1. The center piece of a Python project repo is a Python package. Note, *a* package, not *packages*.
2. Ideally, the package name and the repo name are the same; but they may be different, as is the case in this template repo.
3. Test code is as important as the "package" code itself. There shall be tests.

With that, let's dive into some details.

## Project code structure

[Structure of code is key](https://docs.python-guide.org/writing/structure/). I [wrote about this before](https://zpz.github.io/blog/python-project-tips/), and the main points are

- Put the module code in `src/`.
- Put test code in `tests/` (that is, on the top level, parallel to the module code).

These recommendations have not changed since my previous writing. See [this opinion](https://hynek.me/articles/testing-packaging/) about `src/`.

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

Meta data include package name, author, license, version, and that sort of things. They may be included in a built package (such as a tar ball of the source code intended for distribution) or installed package.

I should point out that *this project is meant for production use*, rather than for providing a tool to be used by other projects. This is important, because in this context I will not **build** the package; rather, I only need to **install** it, and install it in a Docker image, that is.

I've settled on this recommendation:

- Use a `setup.cfg` to hold most meta data.
- Use a short `pyproject.toml` to declare that this project uses `setuptools`.
- Try to put all meta data in either `pyproject.toml` or `setup.cfg`; avoid introducing other meta data files.

My workflow to be described here does not need a `setup.py`.
If we use an equally short `setup.py` in lieu of `pyproject.toml`, things should work just as well. But since [`pyproject.toml` is more forward looking](https://snarky.ca/what-the-heck-is-pyproject-toml/), I've decided to use it.

We will use `pip` to install the package `example`. When `setup.py` does not exist, it requires `pyproject.toml`.

### Where to put the package version

A single source of truth for the version is another subject of heated discussions; see [here](https://packaging.python.org/guides/single-sourcing-package-version/) and [here](https://stackoverflow.com/questions/458550/standard-way-to-embed-version-into-python-package) and [here](https://martin-thoma.com/python-package-versions/).

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

In the Docker workflow, the tag of the built Docker image is far more important than this package version. If the developer does not diligently bump this `__version__` over time, it's largely OK.

## Dependency management and Dockerfile

Dependencies are usually listed in `setup.py` or a dedicated `requirements.txt`. In this workflow, however, my approach is:

- Let `Dockerfile` explicitly install everything needed, and don't mention the dependencies in `setup.py` and don't use a `requirements.txt`.

The main reason has to do with installing additional Linux libraries or compilation tools (`gcc`, `g++`, etc) if any dependency needs them. In those situations, it's most flexible to let `Dockerfile` take control of it all.

I've placed `Dockerfile` in directory `docker/` as opposed to leaving it on the root level. This is not crucial, but it highlights the fact that when this `Dockerfile` is used, the code will not be copied into the image (see sections below), hence the directory `docker/` is self sufficient for the ["build context"](https://docs.docker.com/develop/develop-images/dockerfile_best-practices/#understand-build-context).


## Let the image be built!

Now that the module and test code is ready, and meta data is taken care of, let's build the Docker image. There's a script [build-docker](https://github.com/zpz/docker-project-template-py/blob/master/build-docker) in the root directory, listed below in its entirety:

```
#!/bin/bash

set -e

TINY=zppz/tiny:21.01.02
cmd_find="$(docker run --rm ${TINY} cat /usr/tools/find-image)"
MINI=$(bash -c "${cmd_find}" -- zppz/mini)
cmd_build="$(docker run --rm ${MINI} cat /usr/tools/build-docker)"
PARENT=$(bash -c "${cmd_find}" -- zppz/py3)

bash -c "${cmd_build}" -- --build-arg PARENT=${PARENT} $@
```

Here, we see the techniques described in [Part 1](https://zpz.github.io/blog/python-docker-stack-1/) at play:

1. We use the stable image `tiny` to find the latest versions of the image `mini` and the parent image `py3`.

2. Then, from `mini` we extract the script `build-docker` and run it. The command does not even mention where the `Dockerfile` is, because the script `build-docker` from `mini` knows it by convention.

Note, this script `build-docker` in the project repo can be quite stable, but the script `build-docker` in `mini` may evolve as quickly as needed. We always use the latest version of it via the latest version of `mini`.

Now let's examine the script [build-docker](https://github.com/zpz/docker-mini/blob/master/tools/build-docker) in `mini`.


### The dev/prod image pair

Suppose we run this script in branch `abc`, then two images will be built, named `zppz/docker-project-template-py` and `zppz/docker-project-template-py-abc`, respectively. (Remember `'docker-project-template-py'` is the repo's name.). We'll call the first the "dev image" and the second the "prod image". Their differences are

1. The dev image has all dependencies installed, and nothing else. It does not contain anything from the current project repo.
2. The prod image uses the dev image as the base, and installs the project's Python package (i.e. `example` here) in it.
3. In addition, code checks and tests are run in the just-built prod image. If these fail, the build is considered a failure. The consequence of build failure in a coporate environment is, e.g., no image is pushed to the private Docker image registry.

This design achieves some **critical** effect:

1. The dev and prod images have identical execution environments.
2. The code is tested in the prod image, which is going to be deployed in production.

Some points will become more clear in [Part 3](https://zpz.github.io/blog/python-docker-stack-3/) of this series. For now, let's continue to examine the build code. Please [refer to the code](https://github.com/zpz/docker-mini/blob/master/tools/build-docker) as needed.


### Building the dev image

Building the dev image is relatively simple.
Several things to note include

- The image name is the repo name, i.e. the parent directory name, hence it's easily detected by code.
- The image tag is generated by the "datetime version command" in the image `tiny`.
- The Dockerfile is located in `docker/` by assumption.
- Build args defined in the Dockerfile are passed in. This always includes the base image (or "parent image"), plus others as needed.


### Building the prod image

Building the prod image (called the "branch image" in the code) is slightly more complex. The following points are of note:

- It uses the just-built dev image as the base.
- Its tag is the same as the dev image.
- It uses a Dockerfile that is created by code on-the-fly.


### Building the test image

There's a third image, which is auxciliary and is forgotten after use. Its sole purpose is to run code checks and tests *in the prod image*.
The main points of note include:

- It uses the just-built prod image as the base.
- It installs some packages related to code checks and testing.
- It installs a script called [`check-n-test` from the image `mini`](https://github.com/zpz/docker-mini/blob/master/tools/check-n-test).


### Running code checks and tests

As of this writing, these code checks are run:

- [`radon`](https://pypi.org/project/radon/) for code complexity metrics, including "cyclomatic complexity", "maintainability index", and "Halstead complexity".
- [bandit](https://pypi.org/project/bandit/) for security issues, such as hard-coded credentials.
- [pyflakes](https://pypi.org/project/pyflakes/), mainly detecting name errors and unused variables or imports.
- [pylint](https://pypi.org/project/pylint/) for various code issues.
- [mypy](http://www.mypy-lang.org/) for type annotations.

These checks are not equally important. The security scanner `bandit` is run on both module code and test code. It will halt the build at any serious issue. Similarly, issues found by `pyflakes` also halt the build. On the other hand, the reports of `radon`, `pylint`, and `mypy` are only for reference. These configurations may change, of course depending on the nature of the project and the developer's opinions.

Finally, the test code is run if coverage target is greater than 0. The coverage target is a parameter that can be specified in the script `build-docker` in the project repo. Indeed, during the development of the project, the target of test code coverage often needs to be adjusted, for example, it may be gradually increased as the code gets more stable and mature.

The actual test coverage is reported. If it is below the target, the build fails.

That's all for this post. Please read the other parts:

1. [Part 1](https://zpz.github.io/blog/python-docker-stack-1/): overview.
2. Part 2: building images (this post).
3. Part 3: running containers.
4. Part 4: building images for a separately installable Python package. (In contrast, the images built in "Part 2" do not provide a separately installable package. In order to use the Python package therein, one needs to use these images as base image.)
