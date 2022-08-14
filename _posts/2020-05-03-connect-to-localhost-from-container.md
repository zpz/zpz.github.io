---
layout: post
title: "How to Connect to Localhost from within a Docker Container"
excerpt_separator: <!--excerpt-->
tags: [Docker]
---

I have Docker container A running a server, and container B running a client.
At least for testing, both containers run on the same machine (host).
The client software needs to reach out of its own container and then into the server container.
<!--excerpt-->
The various inter-container connection mechanisms are not usable, because I don't want to *assume* that the server is running in a container, although this is the case in the situation described above. 

In fact, the "into the server container" part is not a problem, as that is easily taken care of by `port mapping`, i.e. the option `-p` of `docker run`. In other words, if the server is running natively on the host machine, the issue is about the same. In still other words,
the problem is in connecting to the host machine from within a Docker container.

Unfortunately, the client software in B can not use `localhost` or `127.0.0.1`; that will loop back into the container itself.

After some research, I figured out one solution. It turned out pretty simple. First, give the host machine's loopback interface an alias IP address (different from `127.0.0.1`). The client software in container B can reach the host machine by connecting to this alias IP address directly. Since this IP may be hard to remember, `docker run` has an option for giving it an alias.


## Step 1

If the host OS is Mac, do this:

```sh
sudo ifconfig lo0 alias 10.254.254.254
```

If Linux, do this:

```sh
sudo ifconfig lo:0 10.254.254.254
```

After this, you can check the effect by

```sh
ifconfig lo:0
```

which will print

```sh
lo:0      Link encap:Local Loopback
          inet addr:10.254.254.254  Mask:255.0.0.0
          UP LOOPBACK RUNNING  MTU:65536  Metric:1
```

whereas before setting the alias, the printout would be

```sh
lo:0      Link encap:Local Loopback
          UP LOOPBACK RUNNING  MTU:65536  Metric:1
```

(To remove the alias, do `sudo ifconfig lo:0 down`.)

I don't know what's good about `10.254.254.254`; it comes from [this post](http://xplus3.net/2016/09/19/reaching-localhost-from-a-docker-container). Other addresses are also possible.

You will want these settings to survive system reboot, i.e. want these settings to be run at system startup.
To do that, put the following block (with blank lines before and after) in the file `/etc/network/interfaces`:

```sh
auto lo:0
allow-hotplug lo:0
iface lo:0 inet static
    address 10.254.254.254
    netmask 255.255.255.0
```

## Step 2

Use these options in the `docker run` command that launches container B:

```sh
docker run --add-host=local_host:10.254.254.254 --add-host=local:10.254.254.254 blah blah
```

Then, within container B, the host machine can be reached by connecting to `local_host`, `local`, or `10.254.254.254` directly.

I also tried `--add-host=localhost:10.254.254.254`, and using `localhost` from within container B worked well. But there might be caveats, as `localhost` is, I guess, used by various programs in the system.

## Update

I have been using a much simpler alternative for quite a while.

I don't remember the theory, but what you need to do is type this command **within* the Docker container (such as B),

```bash
$ ip route show default | awk '/default/ {print $3}'
```

I'm running a Linux Mint 19.03 host machine. This command with the Docker container gave me `172.17.0.1`.

Alternatively, the following should print the same result:

```bash
$ ip -r route list match 0/0 | cut -d' ' -f3
```

If the command `ip` does not exist in your container,
install the Linux package `iproute2`.

Now in container B, use this IP `172.17.0.1` to reach the host machine
and container A. If the service is running on the host machine directly
(no Docker involved), the usage is the same.

If the service is running on a remote machine (in container or not), then get the IP address of the remote machine and use it the same way.

This method does not need anything to be done to the host machine.

For convenience in Python programs, I also created this function:

```python
import subprocess

def get_docker_host_ip():
    z = subprocess.check_output(['ip', '-4', 'route', 'list', 'match', '0/0'])
    z = z.decode()[len('default via ') :]
    return z[: z.find(' ')]
```

(Originally written July 12, 2017; added the simpler alternative in May 2020.)
