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

