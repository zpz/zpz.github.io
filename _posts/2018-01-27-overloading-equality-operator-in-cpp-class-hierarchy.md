---
layout: post
title: "Overloading `operator==` for a C++ Class Hierarchy"
---

Recently I needed to define the equality operator between objects in a class hierarchy. A quick search revealed some discussions on this topic, and the opinion appears to be that this, while certainly doable, does not have a clean, elegant solution.

Techniques involved in this task include `operator==`, virtual functions, API design in a class hierarchy, and the very concept of object equality.

After a few iterations, I've settled with a solution. In fact it's not bad. It's systematic and routine. The code is listed below; some observations follow.


```c++
#include <cassert>
#include <typeinfo>
#include <typeindex>
#include <iostream>
#include <memory>
#include <string>

using namespace std;


class A
{
    private:
        int _a;

    protected:
        virtual bool _equals(A const & other) const
        {
            if (typeid(*this) != typeid(other)) return false;
            return this->_a == other._a;
        }

    public:
        A(int a) : _a{a} {}

        bool operator==(A const & other) const {
            return this->_equals(other);
        }

        bool operator!=(A const & other) const {
            return !(*this == other);
        }
};


class B : public A
{
    private:
        int _b;

    protected:
        virtual bool _equals(A const & other) const
        {
            if (typeid(*this) != typeid(other)) return false;
            auto that = static_cast<B const &>(other);
            if (this->_b != that._b) return false;
            return A::_equals(other);
        }

    public:
        B(int a, int b) : A(a), _b{b} {}
};


class C : public A
{
    private:
        int _c;

    protected:
        virtual bool _equals(A const & other) const
        {
            if (typeid(*this) != typeid(other)) return false;
            auto that = static_cast<C const &>(other);
            if (this->_c != that._c) return false;
            return A::_equals(other);
        }

    public:
        C(int a, int c): A(a), _c{c} {}
};


class D : public C
{
    private:
        int _d;

    protected:
        virtual bool _equals(A const & other) const
        {
            if (typeid(*this) != typeid(other)) return false;
            auto that = static_cast<D const &>(other);
            if (this->_d != that._d) return false;
            return C::_equals(other);
        }

    public:
        D(int a, int c, int d): C(a, c), _d{d} {}
};


class E: public D
{
    public:
        E(int a, int c, int d) : D(a, c, d) {}
};


class F : public E
{
    private:
        int _f;

    protected:
        virtual bool _equals(A const & other) const
        {
            if (typeid(*this) != typeid(other)) return false;
            auto that = static_cast<F const &>(other);
            if (this->_f != that._f) return false;
            return E::_equals(other);
        }

    public:
        F(int a, int c, int d, int f): E(a, c, d), _f{f} {}
};




int main()
{
    A a1(2);
    A a2(2);
    A a3(3);
    B b1{2, 3};
    B b2{2, 3};
    B b3{3, 3};
    B b4(3, 4);
    C c1{8, 4};
    C c2{8, 4};
    C c3{7, 4};
    C c4{7, 5};
    D d1(1, 2, 3);
    D d2(1, 2, 3);
    D d3(1, 2, 4);
    D d4(2, 2, 4);
    D d5(1, 3, 3);
    F f1(1, 2, 3, 4);
    F f2(1, 2, 3, 4);
    F f3(1, 3, 3, 4);
    F f4(1, 2, 3, 4);

    assert(a1 == a2);
    assert(a2 != a3);
    assert(b1 == b2);
    assert(b1 != b3);
    assert(b3 != b4);
    assert(c1 == c2);
    assert(c1 != c3);
    assert(c3 != c4);
    assert(d1 == d2);
    assert(d1 != d3);
    assert(d3 != d4);
    assert(d1 != d5);
    assert(f1 == f2);
    assert(f2 != f3);
    assert(f3 != f4);

    assert(a1 != b2);
    assert(b3 != a2);
    assert(a2 != c2);
    assert(c3 != a1);
    assert(a2 != d1);
    assert(d5 != a1);
    assert(b2 != c3);
    assert(c3 != b1);
    assert(c4 != d1);
    assert(d3 != c2);
    assert(d2 != f1);
    assert(f2 != c3);

    assert(typeid(a1) == typeid(a2));
    assert(typeid(b1) == typeid(b3));
    assert(typeid(c3) == typeid(c4));
    assert(typeid(d1) == typeid(d4));
    assert(typeid(f1) == typeid(f2));
    assert(typeid(a1) != typeid(b1));
    assert(typeid(b1) != typeid(c1));
    assert(typeid(c1) != typeid(d1));
    assert(typeid(d1) != typeid(f3));

    assert(type_index(typeid(a1)) == type_index(typeid(a3)));
    assert(type_index(typeid(b1)) == type_index(typeid(b3)));
    assert(type_index(typeid(c1)) == type_index(typeid(c3)));
    assert(type_index(typeid(d3)) == type_index(typeid(d4)));
    assert(type_index(typeid(a1)) != type_index(typeid(c1)));

    assert(type_index(typeid(a2)).hash_code() == type_index(typeid(a3)).hash_code());
    assert(type_index(typeid(b1)).hash_code() == type_index(typeid(b3)).hash_code());
    assert(type_index(typeid(c1)).hash_code() == type_index(typeid(c3)).hash_code());
    assert(type_index(typeid(d2)).hash_code() == type_index(typeid(d3)).hash_code());
    assert(type_index(typeid(b2)).hash_code() != type_index(typeid(c2)).hash_code());

    A* aa1 = &a1;
    A* aa2 = &a2;
    A* aa3 = &a3;
    A* ab1 = &b1;
    A* ab2 = &b2;
    A* ab3 = &b3;
    A* ab4 = &b4;
    A* ac1 = &c1;
    A* ac2 = &c2;
    A* ac3 = &c3;
    A* ac4 = &c4;
    A* ad1 = &d1;
    A* ad2 = &d2;
    A* ad3 = &d3;
    A* ad4 = &d4;
    A* ad5 = &d5;
    A* af1 = &f1;
    A* af2 = &f2;
    A* af3 = &f3;
    A* af4 = &f4;

    assert(*aa1 == *aa2);
    assert(*aa2 != *aa3);
    assert(*ab1 == *ab2);
    assert(*ab1 != *ab3);
    assert(*ab3 != *ab4);
    assert(*ac1 == *ac2);
    assert(*ac1 != *ac3);
    assert(*ac3 != *ac4);
    assert(*ad1 == *ad2);
    assert(*ad1 != *ad3);
    assert(*ad3 != *ad4);
    assert(*ad1 != *ad5);
    assert(*af1 == *af2);
    assert(*af2 != *af3);
    assert(*af3 != *af4);

    assert(typeid(*aa1) == typeid(*aa2));
    assert(typeid(*ab1) == typeid(*ab3));
    assert(typeid(*ac3) == typeid(*ac4));
    assert(typeid(*ad1) == typeid(*ad4));
    assert(typeid(*af1) == typeid(*af2));
    assert(typeid(*aa1) != typeid(*ab1));
    assert(typeid(*ab1) != typeid(*ac1));
    assert(typeid(*ac1) != typeid(*ad1));
    assert(typeid(*ad1) != typeid(*af3));

    assert(type_index(typeid(*aa1)) == type_index(typeid(*aa3)));
    assert(type_index(typeid(*ab1)) == type_index(typeid(*ab3)));
    assert(type_index(typeid(*ac1)) == type_index(typeid(*ac3)));
    assert(type_index(typeid(*ad3)) == type_index(typeid(*ad4)));
    assert(type_index(typeid(*aa1)) != type_index(typeid(*ac1)));

    assert(type_index(typeid(*aa2)).hash_code() == type_index(typeid(*aa3)).hash_code());
    assert(type_index(typeid(*ab1)).hash_code() == type_index(typeid(*ab3)).hash_code());
    assert(type_index(typeid(*ac1)).hash_code() == type_index(typeid(*ac3)).hash_code());
    assert(type_index(typeid(*ad2)).hash_code() == type_index(typeid(*ad3)).hash_code());
    assert(type_index(typeid(*ab2)).hash_code() != type_index(typeid(*ac2)).hash_code());

    unique_ptr<A> uaa1(new A(move(a1)));
    unique_ptr<A> uab1(new B(move(b1)));
    unique_ptr<A> uab2(new B(move(b2)));
    unique_ptr<A> uac1(new C(move(c1)));

    assert(*uab1 == *uab2);
    assert(*uaa1 != *uab2);
    assert(*uab1 != *uac1);

    assert(typeid(*uab1) != typeid(*uac1));
    assert(typeid(*uab1) == typeid(*uab2));
}
```

All the assertions in this code should pass.

We see the solution is quite systematic and routine. The pattern is clear:

- The root class defines a public function for `operator==`, which sets the API. This function is inherited by every subclass and I didn't see a need to overide or customize this function in any subclass. This function calls a `protected` `virtual` function called `_equals` (of course one can replace this name with whatever they prefer). It's often useful to define a `operator!=` which simply negates the result of `operator==`.
- Every subclass that has its own data members that affect the equality relation of two objects of that subclass should re-define `_equals`, and keep it `protected` and `virtual`.
- All the `operator==`, `operator!=`, and `_quals` functions have a single argument which is a `const` reference to the *root* class, *not* any particular subclass.
- The function `_equals` does exactly three things: 
  1. Check for un-equality according to `typeid`, which determines the [**dynamic type**](https://stackoverflow.com/a/7649711/6178706]() of both the 'current' and the 'other' objects.
  2.  Check for un-equality according to data members, if any, introduced in the 'current' subclass (meaning the subclass in which the the implementation of `_equals` is in question).
  3. Call `_equals` of the parent class (unless the current class is the root class).

Why does the solution take this route?

First, I've used a `public`, non-`virtual` `operator==` to define the API, and delegated implementation details to a `protected` `virtual` function `_equals`. This follows [Herb Sutter's recommendations](http://www.gotw.ca/publications/mill18.htm).

Second, the public API defines the comparison between any two objects in the class hierarchy, on both sides of the `==` sign.

Third, why do we need to check `typeid`? Clearly, the `typeid`s of both objects must match in order for them to be equal. But can we omit this check and achieve the same effect? Suppose `this` is an object of class `C` and `other` is an object of class `D` (a subclass of `C`), then `C::_equals` is called.  When `other` is cast to an object of class `C`, any data members of `other` that are introduced in class `D` are lost. As long as the `C` and `A` parts of `other` match those of `this`, the function will return `true`.  This is of course the wrong answer, because the `D`-specific extra stuff in `other` do not participate in the comparison, as they should.

On the other hand, suppose `this` is a `D` object and `other` is a `C` object, then `D::_equals` is called. If we omit the check on `typeid` and go ahead `cast` `other` to class `D` (of subclass of `C`), it will be a run-time (or compile-time?) error.

Fourth, the rational for comparison based on data members of the `this` class is easy to understand. Actually, after the `typeid` check and `cast`, both `this` and `other` are of the same class, as they actually are but possibly disguised by the declared type of the references. Only when the object `other` is treated as exactly its dynamic type, not any base class (whereas `other` can't be of a subclass of the class of `this`, given that it has passed the `typeid` check), can we fully access all the data members defined in this class of `this`, and define equality based on this data members however we like.

Fifth, why is the call to the parent's `_equals`?  Well, if not, then data members of `this` and `other` that are defined in 'upstream' classes do not affect the determination of equality, which is wrong.

Sixth, why is `_equals` kept `protected` and `virtual` throughout? It needs to be `protected`, rather than `private`, because it needs to be called by subclasses. It needs to be `virtual`, even in a 'leaf' class, because any class in this hierarchy may be subclassed later either in our own code or in user's separate code, and in that situation we need polymorphism to remain functional.

  
  
  
  
  
   
