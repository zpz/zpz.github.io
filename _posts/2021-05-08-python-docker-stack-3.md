---
layout: post
title: "A Docker Stack for Personal and Team Projects in Python --- Part 3"
excerpt_separator: <!--excerpt-->
tags: [python, docker]
---

[Part 1](https://zpz.github.io/blog/python-docker-stack-1/) and [Part 2](https://zpz.github.io/blog/python-docker-stack-2/) of this mini-series have gaven an overview of my Docker stack for Python projects and examined the image building process. This post will dive into *using the image* or, in other words, running the Docker image(s) during development and production.<!--excerpt-->

In the spirit of making it simple, I've devised a single command to do it all. The command is named, surprisingly, `run-docker`. [The entire command](https://github.com/zpz/docker-mini/blob/master/sbin/run-docker) is a short Bash script:

```shell
#!/bin/bash

set -e

# Although we check `--local` anywhere in the command,
# it's recommended to specify it before image name.
use_local=no
for arg in "$@"; do
    if [[ "${arg}" == --local ]]; then
        use_local=yes
        break
    fi
done

TINY=zppz/tiny:21.01.02
if [[ "${use_local}" == yes ]]; then
    MINI=$(bash -c "$(docker run --rm ${TINY} cat /usr/tools/find-local-image)" -- zppz/mini)
else
    MINI=$(bash -c "$(docker run --rm ${TINY} cat /usr/tools/find-image)" -- zppz/mini)
fi

cmd="$(docker run --rm ${MINI} cat /usr/tools/run-docker)"
bash -c "${cmd}" -- $@
```

This suggests the command is used like this:

```shell
$ run-docker [--local] ...
```

The flag `--local` is used when we're not connected to the internet, or deliberately want to use what's available on the local machine, that is, we don't want it to automatically download new versions of images from an image registry. In the following descriptions we'll not worry about this option. With this out of the way, the script is just a few lines:

```shell
#!/bin/bash

set -e
TINY=zppz/tiny:21.01.02
MINI=$(bash -c "$(docker run --rm ${TINY} cat /usr/tools/find-image)" -- zppz/mini)
cmd="$(docker run --rm ${MINI} cat /usr/tools/run-docker)"
bash -c "${cmd}" -- $@
```

We see our classic trick at it again: we use the hardly-ever-need-to-upgrade image `tiny` to find the latest version of the image `mini`, from which we extract the script `run-docker` and use it. So, the command `run-docker` on a development machine or production machine rarely needs upgrad, while its actual behavior may be evolving over time.

Since the real code is the [`run-docker` in `mini`](https://github.com/zpz/docker-mini/blob/master/tools/run-docker), please refer to that as I go over the main points next. A the end of the code is this line:

```
docker run ${opts} ${imagefullname} ${command} ${args}
```

The entire script is about determining all the options to the command `docker run`, and use them on this final line. Some of the options are passed in by the user. Some others are specified or derived in the code.

I'll continue to use the [template repo](https://github.com/zpz/docker-project-template-py) for examples.


## Differences between dev and prod modes

As mentioned in [Part 2](https://zpz.github.io/blog/python-docker-stack-2/), when we run `build-docker` in branch `zepu`, two images are built, namely,
`zppz/docker-project-template-py` and `zppz/docker-project-template-py-zepu`. (In real settings, you may want to use a shorter repo name to save some typing. Besides, the images pushed to registry are usually built in branch `master`, not any other feature branch.) These two images are used in development and production, respectively.

To facilitate development, the dev image does not contain the projct code---it contains dependencies only. As we develop code, we want to run tests to see whether it works; make some code change, run tests again, and iterate that way. We should not need to re-build the image in this process.

The solution to use "live" code is to ["volumn map"](https://docs.docker.com/storage/bind-mounts/) the source code directory into the container using the `-v` option of `docker run`. When typing `run-docker zppz/docker-project-template-py`, we land in a container,

```
zepu@zepu-desktop in ~
$ run-docker zppz/docker-project-template-py

[docker-user in docker-project-template-py] ~/src [zepu]
$ pwd
/home/docker-user/src

[docker-user in docker-project-template-py] ~/src [zepu]
$ ls
README.md  archive/  build-docker*  docker/  pyproject.toml  setup.cfg  src/  tests/

[docker-user in docker-project-template-py] ~/src [zepu]
$ 
```

We see the entire repo directory in the container as directory `~/src`.
It's important to understand that this directory is not copied into the Docker image (which would have been immutable), but rather the "live" code on the host machine "tunneled into" the container. If the code outside of the container is changed, we see the change inside the container. Again, there is no copying taking place. This is achieved by, essentially,

```shell
docker run ... -v ~/work/src/docker-project-template-py:/home/docker-user/src ...
```

I intentionally typed the command `run-docker zppz/docker-project-template-py` in the user home directory instead of the repo's root directory. How does `run-docker` know this image is connected to this repo, and further, this is a dev image so that it should set up volume mapping?

This has to assume certain directory layout on the machine. The assumption `run-docker` uses is

*Project repo directory is directly under `$HOME/work/src/`.*

With this assumption, `run-docker` checks whether directory `$HOME/work/src/docker-project-template-py` exists. If yes, it's a dev image. If no, it's a prod image.

Within the container, we can navigate to the tests directory and run some tests:

```
[docker-user in docker-project-template-py] ~/src [zepu]
$ cd tests/

[docker-user in docker-project-template-py] ~/src/tests [zepu]
$ ls
test_dummy.py

[docker-user in docker-project-template-py] ~/src/tests [zepu]

$ py.test test_dummy.py 
========================= test session starts ==========================
platform linux -- Python 3.8.6, pytest-6.2.3, py-1.10.0, pluggy-0.13.1
rootdir: /home/docker-user/src, configfile: pyproject.toml
plugins: anyio-2.2.0, pudb-0.7.0, cov-2.11.1, asyncio-0.15.1
collected 1 item                                                       

test_dummy.py .                                                  [100%]

========================== 1 passed in 0.14s ===========================
```

It is very convenient to have one terminal open in the dev container, running tests and investigative scripts over and over, and modify the code outside the container, such as in a nice IDE.

There is one question, though: how does the test code find the package `example`? After all, the package is not installed in the dev image, and the source directory is not visible in the `tests/` directory.
The answer is that the `run-docker` command adds the source code directory to `PYTHONPATH` once it determines the image is for development. Essential it does

```shell
docker ... -e PYTHONPATH=/home/docker-user/src/src ...
```

In addition to the source code, `run-docker` volume-map a data directory and a log directory into the container in both dev and prod modes. In a dev container, we see

```
[docker-user in docker-project-template-py] ~
$ ls
data/  log/  src/
```

In a prod container, we see

[docker-user in docker-project-template-py-zepu] ~
$ ls
data/  log/
```

The `data/` and `log/` are maps of relevant directories in `~/work/` on the host machine.
In addition, they are pointed to by environment variables `DATADIR` and `LOGDIR`, respectively. The project code can use these variables, assuming their existence. Environment variables are set by the `-e` option of `docker run`.

## How does this facilitate team work

Notice that I typed `run-docker zppz/docker-project-template-py` without specifying the version of the image. Inside `run-docker`, it detects whether the image version is specified. If not, it will find the latest version by the command `find-image` in the image `tiny`.

Imagine multiple developers are working on this repo. When Mary begins a new day and decides to test some new code, she types

```
$ run-docker zppz/docker-project-template-py
```

without knowing that Chris has successfully merged some code late yesterday. The company's CI system built and tested the code, and pushed new versions of images `zppz/docker-project-template-py` and `zppz/docker-project-template-py-mast` to a private Docker image registry. Nonetheless, the command `run-docker` will know this and will download and use the latest version of the images. Mary does not need to know that she needs to rebuild the image---`run-docker` pulls the latest image for her.

It is also accepted to specify an exact version of the image. For example, we could do

```
$ run-docker zppz/docker-project-template-py:20210502-002343
```

In production, we may want to explicitly specify the version of the image to deploy. Sometimes it's for the need of a rollback.

## Other considerations in `run-docker`

- passing docker args
- passing project code args
- restrict memory use
- use user account
- jupyter nb


That's all for this post. Please read the other parts:

1. [Part 1](https://zpz.github.io/blog/python-docker-stack-1/): overview.
2. [Part 2]((https://zpz.github.io/blog/python-docker-stack-2/)): building images.
3. Part 3: running containers (this article).
4. Part 4: building images for a separately installable Python package. (In contrast, the images built in "Part 2" do not provide a separately installable package. In order to use the Python package therein, one needs to use these images as base image.)
