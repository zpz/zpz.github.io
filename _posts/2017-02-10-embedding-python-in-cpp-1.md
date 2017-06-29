---
layout: post
title: Embedding Python in C++, Part 1
---

Python has very good inter-operability with C and C++.
(In this post I'll just say C++ for simplicity.)
There are two sides to this "inter-op".
The first is that the main program is in Python;
certain performance critical parts are coded in C++, or provided by an existing C++ library,
which is called from the main Python program.
This is known as "extending Python with C++".

The other side is that the main program is in C++, and it calls some Python code.
This is known as "embedding Python in C++".

Both needs arise, but the first is by far more common.
We can find much more online resources about "extending" than about "embedding".
However, the two uses share a lot of common machinery.
There are several tools in this area.
As far as I saw, their documentation and other resources all predominantly talk about "extending".
However, most of them either already support "embedding", or could do so with modest additional work.

Recently I needed to do both "extending" and "embedding". I plan to first write about my "embedding" experience in a few articles.


Setting the stage
=================

My main program is a realtime, low-latency, high-throughput, online sevice in C++. In a critical component, it runs some sophisticated modeling or "machine learning" algorithm to make a decision. I decided to develop the model algorithm in Python in order to tap into its excellent data stack and modeling ecosystem. As a result, I needed to build an interface between the Python and C++ codes. To fix ideas, the Python code is listed below.

The meat of the computation is carried out by the class `Engine`:

```
"""
Module `py4cc_1.py` in package `py4cc`.
"""

import copy
import json
import multiprocessing
import time
import traceback


def big_model(float_features, str_features, int_feature):
    z = int(sum(abs(v) for v in float_features))
    z += sum(len(s.upper()) for s in str_features)
    z += len([' '] * int_feature)
    return z


class Engine:
    def initialize(self, **kwargs):
        pass

    def run_big_model(self, *, float_features, str_features, int_feature):
        try:
            z = big_model(float_features, str_features, int_feature)

            # Emulate a time consuming step,
            # but make the duration deterministic hence reproducible.
            time.sleep((z % 100 + 1) * 0.0001)

            return z, ''
        except Exception as e:
            return -1, traceback.format_exc()
```

The class `Driver` provides C++-facing API. It handles receiving task submissions, providing results to queries, and managing a process pool, in which each process runs a `Engine` instance:

```
""" 
Module `py4cc_1.py` in package `py4cc1` (continued).
"""

# Global variable in its process.
global_engine = Engine()


def initialize(kwargs):
    # `Pool` can only pass in positional arguments.
    # We take a single argument which is a dict,
    # and expand it after this.
    global global_engine
    return global_engine.initialize(**kwargs)


def run_big_model(**kwargs):
    global global_engine
    return global_engine.run_big_model(**kwargs)


class Driver:
    def __init__(self, max_tasks=64, n_subprocesses=-1):
        self._tasks = {}
        self._max_tasks = max_tasks
        self._pool = None
        if n_subprocesses <= 0:
            n_subprocesses = multiprocessing.cpu_count()
        self._n_subproceses = n_subprocesses

    def initialize(self, *, config_json, float_feature_names, str_feature_names, **kwargs):
        self._config_json = json.loads(config_json)
        self._float_feature_names = copy.deepcopy(float_feature_names)
        self._str_feature_names = copy.deepcopy(str_feature_names)

        self._pool = multiprocessing.Pool(
            self._n_subproceses,
            initializer=initialize,
            initargs=[kwargs]
        )

        return self

    def submit(self, *, float_features, str_features, int_feature):
        """
        Returns:
            tuple (flag, key, message)
        """
        assert len(self._float_feature_names) == len(float_features)
        assert len(self._str_feature_names) == len(str_features)
        try:
            if len(self._tasks) >= self._max_tasks:
                return 1, 0, 'Model engine is working at capacity. Try again later.'

            result = self._pool.apply_async(
                run_big_model,
                kwds={'float_features': float_features,
                      'str_features': str_features,
                      'int_feature': int_feature},
            )
            key = id(result)
            self._tasks[key] = result

            return 0, key, ''
        except Exception as e:
            return 9, 0, traceback.format_exc()

    def retrieve(self, key):
        """
        Args:
            key: the key returned from `submit`, used to identify the particular task.

        Returns:
            tuple (flag, value, message).
                Usually `flag == 0` combined with `value > 0` is a sure sign of success.
        """
        try:
            result = self._tasks[key]
            if not result.ready():
                return 1, -1, 'The requested task is not finished'

            val, msg = result.get(0)
            del self._tasks[key]
            if val > 0:
                return 0, val, msg
            else:
                return 2, val, msg
        except Exception as e:
            return 9, -1, traceback.format_exc()

    def finalize(self):
        pass
```

A few points of note:

1. Because this code is totally CPU-bound, I use multiple processes to increase throughput and to serve the multiple threads of the C++ program.
1. At design stage there was a choice to be made between using `multiprocessing` and `concurrent.futures`. One reason for using `multiprocessing` is that it allows custom initialization of the child processes, while `concurrent.futures` does not (yet) support this directly.
1. I use a "submit/retrieve" style to handle task requests initiated from any C++ thread: the thread "submits" a modeling task to the `Driver`, and later comes back to query the result. If submission is not accepted or result is not ready, it waits a little bit and tries again.
1. `Driver` launches a "process pool". The pool accepts task submissions and returns `AsyncResult` objects.
1. I use the `id` (i.e. memory address) of the `AsyncResult` as a receipt for the submitter (a C++ thread), which later provides the `id` in calls to `retrieve` to identify the task whose result is being requested. The keys will have no collision because any key could be used again only after that result has been removed from `_tasks`.
1. In the methods `submit` and `retrieve`, I wrap all code in `try/except` statements to capture any exception, so that to the consumer (C++ threads), the Python code never raises exceptions. The returned value contains exit code and error messages, if any, in addition to the true values of interest.
1. In this example code, I intentionally used a variety of data types (string, string list, numerical list, numerical scalar, etc) in order to test functionalities of the embedding tools.



Testing it in Python
====================

I wrote a Python program to verify that it works. The test code primarily does the following things:

1. Make up some testing data.
1. Create a number of threads, each processing a subset of the testing data.
1. Each thread loops through its data points. In each iteration, it submits the data to `Driver`, then queries about the result. Once the result is ready, it retrieves the result and moves to the next data point. When the result is not ready, it `sleep`s, allowing other threads to execute.

The test code is available at [https://github.com/zpz/python/tree/master/py4cc/tests](https://github.com/zpz/python/tree/master/py4cc/tests). The test program ran with no issues.


First attempt at using raw Python/C API
=======================================

A natural approach uses Python's C API. The official documentation has a [tutorial](https://docs.python.org/3/extending/index.html) as well as a [reference manual](https://docs.python.org/3/c-api/index.html).

To use the API, one must first `#include "Python.h"`, which on my Linux box is located in `/usr/local/include/python3.6m`.

Before anything python related, one needs to call

```
Py_Initialize();
```

to initiate the Python interpreter.  After that, the prevalent business is to create and manipulate variables of type `PyObject *`. (Every Python object is represented by a struct of type `PyObject`.) There are functions to convert basic types between Python and C++, build Python `list` or `dict` objects, call Python functions (which are, of course, `PyObject`s), access attributes of a Python object, and so on.

    
For example,

```
PyObject * pModule = PyImport_Import(PyUnicode_FromString("py4cc1.py4cc"));
```

does what the following accomplishes in Python:

```
import py4cc1.py4cc
```

Top-level functions, classes, and variables in this module are then accessed as attributes of the object `pModule`. For example, to instantiate a `Driver` object, we need to first get the `Driver` class object,

```
PyObject * driver_cls = PyObject_GetAttrString(pModule, "Driver");
```

Then initialize a `Driver` object using the API that calls a Python function (because we would do `Driver()` to initiate a `Driver` object as if `Driver` is just a function):

```
PyObject * noargs = PyTuple_New(0);
PyObject * kwargs = PyDict_New();
PyDict_SetItemString(kwargs, "max_tasks", PyLong_FromLong(64));

PyObject * driver = PyObject_Call(driver_cls, noargs, kwargs);
```
    
As one can see, the API functions are kind of straightforward. But boy, is that tedious. I will not give more details since it's all in the documentation.
    
My C++ header file that corresponds to the Python API is listed below.

```
// File `cc4py_1.h` in `cc4py`.

#ifndef CC4PY_H_
#define CC4PY_H_

#include "Python.h"

#include <string>
#include <vector>


namespace cc4py {
    struct TaskReceipt {
        int flag;
            // 0 -- success; come query the result later with `retrieve`.
            // 1 -- system is full with tasks; try re-submit later.
            // 2+ -- error.
        long key;
            // This is a valid, positive key when `flag` is `0`, 
            // to identify the task just submitted.
            // It is going to be used later to request result of this particular task.
        std::string message;
            // Message when `flag` is nonzero.
    };


    struct TaskResult {
        int flag;
            //  0 -- success; this task is removed from the "driver";
            //       the caller should use the returned `value`.
            //  1 -- the requested task is unfinished; try again later.
            //  2+ -- error; this task is removed from the engine; 
            //       the caller should move on.
        long key;
            // the `key` part of `TaskReceipt` that was returned by `submit`.
        int value;
            // Usually this is a valid positive number only when `flag` is `0`.
        std::string message;
            // Empty if all is well; otherwise inspect and log this message unless `flag` is `1`.
    };


    class Driver
    {
    private:
        // Max number of tasks in the system,
        // including unfinished and finished but un-retrieved ones.
        // Once this capacity is achieved, new task submission is not accepted
        // until some finished tasks have been retrieved (hence removed) from the system.
        int _max_tasks = 64;

        PyObject * _driver = nullptr;
        bool _initialized = false;
        bool _finalized = false;

    public:
        Driver(int max_tasks=64);

        void initialize(
            std::string const & config_json,
            std::vector<std::string> const & float_feature_names,
            std::vector<std::string> const & str_feature_names
            );

        TaskReceipt submit(
            std::vector<double> const & float_features,
            std::vector<std::string> const & str_features,
            const int int_feature
            );

        TaskResult retrieve(const long key);

        void finalize();

        ~Driver();
    };

}   // namespace

#endif  // CC4PY_H_
```

A good part of the corresponding source file is listed below. The complete code is available at [https://github.com/zpz/python/tree/master/cc4py/cc4py_1.cc](https://github.com/zpz/python/tree/master/cc4py/cc4py_1.cc). You can get a feel of the tedious yet predictable style of this approach from this code sample.

```
// File `cc4py_1.cc` in `cc4py`.

#include "cc4py.h"
#include "Python.h"

#include <algorithm>
#include <cassert>
#include <exception>
#include <iostream>
#include <memory>


using namespace cc4py;


PyObject* to_py_float_list(double const * buffer, const int n)
{
    PyObject* list = PyList_New((Py_ssize_t) n);
    for (Py_ssize_t i = 0; i < n; i++) {
        PyList_SetItem(list, i, PyFloat_FromDouble(buffer[i]));
    }
    return list;
}


PyObject* to_py_str_list(char const * const* buffer, int n)
{
    PyObject* list = PyList_New((Py_ssize_t) n);
    for (Py_ssize_t i = 0; i < n; i++) {
        PyList_SetItem(list, i, PyUnicode_FromString(buffer[i]));
    }
    return list;
}


std::vector<char const *> to_c_strs(std::vector<std::string> const& strings)
{
    std::vector<char const *> c_strs;
    c_strs.reserve(strings.size());
    std::transform(std::begin(strings), std::end(strings),
        std::back_inserter(c_strs), std::mem_fn(&std::string::c_str));
    return c_strs;
}


Driver::Driver(int max_tasks)
    : _max_tasks{max_tasks}
{
    // Initialize the Python interpreter.
    Py_Initialize();
}


void Driver::initialize(
    std::string const & model_config_json,
    std::vector<std::string> const & float_feature_names,
    std::vector<std::string> const & str_feature_names
    )
{
    auto pModule = PyImport_Import(PyUnicode_FromString("py4cc1.py4cc"));
    auto driver_cls = PyObject_GetAttrString(pModule, "Driver");

    auto noargs = PyTuple_New(0);
    auto kwargs = PyDict_New();
    PyDict_SetItemString(kwargs, "max_tasks", PyLong_FromLong(_max_tasks));

    _driver = PyObject_Call(driver_cls, noargs, kwargs);

    PyDict_Clear(kwargs);
    PyDict_SetItemString(kwargs, "config_json", PyUnicode_FromString(model_config_json.c_str()));
    PyDict_SetItemString(kwargs, "float_feature_names",
        to_py_str_list(to_c_strs(float_feature_names).data(), float_feature_names.size()));
    PyDict_SetItemString(kwargs, "str_feature_names",
        to_py_str_list(to_c_strs(str_feature_names).data(), str_feature_names.size()));

    auto method = PyObject_GetAttrString(_driver, "initialize");
    PyObject_Call(method, noargs, kwargs);

    _initialized = true;
}


TaskReceipt Driver::submit(
    std::vector<double> const & float_features,
    std::vector<std::string> const & str_features,
    const int int_feature
    )
{
    // ... omitted ...
}


TaskResult Driver::retrieve(const long key)
{
    auto method = PyUnicode_FromString("retrieve");
    auto py_key = PyLong_FromLong(key);
    auto z = PyObject_CallMethodObjArgs(_driver, method, py_key, nullptr);

    auto flag = PyTuple_GetItem(z, 0);
    auto val = PyTuple_GetItem(z, 1);
    auto msg = PyTuple_GetItem(z, 2);

    TaskResult result;
    result.flag = static_cast<int>(PyLong_AsLong(flag));
    result.key = key;
    result.value = static_cast<int>(PyLong_AsLong(val));
    result.message = PyUnicode_AsUTF8(msg);

    return result;
}


void Driver::finalize()
{
    // ... omitted ...
}


Engine::~Driver()
{
    this->finalize();
    Py_Finalize();
}
```


Testing it in C++
=================

The test program for the C++ implementation is analogous to the Python test. An important difference is that the C++ threads need to lock up the code blocks that call `Driver` methods because, unlike Python, C++ threads do execute simultanously on a multi-core machine. If multiple threads access the single Python interpreter at the same time, the program will crash.

The source code is available at
[https://github.com/zpz/python/tree/master/cc4py/test_1.cc](https://github.com/zpz/python/tree/master/cc4py/test_1.cc).


Looking back and forth
======================

Let me emphasize that **the C++ implementation listed above is not complete**; it's just a start. For one thing, there are typically multiple slightly different functions in the API that do the same thing, therefore I expect the implementation can be somewhat cleaner.

A real issue is momery management. For example, my code did not take care of releasing memory that it allocated for strings. However, a much, much bigger burden is taking care of reference counts for Python, which I decided to forgo in the proof-of-concept code. 

[The section entitled "Objects, Types and Reference Counts" in the official documentation](https://docs.python.org/3/c-api/intro.html) basically convinced me that I could hardly get it right, therefore the raw API is not the way to go. There has to be a better way, a way that works on a higher level, hence is cleaner and more concise, and that takes much low-level burden, especially memory management, off the shoulder of the user.

In the next post I will explore third-party tools, which necessarily are built on top of the raw API, that make my job much, much, and **so much** easier.

