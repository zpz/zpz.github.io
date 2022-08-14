---
layout: post
title: "Simple Rotating Log Capture"
tags: [logging]
---

This post concerns simple text-dump logs, not "data logs" that are sent to, say, Kafka for structured treatment.

I agree with [the 12-factor app](https://12factor.net/logs) that programs
should simply print logs to the standard output, and should not worry about 
what file to save the logs to, let alone file size capping and rotation, etc.
Capture, storage, and consumption of the log are concerns of the execution environment.

After some research, I've decided to use a program called [`multilog`](http://cr.yp.to/daemontools/multilog.html),
which is part of the [`deamontools`](http://cr.yp.to/daemontools.html) suit of utilities
for UNIX services.

Suppose our program `myapp` logs to `stdout` or `stderr` periodically, simply do

```sh
myapp 2>&1 | multilog s1000000 n10 /path-to-log/myapp
```

This will save logs in the directory `/path-to-log/myapp`.
The currently active log file is named `current`.
Each log file is size-capped at 1000000 bytes (via the argument `s1000000`), i.e. 1 MB.
Older log files are named after the timestamp of the start (or maybe end?)
of the content of the file.
At most 10 (most recent) log files, including `current`, are kept in the directory,
and older ones are discarded; this is controlled by the argument `n10`.


If we want to observe the terminal printout in the meantime,
we can do

```sh
myapp 2>&1 | tee >(multilog s1000000 n10 /path-to-log/myapp)
```

There are a number of ways to customize the behavior of `multilog`.
For example, it can selectively capture lines by pattern matching.
It can also insert a timestamp at the beginning of each line if you so wish.

Some useful resources:

- [StackExchange question](https://superuser.com/questions/291368/log-rotation-of-stdout)
- [StackExchange Q/A](https://unix.stackexchange.com/questions/326127/how-do-i-append-prepend-a-timestamp-to-grep-output/326166#326166)
- [logging](http://jdebp.eu/FGA/daemontools-family.html#Logging)

Several alternatives appear to be similar and equally adequate,
including

- [`cyclog`](http://jdebp.eu/Softwares/nosh/guide/cyclog.html) in [`nosh`](http://jdebp.eu/Softwares/nosh/)
- [`tinylog`](http://b0llix.net/perp/site.cgi?page=tinylog.8) in [`perp`](http://b0llix.net/perp/)
- [`s6-log`](http://skarnet.org/software/s6/s6-log.html) in [`s6`](http://skarnet.org/software/s6/index.html)
- [`svlogd`](http://smarden.org/runit/svlogd.8.html) in [`runit`](http://smarden.org/runit/index.html)
- [`multilog`](http://untroubled.org/daemontools-encore/multilog.8.html) in [`daemontools-encore`](http://untroubled.org/daemontools-encore/)

I chose the `multilog` in `daemontools` because this is a package (called `daemontools`) that is
`apt-get` installable on `debian`.

If your program is in Python, I had some 
[tips on Python logging here]({{ site.baseurl }}/blog/python-project-tips/#logging)
that will work well with such `stdout` capturing.

If you want to observe the terminal printout as well as capture it in files, you may be tempted to use

```sh
python myapp.py 2>&1 | tee >(multilog s1000000 n10 /path-to-log/myapp)
```

To your surprise, you won't see printouts. Instead, you need to use

```sh
python -u myapp.py 2>&1 | tee >(multilog s1000000 n10 /path-to-log/myapp)
```

This has to do with [buffering](https://stackoverflow.com/questions/21662783/linux-tee-is-not-working-with-python).



