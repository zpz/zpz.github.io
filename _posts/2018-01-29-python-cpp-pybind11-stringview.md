---
layout: post
title: Python, C++, Pybind11, and string_view
---


```C++
#include <pybind11/pybind11.h>

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

        void print() const {
            std::cout << _value << std::endl;
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
        .def("print", &Item::print)
        .def_property_readonly("value", &Item::value);

    py::class_<Row>(m, "Row")
        .def(py::init<std::vector<Item>>())
        .def("print", &Row::print);
}
```

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


```
class Item {                                                                                            
    public:                                                                                             
        Item(string_view value) : _value{value} {}                                                      
                                                                                                        
        Item(string value) : _string_value{value}, _value{_string_value} {}                             
                                                                                                        
        string_view value() const {                                                                     
            return _value;                                                                              
        }                                                                                               
                                                                                                        
        void print() const {                                                                            
            std::cout << _value << std::endl;                                                           
        }                                                                                               
                                                                                                        
    private:                                                                                            
        string _string_value;                                                                           
        string_view _value;                                                                             
};        
```

```
    py::class_<Item>(m, "Item")                                                                         
        .def(py::init<string>())                                                                        
        .def("print", &Item::print)                                                                     
        .def_property_readonly("value", &Item::value);    
```


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

