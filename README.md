# oop_ptr

smart pointer for managing objects that utilize object oriented programming paradigm of C++

# Problem

Whenever you wish to utilize oriented programming paradigm of C++ you are sometimes forced to use pointers:

``` c++
class BaseClass {
    public:
    int a;
    virtual void foo() {
        printf("base\n");
    }
    virtual ~BaseClass() = default;
};

class ChildClass : public BaseClass {
    public:
    int b;
    void foo() override {
        printf("child\n");
    }
}

int main() {
    ChildClass c;
    BaseClass b = c; // c object will get trimmed to b object
    b.foo(); // "base" will be printed


    ChildClass* cptr = new ChildClass();
    BaseClass* bptr = cptr;
    bptr->foo(); // "child" will be printed
}
```

Problem arises when you want to store bunch of objects that inherit from some base class in a containter like `std::vector`:

``` c++
std::vector<BaseClass*> bcv;
bcv.push_back(new ChildClass());

bcv2 = bcv; // now both bcv[0] and bcv2[0] point to the same object
// potential double free can happen if you are not carefull
```

Now try putting a vector like that as a member in a class that you wish to be __Copy Assignable__ or __Copy Constructable__. You will not only need to define a destructuor (to free dynamically allocated data), but also Copy Assignment constructor and operator.

# Solution

Here comes `oop_ptr`.

With `oop_ptr` you can treat all objects as trivially constuctable, copiable and destructable despite them being dynamically alocated with `new` keyword.

Only caveat is, that you will need to define a special function in every non-abstract object in your inheritance hierarchy. Luckily `oop_ptr.hpp` contains macros to speed up the process.

``` c++
#include <oop_ptr.hpp>

class BaseClass {
    public:
    int a;
    virtual void foo() {
        printf("base\n");
    }
    virtual ~BaseClass() = default;

    oop_ptr_base_declare(BaseClass);
};

class ChildClass : public BaseClass {
    public:
    int b;
    void foo() override {
        printf("child\n");
    }
    oop_ptr_child_declare(BaseClass);
}

oop_ptr_define(BaseClass, BaseClass)
oop_ptr_define(BaseClass, ChildClass)

int main() {
    oop_ptr<BaseClass> boop = oop_ptr<BaseClass>(new ChildClass()); // boop will take ownership over newly allocated ChildClass instance
    boop->foo(); // "child" will be printed

    boop2 = boop; // boop2 will allocate it's own instance of ChildClass using ChildClass default copy constructor
    boop2->foo(); // "child" will be printed

    poob->b = 1;
    poob2->b = 2;
    assert(poob->b != boop2->b);

    // no need to delete as both boop and boop2 will automatically dealocate managed resources when exiting scope
}
```

# Features

## 1. Made to be consistent with smart pointers form C++ standard library:
with methods like `release()` and `get()` and overloaded `->` and `*` operators

## 2. Customizable:
with `#define`s you can include / exclude methods, operators and constructors from `oop_ptr` definition

# Documentation

> [!WARNING]  
> Documentation not complite

``` c++
template<
    typename T
> class oop_ptr;
```

## Constructors

* `oop_ptr()` - initializes managed object pointer to `nullptr`
* `oop_ptr(T* ptr)` - takes ownership over object pointed by `ptr`
* `oop_ptr(const T& obj)` - copies `obj` into newly allocated memory

## Operators

* `operator= (const oop_ptr<T>& other)` - copy assignment operator
* `operator= (oop_ptr<T>&& other)` - move assignment operator
* `operator bool` - checks if there is an associated managed object
* `operator*`, `operator->` - dereferences pointer to the managed object

## Methods

* `release()` - returns a pointer to the managed object and releases the ownership
* `get()` - returns a pointer to the managed object without releasing the ownership
* `gettable<U>()` - checks if managed object is an instance of `U` (with `dynamic_cast`)
* `release<U>()` - returns a pointer to the managed object, dynamic_casted to `U` and releases the ownership
* `get<U>()` - returns a pointer to the managed object, dynamic_casted to `U`, without releasing the ownership

# Future

With new [reflection system for C++ 26](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2023/p2996r0.html) there should be a way to do all this without the need to define special functions in class definitions. I have plans for reimplementing oop_ptr using new reflection system as a separate smart pointer (`oop_ptr26`). Existing implementation will remain to be used in projects that don't want to switch to version 26 of C++.

Look into [issues](https://github.com/pwalig/oop_ptr/issues) to see all planned features.
