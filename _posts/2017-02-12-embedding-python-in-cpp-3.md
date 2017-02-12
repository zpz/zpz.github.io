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
# File `stl.py` in package `py4cc2`.

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
    # then insert a new lement 'total': total.
    print('before `mapadd`,', show(x, False))
    #total = sum(x.values())
    total = 0
    for k in x:
        v = x[k]
        total += v
        x[k] = v + 1
    x['total'] = total
    print('after `mappad`,', show(x, False))
    return x
```



# Scenario 1: cast a C++ object to a Python object

Code:

```
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
    std::map<std::string, double> doublemap{{"first", 1.1}, {"second", 2.2}, {"third", 3.3}};
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

```
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
2. This requires `#include "pybind11/std.h"`. 
3. The cast produces a Python object. One can then call the objec'ts Python method (such as `__str__` or `__len__`) by obtaining the method via `.attr(method_name)`, followed by the function-call syntax, `method_object(arguments)`.
3. While calling a Python function (or method), parameters of basic types may not need an explicit `cast`, like `3` in the call to `__index__`. The context is clear enough so that the system may be conducting a cast implicitly.
4. Calls to Python methods and functions produce Python objects.  Use `python_object.cast<cpp_type>()` to cast a Python object to a C++ object.


# Scenario 2:

Code:

```
#include "Python.h"
#include "pybind11/pybind11.h"
#include "pybind11/stl.h"

#include "util.h"

#include <iostream>
#include <map>
#include <vector>
#include <string>

namespace py = pybind11;


void test_copy()
{
    std::vector<int> intvec{1, 3, 5};
    std::vector<std::string> strvec{"abc", "def", "this is good"};
    std::map<std::string, double> doublemap{{"first", 1.1}, {"second", 2.2}, {"third", 3.3}};

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

```
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


# "Opaque" object binding

```
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


# Scenario 3

Code:

```
#include "Python.h"
#include "pybind11/pybind11.h"
//#include "pybind11/stl.h"
    // incuding this will enforce passing by value, even if using `&x` to pass

// Do not `#include stl.h`,
// and do `import _cc11binds`,
// then `&x` will pass by ref, but `x` wil still pass by value.
// STL containers become custom type on Python side.

// Do `#include stl.h`,
// then `&x` and `x` will both pass by value,
// even with `import _cc11binds`.
// STL containers become native Python `list`, `dict`, etc on Python side.

// With
// `const std::vector<..> & x`
// Passing `&x` into Python will ignore `const`.

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
    std::map<std::string, double> doublemap{{"first", 1.1}, {"second", 2.2}, {"third", 3.3}};

    auto module = py::module::import("py4cc2.stl");
    py::module::import("py4cc2._cc11binds");
    auto cumsum = module.attr("cumsum");
    auto mapadd = module.attr("mapadd");

    // Pass pointers.

    std::cout << "=== pass as `&x` ===" << std::endl;

    std::cout << std::endl;
    test_noconst(intvec, cumsum);
    
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

```
$ ./test_ref 
=== pass as `&x` ===

before `cumsum`, in C++ --- [1, 3, 5]
before `cumsum`, in Python --- <class 'py4cc2._cc11binds.IntVector'>: IntVector[1, 3, 5]
after `cumsum`, in Python --- <class 'py4cc2._cc11binds.IntVector'>: IntVector[1, 4, 9]
after `cumsum`, in C++ --- [1, 4, 9]
before `cumsum`, in C++ --- [1, 4, 9]
before `cumsum`, in Python --- <class 'py4cc2._cc11binds.IntVector'>: IntVector[1, 4, 9]
after `cumsum`, in Python --- <class 'py4cc2._cc11binds.IntVector'>: IntVector[1, 5, 14]
after `cumsum`, in C++ --- [1, 5, 14]

before `cumsum`, in C++ --- [abc, def, this is good]
before `cumsum`, in Python --- <class 'py4cc2._cc11binds.StringVector'>: StringVector[abc, def, this is good]
after `cumsum`, in Python --- <class 'py4cc2._cc11binds.StringVector'>: StringVector[abc, abcdef, abcdefthis is good]
after `cumsum`, in C++ --- [abc, abcdef, abcdefthis is good]

before `mapadd`, in C++ --- {first:1.1, second:2.2, third:3.3}
before `mapadd`, in Python --- <class 'py4cc2._cc11binds.StringDoubleMap'>: StringDoubleMap{first: 1.1, second: 2.2, third: 3.3}
after `mappad`, in Python --- <class 'py4cc2._cc11binds.StringDoubleMap'>: StringDoubleMap{first: 2.1, second: 3.2, third: 4.3, total: 6.6}
after `mapadd`, in C++: {first:2.1, second:3.2, third:4.3, total:6.6}

=== pass as `x`===

before `cumsum`, in C++ --- [1, 5, 14]
before `cumsum`, in Python --- <class 'py4cc2._cc11binds.IntVector'>: IntVector[1, 5, 14]
after `cumsum`, in Python --- <class 'py4cc2._cc11binds.IntVector'>: IntVector[1, 6, 20]
after `cumsum`, in C++ --- [1, 5, 14]

before `cumsum`, in C++ --- [abc, abcdef, abcdefthis is good]
before `cumsum`, in Python --- <class 'py4cc2._cc11binds.StringVector'>: StringVector[abc, abcdef, abcdefthis is good]
after `cumsum`, in Python --- <class 'py4cc2._cc11binds.StringVector'>: StringVector[abc, abcabcdef, abcabcdefabcdefthis is good]
after `cumsum`, in C++ --- [abc, abcdef, abcdefthis is good]

before `mapadd`, in C++ --- {first:2.1, second:3.2, third:4.3, total:6.6}
before `mapadd`, in Python --- <class 'py4cc2._cc11binds.StringDoubleMap'>: StringDoubleMap{first: 2.1, second: 3.2, third: 4.3, total: 6.6}
after `mappad`, in Python --- <class 'py4cc2._cc11binds.StringDoubleMap'>: StringDoubleMap{first: 3.1, second: 4.2, third: 5.3, total: 16.2}
after `mapadd`, in C++: {first:2.1, second:3.2, third:4.3, total:6.6}

=== test const ===

before `cumsum`, in C++ --- [1, 5, 14]
before `cumsum`, in Python --- <class 'py4cc2._cc11binds.IntVector'>: IntVector[1, 5, 14]
after `cumsum`, in Python --- <class 'py4cc2._cc11binds.IntVector'>: IntVector[1, 6, 20]
after `cumsum`, in C++ --- [1, 6, 20]

=== test noconst ===

before `cumsum`, in C++ --- [1, 6, 20]
before `cumsum`, in Python --- <class 'py4cc2._cc11binds.IntVector'>: IntVector[1, 6, 20]
after `cumsum`, in Python --- <class 'py4cc2._cc11binds.IntVector'>: IntVector[1, 7, 27]
after `cumsum`, in C++ --- [1, 7, 27]
```


# Scenario 4


Code:

```
#include "Python.h"
#include "pybind11/pybind11.h"

// Calling by ref has trouble with passing by keywords.
// The following will not work:
//
//  auto kwargs = py::dict(py::arg("x") = &x);
//  f(**kwargs);
//
//  f(py::arg("x") = &x);
//
// However, this works:
//
//  auto args = py::list();
//  args.append(&x);
//  f(*args);
// This will pass `x` by reference.

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

```
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




The complete code is available at [https://github.com/zpz/python/tree/master/cc4py2/cc4py.cc](https://github.com/zpz/python/tree/master/cc4py2/cc4py.cc). Please feel the contrast yourself with the older version at [https://github.com/zpz/python/tree/master/cc4py1/cc4py.cc](https://github.com/zpz/python/tree/master/cc4py1/cc4py.cc).

In the next post I will show how to pass a C++ vector to Python, where it behaviors like a Python list, without copying.
