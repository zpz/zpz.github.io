---
layout: post
title: "Speeding up Python Numerics: an Entr&eacute;e with C and cffi"
excerpt_separator: <!--excerpt-->
tags: [python]
---

In [an appetizer served back in April, 2017]({{ site.baseurl }}/blog/speeding-up-python-with-cython/), I demonstrated using `Cython` to achieve 100+ times speed-up to numerical bottlenecks in Python code. I have not used Cython in actual projects since then. Instead, I have had some experiences using `C++` to extend Python. Compared with Cython, which is deeply intertwined with Python, using a separate language to write Python extensions has advantages, such as <!--excerpt-->

1. There is no need to learn Cython, which is nevertheless a different language than Python, yet has no value outside of Python projects. On the other hand, a totally separate language
(if suitable for writing Python extensions) has its own value in non-Python projects.
Either we already know the language, or we learn it for its own merits.
The reward of the skill investment may be broader than what can be gained with Cython.

2. Performance tuning tends to be cleaner than Cython. Because Cython is a superset of Python, one would tend to *gradually* add type declarations and other tweaks to the Cython code based on performance observations and profiling. While super convenient, eventually it also tends to take more time.
With a separate, statically-typed language, we start clean and complete with respect to *language*-wise benefits.

3. If the extension code is substantial, we can maintain it as a separate library independent of Python, thus having its own life and broader applications. This is not the case with extensions written in Cython, which, as far as I know, can not be used outside of Python.

In this post, I will demonstrate using `C` to write Python extensions.
I'm going to use the same example that was used in the previous article.
The C/Python inter-op tool used will be 
[`cffi`](https://cffi.readthedocs.io/en/latest/#).
The objectives of this blogpost are two fold.
First, figure out the basics of this workflow to make it work for the simple example.
Second, find out whether it achieves performance that is comparable to the Cython solution.
My skill at `cffi` at this time stops at making this example work,
therefore this is not an intro or tutorial about `cffi`.

## Set it up

The problem is to calculate the day of week given a UNIX timestamp.
The Python code is simple.

```python
# File `src/python/datex/version01.py`.

from datetime import datetime


def weekday(ts):
    dt = datetime.utcfromtimestamp(ts)
    wd = dt.isoweekday()
    return wd


def weekdays(ts):
    return [weekday(t) for t in ts] 
```

In an attempt to speed things up,
I expanded the logic of the "weekday" problem to the point that it now is just a numerical calculation that bares little sign of a *datetime* problem.


```python
# File `src/python/datex/version03.py`.

def weekday(ts):
    ts0 = 1489363200   # 2017-03-13 0:0:0 UTC, Monday
    weekday0 = 1       # ISO weekday: Monday is 1, Sunday is 7

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

This version was migrated to Cython, mainly by adding type info:

```python
# File `src/python_ext/datex/cy/_version09.pyx`.

import numpy as np
cimport numpy as np
cimport cython


@cython.cdivision(True)
cdef long weekday(long ts):
    cdef long ts0, weekday0, DAY_SECONDS, WEEK_SECONDS
    cdef long ts_delta, td, nday, weekday

    ts0 = 1489363200   # 2017-03-13 0:0:0 UTC, Monday
    weekday0 = 1       # ISO weekday: Monday is 1, Sunday is 7

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


def weekdays(long[:] ts):
    cdef long n = len(ts)
    cdef np.ndarray[np.int64_t, ndim=1] out = np.empty(n, np.int64)
    cdef long[:] oout = memoryview(out)
    cdef long i, t, z
    for i in range(n):
        t = ts[i]
        z = weekday(t)
        oout[i] = z
    return out
```

At this point, the files are laid out like this:

```
├── setup.py
├── src
│   ├── python
│   │   ├── datex
│   │   │   ├── cy
│   │   │   │   ├── __init__.py
│   │   │   ├── __init__.py
│   │   │   ├── version01.py
│   │   │   └── version03.py
│   ├── python_ext
│   │   └── datex
│   │       ├── cy
│   │       │   └── _version09.pyx
└── tests
    ├── datex
    │   ├── __init__.py
    │   └── test_1.py
```

Here is the content of `setup.py`:

```
# File `setup.py`.

from setuptools import setup, Extension, find_packages
from Cython.Build import cythonize
import numpy


debug = False

numpy_include_dir = numpy.get_include()

cy_options = {
    'annotate': debug,
    'compiler_directives': {
        'profile': debug,
        'linetrace': debug,
        'wraparound': debug,
        'boundscheck': debug,
        'initializedcheck': debug,
        'language_level': 3,
    },
}

cy_extensions = cythonize([
    Extension(
        'datex.cy._version09', 
        sources=['src/python_ext/datex/cy/_version09.pyx'],
        include_dirs=[numpy_include_dir,],
        define_macros=[('CYTHON_TRACE', '1' if debug else '0')],
        extra_compile_args=['-O3', '-Wall'],
        ),
    ],
    **cy_options
    )

setup(
    name='datex',
    version='0.1.0',
    package_dir={'': 'src/python'},
    packages=find_packages(where='src/python'),
    ext_modules=cy_extensions,
)
```

Type the following to build and install the package:

```shell
$ pip install --user .
```

## Performance baseline

Below is a script to benchmark the code:

```python
# File `test_1.py`.

from functools import partial
import numpy

from zpz.profile import Timer

from datex import version01, version03
from datex import cy


def time_it(fn, timestamps, repeat=1):
    tt = Timer().start()
    for _ in range(repeat):
        z = fn(timestamps)
    t = tt.stop().seconds
    name = fn.__module__ + '.' + fn.__name__
    print('{: <42}:  {: >8.4f} seconds'.format(name, t))


def do_all(fn, n):
    timestamps_np = numpy.random.randint(
        10000000, 9999999999, size=n, dtype=numpy.int64)

    functions = [
        (version01.weekdays, timestamps_np),
        (version03.weekdays, timestamps_np),
        (cy.version09.weekdays, memoryview(timestamps_np)),
    ]

    for f, ts in functions:
        fn(f, ts)


def benchmark(n, repeat):
    do_all(partial(time_it, repeat=repeat), n)
    

if __name__ == "__main__":
    from argparse import ArgumentParser
    p = ArgumentParser()
    p.add_argument('--n', type=int, default=1000000)
    p.add_argument('--repeat', type=int, default=1)
    args = p.parse_args()

    benchmark(args.n, args.repeat)
```

I have omitted code that verifies correctness.
The package `zpz` contains some utilities and can be
[found in my Github account](https://github.com/zpz/utilities.py).

Let's run it to get a performance baseline:

```
$ python test_1.py --n 10000000
datex.version01.weekdays              :   5.1472 seconds
datex.version03.weekdays              :  17.2375 seconds
datex.cy._version09.weekdays          :   0.0563 seconds
```

I don't know why 'version03' is three times slower than 'version01,
but that's not a concern in this post.


## Implement the numerical bottleneck in C

The numerical part of the Python code is migrated to C with little change:

```c
/* File `src/c/datex/c_version01.c`. */

long weekday(long ts) 
{
    long ts0, weekday0, DAY_SECONDS, WEEK_SECONDS;
    long ts_delta, td, nday, weekday;

    ts0 = 1489363200;   /* 2017-03-13 0:0:0 UTC, Monday */
    weekday0 = 1;       /* ISO weekday: Monday is 1, Sunday is 7 */

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

Here is a header file:

```c
/* File `src/c/datex/c_version01.h`. */

long weekday(long ts);

void weekdays(long const * ts, long * out, long n);
```

I have intentionally placed these two files in a folder that is unrelated to the Python package,
as if they are an independent library in C.


## Enter `cffi`

There are two parts of work in using `cffi` to develop Python extensions in C.
The first part is using `cffi` to build C code into a shared library
(a '*.so' file in Linux), while exposing specified functions.
The second part is using the exposed functions via the `cffi` package; this part is regular Python code.

Let's see the first part, which is a simple Python script:

```python
# File `src/python_ext/datex/c/_version01_build.py`.

from cffi import FFI


ffibuilder = FFI()

ffibuilder.cdef(open('src/c/datex/c_version01.h').read())

ffibuilder.set_source(
    "datex.c._version01",
    '',  # source code
    sources=['src/c/datex/c_version01.c'],
    include_dirs=['src/c/datex'],
)

# The paths in the code above are relative to the location
# of `setup.py`, which calls this script.
```

This file is not a module in the package `datex`.
It contains instructions for building the shared library, and after that it's mission is finished.

Let's walk through this script.
It creates an object of class `cffi.FFI`, then calls the methods
`cdef` and `set_source` on this object.
`FFI.cdef` specifies the signature of functions that are to be exposed to Python.
One often chooses to directly write the signatures in this call.
In our case, the header file of the C library contains exactly what we need,
so we call `FFI.cdef` with the content of the entire header file.

`FFI.set_source` specifies the source code to be compiled into a shared library.
The first argument specifies the Python module's name for the shared library.
The resultant shared library will be known as module `datex.c._version01`.
The second argument is source code we have written to bridge Python and the external C library.
In the current case, we totally rely on the external library, and no bridging code is needed.
The other arguments specify source and header files needed to build the shared library.

This script is most conveniently processed by `setup.py`.

At this point, the files are laid out like this:

```
├── setup.py
├── src
│   ├── c
│   │   └── datex
│   │       ├── c_version01.c
│   │       └── c_version01.h
│   ├── python
│   │   ├── datex
│   │   │   ├── c
│   │   │   │   ├── __init__.py
│   │   │   │   └── version01.py
│   │   │   ├── cy
│   │   │   ├── __init__.py
│   │   │   ├── version01.py
│   │   │   └── version03.py
│   ├── python_ext
│   │   └── datex
│   │       ├── c
│   │       │   └── _version01_build.py
│   │       ├── cy
└── tests
    ├── datex
    │   ├── __init__.py
    │   └── test_1.py
```

The file `setup.py` has been augmented to build the C extension:

```
# File `setup.py`.

from setuptools import setup, Extension, find_packages
from Cython.Build import cythonize
import numpy
from cffi import FFI

cy_extensions = ...

cffi_extensions = [
    'src/python_ext/datex/c/_version01_build.py:ffibuilder',
    ]

setup(
    name='datex',
    version='0.1.0',
    package_dir={'': 'src/python'},
    packages=find_packages(where='src/python'),
    ext_modules=cy_extensions,
    cffi_modules=cffi_extensions,
)
```

Again, build and install the package `datex` by

```
$ pip install --user .
```

Now the module `datex.c._version01` is ready for use,
and it provides two functions: `weekday` and `weekdays`.
The function `weekday` takes a `long` and returns a `long`;
it can be used directly from Python.
The other function, `weekdays`, takes pointers to arrays.
This obviously can not be used from Python directly---we need to use the help of `cffi`
to access this function.

A critical point about efficiency is that we want to avoid *copying* arrays between the
Python/C language interface.
Luckily, this can be achieved by `numpy`.
Because a numerical array in `numpy` is stored in a contiguous block of memory,
a pointer to the memory block can be passed to the C side, achieving zero-copy inter-op.

Here is the code:

```python
# File `src/python/datex/c/version01.py`.

from cffi import FFI
import numpy as np

from ._version01 import lib

weekday = lib.weekday

ffi = FFI()

def weekdays(ts: np.ndarray) -> np.ndarray:
    n = len(ts)
    out = np.zeros(n, dtype=np.int64)
    p_in = ffi.cast("long *", ffi.from_buffer(ts))
    p_out = ffi.cast("long *", ffi.from_buffer(out))
    lib.weekdays(p_in, p_out, n)
    return out
```

Here I provide a module `version01` as the "face" of the module `_version01`.
In addition to a direct import and exposure of the function `weekday` in `_version01`,
`version01` provides a wrapper function for `weekdays` in `_version01`.
Note that functions in `_version01` are accessed via its member `lib`.

Now we are ready to test it out.
First, add the function `datex.c.version01.weekdays` to the testing code:

```python
...

from datex import cy, c

...

def do_all(fn, n):
    ...

    functions = [
        (version01.weekdays, timestamps_np),
        (version03.weekdays, timestamps_np),
        (cy.version09.weekdays, memoryview(timestamps_np)),
        (c.version01.weekdays, timestamps_np),
    ]

    ...

...
```

Then launch the script:

```
$ python test_1.py --n 1000000
datex.version01.weekdays              :    0.5243 seconds
datex.version03.weekdays              :    1.6563 seconds
datex.cy._version09.weekdays          :    0.0037 seconds
datex.c.version01.weekdays            :    0.0226 seconds
```

Ooops! A little disappointing. The C version is seven times slower than the Cython version!
The function `c.version01.weekdays` is definitely passing only the pointer of the `numpy` array
to the C function, and on the C side, the algorithm is identical to the Cython solution.
So, what's slowing it down?


## Dig into performance

This certainly puzzled me and triggered some digging.
It is interesting to see that if I rrn the function twice, the second time will be much faster than the first:

```
$ python test_1.py --n 1000000
datex.version01.weekdays              :    0.5053 seconds
datex.version03.weekdays              :    1.6358 seconds
datex.cy._version09.weekdays          :    0.0037 seconds
datex.c.version01.weekdays            :    0.0221 seconds
datex.c.version01.weekdays            :    0.0057 seconds
```

To be sure, I tried using different input data and array sizes in the second call than the first,
and the pattern was the same.
Furthermore, this pattern appears in each run of the program, that is,
the second time the program is run, the first call is still a few times slower than the second call.
This seems to suggest that there is some in-memory caching going on related to compiling the function `weekdays`.

To see more details, let's line-profile the function `weekdays`:

```python
# File `src/python/datex/c/version01.py`.

...

from zpz.profile import lineprofiled

...

@lineprofiled()
def weekdays(ts: np.ndarray) -> np.ndarray:
    ...
    
...
```

Run the script again with two calls to `weekdays`:

```
$ python test_1.py --n 1000000
datex.version01.weekdays              :    0.5281 seconds
datex.version03.weekdays              :    1.6649 seconds
datex.cy._version09.weekdays          :    0.0041 seconds
Timer unit: 1e-06 s

Total time: 0.381006 s
File: /home/docker-user/.local/lib/python3.6/site-packages/datex/c/version01.py
Function: weekdays at line 13

Line #      Hits         Time  Per Hit   % Time  Line Contents
==============================================================
    13                                           @lineprofiled()
    14                                           def weekdays(ts: np.ndarray) -> np.ndarray:
    15         1          3.0      3.0      0.0      n = len(ts)
    16         1        552.0    552.0      0.1      out = np.zeros(n, dtype=np.int64)
    17         1     375850.0 375850.0     98.6      p_in = ffi.cast("long *", ffi.from_buffer(ts))
    18         1         11.0     11.0      0.0      p_out = ffi.cast("long *", ffi.from_buffer(out))
    19         1       4588.0   4588.0      1.2      lib.weekdays(p_in, p_out, n)
    20         1          2.0      2.0      0.0      return out

datex.c.version01.weekdays            :    0.3871 seconds
Timer unit: 1e-06 s

Total time: 0.005146 s
File: /home/docker-user/.local/lib/python3.6/site-packages/datex/c/version01.py
Function: weekdays at line 13

Line #      Hits         Time  Per Hit   % Time  Line Contents
==============================================================
    13                                           @lineprofiled()
    14                                           def weekdays(ts: np.ndarray) -> np.ndarray:
    15         1          3.0      3.0      0.1      n = len(ts)
    16         1        527.0    527.0     10.2      out = np.zeros(n, dtype=np.int64)
    17         1         17.0     17.0      0.3      p_in = ffi.cast("long *", ffi.from_buffer(ts))
    18         1          6.0      6.0      0.1      p_out = ffi.cast("long *", ffi.from_buffer(out))
    19         1       4592.0   4592.0     89.2      lib.weekdays(p_in, p_out, n)
    20         1          1.0      1.0      0.0      return out

datex.c.version01.weekdays            :    0.0062 seconds
```

The profiling shows that in the first call to `weekdays`,
the first call to `ffi.cast` took 375850 time units, compared to 11 time units by the second call.
In the second call to `weekdays`, the two calls to `ffi.cast` took 17 and 6 time units, respectively.
This supports my theory of compiler caching to some extent.

I did not dig deeper along this line.
However, note that compilation overhead should be roughly constant (and small).
As the input size of the function increases, the compilation overhead should become less and less significant compared to the "real" computation.

For the purpose of timing the function `weekdays`, I added this line before the timing code

```
_ = c.version01.weekdays(np.array([1,2,3,4]))
```

to "warm it up".

Then remove the profiling code, and run the program with larger input data sizes:

```
$ python test_1.py --n 1000000
datex.version01.weekdays              :    0.5477 seconds
datex.version03.weekdays              :    1.7806 seconds
datex.cy._version09.weekdays          :    0.0042 seconds
datex.c.version01.weekdays            :    0.0061 seconds

docker-user@py3x in ~/work/src/py-extensions/tests/datex [develop]
$ python test_1.py --n 10000000
datex.version01.weekdays              :    5.5080 seconds
datex.version03.weekdays              :   18.0634 seconds
datex.cy._version09.weekdays          :    0.0569 seconds
datex.c.version01.weekdays            :    0.0679 seconds

docker-user@py3x in ~/work/src/py-extensions/tests/datex [develop]
$ python test_1.py --n 100000000
datex.version01.weekdays              :   54.6860 seconds
datex.version03.weekdays              :  175.0819 seconds
datex.cy._version09.weekdays          :    0.5647 seconds
datex.c.version01.weekdays            :    0.6544 seconds

docker-user@py3x in ~/work/src/py-extensions/tests/datex [develop]
$ python test_1.py --n 100000000 --repeat 10
datex.cy._version09.weekdays          :    5.8193 seconds
datex.c.version01.weekdays            :    6.6898 seconds
```

According to this benchmark, the C/`cffi` solution is about 15% slower than the Cython solution.
I doubt this statement is generalizable.
I would say the performances of the two solutions are "comparable".