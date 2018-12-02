---
layout: post
title: "Speeding up Python with Cython---Sequel 1: cffi"
excerpt_separator: <!--excerpt-->
tags: [python]
---

In [a post published in April, 2017]({{ site.baseurl }}/speeding-up-python-with-cython-appetizer/), I demonstrated using `Cython` to achieve 100+ times speed-up to numerical bottlenecks in Python code. I have not used Cython in actual projects since then. Instead, I have had some experiences using `C++` to extend Python. Compared with Cython, which is deeply intertwined with Python, using a separate language to write Python extensions has advantages. <!--excerpt-->
In my opinion, there are three major advantages:

    1. There is no need to learn Cython, which is essentially a different language than Python, yet has no value in non-Python projects. On the other hand, if we use a totally separate language to write Python extensions, the language has its own value in projects unrelated to Python.
    
    2. Performance tuning tends to be cleaner than Cython. Because Cython is a superset of Python, one would tend to *gradually* add type declarations and other tweaks to the Cython code. While super convenient, it also tends to take more time eventually.

    3. If the extension code is substantial, one can maintain it as a separate library independent of Python, thus having its use life and broader applications.

In this post, I will demonstrate using `C` to write Python extensions.
I'm going to use the same example as the Cython article, which is a simple, numerical problem. The C/Python inter-op tool used will be 
[`cffi`](https://cffi.readthedocs.io/en/latest/#).
This is not a `cffi` tutorial.
The objectives of writing this note are two fold.
First, figure out the basics of this workflow to make it work.
Second, find out whether it can achieve performance (for this specific problem) that is comparable to Cython.

# Set it up

The problem is to calculate the day of week given a UNIX timestamp.
I'll copy some code from the Cython post.

```python
# Module `pyx.datex.version01`.

from datetime import datetime


def weekday(ts):
    dt = datetime.utcfromtimestamp(ts)
    wd = dt.isoweekday()
    return wd


def weekdays(ts):
    return [weekday(t) for t in ts] 
```

```python
# Module `pyx.datex.version03`.

def weekday(ts):
    ts0 = 1489363200   # 2017-03-13 0:0:0 UTC, Monday
    weekday0 = 1   # ISO weekday: Monday is 1, Sunday is 7

    DAY_SECONDS = 86400
    WEEK_SECONDS = 604800

    ts_delta = ts - ts0
    if ts_delta < 0:
        ts_delta += ((-ts_delta) // WEEK_SECONDS + 1) * WEEK_SECONDS

    td = ts_delta % WEEK_SECONDS
    nday = td // DAY_SECONDS
    weekday = weekday0 + nday
    if weekday > 7:
        weekday = weekday - 7
    return weekday


def weekdays(ts):
    return [weekday(t) for t in ts]
```

# The bottleneck in C

```c
/* File `c_version01.c` in `src/c/datex/`. */

long weekday(long ts) 
{
    long ts0, weekday0, DAY_SECONDS, WEEK_SECONDS;
    long ts_delta, td, nday, weekday;

    ts0 = 1489363200;   // 2017-03-13 0:0:0 UTC, Monday
    weekday0 = 1;       // ISO weekday: Monday is 1, Sunday is 7

    DAY_SECONDS = 86400;
    WEEK_SECONDS = 604800;

    ts_delta = ts - ts0;
    if (ts_delta < 0) {
        ts_delta += ((-ts_delta) / WEEK_SECONDS + 1) * WEEK_SECONDS;
    }

    td = ts_delta % WEEK_SECONDS;
    nday = td / DAY_SECONDS;
    weekday = weekday0 + nday;
    if (weekday > 7) {
        weekday = weekday - 7;
    }
    return weekday;
}


void weekdays(long const * ts, long * out, long n)
{
    for (long i = 0; i < n; i++) {
        out[i] = weekday(ts[i]);
    }
}
```

```c
/* File `c_version01.h` in `src/c/datex/`. */

long weekday(long ts);

void weekdays(long const * ts, long * out, long n);
```

# Enter `cffi`

```python
# Module `pyx.datex.c._version01_build.py`.

from cffi import FFI
ffibuilder = FFI()


# ffibuilder.cdef('''
#     long weekday(long ts);
#     void weekdays(long const * ts, long * out, long n);
#     ''')
ffibuilder.cdef(open('src/c/datex/c_version01.h').read())

ffibuilder.set_source(
    "datex.c._version01",
    '',
    sources=['src/c/datex/c_version01.c'],
    include_dirs=['src/c/datex'],
)

# The paths in the code above are relative to the location
# of `setup.py`, which calls this script.
```

```python
# Module `pyx.datex.c.version01.py`.

from cffi import FFI
import numpy as np

from ._version01 import lib

weekday = lib.weekday

ffi = FFI()


def weekdays(ts):
    n = len(ts)
    out = np.zeros(n, dtype=np.int64)
    p_in = ffi.cast("long *", ffi.from_buffer(ts))
    p_out = ffi.cast("long *", ffi.from_buffer(out))
    lib.weekdays(p_in, p_out, n)
    return out

# _ = weekdays(np.array([1234]))
```