---
layout: post
title: Embedding Python in C++, Part 3
---

In this post I will explore passing STL containers to Python in a variety of ways. The focus is to sort out how to pass containers by value and by reference. The `pybind11` documentation is limited, and not totally clear to me on this topic, therefore many of the findings below were obtained via trials. Some things turned out simpler than what the documentation suggests. 

Note that I do not know whether and how the behaviors listed below will change as the project evolves. 

There are two headers relavent to this topic: `pybind11/stl.h` and `pybind11/stl_bind.h`.



# Python testing code

Below is the Python code I used to test operations on the objects passed in from C++. This is pure Python code with no hint that it is going to be called from C++. This is the sole module, besides `__init__.py`, in package `py4cc2`. The package is found on `PYTHONPATH` and is located independently of the C++ code that would call it.

```
to be pasted
```



# Scenario 1: explicitly cast a C++ object to a Python object

Code:

```
// File `test_basic_1.cc`.

#include "Python.h"
#include "pybind11/pybind11.h"
#include "pybind11/stl.h"  // needed for explicit call of `cast` on STL containers; "stl_bind.h" won't work.

#include <cassert>
#include <iostream>
#include <map>
#include <string>
#include <tuple>
#include <vector>

namespace py = pybind11;

template<typename T>
void pyprint(const T& x)
{
    py::object y = py::cast(x);
    std::cout 
        << "Python type: " << y.attr("__class__").attr("__name__").cast<std::string>()
        << std::endl
        << "  __repr__: " << y.attr("__repr__")().cast<std::string>()
        << std::endl
        << "  __str__: " << y.attr("__str__")().cast<std::string>()
        << std::endl
        << "  __len__: " << y.attr("__len__")().cast<int>()
        << std::endl << std::endl;
}

void test_basic()
{
    std::vector<int> intvec{1, 3, 5};
    std::vector<std::string> strvec{"abc", "def", "this is good"};
    std::map<std::string, double> doublemap{{"first", 1.1}, {"second", 2.2}, {"third", 3.3}};
    std::tuple<int, double, std::string> misctuple = std::make_tuple(123, 3.1415926, "Captain Cook");

    pyprint<>(intvec);
    pyprint<>(strvec);
    pyprint<>(doublemap);
    pyprint<>(misctuple);

    auto z = py::cast(intvec).attr("index")(3);
    auto zz = z.cast<int>();
    std::cout << " 3 is at index " << zz << std::endl;
    assert(zz == 1);
}


int main()
{
    Py_Initialize();
    test_basic();
    return 0;
}
```


Output:

```
$ ./test_basic_1 
Python type: list
  __repr__: [1, 3, 5]
  __str__: [1, 3, 5]
  __len__: 3

Python type: list
  __repr__: ['abc', 'def', 'this is good']
  __str__: ['abc', 'def', 'this is good']
  __len__: 3

Python type: dict
  __repr__: {'first': 1.1, 'second': 2.2, 'third': 3.3}
  __str__: {'first': 1.1, 'second': 2.2, 'third': 3.3}
  __len__: 3

Python type: tuple
  __repr__: (123, 3.1415926, 'Captain Cook')
  __str__: (123, 3.1415926, 'Captain Cook')
  __len__: 3

 3 is at index 1
```

Observations:

1. `py::cast(x)` converts a C++ object to a Python object: `std::vector` --> `list`, `std::map` --> `dict`, `std::tuple` --> `tuple`.
2. This requires `#include "pybind11/std.h"`. 
3. The cast produces a Python object. One can then call the objec'ts Python method (such as `__str__` or `__len__`) by obtaining the method via `.attr(method_name)`, followed by the function-call syntax, `method_object(arguments)`.
3. While calling a Python function (or method), parameters of basic types may not need an explicit `cast`, like `3` in the call to `__index__`. The context is clear enough so that the system may be conducting a cast implicitly.
4. Calls to Python methods and functions produce Python objects.  Use `python_object.cast<cpp_type>()` to cast a Python object to a C++ object.


# Scenario 2:


The complete code is available at [https://github.com/zpz/python/tree/master/cc4py2/cc4py.cc](https://github.com/zpz/python/tree/master/cc4py2/cc4py.cc). Please feel the contrast yourself with the older version at [https://github.com/zpz/python/tree/master/cc4py1/cc4py.cc](https://github.com/zpz/python/tree/master/cc4py1/cc4py.cc).

In the next post I will show how to pass a C++ vector to Python, where it behaviors like a Python list, without copying.
