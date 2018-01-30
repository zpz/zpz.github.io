---
layout: post
title: Python, C++, Pybind11, and string_view
---

The other day I was using the excellent [pybind11](https://github.com/pybind/pybind11) to bridge some Python code and C++ code.
The C++ code was performance critical, hence I used `string_view` (standardized in C++17) to avoid copying wherever possible.
While calling this C++ code from Python via `pybind11` bindings, some baffling behaviors were observed related to the use of `string_view`.

The (simplified) C++ code started like this:


```C++
// File '_example.cc'.

#include <pybind11/pybind11.h>
#include <pybind11/stl.h>

#include <iostream>
#include <string>
#include <variant>
#include <vector>
#include <experimental/string_view>

using string_view = std::string_view;
using string = std::string;


class Item {
    public:
        Item(string_view value) : _value{value} {}

        string_view value() const {
            return _value;
        }

    private:
        string_view _value;
};


class Row {
    public:
        Row(std::vector<Item> items) : _items(items) {}

        void print() const {
            int i = 0;
            for (auto const & v : _items) {
                if (i > 0) std::cout << ", ";
                std::cout << v.value();
                i++;s
            }
            std::cout << std::endl;
        }

    private:
        std::vector<Item> _items;
};



namespace py = pybind11;


PYBIND11_MODULE(_example, m)
{
    py::class_<Item>(m, "Item")
        .def(py::init<string_view>())
        .def_property_readonly("value", &Item::value);

    py::class_<Row>(m, "Row")
        .def(py::init<std::vector<Item>>())
        .def("print", &Row::print);
}
```

The header file `pybind11/stl.h` is included to perform automatic conversion from a Python `list` to C++ `vector`
in calls to the `Row` constructor; otherwise it does not play any role in the behaviors we'll discuss below.

Let's use the binding to the class `Item` first, in a Python interpreter:
 
```
>>> from _example import Item
>>> x = 'abcd'
>>> y = Item(x)
>>> y.value
'e\x00\x00j'
>>> y.value
'e\x00\x00j'
>>> x = 'xyz'
>>> y = Item(x)
>>> y.value
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
UnicodeDecodeError: 'utf-8' codec can't decode byte 0xcf in position 2: unexpected end of data
'utf-8' codec can't decode byte 0xcf in position 2: unexpected end of data
>>> 
```

Ooops! Doesn't look too good. We're using Python `str` here. Let's use some `bytes`:

```
>>> x = b'abcd'
>>> x
b'abcd'
>>> y = Item(x)
>>> y.value
'abcd'
>>> x = 'xyz'.encode()
>>> y = Item(x)
>>> y.value
'xyz'
>>> 
```

Better! Since the string values are not used other than initiating the `Item` objects, let's use literals directly the `Item` initializer:

```
>>> y = Item(b'abcd')
>>> y.value
'e\x00\x00j'
>>> y = Item('xyz'.encode())
>>> y.value
'e\x00\x00'
>>> y.value
'e\x00\x00'
>>> y.value
'\x00|\t'
>>> y.value
'e\x00\x00'
>>> y.value
'e\x00\x00'
>>> y.value
'\x00|\t'
>>> 
```

Ooops! Not too good. What's happening here?

I took it to the `pybind11` support channel at [gitter](https://gitter.im/pybind/Lobby) and asked the folks there. They were very responsive and helpful (Thanks folks!). Once they gave me the pointer, things are not hard to understand.

The `string_view` in C++ is a pointer to some contiguous bytes along with the length. It has no idea about string, encoding, or that sort of things. This corresponds to `bytes` in Python, not `str`. In the code block that works, I save `bytes` in `x`, then call `Item(x)`. In this call, the `pybind11` binding code passes a view of the Python `bytes` to C++ without copying. Because `x` lives beyond the subsequent calls to `y.value`, the piece of memory holding `x` value does not change, hence `y.value` is stable.

In the last code block, although `bytes` are used to initiate `Item`, the literal values are temporaries that may be garbage-collected anytime after the call to `Item()`. By the we check `y.value`, chances are `y._value` (on the C++ side) points to some memory (on the Python side) that has been reused for something out of our control. That's why we see garbage, and the garbage changes!

In the code above the block that works, I save some `str` (not `bytes`) in `x`, and call `Item(x)`. The persistent `x` does not help here---during the call, a `bytes` object is created based on `x`, and a view of the **temporary** `bytes` is passed to C++. Gargabe kicks in again.

OK, I understand it. Now, how do I fix it? An easy solution is to have a version in C++ that accepts `string`, not `string_view`, and expose only the `string` version to Python. The changed class `Item` looks like this:

```
class Item {
    public:
        Item(string_view value) : _value{value} {}                                                      
                                                                                                        
        Item(string value) : _string_value{value}, _value{_string_value} {}                             
                                                                                                        
        string_view value() const {                                                                     
            return _value;                                                                              
        }                                                                                               
                                                                    
    private:
        string _string_value;                                                                           
        string_view _value;                                                                             
};
```

The binding code becomes

```
    py::class_<Item>(m, "Item")
        .def(py::init<string>())                                                           
        .def_property_readonly("value", &Item::value);    
```

This works, as is easily verified:

```
>>> y = Item(b'abcd')
>>> y.value
'abcd'
>>> y.value
'abcd'
>>> y.value
'abcd'
>>> y = Item('xyz'.encode())
>>> y.value
'xyz'
>>> 
>>> y.value
'xyz'
>>> z = [Item(b'abc'), Item(b'def'), Item(b'123'), Item(b'rst')]
>>> [zz.value for zz in z]
['abc', 'def', '123', 'rst']
>>> 
```

However, I did not want to complicate the C++ code. I wanted to keep the C++ code absolutely clean and performant. Whatever issue Python has, it better solves it itself. I removed the `string` version of `Item::Item` and reverted to the original version.

The solution points straight at the cause: temporaries. Let's save the temporaries and make sure they outlive their use. I created a Python file `example.py`, which `import`s `Item` from the dynamic library `_example` and does thing about it:

```
from _example import Item, Row


item_init_original = Item.__init__

def item_init(self, value):
    if isinstance(value, bytes):
        self._value = value
        item_init_original(self, self._value)
    elif isinstance(value, str):
        self._value = value.encode()
        item_init_original(self, self._value)
    else:
        item_init_original(self, value)                                                

Item.__init__ = item_init
```

So I modified the `__init__` method of the class `Item`. Because we're adding members to the class (namely, `self._value`), we need to use `py::dynamic_attr` in the binding code:

```
    py::class_<Item>(m, "Item", py::dynamic_attr())
        .def(py::init<string_view>())                                             
        .def("print", &Item::print)                                               
        .def_property_readonly("value", &Item::value); 
```

I was a little unsure about hacking `__init__`, but the result is assuring:

```
>>> from example import Item
>>> y = Item('abcd')
>>> y.value
'abcd'
>>> y.value
'abcd'
>>> y.value
'abcd'
>>> y = [Item(v) for v in ('abc', 'def', 'xy1', '234#')]
>>> [v.value for v in y]
['abc', 'def', 'xy1', '234#']
>>> [v.value for v in y]
['abc', 'def', 'xy1', '234#']
>>> [v.value for v in y]
['abc', 'def', 'xy1', '234#']
>>> y = Item('xyz'.encode())
>>> y.value
'xyz'
>>> y.value
'xyz'
>>> y.value
'xyz'
>>> 
```

Now on to `Row`, which is constructed by a `vector` of `Item`s:

```
>>> from example import Item, Row
>>> values = ['abcd', 'xyz'.encode(), b'##12a', b'!#@<>_x']
>>> items = [Item(v) for v in values]
>>> row = Row(items)
>>> row.print()
abcd, xyz, ##12a, !#@<>_x
>>> row.print()
abcd, xyz, ##12a, !#@<>_x
>>> 
```
Nice.
```
>>> row = Row([Item(v) for v in values])
>>> row.print()
�+��, xyz, ##12a, !#@<>_x
>>> row.print()
x(��, xyz, ##12a, !#@<>_x
>>> row.print()
((��, xyz, ##12a, !#@<>_x
>>> row.print()
((��, xyz, ##12a, !#@<>_x
>>> row.print()
�)��, xyz, ##12a, !#@<>_x
>>> row.print()
�t>�, xyz, ##12a, !#@<>_x
>>> row.print()
�)��, xyz, ##12a, !#@<>_x
>>> 
```

Oooops! What's wrong, again?

Although we've saved the `str` or `bytes` in `Item`, now the `Item`s themselves are temporaries in the call to `Row()`, hence become target of garbage collection. Once an `Item` object is garbage collected, its data members are, of course, discarded as well.

The solution is similar. We need to persist some things in `Row`:

```
row_init_original = Row.__init__
                                                                                                                
def row_init(self, items):
    self._items = [v if isinstance(v, Item) else Item(v) for v in items]
    row_init_original(self, self._items)

Row.__init__ = row_init
```

Correspondingly, `py::dynamic_attr` is added to the binding code:

```
    py::class_<Row>(m, "Row", py::dynamic_attr())
        .def(py::init<std::vector<Item>>())
        .def("print", &Row::print);
```

 Does this fix it?

```
>>> from example import Item, Row
>>> values = ['abcd', 'xyz'.encode(), b'##12a', b'!#@<>_x']
>>> row = Row([Item(v) for v in values])
>>> row.print()
abcd, xyz, ##12a, !#@<>_x
>>> 
>>> row = Row(values)
>>> row.print()
abcd, xyz, ##12a, !#@<>_x
>>> 
```

I guess so!
