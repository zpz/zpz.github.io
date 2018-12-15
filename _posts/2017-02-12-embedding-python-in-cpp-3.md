---
layout: post
title: Embedding Python in C++, Part 3
excerpt_separator: <!--excerpt-->
tags: [Python, C++]
---

In this post I will explore passing STL containers to Python in a variety of ways. The focus is to sort out how to pass containers by value and by reference.
<!--excerpt-->
The `pybind11` documentation is not totally clear to me on this topic, therefore some of the findings below were obtained via trials.

There are two headers relevant to this topic: `pybind11/stl.h` and `pybind11/stl_bind.h`. `stl.h` is responsible for converting a C++ STL container to a native Python object (such as `list` and `dict`) by copying, whereas `stl_bind.h` is responsible for passing a C++ STL container to Python in a custom class (not the native `list` and `dict`) that provides Pythonic behavior (such as `__getitm__` and `__setitem__`). This custom class "wraps" the C++ STL container and avoids data copying, hence enables "passing by reference" between C++ and Python.



## Python testing code

The Python code below operations on the objects that are passed in from C++. However, this is pure Python code unaware that it is going to be called from C++. This module resides in package `py4cc`. The package is found on `PYTHONPATH` and is located independently of the C++ code that would call it.

```python
# File `stl.py` in package `py4cc`.

def show(x, pp=True):
    s = 'in Python --- ' + str(type(x)) + ': ' + str(x)
    if pp:
        print(s)
    else:
        return s


def cumsum(x):
    # `cumsum` for sequence `x` whose elements have meaningful `+` operation.
    print('before `cumsum`,', show(x, False))
    for i in range(1, len(x)):
        x[i] = x[i-1] + x[i]
    print('after `cumsum`,', show(x, False))
    return x


def mapadd(x):
    # Add 1 to each value of a `dict`,
    # then insert a new element 'total' with the sum of the original elements.
    print('before `mapadd`,', show(x, False))
    total = 0
    for k in x:
        v = x[k]
        total += v
        x[k] = v + 1
    x['total'] = total
    print('after `mappad`,', show(x, False))
    return x
```



## Scenario 1: cast a C++ object to a Python object

I tested two situations. First, explicitly cast a C++ STL container to a Python object, then call the objects Python methods. The code remains in C++; there is no Python packages or modules involved. Second, pass a C++ STL container to a Python function (called in C++ code), then inspect the input argument in the Python code of said function.

Note that `stl.h` is `#include`d. Also not that at one point the Python module listed above is `import`ed.

Code:

```cpp
// File `test_cast.cc` in `cc11bind`.

#include "Python.h"
#include "pybind11/pybind11.h"
#include "pybind11/stl.h" 
    // needed for explicit or implicit `cast` of STL containers.
    // "stl_bind.h" won't work.

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

void test_cast()
{
    std::vector<int> intvec{1, 3, 5};
    std::vector<std::string> strvec{"abc", "def", "this is good"};
    std::map<std::string, double> doublemap{ {"first", 1.1}, {"second", 2.2}, {"third", 3.3} };
    std::tuple<int, double, std::string> misctuple = std::make_tuple(123, 3.1415926, "Captain Cook");

    // explicit cast

    pyprint<>(intvec);
    pyprint<>(strvec);
    pyprint<>(doublemap);
    pyprint<>(misctuple);

    auto z = py::cast(intvec).attr("index")(3);
    auto zz = z.cast<int>();
    std::cout << "3 is at index " << zz << " in " << std::endl;
    assert(zz == 1);

    // implicit cast

    auto show = py::module::import("py4cc2.stl").attr("show");

    std::cout << std::endl;
    show(intvec);
    std::cout << std::endl;
    show(strvec);
    std::cout << std::endl;
    show(doublemap);
    std::cout << std::endl;
    show(misctuple);
}


int main()
{
    Py_Initialize();
    test_cast();
    return 0;
}
```

Output:

```bash
$ ./test_cast 
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

3 is at index 1 in 

in Python --- <class 'list'>: [1, 3, 5]

in Python --- <class 'list'>: ['abc', 'def', 'this is good']

in Python --- <class 'dict'>: {'first': 1.1, 'second': 2.2, 'third': 3.3}

in Python --- <class 'tuple'>: (123, 3.1415926, 'Captain Cook')
```

Observations:

1. `py::cast(x)` converts a C++ object to a Python object: `std::vector` --> `list`, `std::map` --> `dict`, `std::tuple` --> `tuple`.
2. This requires `#include "pybind11/stl.h"`. 
3. The cast produces a Python object. One can then call the object's Python method (such as `__str__` or `__len__`) by obtaining the method via `.attr(method_name)`, followed by the function-call syntax, `method_object(arguments)`.
3. While calling a Python function or method, parameters of basic types may not need an explicit `cast`, like `3` in the call to `__index__`. The context is clear enough so that the system conducts a cast implicitly.
4. With `stl.h` `#include`d, passing a STL container to a Python function does not need an explicit `py::cast(x)`. Casting is done implicitly.
5. Calls to Python methods and functions return Python objects.  Use `python_object.cast<cpp_type>()` to cast a Python object to a C++ object.



## Scenario 2: pass by value

I verified that with `#include "pybind11/stl.h"`, passing is by value, as demonstrated below.

Code:

```cpp
// File `test_copy.cc` in `cc11bind`.

#include "Python.h"
#include "pybind11/pybind11.h"
#include "pybind11/stl.h"

#include "util.h"  // Provides `print_vec`, `print_map`. On github.

#include <iostream>
#include <map>
#include <vector>
#include <string>

namespace py = pybind11;


void test_copy()
{
    std::vector<int> intvec{1, 3, 5};
    std::vector<std::string> strvec{"abc", "def", "this is good"};
    std::map<std::string, double> doublemap{ {"first", 1.1}, {"second", 2.2}, {"third", 3.3} };

    auto module = py::module::import("py4cc2.stl");
    auto cumsum = module.attr("cumsum");
    auto mapadd = module.attr("mapadd");

    std::cout << "before `cumsum`, in C++ --- ";
    print_vec<>(intvec);
    cumsum(intvec);
    std::cout << "after `cumsum`, in C++ --- ";
    print_vec<>(intvec);

    std::cout << std::endl;
    std::cout << "before `cumsum`, in C++ --- ";
    print_vec<>(strvec);
    cumsum(strvec);
    std::cout << "after `cumsum`, in C++ --- ";
    print_vec<>(strvec);

    std::cout << std::endl;
    std::cout << "before `mapadd`, in C++ --- ";
    print_map<>(doublemap);
    mapadd(doublemap);
    std::cout << "after `mapadd`, in C++: ";
    print_map<>(doublemap);
    std::cout << std::endl;
}


int main()
{
    Py_Initialize();
    test_copy();
    return 0;
}
```

Output:

```bash
$ ./test_copy 
before `cumsum`, in C++ --- [1, 3, 5]
before `cumsum`, in Python --- <class 'list'>: [1, 3, 5]
after `cumsum`, in Python --- <class 'list'>: [1, 4, 9]
after `cumsum`, in C++ --- [1, 3, 5]

before `cumsum`, in C++ --- [abc, def, this is good]
before `cumsum`, in Python --- <class 'list'>: ['abc', 'def', 'this is good']
after `cumsum`, in Python --- <class 'list'>: ['abc', 'abcdef', 'abcdefthis is good']
after `cumsum`, in C++ --- [abc, def, this is good]

before `mapadd`, in C++ --- {first:1.1, second:2.2, third:3.3}
before `mapadd`, in Python --- <class 'dict'>: {'first': 1.1, 'second': 2.2, 'third': 3.3}
after `mappad`, in Python --- <class 'dict'>: {'first': 2.1, 'second': 3.2, 'third': 4.3, 'total': 6.6}
after `mapadd`, in C++: {first:1.1, second:2.2, third:3.3}
```

Observations:

1. The passing is by value. While the Python function modifies the object that is passed in, the original object in C++ remains unchanged.
2. The input argument on the Python side is "native" types like `list` and `dict`. This nativeness is only possible with value copying.



## Binding code to enable pass-by-reference

In order to pass a C++ STL container to Python "by reference", one needs to "bind" the container type of interest to a certain Python class (define in C++ using `pybind11` machinery), which "wraps" the STL container without data copying, and provides methods expected by Python code.

`pybind11` provides these binding classes. One needs to expose these classes to Python in a Python module that is written in C++. The following is my module for this purpose. The file is located in the package `py4cc2`. Compile this file to create a shared library in the package folder, and `import` it in Python or C++.

```cpp
// File `_cc11binds.cc` in package `py4cc`.

// Compile:
// c++ -O3 -shared -std=c++11 -fPIC -I /usr/local/include/python3.6m \
//      `python-config --cflags --ldflags` _cc11binds.cc -o _cc11binds.so

#include <pybind11/pybind11.h>
#include <pybind11/stl_bind.h>

#include <map>
#include <string>
#include <vector>

namespace py = pybind11;

PYBIND11_PLUGIN(_cc11binds) {
    py::module m("_cc11binds", "C++ type bindings created by py11bind");
    py::bind_vector<std::vector<int>>(m, "IntVector");
    py::bind_vector<std::vector<std::string>>(m, "StringVector");
    py::bind_map<std::map<std::string, double>>(m, "StringDoubleMap");
    // 'bind_map` does not create method `values`.

    return m.ptr();
}
```


## Scenario 3: pass by reference, but watch out!

The code below demonstrates various combinations of `#include "pybind11/stl.h"` and `#include "pybind11/stl_bind.h"`, with variables passed by name (`x`) or by pointer (`&x`). It often demonstrates the (non)effect of `const` on objects that get passed to Python.

Code:

```cpp
// File `test_ref` in `cc11bind`.

#include "Python.h"
#include "pybind11/pybind11.h"

#include "util.h"

#include <iostream>
#include <map>
#include <string>
#include <vector>

namespace py = pybind11;


void test_const(const std::vector<int> & intvec, py::object cumsum)
{
    std::cout << "before `cumsum`, in C++ --- ";
    print_vec<>(intvec);
    cumsum(&intvec);
    std::cout << "after `cumsum`, in C++ --- ";
    print_vec<>(intvec);
}


void test_noconst(std::vector<int> & intvec, py::object cumsum)
{
    std::cout << "before `cumsum`, in C++ --- ";
    print_vec<>(intvec);
    cumsum(&intvec);
    std::cout << "after `cumsum`, in C++ --- ";
    print_vec<>(intvec);
}


void test_ref()
{
    std::vector<int> intvec{1, 3, 5};
    std::vector<std::string> strvec{"abc", "def", "this is good"};
    std::map<std::string, double> doublemap{ {"first", 1.1}, {"second", 2.2}, {"third", 3.3} };

    auto module = py::module::import("py4cc2.stl");
    py::module::import("py4cc2._cc11binds");        // ATTENTION! Python import.
    auto cumsum = module.attr("cumsum");
    auto mapadd = module.attr("mapadd");

    // Pass pointers.

    std::cout << "=== pass as `&x` ===" << std::endl;
    
    std::cout << std::endl;
    std::cout << "before `cumsum`, in C++ --- ";
    print_vec<>(intvec);
    cumsum(&intvec);
    std::cout << "after `cumsum`, in C++ --- ";
    print_vec<>(intvec);

    std::cout << std::endl;
    std::cout << "before `cumsum`, in C++ --- ";
    print_vec<>(strvec);
    cumsum(&strvec);
    std::cout << "after `cumsum`, in C++ --- ";
    print_vec<>(strvec);

    std::cout << std::endl;
    std::cout << "before `mapadd`, in C++ --- ";
    print_map<>(doublemap);
    mapadd(&doublemap);
    std::cout << "after `mapadd`, in C++: ";
    print_map<>(doublemap);


    // Pass values.
    std::cout << std::endl << "=== pass as `x`===" << std::endl;

    std::cout << std::endl;
    std::cout << "before `cumsum`, in C++ --- ";
    print_vec<>(intvec);
    cumsum(intvec);
    std::cout << "after `cumsum`, in C++ --- ";
    print_vec<>(intvec);

    std::cout << std::endl;
    std::cout << "before `cumsum`, in C++ --- ";
    print_vec<>(strvec);
    cumsum(strvec);
    std::cout << "after `cumsum`, in C++ --- ";
    print_vec<>(strvec);

    std::cout << std::endl;
    std::cout << "before `mapadd`, in C++ --- ";
    print_map<>(doublemap);
    mapadd(doublemap);
    std::cout << "after `mapadd`, in C++: ";
    print_map<>(doublemap);

    // Test constness violation
    std::cout << std::endl << "=== test const ===" << std::endl << std::endl;
    test_const(intvec, cumsum);

    std::cout << std::endl << "=== test noconst ===" << std::endl << std::endl;
    test_noconst(intvec, cumsum);
}


int main()
{
    Py_Initialize();
    test_ref();
    return 0;
}
```

Output:

```bash
$ ./test_ref 
=== pass as `&x` ===

before `cumsum`, in C++ --- [1, 3, 5]
before `cumsum`, in Python --- <class 'py4cc2._cc11binds.IntVector'>: IntVector[1, 3, 5]
after `cumsum`, in Python --- <class 'py4cc2._cc11binds.IntVector'>: IntVector[1, 4, 9]
after `cumsum`, in C++ --- [1, 4, 9]

before `cumsum`, in C++ --- [abc, def, this is good]
before `cumsum`, in Python --- <class 'py4cc2._cc11binds.StringVector'>: StringVector[abc, def, this is good]
after `cumsum`, in Python --- <class 'py4cc2._cc11binds.StringVector'>: StringVector[abc, abcdef, abcdefthis is good]
after `cumsum`, in C++ --- [abc, abcdef, abcdefthis is good]

before `mapadd`, in C++ --- {first:1.1, second:2.2, third:3.3}
before `mapadd`, in Python --- <class 'py4cc2._cc11binds.StringDoubleMap'>: StringDoubleMap{first: 1.1, second: 2.2, third: 3.3}
after `mappad`, in Python --- <class 'py4cc2._cc11binds.StringDoubleMap'>: StringDoubleMap{first: 2.1, second: 3.2, third: 4.3, total: 6.6}
after `mapadd`, in C++: {first:2.1, second:3.2, third:4.3, total:6.6}

=== pass as `x`===

before `cumsum`, in C++ --- [1, 4, 9]
before `cumsum`, in Python --- <class 'py4cc2._cc11binds.IntVector'>: IntVector[1, 4, 9]
after `cumsum`, in Python --- <class 'py4cc2._cc11binds.IntVector'>: IntVector[1, 5, 14]
after `cumsum`, in C++ --- [1, 4, 9]

before `cumsum`, in C++ --- [abc, abcdef, abcdefthis is good]
before `cumsum`, in Python --- <class 'py4cc2._cc11binds.StringVector'>: StringVector[abc, abcdef, abcdefthis is good]
after `cumsum`, in Python --- <class 'py4cc2._cc11binds.StringVector'>: StringVector[abc, abcabcdef, abcabcdefabcdefthis is good]
after `cumsum`, in C++ --- [abc, abcdef, abcdefthis is good]

before `mapadd`, in C++ --- {first:2.1, second:3.2, third:4.3, total:6.6}
before `mapadd`, in Python --- <class 'py4cc2._cc11binds.StringDoubleMap'>: StringDoubleMap{first: 2.1, second: 3.2, third: 4.3, total: 6.6}
after `mappad`, in Python --- <class 'py4cc2._cc11binds.StringDoubleMap'>: StringDoubleMap{first: 3.1, second: 4.2, third: 5.3, total: 16.2}
after `mapadd`, in C++: {first:2.1, second:3.2, third:4.3, total:6.6}

=== test const ===

before `cumsum`, in C++ --- [1, 4, 9]
before `cumsum`, in Python --- <class 'py4cc2._cc11binds.IntVector'>: IntVector[1, 4, 9]
after `cumsum`, in Python --- <class 'py4cc2._cc11binds.IntVector'>: IntVector[1, 5, 14]
after `cumsum`, in C++ --- [1, 5, 14]

=== test noconst ===

before `cumsum`, in C++ --- [1, 5, 14]
before `cumsum`, in Python --- <class 'py4cc2._cc11binds.IntVector'>: IntVector[1, 5, 14]
after `cumsum`, in Python --- <class 'py4cc2._cc11binds.IntVector'>: IntVector[1, 6, 20]
after `cumsum`, in C++ --- [1, 6, 20]
```

Observations:

1. The Python module `_cc11binds` created in the last section is `import`ed.
2. The C++ code does not `#include "pybind11/stl_bind.h"`, nor `#include "pybind11/stl.h"`.
3. When a STL container is passed by pointer (`&x`), the passing is by reference. Modifications to the input argument in Python functions are reflected in the original C++ object that got passed to Python.
4. However, if the STL container is passed by name (`x`), the passing is by value, although the input argument is of the custom class rather than native Python container types.

More observations (code modification and output are not show):

1. With `#include "pybind11/stl.h"`, the passing is by value, be the variable passed by name (`x`) or pointer (`&x`), even if `_cc11binds` is `import`ed.
2. In this case, the passed-in object in the Python function is of native Python types like `list` and `dict`.

This seems to suggest that `import _c11binds` without `#include "pybind11/stl.h"` provides the best of both worlds: pass by value with `x`, and by reference with `&x`. However, notice that the passed-in object in Python is not of a built-in type, and the custom type does not need to provide the complete interface of the built-in type. Indeed, the bound `map` (corresponding to Python `dict`) does not have the `values()` method, for example. Moreover, passing-by-reference has restrictions in relation to named arguments; see the next section.

Observations regarding C++ const correctness:

1. If a "const" container is passed to Python by reference, then it can be modified in Python and the modifications are reflected in the original C++ container. In other words, "const-ness" is ignored. This is undesirable. Hopefully this will be fixed in the future.



## Scenario 4: pass-by-reference in named arguments


Code:

```cpp
// File `test_kwargs.cc` in `cc11bind`.

#include "Python.h"
#include "pybind11/pybind11.h"

#include "util.h"

#include <iostream>
#include <map>
#include <string>
#include <vector>

namespace py = pybind11;


void test_ref()
{
    std::vector<int> intvec{1, 3, 5};

    auto module = py::module::import("py4cc2.stl");
    py::module::import("py4cc2._cc11binds");
    auto cumsum = module.attr("cumsum");

    auto kwargs = py::dict(py::arg("x") = intvec);
    std::cout << "before `cumsum`, in C++ --- ";
    print_vec<>(intvec);
    cumsum(**kwargs);
    std::cout << "after `cumsum`, in C++ --- ";
    print_vec<>(intvec);

    std::cout << std::endl;
    std::cout << "before `cumsum`, in C++ --- ";
    print_vec<>(intvec);
    cumsum(py::arg("x") = intvec);
    std::cout << "after `cumsum`, in C++ --- ";
    print_vec<>(intvec);

    std::cout << std::endl;
    std::cout << "before `cumsum`, in C++ --- ";
    print_vec<>(intvec);
    cumsum(**py::dict(py::arg("x") = intvec));
    std::cout << "after `cumsum`, in C++ --- ";
    print_vec<>(intvec);

    auto args = py::list();
    args.append(intvec);
    std::cout << std::endl;
    std::cout << "before `cumsum`, in C++ --- ";
    print_vec<>(intvec);
    cumsum(*args);
    std::cout << "after `cumsum`, in C++ --- ";
    print_vec<>(intvec);

    auto args2 = py::list();
    args2.append(&intvec);
    std::cout << std::endl;
    std::cout << "before `cumsum`, in C++ --- ";
    print_vec<>(intvec);
    cumsum(*args2);
    std::cout << "after `cumsum`, in C++ --- ";
    print_vec<>(intvec);
}


int main()
{
    Py_Initialize();
    test_ref();
    return 0;
}
```

Output:

```bash
$ ./test_kwargs 
before `cumsum`, in C++ --- [1, 3, 5]
before `cumsum`, in Python --- <class 'py4cc2._cc11binds.IntVector'>: IntVector[1, 3, 5]
after `cumsum`, in Python --- <class 'py4cc2._cc11binds.IntVector'>: IntVector[1, 4, 9]
after `cumsum`, in C++ --- [1, 3, 5]

before `cumsum`, in C++ --- [1, 3, 5]
before `cumsum`, in Python --- <class 'py4cc2._cc11binds.IntVector'>: IntVector[1, 3, 5]
after `cumsum`, in Python --- <class 'py4cc2._cc11binds.IntVector'>: IntVector[1, 4, 9]
after `cumsum`, in C++ --- [1, 3, 5]

before `cumsum`, in C++ --- [1, 3, 5]
before `cumsum`, in Python --- <class 'py4cc2._cc11binds.IntVector'>: IntVector[1, 3, 5]
after `cumsum`, in Python --- <class 'py4cc2._cc11binds.IntVector'>: IntVector[1, 4, 9]
after `cumsum`, in C++ --- [1, 3, 5]

before `cumsum`, in C++ --- [1, 3, 5]
before `cumsum`, in Python --- <class 'py4cc2._cc11binds.IntVector'>: IntVector[1, 3, 5]
after `cumsum`, in Python --- <class 'py4cc2._cc11binds.IntVector'>: IntVector[1, 4, 9]
after `cumsum`, in C++ --- [1, 3, 5]

before `cumsum`, in C++ --- [1, 3, 5]
before `cumsum`, in Python --- <class 'py4cc2._cc11binds.IntVector'>: IntVector[1, 3, 5]
after `cumsum`, in Python --- <class 'py4cc2._cc11binds.IntVector'>: IntVector[1, 4, 9]
after `cumsum`, in C++ --- [1, 4, 9]
```

Observations (some code and output are not shown):

1. Pass-by-reference does not work in named arguments, i.e, the following do not work:

   ```cpp
   auto kwargs = py::dict(py::arg("x") = &x);
   f(**kwargs);

   f(py::arg("x") = &x);
   ```
   
   This usage causes memory errors at destruction time.
  
   If the `&x` above is replaced by `x`, the code will work but that passes by value.

2. Pass-by-reference works in "star" arguments

   ```cpp
   auto args = py::list();
   args.append(&x);
   f(*args);
   ```
   
   This works, and passes by reference. Previously I have demonstrated that pass-by-reference works in direct positional arguments.

Digging deeper, the two failing scenarios, namely `py::dict(py::arg("x") = &x)`  and `f(py::arg("x") = &x)` can be explained. My understanding is that the two are the same issue, and what's going on is as follows:

1. The construct `py::arg("x") = &x)` pulls `x` into an Python object. In the first scenario it's the Python `dict` being defined, whereas in the second it's a temporary object in the function call.
2. When this Python object goes out of scope (which happens in the C++ code), in the eyes of Python the reference count of `x` reduces to 0, hence Python frees `x`. The reference count reduction is performed in the destructor of the C++ object that is defined by `pybind11` for `py::arg("x") = &x`.
3. However, when `x` goes out of scope in the C++ code, C++ frees its memory as well. I do not know which of Python and C++ frees it first, but double free of this one object crashes the program.

The solution is to tell `pybind11` "I am giving you a *reference*; the object's life-time is managed in C++; do not decrease its reference count in Python, instead make sure it is just a pass-through in Python". The code in both scenarios is to replace `&x` by `py::cast(&x, py::return_value_policy::reference)`.

`pybind11` provides methods `ref_count`, `inc_ref`, and `dec_ref` for every `py::object` instance. If you are in the mode of digging, these methods can show you the dynamics.

<!-- The complete code is available at [https://github.com/zpz/cppy/tree/master/py4cc](https://github.com/zpz/cppy/tree/master/py4cc/) and [https://github.com/zpz/cppy/tree/master/cc11bind](https://github.com/zpz/cppy/tree/master/cc11bind/). -->
