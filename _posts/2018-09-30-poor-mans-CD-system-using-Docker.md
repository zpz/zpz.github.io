---
layout: post
title: A Poor Man's Continuous Deployment System Using Docker
excerpt_separator: <!--excerpt-->
tags: [Docker, CICD]
---

I've implemented a "continuous deployment" (CD) system centered on Docker.
<!--excerpt-->
I have not studied any standard approaches to CD.
In this "system", automated updates of things are not triggered by events such as code commit,
but rather by scheduled cron jobs.
There is no test suits to be run automatically---this is not because I do not consider
testing to be essential, but rather is aligned with the reality of my (team's) code.
On the other hand, the type of deployed code that is of concern here is mostly batch-mode
pipelines, for which a crash in production is not the end of the world.
There is no doubt that my solution is crude, but it is practical.
For its purpose, it does a pretty good job.

## The setting

Suppose I and a small team develop back-end data pipelines.
All of our code is hosted on Github under organization `my-org`.
We have three repos:

- `infra`: this is a general framework/utility library that our other work uses.
- `app1`: this is a particular work component, which makes use of `infra`.
- `my-docker`: this contains Docker image definitions and related tools for `infra`, `app1` (and other repos in a position similar to that of `app1`).

All the code is in Python.
There is a top-level directory in repo `app1`, named `scripts`, that contains pipeline scripts that are deployed in production.

Our Docker images are stored in AWS Elastic Container Service (ECS).

There are several goals for this automation system:

1. Docker images are automatically re-built when appropriate. There are three occasions that demand image rebuild:
   1. Image definition in `my-docker` has changed.
   2. Some code in `infra` or `app1` has changed, and the code is to be "sealed" in an image.
   3. An upstream image has been re-built, requiring the rebuild of a downstream image.
2. On a development machine (or deployment machine), simple commands are defined for launching Docker images.
   They automatically launch the latest version that exists in ECS. If the latest version is already present on the local machine, the launch is fast. Otherwise, downloading will happen, which takes some time.
3. Once new code is merged into the `develop` branch of the repo `app1`, deployed pipelines in `app1/scripts/`,
   triggered by cron, automatically run in a properly rebuilt image that contain the new code.


## The `my-docker` repo

This repo started like this:

```
my-docker/
  |-- py3dev/
  |     |-- Dockerfile
  |     |-- build.sh
  |-- infra-prod/
  |     |-- Dockerfile
  |     |-- build.sh
  |-- infra-dev/
  |     |-- Dockerfile
  |     |-- build.sh
  |-- app1-prod/
  |     |-- Dockerfile
  |     |-- build.sh
  |-- app1-dev/
  |     |-- Dockerfile
  |     |-- build.sh
  |-- build.sh
  |-- run-docker
  |-- common.sh
  |-- pipeline
  |-- pyscript
  |-- auto-build
  |-- auto-build.sh
```

Some explanation of the relations between our images is in order.

`py3dev` is the base of them all.
It contains Python 3.6, some common Python packages, `Jupyter` notebook, utilities, a nice editor (`neovim`, that is), and so on.
In addition, it has a user account `docker-user`. All code execution in our containers is via this user.

`infra-dev` is used for developing the code in repo `infra`. Importantly, it does not contain `infra` code,
but rather volume-map the `infra` code directory on the local machine into the container, so that code changes can happen both from insider and outside of the container, and test run always uses the latest in-development code.
If `infra` depends on some third-party libraries, they are installed in `infra-dev`. This image is based on `py3dev`.

`infra-prod` is the 'production' or 'deployed' image of `infra`. It is based on `infra-dev` (which contains third-party dependencies of `infra`) and, importantly, contains, say, the `develop` branch of the `infra` code. The `infra` code sealed in the `infra-prod` image is read-only, appropriate for production use.

`app1-dev` is used for developing the code in repo `app1`. It contains third-party dependencies of `app1`,
is based on `infra-prod`, and volume-maps the source code of `app1` from the local machine into the container.

`app1-prod` is based on `app1-dev`. In addition to the dependences installed in `app1-dev`, it contains a read-only copy of the `app1` code.


## Building the images

Let's not worry about auto-build for now.
Let's use `build.sh` to build all the images.

`build.sh` is simple. Below is the entire script:

```bash
set -Eeuo pipefail

# The sole optional argument is 'push'.

thisfile="${BASH_SOURCE[0]}"
thisdir="$( cd $( dirname ${thisfile} ) && pwd )"

(cd "${thisdir}"/py3dev && bash build.sh $@)

repos=( infra app1 )
for repo in "${repos[@]}"; do
    (cd "${thisdir}"/${repo}-dev && bash build.sh $@)
    (cd "${thisdir}"/${repo}-prod && bash build.sh $@)
done
```

The `build.sh` scripts in the image-specific directories are all similar.
They all "source" in the common utility file `common.sh`, and take one optional argument
indicating whether to push the images to AWS ECS.

The entire `py3dev/build.sh` is as follows:

```bash
set -Eeuo pipefail

# The sole optional argument is 'push'.

thisfile="${BASH_SOURCE[0]}"
thisdir="$( cd "$( dirname "${thisfile}" )" && pwd )"
parentdir="$( dirname "${thisdir}" )"
source "${parentdir}/common.sh"

PARENT_IMAGE=python:3.6-slim-stretch

cp -f "${parentdir}/pipeline" "${thisdir}/"
cp -f "${parentdir}/pyscript" "${thisdir}/"
build_simple "${thisdir}" "${PARENT_IMAGE}" py3dev $@
rm -f "${thisdir}/pipeline" "${thisdir}/pyscript"
```

The script calls the function `build_simple` the is defined in `common.sh` and takes one optional argument `push`.
As for the scripts `pipeline` and `pyscript`, we'll get to them alter.

Here is the script `infra-dev/build.sh`:

```bash
set -Eeuo pipefail

# The sole optional argument is 'push'.

thisfile="${BASH_SOURCE[0]}"
thisdir="$( cd "$( dirname "${thisfile}" )" && pwd )"
parentdir="$( dirname "${thisdir}" )"
source "${parentdir}/common.sh"

parent_name=py3dev
parent_version=$(pull_aws_latest ${parent_name})
parent_image=${parent_name}:${parent_version}

build_dev "${thisdir}" "${parent_image}" $@
```

The commands `pull_aws_latest` and `build_dev` are defined in `common.sh`.
The former pulls the latest version of the specified image from AWS ECS (if not already present on the local machine) and returns its tag. The tag is used to specify the parent image for the image being built.

Here is the script 'infra-prod/buil.sh':

```bash
set -Eeuo pipefail

# Sole optional argument is 'push'.

thisfile="${BASH_SOURCE[0]}"
thisdir="$( cd "$( dirname "${thisfile}" )" && pwd )"
parentdir="$( dirname "${thisdir}" )"
source "${parentdir}/common.sh"

parent_name=infra-dev
parent_version=$(pull_aws_latest ${parent_name})
parent_image=${parent_name}:${parent_version}

build_prod "${thisdir}" "${parent_image}" $@
```

Note that it specifies `infra-dev` as the parent image.
`build_branch` is another function defined in `common.sh` (we'll get to it).

The script `app1-dev/build.sh` is identical to `infra-dev/build.sh` except that
`parent_name` in the script is defined to be `infra-prod`.

The script `app1-prod/build.sh` is identical to `infra-prod/build.sh` except that
`parent_name` in the script is defined to be `app1-dev`.


## Utility functions in `common.sh`

Now it's time to demystify the functions defined in `common.sh`.

We start with the few functions related to finding the latest version of an image on AWS ECS,
and on the location machine.

Note that our images on ECS are located at `my-org/py3dev`, `my-org/infra-dev`, `my-org/infra-prod`,
`my-org/app1-dev`, and `my-org/app1-prod` at the address pointed to by `$ECR_URL`.


```bash
function log_in_aws {
    $(aws ecr get-login --no-include-email --region ${AWS_DEFAULT_REGION}) > /dev/null 2>&1
}


# Find latest tag of specified repo AWS.
# This assumes all tag names are already sort-able, so
# this does not get 'latest' by timestamp, rather just by tag.
function find_aws_latest_tag {
    log_in_aws
    name="my-org/$1"
    z=$(aws ecr list-images \
        --repository-name ${name} \
        --query 'sort_by(imageIds,& imageTag)[-1].imageTag')

    # Remove quotes.
    z=${z#*\"}
    z=${z%\"*}
    echo "$z"
}


# Find latest tag of specified repo on the local machine.
# This assumes all tag names are already sort-able, so
# this does not get 'latest' by timestamp, rather just by tag.
function find_latest_tag {
    docker images "$1" --format {% raw %}"{{.Tag}}"{% endraw %} | sort | tail -n 1
}
```

{% comment %}
The 'raw', 'endraw' stuff is to work around Liquid templating.
{% endcomment %}

After some experiments, we settled on tagging images by the UTC timestamp of its build time in this format:
`20180923T082316Z`. This records the full year, month, day, hour, minute, second, in UTC.
With this naming convention, we are able to which image is the newest solely by its tag, ignoring its created/built/pushed-at timestamps.

(At first, we retrieved the tags from AWS ECS with the latest `pushedAt` timestamp, and found the largest tag among them. We tripped on a tricky corner case. At one point, we made a small change and pushed a new image.
Then reverted that change, re-built, re-tagged, and re-pushed. This resulted in the latest tag associated with the
second-to-latest `pushedAt` time. Then, by restricting our search to the latest `pushedAt` tags, we could not find the truely latest tag; the our code had the wrong idea about whether the AWS iamge is up-to-date.
Since then, we've switched to this universal-tag-search approach.)

Part of the motivation for this format is that Github returns a repo's commit time in this format.
Using the same format eases comparison (see below).

Below we'll see that the code constructs the tag in this format when building an image.

Next we look at the functions that deal with pulling/pushing images from/to AWS ECS.

```bash
# Pull from AWS and tag it.
function pull_aws {
    log_in_aws
    name="$1"
    version="$2"
    set -x
    docker pull "${ECR_URL}/my-org/${name}:${version}" >&2
    docker tag "${ECR_URL}/my-org/${name}:${version}" "${name}:${version}"
    set +x
}


# Find the latest version on AWS.
# Check if it's present locally.
# If not, pull it.
# Return the tag.
function pull_aws_latest {
    name="$1"
    version="$(find_aws_latest_tag ${name})"
    if [[ $(docker images "${name}:${version}" -q) == "" ]]; then
        echo Latest version of "my-org/${name}" on AWS is "${version}" >&2
        echo Could not find "${name}:${version}" locally\; pulling from AWS... >&2
        pull_aws "${name}" "${version}"
    fi
    echo "${version}"
}

function push_to_aws {
    name="$1"
    version="$2"
    log_in_aws

    tag=${ECR_URL}/my-org/${name}:${version}

    set -x
    docker tag ${name}:${version} ${tag}
    docker push ${tag} >&2
    set +x
}
```

Note that when we push a local image to AWS or pull an AWS image to local machine,
we make sure the local and AWS images have the same tag.

In the `build.sh` scripts, we call `build_dev` and `build_prod` to build development and production images, respectively. The difference between these functions is that the latter takes care of downloading the `develop` branch of code from Github, unpacking it in the current directory (the Docker build context). Subsequenctly,
the Dockerfile is responsible for copying that code into the image, building and installing as appropriate.

The image `py3dev` does not have a `development` vs `production` distinction. It is built by the function `build_simple`.

Below is the code.

```bash
function build_simple {
    build_dir="$1"
    PARENT_IMAGE="$2"
    NAME="$3"
    if [[ $# > 3 ]]; then
        push="$4"
    else
        push=""
    fi

    VERSION="$(date -u +%Y%m%dT%H%M%SZ)"
    # UTC. This command with these arguments work the same on Mac and Linux.
    # Version format is like this:
    #    20180923T081243Z
    # which indicates full datetime accurate to seconds in UTC.
    # This format is chosen to facilitate comparison with Github commit time;
    # see 'get_lastest_branch_commit_date' in 'auto-build.sh'.

    FULLNAME="${NAME}:${VERSION}"

    set -x
    # docker build --build-arg PARENT_IMAGE="${PARENT_IMAGE}" -t "${FULLNAME}" --no-cache "${build_dir}"
    # TODO: remove '--no-cache' once things work fine.
    docker build --build-arg PARENT_IMAGE="${PARENT_IMAGE}" -t "${FULLNAME}" "${build_dir}" >&2
    set +x
    echo

    if [[ "${push}" == "push" ]]; then
        push_to_aws $NAME $VERSION
    fi
}


function build_dev {
    build_dir="$1"
    parent_image="$2"
    image_name="$(basename ${build_dir})"
    if [[ "${image_name}" != *-dev ]]; then
        echo "'build_dev' called from directory '"${image_name}"'"
        return 1
    fi
    shift
    shift
    # optional 3rd argument is 'push'.
    build_simple ${build_dir} ${parent_image} ${image_name} $@
}


function build_prod {
    build_dir="$1"
    parent_image="$2"
    image_name="$(basename ${build_dir})"
    if [[ "${image_name}" != *-prod ]]; then
        echo "'build_prod' called from directory '"${image_name}"'"
        return 1
    fi
    shift
    shift
    # otional 3rd argument is 'push'

    repo="${image_name%-prod}"
    branch=develop

    URL=https://github.com/my-org/${repo}/archive/${branch}.zip
    GIT_TOKEN="Authorization: token ${GITHUB_ACCESS_TOKEN}"
    rm -rf "${build_dir}/src.zip" "${build_dir}/${repo}-develop" "${build_dir}/src"
    curl -skL --retry 3 -H "${GIT_TOKEN}" ${URL} -o "${build_dir}/src.zip"
    (cd "${build_dir}" && unzip src.zip && mv -f ${repo}-${branch} src)
    rm -f "${build_dir}/src.zip"

    build_simple "${build_dir}" "${parent_image}" "${image_name}" $@

    rm -rf "${build_dir}/src"
}
```

## Run Docker containers

Having built the images, let's develop a script to launch Docker containers based on the images,
while taking care of any setup about the container that can be automated. Some main concerns include:

- Set up environment variables that should be available to programs running inside the container.
- Set up volumn mapping between the host machine and the container. For example, a "development" container
  needs to have source code mapped into the container; for "production" containers, it's important
  to provide mapping for data, config, and logging directories.
- Check the version of the image on the local machine as well as in AWS ECS, and pull the latest from AWS
  as needed.
- Provide support for running pipelines in a production container, such as take care of logging file rotation.
- Cater to the need of particular commonly-used programs.

The script is named `run-docker`. The general pattern of its usage is like this:

```bash
run-docker [docker-options] image-name command [args]
```

The content of `run-docker` is as follows:

```bash
#!/usr/bin/env bash

thisfile="${BASH_SOURCE[0]}"
thisdir="$( cd "$( dirname "${thisfile}" )" && pwd )"
source "${thisdir}/common.sh"

# set -Eeuo pipefail
set -o errexit
set -o nounset
set -o pipefail

# For all the directory and file names touched by this script,
# space in the name is not supported.
# Do not use space in directory and file names in ${HOME}/work and under it.

USAGE=$(cat <<'EOF'
Usage:
   run-docker [--local] image-name [command [...] ]
   run-docker [--local] image-name pipeline py-script-name [...]
   run-docker [--local] image-name pyscript py-script-name [...]

where 

`image-name` is like 'py3dev', 'infra-dev', 'infra-prod', 'app1-dev', etc.

`command` is the command to be run within the container, followed by arguments to the command.
(Default: /bin/bash)

`py-script-name`: name of the Python script, including the extension '.py'.
This script must reside directly under the 'script' folder in the project's repository.

`...`: additional arguments for `command` or `py-script-name`.

If `--local` is present, use local image and do not try to pull the latest from AWS.
All Docker options appear before `image-name`; after `image-name` are command and it arguments.
EOF
)

imagename=""
uselocal="no"
command=/bin/bash
args=""
opts=""

# Parse arguments. After mandatory arguments are obtained,
# remaining arguments are stored, to be passed on.
while [[ $# > 0 ]]; do
    if [[ "${imagename}" == "" ]]; then
        if [[ "$1" == '--local' ]]; then
            uselocal="yes"
        else
            imagename="$1"
        fi
        shift
    else
        # After `image-name`.
        command="$1"
        shift
        args="$@"
        break
    fi
done

if [[ "${imagename}" == "" ]]; then
    echo "${USAGE}"
    exit 1
fi


# Check whether the latest version of the Docker image on AWS is available locally.
# If not, pull it from AWS and tag appropriately.
if [[ "${uselocal}" == "no" ]]; then
    imageversion=$(pull_aws_latest "${imagename}")
    echo using latest image "${imagename}:${imageversion}" in sync with AWS
else
    imageversion=$(find_latest_tag ${imagename})
    echo using latest local image "${imagename}:${imageversion}"
fi

projname="${imagename%%-*}"
# Delete longest match of "-*" from back of string.

variantname="${imagename#*-}"
# Delete shortest match of "*-" from front of string.
# Value is 'dev' or 'prod'.

if [[ "${command}" == "pipeline" ]]; then
    if [[ $# < 1 ]]; then
        echo "${USAGE}"
        exit 1
    fi
    scriptname="$1"
fi

if [[ $(uname) == Linux && $(id -u) != 1000 ]]; then
    # Linux box.
    uid=$(id -u)
    dockeruser=${uid}
    opts="${opts} -e USER=${dockeruser} -u ${dockeruser}:docker -v /etc/group:/etc/group:ro -v /etc/passwd:/etc/passwd:ro"
else
    dockeruser='docker-user'
    opts="${opts} -e USER=${dockeruser} -u ${dockeruser}"
fi
dockerhomedir='/home/docker-user'
dockerworkdir="${dockerhomedir}/work"
hostworkdir="${HOME}/work"
workdir="${dockerworkdir}"

if [[ "${variantname}" == "dev" ]]; then
    SRCDIR="src/${projname}"
    opts="${opts} -v ${hostworkdir}/${SRCDIR}:${dockerworkdir}/${SRCDIR}"
    opts="${opts} -e SRCDIR=${dockerworkdir}/${SRCDIR}"
    opts="${opts} -e PYTHONPATH=${dockerworkdir}/src/${projname}"
    opts="${opts} -e SCRIPTDIR=${dockerworkdir}/src/${projname}/scripts"
    workdir="${dockerworkdir}/${SRCDIR}"
else
    opts="${opts} -e SCRIPTDIR=/usr/local/bin/my-org/${projname}"
fi

if [[ "${command}" == "notebook" ]]; then
    opts="${opts} --expose=8888 -p 8888:8888"
    workdir="${dockerworkdir}/tmp"
    command="jupyter notebook --port=8888 --no-browser --ip=0.0.0.0 --NotebookApp.notebook_dir='${workdir}' --NotebookApp.token=''"
elif [[ "${command}" == "py.test" ]]; then
    args="-p no:cacheprovider ${args}"
elif [[ "${command}" == "pipeline" ]]; then
    :  # do nothing, but stay away from '-it'
else
    opts="${opts} -it"
fi

opts="${opts}
-e HOME=${dockerhomedir}
--workdir ${workdir}
-e IMAGE_NAME=${imagename}
-e IMAGE_VERSION=${imageversion}
-e TZ=America/Los_Angeles
--rm --init"

LOGDIR=log/"${imagename}"
if [[ "${command}" == "pipeline" ]]; then
    LOGDIR="${LOGDIR}/${scriptname}"
fi
mkdir -p "${hostworkdir}/${LOGDIR}"
opts="${opts} -v ${hostworkdir}/${LOGDIR}:${dockerworkdir}/${LOGDIR}"
opts="${opts} -e LOGDIR=${dockerworkdir}/${LOGDIR}"

DATADIR="data/${imagename}"
mkdir -p "${hostworkdir}/${DATADIR}"
opts="${opts} -v ${hostworkdir}/${DATADIR}:${dockerworkdir}/${DATADIR}"
opts="${opts} -e DATADIR=${dockerworkdir}/${DATADIR}"

CFGDIR="config/${imagename}"
mkdir -p "${hostworkdir}/${CFGDIR}"
opts="${opts} -v ${hostworkdir}/${CFGDIR}:${dockerworkdir}/${CFGDIR}"
opts="${opts} -e CFGDIR=${dockerworkdir}/${CFGDIR}"

TMPDIR="tmp"
mkdir -p "${hostworkdir}/${TMPDIR}"
opts="${opts} -v ${hostworkdir}/${TMPDIR}:${dockerworkdir}/${TMPDIR}"
opts="${opts} -e TMPDIR=${dockerworkdir}/${TMPDIR}"

#set -x
docker run ${opts} ${imagename}:${imageversion} ${command} ${args}
```

There are quite a few things going on here, and I'm not going to wark through it.
(It is somewhat simplified from the actual version I use, but all the patterns are here.)
It will be helpful to know that I assume the project directory structure on the development machine
follows the 
[recommendations here]({{ site.baseurl }}/blog/python-project-tips/#directory-structure).

This script gives special attention to two commands, namely `pipeline` and `pyscript`.
They are installed in the image `py3dev` as shown in `py3dev/build.sh`.
I'll talk about them next.


## The special commands `pipeline` and `pyscript`
<a name="pipeline-and-pyscript"></a>

Suppose there is a script `app1/scripts/do-something.py`. `pipeline` is a command installed into
`/usr/local/bin` in image `py3dev`. Its intended use is like this:

```bash
run-docker app1-prod pipeline do-something.py [...args...]
```

Typically this is launched as a cron job. Note that `pipeline` is a command residing inside the container,
on the standard command path, hence this is Docker's standard way of launching a command inside the container
without first landing on an interactive console in the container. How does `pipeline` find `do-something.py`?
Well, it assumes the script is in the directory pointed to by `$SCRIPTDIR`, which is set up by `run-docker`.
The build spec `app1-prod/Dockerfile` make sure to copy (or "install") that script into the correct location.
In `run-docker`, notice that `$SCRIPTDIR` points to different locations depending on whether the image is
`*-dev` or `*-prod`.

`pipeline` handles logging and adds some contextual info
to the log (things like "xxx is starting with arguments ... at 2018-09-28 01:02:03 PST ...").

In contrast, `pyscript` is intended for one-off use. It does not handle logging; any printout of the program
appears in the console directly. `pyscript` also finds `do-something.py` via `$SCRIPTDIR`.
It is used in the same way:

```bash
run-docker app1-prod pyscript do-something.py [...args...]
```

Below is the script `pipeline`:

```bash
#!/usr/bin/env bash

set -o errexit
set -o nounset
set -o pipefail

USAGE=$(cat <<'EOF'
This script resides within a Docker image, and calls a Python program
in `${SCRIPTDIR}`.

Usage:
    pipeline task [args]

`task` indicates the Python script that is to be run.
For example, if `task` is `abc.py`, then `${SCRIPTDIR}/abc.py` will be run.

`args` are optional arguments to be passed on to the Python script.
EOF
)

if [[ $# < 1 ]]; then
    echo "${USAGE}"
    exit 1
fi

if [[ $# < 1 ]]; then
    echo "${USAGE}"
    exit 1
fi

task="$1"
shift

: "${SCRIPTDIR:?Environment variable \'SCRIPTDIR\' is not set}"
: "${LOGDIR:?Environment variable \'LOGDIR\' is not set}"
: "${DATADIR:?Environment variable \'DATADIR\' is not set}"

: "${IMAGE_NAME:?}"
: "${IMAGE_VERSION:?}"


PYSCRIPT="${SCRIPTDIR}"/${task}

echo | multilog s1000000 n30 "${LOGDIR}"
echo | multilog s1000000 n30 "${LOGDIR}"
echo "========================================" | multilog s1000000 n30 "${LOGDIR}"
date --utc +'%Y-%m-%d %H:%M:%S UTC' | multilog s1000000 n30 "${LOGDIR}"
echo in Docker image "${IMAGE_NAME}:${IMAGE_VERSION}" | multilog s1000000 n30 "${LOGDIR}"
echo "$(env)" | multilog s1000000 n30 "${LOGDIR}"
echo | multilog s1000000 n30 "${LOGDIR}"
echo starting task \`${task}\` | multilog s1000000 n30 "${LOGDIR}"
if [[ "${task}" != *.py ]]; then
    echo Unrecognized script "'${task}'" --- did you forget the extension? | multilog s1000000 n30 "${LOGDIR}"
    echo Aborting... | multilog s1000000 n30 "${LOGDIR}"
    exit 1
fi
echo python -u ${PYSCRIPT} $@ | multilog s1000000 n30 "${LOGDIR}"
echo "----------------------------------------" | multilog s1000000 n30 "${LOGDIR}"
echo | multilog s1000000 n30 "${LOGDIR}"

python -u ${PYSCRIPT} $@ 2>&1 | multilog s1000000 n30 "${LOGDIR}"

echo | multilog s1000000 n30 "${LOGDIR}"
echo "----------------------------------------" | multilog s1000000 n30 "${LOGDIR}"
date --utc +'%Y-%m-%d %H:%M:%S UTC' | multilog s1000000 n30 "${LOGDIR}"
echo task \`${task}\` finished | multilog s1000000 n30 "${LOGDIR}"
echo "========================================" | multilog s1000000 n30 "${LOGDIR}"
```

The log rotation tool `multilog` (see 
[Simple Rotating Log Capture]({{ site.baseurl }}/blog/log-rotation-of-stdout/))
is installed in `py3dev/Dockerfile`.

The script `pyscript` is slightly simpler since it does not worry about logging:

```bash
#!/usr/bin/env bash

set -o errexit
set -o nounset
set -o pipefail

USAGE=$(cat <<'EOF'
This script resides within a Docker image, and calls a Python program
in `${SCRIPTDIR}`.

Usage:
    py-script task [args]

`task` indicates the Python script that is to be run.
For example, if `task` is `abc.py`, then `${SCRIPTDIR}/abc.py` will be run.

`args` are optional arguments to be passed on to the Python script.
EOF
)

if [[ $# < 1 ]]; then
    echo "${USAGE}"
    exit 1
fi


task="$1"
shift

: "${SCRIPTDIR:?Environment variable \'SCRIPTDIR\' is not set}"


PYSCRIPT="${SCRIPTDIR}"/${task}

if [[ "${task}" != *.py ]]; then
    echo Unrecognized script "'${task}'" --- did you forget the extension?
    echo Aborting...
    exit 1
fi

python -u ${PYSCRIPT} $@
```

Below is the segment of `py3dev/Dockerfile` that installs `pipeline` and `pyscript`:

```bash
COPY pipeline /usr/local/bin/
RUN chmod +x /usr/local/bin/pipeline
COPY pyscript /usr/local/bin/
RUN chmod +x /usr/local/bin/pyscript
```

In `app1-prod/Dockerfile`, the scripts under `app1/scripts/` are copied into `/usr/local/bin/my-org/app1`,
as is hinted at in `run-docker`.


## Auto-build the images

Now how to build and use the images is behind us, we can finally turn to **auto** build the images. The main points of the idea are

1. Building images need to use the latest code of the repo `my-docker`. A simple way to do this in a cron job
   is by downloading the latest code everytime the cron job runs.
2. Provide a script in `my-docker` that does version checking, image building, and all that.
3. Provide a more stable, simple script that handles downloading the code of `my-docker`. This script is what is called by cron.

The downloader is simple as the following:

```bash
#!/usr/bin/env bash

thisfile="${BASH_SOURCE[0]}"
thisdir="$( cd $( dirname ${thisfile} ) && pwd )"

export GITHUB_ACCESS_TOKEN="fill this out in deployed copy"
: "${GITHUB_ACCESS_TOKEN:?}"

# Also make sure other environment variables required by `common.sh` are defined,
# possibly by 'sourcing' in a file containing definition of the variables.

echo
echo ======================================================
echo $(date)
echo fetching the latest of \'my-docker:master\'...
echo

URL=https://github.com/my-org/my-docker/archive/master.zip
GIT_TOKEN="Authorization: token ${GITHUB_ACCESS_TOKEN}"
cd /tmp
rm -f my-docker.zip
curl -skL --retry 3 -H "${GIT_TOKEN}" ${URL} -o my-docker.zip
rm -rf my-docker-master
unzip my-docker.zip

echo
echo ----------------------------------------
echo finished fetching \'my-docker:master\'
echo starting to build images...
echo

cd my-docker-master
bash auto-build.sh

echo
echo finished building images.
echo $(date)
echo -----------------------------------------
```

The heavy-lifter `auto-build.sh` is listed below.

```bash
set -Eeuo pipefail

thisfile="${BASH_SOURCE[0]}"
thisdir="$( cd $( dirname ${thisfile} ) && pwd )"
source "${thisdir}/common.sh"

: "${GITHUB_ACCESS_TOKEN:?}"

GIT_TOKEN="Authorization: token ${GITHUB_ACCESS_TOKEN}"


function find_latest_branch_commit {
    repo="$1"
    branch="$2"
    url="https://api.github.com/repos/my-org/${repo}/git/refs/heads/${branch}"
    z=$(curl -s -H "${GIT_TOKEN}" ${url})
    sha=$(grep '"sha": ' <<< "$z")
    sha=${sha##*\"sha\": \"}
    sha=${sha%\",}
    echo "${sha}"
}


function get_commit {
    repo="$1"
    sha="$2"
    url="https://api.github.com/repos/my-org/${repo}/git/commits/${sha}"
    z=$(curl -s -H "${GIT_TOKEN}" ${url})
    echo "${z}"
}


function get_latest_branch_commit {
    repo="$1"
    branch="$2"
    get_commit "${repo}" "$(find_latest_branch_commit $repo $branch)"
}


function get_latest_branch_commit_date {
    z="$(get_latest_branch_commit $@)"
    z=$(grep "\"date\": \"" <<< "$z" | head -1)
    z="${z#*\"date\": \"}"
    z="${z%\"}"    # string formatted like '2018-09-04T23:12:50Z'
    z="${z//-/}"
    z="${z//:/}"   # string formatted like '20180904T231250Z'
    echo "$z"
}


function get_latest_aws_tag_date {
    z=$(find_aws_latest_tag "$1")    # tag is formmated like '20180904T231250Z'
    echo "$z"
}


function main {
    if [[ "$(get_latest_branch_commit_date my-docker master)" > "$(get_latest_aws_tag_date py3dev)" ]]; then
        echo
        echo "'my-docker:master' has updated; re-building everything..."
        echo
        bash "${thisdir}/build.sh"
    else
        if [[ "$(get_latest_branch_commit_date infra develop)" > "$(get_latest_aws_tag_date infra-prod)" ]]; then
            echo
            echo "'infra:develop' has updated; re-building most images..."
            echo

            (cd "${thisdir}"/infra-prod && bash build.sh push)

            repos=( app1 )
            for repo in "${repos[@]}"; do
                (cd "${thisdir}"/${repo}-dev && bash build.sh push)
                (cd "${thisdir}"/${repo}-prod && bash build.sh push)
            done
        else
            repos=( app1 )
            for repo in "${repos[@]}"; do
                if [[ "$(get_latest_branch_commit_date ${repo} develop)" > "$(get_latest_aws_tag_date ${repo}-prod)" ]]; then
                    echo
                    echo "'${repo}:develop' has updated; re-building '${repo}-prod'"
                    echo
                    (cd "${thisdir}/${repo}" && bash build.sh push)
                fi
            done
        fi
    fi
    return 0
}

main
```
