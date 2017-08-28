---
layout: post
title: "How to Connect to Localhost from within a Docker Container"
---

I have Docker container A running a server, and container B running a client.
At least for testing, both containers run on the same machine (host).
The client software needs to reach out of its own container and then into the server container.
The various inter-container connection mechanisms are not usable, because I don't want to assume that the server is running in a container.

In fact, the "into the server container" part is not a problem, as that is easily taken care of by `port mapping`, i.e. the option `-p` of `docker run`. In other words, if the server is runing natively on the host machine, the issue is about the same, namely, how to connect to the host machine from within a Docker container.

Problem is, the client software can not use `localhost` or `127.0.0.1`; that will loop back into the container itself.

After some research, I figured out one solution. It turned out pretty simple. First, give the host machine's loopback interface an alias IP address (different from `127.0.0.1`). The client software in container B can reach the host machine by connecting to this alias IP address directly. Since this IP may be hard to remember, `docker run` has an option for giving it an alias.


Step 1
====

If the host OS is Mac, do this:

```
sudo ifconfig lo0 alias 10.254.254.254
```

If Linux, do this:

```
sudo ifconfig lo:0 10.254.254.254
```

I don't know what's good about `10.254.254.254`; it comes from [this post](http://xplus3.net/2016/09/19/reaching-localhost-from-a-docker-container). Other addresses are also possible.

These settings can not survive system reboot. I will update this post once I figure out
how to persist them.

Step 2
====

Use these options in the `docker run` command that launches container B:

```
docker run --add-host=local_host:10.254.254.254 --add-host=local:10.254.254.254 blah blah
```

Then, within container B, the host machine can be reached by connecting to `local_host`, `local`, or `10.254.254.254` directly.

I also tried `--add-host=localhost:10.254.254.254`, and using `localhost` from within container B worked well. But there might be caveats, as `localhost` is, I guess, used by various programs in the system.

