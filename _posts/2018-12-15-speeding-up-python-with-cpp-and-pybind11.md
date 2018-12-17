---
layout: post
title: "Speeding up Python Numerics: the Main Course with C++ and pybind11"
excerpt_separator: <!--excerpt-->
tags: [python]
---

In two previous posts, I described using
[cython]({{ site.baseurl }}/blog/speeding-up-python-with-cython/) and
[C]({{ site.baseurl }}/blog/speeding-up-python-with-c-and-cffi/), respectively,
to speed up a Python program that involves simple numerical computations.
In this post, I will use the "big brother" of the industrial languages---C++---to
speed up the same program.<!--excerpt-->
For Python/C++ inter-op, I will use the excellent
[`pybind11`](https://github.com/pybind/pybind11).

This task consists of three components or steps:

1. Develop (or obtain) the C++ code, independent of Python. (That is, it's perfectly fine
if we use a C++ library that was not originally intended to be called from Python.)
2. Develop "binding" code using `pybind11`. This code will make selected functions, classes, and methods in the C++ code accessible from Python. Building this code will generate a '.so' file (on Linux) that is a
**proper Python module**, that is, it is understood by the Python interpreter at run-time
without needing any other tool.
3. Use the generated module in Python. Again, this access does not need `pybind11` or any other tool.

## The C++ implementation

The C++ implementation consists of a header file

```cpp
// File `src/cc/datex/cc_version01.h`.

#ifndef _CC_VERSION01_
#define _CC_VERSION01_

#include <vector>

long weekday(long ts);

std::vector<long> weekdays(std::vector<long> const & ts);

#endif
```

and a source file

```cpp
// File `src/cc/datex/cc_version01.cc`.

#include <vector>

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


std::vector<long> weekdays(std::vector<long> const & ts)
{
    long n = ts.size();
    std::vector<long> out(n);
    for (long i = 0; i < n; i++) {
        out[i] = weekday(ts[i]);
    }
    return out;
}
```

The function `weekday` is a straightforward port of the Python version
(and is identical to the C version in the
[previous post]({{ site.baseurl }}/blog/speeding-up-python-with-c-and-cffi/)).
The function `weekdays` takes a vector, calls `weekday` on each element of the vector, 
and returns the result in a new vector.

To highlight the notion that this C++ implementation can be a library totally unaware of Python, these two files are stored in a directory separate from the Python package.

## Python bindings for the C++ code

The binding code is C++ code that uses `pybind11` to define a Python module
and specify content of the module. My initial attempt is as follows:

```cpp
// File `src/python_ext/datex/cc/version01.cc`.

#include "datex/cc_version01.h"
#include <pybind11/pybind11.h>
#include <pybind11/stl.h>

PYBIND11_MODULE(version01, m)
{
    m.def("weekday", &weekday);
    m.def("weekdays", &weekdays);
}
```

This code trivially exposes the C++ functions `weekday` and `weekdays` to Python
in a new module named "version01". (I would rather name the module "_version01",
but pybind11 does not appear to allow it.)

Pay special attention to

```cpp
#include <pybind11/stl.h>
```

With this statement, collections of compatible types are automatically converted
between Python and C++. For example, Python `list` and `tuple` correspond to C++ `vector`,
whereas Python `dict` corresponds to C++ `map`.
In the case of `weekdays`, the input from Python can be a list or a `numpy` array (or possibly other iterables);
these will be converted to a `vector` as input to the C++ function.
On the other hand, the `vector` returned from the C++ function will become a `list` coming out of the Python function `weekdays` in the interface module `version01`.

At this point, the directory structure looks like this:

```
├── setup.py
├── src
│   ├── c
│   ├── cc
│   │   └── datex
│   │       ├── cc_version01.cc
│   │       └── cc_version01.h
│   ├── python
│   │   ├── datex
│   │   │   ├── c
│   │   │   ├── cc
│   │   │   │   ├── __init__.py
│   │   │   ├── cy
│   │   │   ├── __init__.py
│   │   │   ├── version01.py
│   │   │   └── version03.py
│   ├── python_ext
│   │   └── datex
│   │       ├── c
│   │       ├── cc
│   │       │   ├── version01.cc
│   │       ├── cy
└── tests
    ├── datex
    │   ├── __init__.py
    │   └── test_1.py
```

Below is the content of `setup.py` (skipping non-C++ details):

```python
from setuptools import setup, Extension, find_packages

cy_extensions = ...

cffi_extensions = ...

cc_options = ['--std=c++17', '-O3', '-Wall', '-Wextra', '-Wfatal-errors']

cc_extensions = [
    Extension(
        'datex.cc.version01',
        sources=['src/python_ext/datex/cc/version01.cc',
                 'src/cc/datex/cc_version01.cc'],
        include_dirs=['src/cc'],
        extra_compile_args=cc_options,
        ),
    ]

setup(
    name='datex',
    version='0.1.0',
    package_dir={'': 'src/python'},
    packages=find_packages(where='src/python'),
    ext_modules=cc_extensions + cy_extensions,
    cffi_modules=cffi_extensions,
)
```

Typing

```bash
$ pip install --user .
```

will install the package `datex`.

To ease import, put this line in the file `src/python/datex/cc/__init__.py`:

```python
# File `src/python/datex/cc/__init__.py`.

from . import version01
```

The `version01` being imported here is the dynamic library generated by the C++ extension,
and there is no Python source file corresponding to it.

Let's check the speed of the C++ extension in comparison with the Python version as well as the C and cython extensions. Below is the benchmarking code:

```python
# File `tests/datex/test_1.py`.

from functools import partial
import numpy

from zpz.profile import Timer

from datex import version01, version03
from datex import cy, c, cc


def check_it(fn, timestamps):
    # Check correctness against the original Python version.

    z = fn(timestamps)
    z0 = version01.weekdays(timestamps)
    assert all(a == b for a,b in zip(z, z0))


def time_it(fn, timestamps, repeat=1):
    tt = Timer().start()
    for _ in range(repeat):
        z = fn(timestamps)
    t = tt.stop().seconds
    name = fn.__module__ + '.' + fn.__name__
    print('{: <42}:  {: >8.4f} seconds'.format(name, t))


def do_all(fn, n):
    timestamps_np = numpy.random.randint(10000000, 9999999999, size=n, dtype=numpy.int64)

    functions = [
        (version01.weekdays, timestamps_np),
        (version03.weekdays, timestamps_np),
        (cy.version09.weekdays, memoryview(timestamps_np)),
        (c.version01.weekdays, timestamps_np),
        (cc.version01.weekdays, memoryview(timestamps_np)),
    ]

    # Cache JIT type of work so that it does not distort benchmarks.
    _ = c.version01.weekdays(timestamps_np[:10])

    for f, ts in functions:
        fn(f, ts)


def test_all():
    # This is called by `py.test` to verify that code runs and is correct.
    do_all(check_it, 10)


def benchmark(n, repeat):
    do_all(partial(time_it, repeat=repeat), n)


if __name__ == "__main__":
    # Running the script (i.e. not by `py.test`) will do time benchmarking.

    from argparse import ArgumentParser
    p = ArgumentParser()
    p.add_argument('--n', type=int, default=10000000)
    p.add_argument('--repeat', type=int, default=1)
    args = p.parse_args()

    benchmark(args.n, args.repeat)
```

Here's the benchmark outcome:

```
$ python test_1.py 
datex.version01.weekdays                  :    5.2313 seconds
datex.version03.weekdays                  :   17.4581 seconds
datex.cy._version09.weekdays              :    0.0590 seconds
datex.c.version01.weekdays                :    0.0711 seconds
datex.cc.version01.weekdays               :    0.4835 seconds
```

I have passed a `memoryview` to the function `datex.cc.version01.weekdays`.
In another run where I passed a `numpy` array directly to the function,
it took three times as long (1.40 seconds, specifically) for this particular input size.

Although the C++ extension is clearly faster than the Python versions,
it falls far behind the C and Cython extensions.
The slowness is mainly due to array copying at the Python/C++ boundary,
when `pybind11/stl.h` is at work.


## Vectorize it

Pybind11 provides an option to 
[*vectorize* a function](https://pybind11.readthedocs.io/en/master/advanced/pycpp/numpy.html?highlight=vectorize#vectorizing-functions) using `numpy`. 
All I need to do is add a vectorized function signature in the binding code:

```cpp
// File `src/python_ext/datex/cc/version01.cc`.

#include "datex/cc_version01.h"
#include <pybind11/pybind11.h>
#include <pybind11/stl.h>
#include <pybind11/numpy.h>

namespace py = pybind11;

PYBIND11_MODULE(version01, m)
{
    m.def("weekday", &weekday);
    m.def("weekdays", &weekdays);
    m.def("vectorized_weekday", py::vectorize(weekday));
}
```

Note the inclusion of `<pybind11/numpy.h>` and the use of `py::vectorize` on the "scalar" function `weekday`.

Then add this new function to the benchmark code:

```python
# File `tests/datex/test_1.py`.

...

def do_all(fn, n):
    ...

    functions = [
        (version01.weekdays, timestamps_np),
        (version03.weekdays, timestamps_np),
        (cy.version09.weekdays, memoryview(timestamps_np)),
        (c.version01.weekdays, timestamps_np),
        (cc.version01.weekdays, memoryview(timestamps_np)),
        (cc.version01.vectorized_weekday, timestamps_np),
    ]

    ...

...
```

Now check the speed:

```
$ python test_1.py 
datex.version01.weekdays                  :    5.6730 seconds
datex.version03.weekdays                  :   17.5845 seconds
datex.cy._version09.weekdays              :    0.0571 seconds
datex.c.version01.weekdays                :    0.0703 seconds
datex.cc.version01.weekdays               :    0.4766 seconds
datex.cc.version01.vectorized_weekday     :    0.0688 seconds
```

The vectorized version is about **seven times as fast** as the copied-vector version.
Its speed is comparable to that of the C extension (via `cffi`),
but is slightly slower than the Cython version.

Notice that I did not use `memoryview` while calling `vectorized_weekday`.
Using `memoryview` makes no difference here.


## Use `numpy` to achieve zero-copy array passing

I imagine `vectorize` is only applicable to certain types of functions.
To have full control, I can use
[`numpy` arrays provided by `pybind11`](https://pybind11.readthedocs.io/en/master/advanced/pycpp/numpy.html?highlight=vectorize#arrays),
called `pybind11::array`,
as input arguments and return values,
mixed with arguments of other types, and manipulate these arrays however I need to. Below is my second version of the C++ extension.

```cpp
// File `src/python_ext/datex/cc/version02.cc`.

#include <datex/cc_version01.h>
#include <pybind11/pybind11.h>
#include <pybind11/numpy.h>

namespace py = pybind11;


py::array_t<long> weekdays_py(py::array_t<long> ts)
{
    long n = ts.size();
    py::array_t<long> out = py::array_t<long>(n);
    auto p_ts = ts.data();
    auto p_out = out.mutable_data();
    for (long i = 0; i < n; i++) {
        p_out[i] = weekday(p_ts[i]);
    }
    return out;
}


PYBIND11_MODULE(version02, m)
{
    m.def("weekday", &weekday);
    m.def("weekdays", &weekdays_py);
}
```

Note the absence of `#include <pybind11/stl.h>`.
Automatic conversions done by the inclusion of that header tend to interfere with the desire for full control.

Now add `version02` to the benchmark code,

```python
```python
# File `tests/datex/test_1.py`.

...

def do_all(fn, n):
    ...

    functions = [
        (version01.weekdays, timestamps_np),
        (version03.weekdays, timestamps_np),
        (cy.version09.weekdays, memoryview(timestamps_np)),
        (c.version01.weekdays, timestamps_np),
        (cc.version01.weekdays, memoryview(timestamps_np)),
        (cc.version01.vectorized_weekday, timestamps_np),
        (cc.version02.weekdays, timestamps_np),
    ]

    ...

...
```

and check the speed,

```
$ python test_1.py 
datex.version01.weekdays                  :    5.4988 seconds
datex.version03.weekdays                  :   18.0567 seconds
datex.cy._version09.weekdays              :    0.0570 seconds
datex.c.version01.weekdays                :    0.0796 seconds
datex.cc.version01.weekdays               :    0.4754 seconds
datex.cc.version01.vectorized_weekday     :    0.0688 seconds
datex.cc.version02.weekdays               :    0.0698 seconds
```

This version---`cc.version02.weekdays`---runs at the same speed as the *vectorized* version.

To recap, `pybind11::vectorize` and `pybind11::array` are suitable in different situations,
and both are straightforward to use.

Pybind11 shines in more complex use cases, notably in 
[object-oriented code](https://pybind11.readthedocs.io/en/master/classes.html).
This has not been needed in the example of this post.
However, if you are interested, check out the documentation and give it a spin!

All the code in this post can be found at
[https://github.com/zpz/experiments.py](https://github.com/zpz/experiments.py).