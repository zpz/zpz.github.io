---
layout: post
title: Embedding Python in C++, Part 2
tags: [Python, C++]
---

In my search for an alternative to raw Python/C API for embedding Python in C++, I had several requirements:

1. It must have good support for C++ (i.e. not just C).
2. It must provide good control and flexibility. This is a little vague, but what this means is that I like step-by-step programming with full understanding; I don't like tools that give me a black-box feel of magic.
3. It must handle Python exceptions smoothly, meaning un-captured exceptions in my Python code should propagate into the C++ calling side as C++ exceptions. This way, I may choose to raise exceptions in Python instead of returning error codes.
4. Big plus if it's possible to pass data from C++ to Python by reference. I did not have an immediate need to modify an object in Python and have the modification visible in C++, but I would like to be able to pass read-only data without copying.
5. The tool should be an actively and seriously developed and maintained open source project with reasonable traction.

I found several tools that enjoyed decent attention, including

- [cffi](http://cffi.readthedocs.io/en/latest/)
- [cython](cython.org)
- [pybind11](https://github.com/pybind/pybind11)
- [PyCXX](cxx.sourceforge.net)

The documentation for all these tools give much more attention to "extending" than "embedding". It may not be easy to find out how to start embedding Python using these tools.

I did find explicit mentions to "embedding" using [cffi](http://cffi.readthedocs.io/en/latest/embedding.html) and [cython](https://github.com/cython/cython/wiki/EmbeddingCython). A quick look at the descriptions suggested that these two options did not meet the 2nd requirement listed above.

The [documentation for PyCXX](http://cxx.sourceforge.net/PyCXX-Python3.html) is clean and to the point. I found such section titles (and content) as ["We avoid programming with Python object pointers"](http://cxx.sourceforge.net/PyCXX-Python3.html#h2_no_pointers) and ["The basic concept is to wrap Python pointers"](http://cxx.sourceforge.net/PyCXX-Python3.html#h2_basic_concepts) enlightening. Unfortunately, I decided to skip this tool without actually trying it, for reasons in its model of development and distribution. The code is distributed on [`sourceforge.net`](http://cxx.sourceforge.net/) as a tar-ball. It appears to have a sole maintainer and developer. The revision history indicates it's being steadily maintained and developped. Beyond that, I did not get to see a list of issues, frequency of commits, level of traction, plan for future development, and so on. These are unfortunate reasons for skipping a capable tool, and it seems to be one.

So I ended up trying `pybind11`. This is a successor to `boost.python`. The codebase is substantial, development activity is high, level of interest is good (1771 stars on `github` as of today). The [list of features](https://github.com/pybind/pybind11/blob/master/README.md)
are **very** impressive. The documentation put a lot of emphasis on "object-oriented code", such as defining a class in C++, and inheriting it in Python, and things even fancier than that. This did feel overkill for my need at the moment and taste in general. As can be seen in 
[part 1]({{ site.baseurl }}/blog/embedding-python-in-cpp-1/) (class `Driver` to be specific), I preferred to have a rather clean separation of the Python and C++ codes, with a small API in between. But my idea could change if I ended up learning and using this tool a lot.

The [documentation](http://pybind11.readthedocs.io/en/master/) is serious but still incomplete. It took some imagination and trials to fill the gaps. I managed to make it work for me.

The C++ header file is identical to the first version in 
[part 1]({{ site.baseurl }}/blog/embedding-python-in-cpp-1/) except for two lines. One is an extra include

```cpp
#include "pybind11/pybind11.h"
```

The other is replacing

```cpp
PyObject * _engine = nullptr;
```

by

```cpp
py::object _engine;
```

This is an important difference: with Python/C API, we work with bare C pointers; with `pybind11`, we work with `C++` objects that manages bare pointers for us behind the scene.

The source file is across-the-board cleaner and simpler than the previous version. Here is the method `initialize`:

```cpp
void Driver::initialize(
    std::string const & model_config_json,
    std::vector<std::string> const & float_feature_names,
    std::vector<std::string> const & str_feature_names
    )
{
    auto driver_cls = py::module::import("py4cc1.py4cc").attr("Driver");
    _driver = driver_cls(py::arg("max_tasks") = _max_tasks);

    auto kwargs = py::dict(
        py::arg("config_json") = py::cast(model_config_json),
        py::arg("float_feature_names") = float_feature_names,
        py::arg("str_feature_names") = str_feature_names
    );
    _driver.attr("initialize")(**kwargs);

    _initialized = true;
}
```

Note how the tedious type conversions (`PyLong_FromLong` and the like) are gone or much simplified with a `py::cast(...)`. Also note the use of named arguments. C++ vectors are converted to Python lists automatically. The fact that one can use the `**kwargs` syntax is supprisingly delightful. Calling a method involves getting the method as an attribute (`_driver.attr("initialize")`), then calling the returned callable object directly (`_driver.attr("initialize")(**kwargs)`). No more `PyObject_Call(...)` gymnastics.

The method `retrieve` goes like this:

```cpp
TaskResult Driver::retrieve(const long key)
{
    py::tuple z = _driver.attr("retrieve")(key);

    TaskResult result;
    result.flag = z[0].cast<int>();
    result.key = key;
    result.value = z[1].cast<int>();
    result.message = z[2].cast<std::string>();

    return result;
}
```

Here we see tuple items are accessed in the natural way (`z[0]`) instead of `PyTuple_GetItem(z, 0)`, and Python-to-C++ conversion uses the pattern `py_object.cast<target_type>()`.

<!-- The complete code is available at [https://github.com/zpz/cppy/tree/master/cc4py/cc4py_2.cc](https://github.com/zpz/cppy/tree/master/cc4py/cc4py_2.cc). Please feel the contrast yourself with the older version at [https://github.com/zpz/cppy/tree/master/cc4py/cc4py_1.cc](https://github.com/zpz/cppy/tree/master/cc4py/cc4py_1.cc). -->

In the next post I will explore various ways of passing C++ STL containers to Python.

