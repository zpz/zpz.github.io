---
layout: post
title: Embedding Python in C++, Part 1
---

Python has very good inter-operability with C and C++. (In this post I'll just say C++ for simplicity.) There are two situations to this "inter-op". The first is that the main program is in Python; certain performance critical parts are coded in C++, or provided by an existing C++ library, which is called from the main Python program. This is known as "extending Python with C++".

The second situation is that the main program is in C++, and it calls some Python code. This is known as "embedding Python in C++".

Both needs arise, but the first ("extension") is by far more common. We can find much more online resources about "extension" than about "embedding". However, the two uses share a lot of the tooling. Recently I needed to do both "extension" and "embedding". I plan to first write about my "embedding" experience in a few posts.


Description of the problem
==========================

My main program is a realtime, low-latency, high-throughput, online sevice in C++. In a critical component, it runs some sophisticated modeling or "machine learning" algorithm to make a decision. I dediced to develop the modeling algorithm in Python in order to tap into its excellent data stack and modeling ecosystem. As a result, I needed to build an interface between the Python and C++ codes. To fix ideas, the Python code is listed below.

The meat of the computation is carried out by class `Engine`:

```
"""
Module `py4cc.py` in package `py4cc1`.
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

Class `Driver` provides C++-facing API. It handles task reception, result provision, and management of a process pool; each process runs a `Engine` instance:

```
""" 
Module `py4cc.py` in package `py4cc1` (continued).
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
        if n_subprocesses < 0:
            n_subprocesses = multiprocessing.cpu_count()
        self._n_subproceses = n_subprocesses
        self._engine = None   # provide a sequential option for testing

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
        assert self._n_subproceses > 0
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
            # A valid positive `key` occurs along with a flag of `0`.
            # key will have no collision because any key (i.e. memory address)
            # could be used again only after that result has been removed
            # from `_tasks`.

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
        assert self._n_subproceses > 0
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
2. I use a "submit/retrieve" style to handle task requests initiated from any C++ thread: the thread "submits" a modeling task to the `Driver`, and later come back to query the result. If submission is not accepted or result is not ready, it waits a little bit and tries again.
3. `Driver` launches a "process pool". The pool accepts task submissions and returns `AsyncResult` objects.
4. We use the `id` (i.e. memory address) of the `AsyncResult` as a receipt for the submitter (a C++ thread),
which later provides the `id` in calls to `retrieve` to identify the task whose result is being requested.
5. In methods `submit` and `retrieve`, we wrap all code in `try/except` statements to capture any exception, so that to the consumer (C++ threads), the Python code never raises exceptions. The returned value contains exit code and error messages, if any, in addition to true values of interest.
6. In this example code, I intentionally used a variety of data types (string, string list, numerical list, numerical scalar, etc) in order to test the embedding tools in later steps.


Test it in Python
=================

I wrote a Python program to verify it works.


```
"""
File `test.py`.
"""

import threading
import time

from faker import Faker
import numpy as np

from py4cc1.py4cc import Driver, big_model


rnd_seed = 3333

config_json = '{"model_name": "time killer"}'
float_feature_names = ['value_1', 'value_2']
str_feature_names = ['name', 'sentence', 'id']


def makedata(N):
    fake = Faker()
    fake.seed(rnd_seed)

    data = [None] * N
    for idx in range(N):
        data[idx] = {
            'float_features': [fake.pyfloat(), fake.pyfloat()],
            'str_features': [fake.name(), fake.sentence(), fake.uuid4()],
            'int_feature': fake.pyint()
        }

    answers = np.array([big_model(**z) for z in data], int)
    # Correct result for verification.

    return data, answers


def make_driver(n_subprocesses=-1):
    driver = Driver(n_subprocesses=n_subprocesses)
    driver.initialize(config_json=config_json,
                      float_feature_names=float_feature_names,
                      str_feature_names=str_feature_names)
    return driver


def do_one_thread(driver, data, results, idx_from, idx_to):
    for idx in range(idx_from, idx_to):
        while True:
            flag, key, msg = driver.submit(**data[idx])
            if flag == 0:
                assert key > 0
                break
            elif flag == 1:
                time.sleep(0.001)
            else:
                raise Exception(msg)

        time.sleep(0.001)

        while True:
            flag, val, msg = driver.retrieve(key)
            if flag == 0:
                results[idx] = val
                break
            elif flag == 1:
                time.sleep(0.001)
            else:
                raise Exception(msg)


def do_threads(data):
    driver = make_driver()

    N = len(data)
    results = np.zeros(N, int)

    n_threads = 16
    idx_from = [0] * n_threads
    idx_to = [N] * n_threads
    for i in range(n_threads - 1):
        last_to = N * (i+1) // n_threads
        idx_to[i] = last_to
        idx_from[i+1] = last_to

    threads = [
        threading.Thread(target=do_one_thread,
                         args=[driver, data, results, idx_from[i], idx_to[i]])
        for i in range(n_threads)
        ]

    time0 = time.perf_counter()
    for t in threads:
        t.start()
    for t in threads:
        t.join()
    time1 = time.perf_counter()

    driver.finalize()
    
    print('total time:', time1 - time0, 'seconds')
    print('processed', int(N / (time1 - time0)), 'cases per second')

    return results


def main():
    N = 1000
    data, answers = makedata(N)

    results_th = do_threads(data)
    assert all(results_th == answers)


if __name__ == '__main__':
    main()

```

Run it:

```
$ python test.py 
total time: 1.298026571000264 seconds
processed 770 cases per second
```

No issues.


First attempt using raw Python/C API
====================================


Now it's time to familiarize myself with the  Python/C API. The official documentation has a [tutorial](https://docs.python.org/3/extending/index.html) as well as a [reference manual](https://docs.python.org/3/c-api/index.html).

To use the API, one must first `include` `Python.h`, which on my Linux box is located in `/usr/local/include/python3.6m`.

Before anything else, one needs to call

```
Py_Initialize();
```

to initiate the Python interpreter.  After that, the prevalent business is to create and manipulate variables of type `PyObject *`. (Every Python object is represented by a struct of type `PyObject`.) There are functions to convert basic types between Python and C++, build Python `list` or `dict` objects, call Python functions (which are, of course, `PyObject`s), access attributes of a Python object, and so on.

    
For example,

```
auto pModule = PyImport_Import(PyUnicode_FromString("py4cc1.py4cc"));
```

(where `auto` would resolve to `PyObject *`) does what the following line of code accomplishes in Python:

```
import py4cc1.py4cc
```

Top-level functions, classess, and variables in this module are then accessed as attributes of the object `pModule`. For example, to instantiate a `Driver` object, we need to first get the `Driver` class object,

```
auto engine_cls = PyObject_GetAttrString(pModule, "Driver");
```

Then initiate a `Driver` object using the API that calls a Python function (because we would do `Driver()` to initiate a `Driver` object as if `Driver` is just a function):

```
auto noargs = PyTuple_New(0);
auto kwargs = PyDict_New();
PyDict_SetItemString(kwargs, "max_tasks", PyLong_FromLong(64));

auto engine = PyObject_Call(engine_cls, noargs, kwargs);
```
    
As one can see, the API functions are kind of straightfoward. But boy, is that tedious. I will not give more details since it's all in the documentation.
    
My C++ header file that corresponds to the Python API is listed below.

```
// File `cc4py.h`.

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
            // This is a valid, positive key when `flag` is `0`, to identify the task just submitted.
            // It is going to be used later to request result of this particular task.
        std::string message;
            // Message when `flag` is nonzero.
    };


    struct TaskResult {
        int flag;
            //  0 -- success; this task is removed from the engine; the caller should use the returned `value`.
            //  1 -- the requested task is unfinished; try again later.
            //  2+ -- error; this task is removed from the engine; the caller should move on.
        long key;
            // `key` part of `TaskReceipt` that was returned by `submit`.
        int value;
            // Usually this is a valid positive number only when `flag` is `0`.
        std::string message;
            // Empty if all is well; otherwise inspect and log this message unless `flag` is `1`.
    };


    class Engine
    {
    private:
        // Max number of tasks in the engine,
        // including unfinished and finished but un-retrieved ones.
        // Once this capacity is achieved, the task submission is not accepted
        // until some finished tasks have been retrieved (hence removed) from the engine.
        int _max_tasks = 64;

        PyObject * _engine = nullptr;
        bool _initialized = false;
        bool _finalized = false;

    public:
        Engine(int max_tasks=64);

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

        ~Engine();
    };

}   // namespace

#endif  // CC4PY_H_
```

The corresponding source file is this:

```
// File `cc4py.cc`.

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


Engine::Engine(int max_tasks)
    : _max_tasks{max_tasks}
{
    // Initialize the Python interpreter.
    Py_Initialize();
}


void Engine::initialize(
    std::string const & model_config_json,
    std::vector<std::string> const & float_feature_names,
    std::vector<std::string> const & str_feature_names
    )
{
    auto pModule = PyImport_Import(PyUnicode_FromString("py4cc1.py4cc"));
    auto engine_cls = PyObject_GetAttrString(pModule, "Driver");

    auto noargs = PyTuple_New(0);
    auto kwargs = PyDict_New();
    PyDict_SetItemString(kwargs, "max_tasks", PyLong_FromLong(_max_tasks));

    _engine = PyObject_Call(engine_cls, noargs, kwargs);

    PyDict_Clear(kwargs);
    PyDict_SetItemString(kwargs, "config_json", PyUnicode_FromString(model_config_json.c_str()));
    PyDict_SetItemString(kwargs, "float_feature_names",
        to_py_str_list(to_c_strs(float_feature_names).data(), float_feature_names.size()));
    PyDict_SetItemString(kwargs, "str_feature_names",
        to_py_str_list(to_c_strs(str_feature_names).data(), str_feature_names.size()));

    auto method = PyObject_GetAttrString(_engine, "initialize");
    PyObject_Call(method, noargs, kwargs);

    _initialized = true;
}


TaskReceipt Engine::submit(
    std::vector<double> const & float_features,
    std::vector<std::string> const & str_features,
    const int int_feature
    )
{
    auto method = PyObject_GetAttrString(_engine, "submit");

    auto kwargs = PyDict_New();
    PyDict_SetItemString(kwargs, "float_features", to_py_float_list(float_features.data(), float_features.size()));
    PyDict_SetItemString(kwargs, "str_features", to_py_str_list(to_c_strs(str_features).data(), str_features.size()));
    PyDict_SetItemString(kwargs, "int_feature", PyLong_FromLong(int_feature));

    auto args = PyTuple_New(0);

    auto z = PyObject_Call(method, args, kwargs);
    auto flag = PyTuple_GetItem(z, 0);
    auto key = PyTuple_GetItem(z, 1);
    auto msg = PyTuple_GetItem(z, 2);

    TaskReceipt result;
    result.flag = static_cast<int>(PyLong_AsLong(flag));
    result.key = PyLong_AsLong(key);
    result.message = PyUnicode_AsUTF8(msg);

    return result;
}


TaskResult Engine::retrieve(const long key)
{
    auto method = PyUnicode_FromString("retrieve");
    auto py_key = PyLong_FromLong(key);
    auto z = PyObject_CallMethodObjArgs(_engine, method, py_key, nullptr);

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


void Engine::finalize()
{
    if (_initialized && !_finalized) {
        if (_engine) {
            PyObject_CallMethod(_engine, "finalize", nullptr);
        }
        _finalized = true;
    }
}


Engine::~Engine()
{
    this->finalize();
    Py_Finalize();
}
```


Test it in C++
==============

My testing program is listed below.

```
// File `test.cc`.

#include "cc4py.h"

#include <cassert>
#include <chrono>
#include <cmath>
#include <exception>
#include <iostream>
#include <memory>
#include <mutex>
#include <random>
#include <string>
#include <thread>
#include <vector>


std::mutex py_lock;
//std::mutex cout_lock;


struct Datapoint {
    std::vector<double> float_features;
    std::vector<std::string> str_features;
    int int_feature;
};


Datapoint make_point(
    double x, double y, double z,
    std::string a, std::string b,
    int k)
{
    return std::move(
        Datapoint{
            std::vector<double>{x, y, z},
            std::vector<std::string>{a, b},
            k
        }
    );
}


std::vector<Datapoint> make_data(uint n)
{
    std::vector<Datapoint> data;
    data.push_back(make_point(1.1, 11.2, .3, "ab", "cda", 27));
    data.push_back(make_point(2.3, 1.23, 11.3, "abc", "csd", 29));
    data.push_back(make_point(1.88, 51.2, 1.34, "attb", "cdvfsdfv", 7));
    data.push_back(make_point(1.1, 33.2, 1.003, "ab", "cdv11", 127));
    data.push_back(make_point(10.1, 1.2, 2.3, "ab0", "cda88", 270));
    data.push_back(make_point(1.223, 21.2, 3.3, "abcsdf", "cd6", 129));
    data.push_back(make_point(0.88, 1.02, 10.3, "atewwtb", "cd34vv", 72));
    data.push_back(make_point(3.1, 33.2, 4.3, "abaer", "cdaav", 17));
    assert(n == data.size());
    return std::move(data);
}


int answer(const Datapoint & dp)
{
    int z = static_cast<int>(
        fabs(dp.float_features[0]) +
        fabs(dp.float_features[1]) +
        fabs(dp.float_features[2]));
    z += dp.str_features[0].size() + dp.str_features[1].size();
    z += dp.int_feature;
    return z;
}


void wait(int millisec=5)
{
    std::this_thread::sleep_for(std::chrono::milliseconds(millisec));
}


void do_one_thread(
    cc4py::Engine * engine,
    Datapoint const * data_in,
    int * data_out,
    int idx_from,
    int idx_to)
{
    for (int idx=idx_from; idx < idx_to; idx++) {
        long key = 0;
        while (1) {
            py_lock.lock();
            auto receipt = engine->submit(
                data_in[idx].float_features,
                data_in[idx].str_features,
                data_in[idx].int_feature
            );
            py_lock.unlock();
            if (receipt.flag == 0) {
                key = receipt.key;
                break;
            } else if (receipt.flag == 1) {
                wait(3);
            } else {
                std::cout << "submission failed at index " << idx << ": "
                        << receipt.message << std::endl;
                break;
            }
        }
        if (key == 0) {
            data_out[idx] = -1.0;
            continue;
        }

        wait(2);

        while (1) {
            py_lock.lock();
            auto result = engine->retrieve(key);
            py_lock.unlock();
            if (result.flag == 0) {
                data_out[idx] = result.value;
                break;
            } else if (result.flag == 1) {
                wait(3);
            } else {
                std::cout << "retrieval failed at index " << idx << ": "
                        << result.message << std::endl;
                data_out[idx] = -1.0;
                break;
            }
        }
    }
}


int main(int const argc, char const * const * const argv)
{
    int N = 8;
    auto dataset = make_data(N);

    int NTHREADS = 2;
    int idx_from[NTHREADS] = {0, 4};
    int idx_to[NTHREADS] = {4, N};

    std::vector<int> results(N);

    cc4py::Engine engine;

    engine.initialize(
        "{\"a\": 38, \"b\": [2, 3, 4]}",
        std::vector<std::string>{"float1", "float2", "float3"},
        std::vector<std::string>{"str1", "str2"}
    );

    std::vector<std::thread> cc_threads(NTHREADS);
    for (int i=0; i < NTHREADS; i++) {
        cc_threads[i] = std::thread(
            do_one_thread,
            &engine,
            dataset.data(),
            results.data(),
            idx_from[i],
            idx_to[i]);
    }
    for (auto it=cc_threads.begin(); it != cc_threads.end(); ++it) {
        it->join();
    }

    engine.finalize();

    for (int i = 0; i < N; ++i) {
        assert(answer(dataset[i]) == results[i]);
    }

    return 0;
}
```

Here is the `Makefile`:

```
CC = g++
CCFLAGS = -std=c++14 -Wall -O2 -static
INCLUDES = -I./ -I/usr/local/include/python3.6m
LIBS = -lpython3.6m -lpthread


all: test

test: test.o cc4py.o
	$(CC) $(LLFLAGS) $^ $(LIBS) -o $@


%.o: %.cc
	$(CC) -c $(CCFLAGS) $(INCLUDES) $< -o $@

clean:
	rm -f *.o test
```

The test program ran with no issues.


Looking back and forth
======================

Let's look back at the use of Python/C API. While the test program ran with no issues, I knew it's just a start. For one thing, there are typically multiple slightly different functions in the API that do the same thing, therefore I expect the code listed above can be made cleaner.

A far more serious issue is momery management. As a start, my code does not take care of release memory that it allocates for strings. This has to be wrong. There maybe other pitfalls.

It turned out a much, much bigger pitfall is taking care of reference counts for Python. Reading [the section of the offidical doc entitled "Objects, Types and Reference Counts"](https://docs.python.org/3/c-api/intro.html) basically convinced me that I could never get it right (unless I work really hard at it), and the raw API is not the way to go.

In the next post I will explore third-party tools that are built on top of the raw API and make my job much, much, and so much easier.

The code listed in this post is available at [https://github.com/zpz/python/tree/master/py4cc1](https://github.com/zpz/python/tree/master/py4cc1) and [https://github.com/zpz/python/tree/master/cc4py1](https://github.com/zpz/python/tree/master/cc4py1).


