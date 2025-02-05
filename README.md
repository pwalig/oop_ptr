# oop_ptr

smart pointer for managing objects that belong to inheritance hierarchies

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

int main(void) {
    ChildClass c;
    BaseClass b = c; // c object will get trimmed to b object
    b.foo(); // "base" will be printed


    ChildClass* cptr = new ChildClass();
    BaseClass* bptr = cptr;
    bptr->foo(); // "child" will be printed

    return 0;
}
```

Problem arises when you want to store bunch of objects that inherit from some base class in a containter like `std::vector`:

``` c++
std::vector<BaseClass*> bcv;
bcv.push_back(new ChildClass());

bcv2 = bcv; // now both bcv[0] and bcv2[0] point to the same object
// potential double free and other abnormalities can happen if you are not carefull
```

Now try putting a vector like that as a member in a class that you wish to be *Copy Assignable* or *Copy Constructable*. You will not only need to define a *destructuor* (to free dynamically allocated data), but also *Copy Assignment* *constructor* and *operator*.

# Solution

Here comes `oop_ptr`.

`oop_ptr` makes all objects easily constuctable, copiable and destructable.

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

int main(void) {
    oop_ptr<BaseClass> boop = oop_ptr<BaseClass>(new ChildClass()); // boop will take ownership over newly allocated ChildClass instance
    boop->foo(); // "child" will be printed

    boop2 = boop; // boop2 will allocate it's own instance of ChildClass using ChildClass default copy constructor
    boop2->foo(); // "child" will be printed

    poob->b = 1;
    poob2->b = 2;
    assert(poob->b != boop2->b); // boop and boop2 are independent - point to different objects

    // no need to delete as both boop and boop2 will automatically dealocate managed resources when exiting scope
    return 0;
}
```

# Features

## 1. Made to be consistent with smart pointers form C++ standard library:
with methods like `release()` and `get()` and overloaded `->` and `*` operators

## 2. Customizable:
with `#define`s you can include / exclude methods, operators and constructors from `oop_ptr` definition

## 3. Simple and understandable:
easy to analyze source code

# Documentation

``` c++
template<
    typename T
> class oop_ptr;
```

## Constructors

* `oop_ptr()` - initializes managed object pointer to `nullptr`
* `oop_ptr(T* ptr)` - takes ownership over object pointed by `ptr`
* `oop_ptr(const T* ptr)` - copies object pointed by `ptr` into newly allocated memory, disabled by default, define `OOP_PTR_PTR_COPY_CONSTRUCTOR` to enable
* `oop_ptr(const T& obj)` - copies `obj` into newly allocated memory
* `oop_ptr(const oop_ptr<T>& other)` - copy constructor
* `oop_ptr(oop_ptr<T>&& other)` - move constructor

## Operators

* `operator= (const oop_ptr<T>& other)` - copy assignment operator
* `operator= (oop_ptr<T>&& other)` - move assignment operator
* `operator bool` - checks if there is an associated managed object
* `operator*`, `operator->` - dereferences pointer to the managed object
* `operator=(const T& obj)` - destroyes currently managed object (if any) and creates a copy of supplied object, disabled by default, to enable define `OOP_PTR_OBJECT_COPY_OPERATOR`
* `operator=(const T& obj)` - destroyes currently managed object (if any) and creates a copy of object pointed by `ptr`, disabled by default, to enable define `OOP_PTR_PTR_COPY_OPERATOR`
* `operator=(T* ptr)` - destroyes currently managed object (if any) and takes ownership over object pointed by `ptr`, disabled by default, to enable define `OOP_PTR_PTR_MOVE_OPERATOR`

## Methods

* `release()` - returns a pointer to the managed object and releases the ownership
* `get()` - returns a pointer to the managed object without releasing the ownership
* `gettable<U>()` - checks if managed object is an instance of `U` (with `dynamic_cast`)
* `release<U>()` - returns a pointer to the managed object, dynamic_casted to `U` and releases the ownership
* `get<U>()` - returns a pointer to the managed object, dynamic_casted to `U`, without releasing the ownership

## Macros

In following definitions replace `b` with name of the base class and `c` with name of the child class.

* `oop_ptr_base_declare(b);` - meant to be put in base class declaration (for pure virtual method type: `oop_ptr_base_declare(b) = 0;`)
* `oop_ptr_child_declare(b);` - meant to be put in child class declaration
* `oop_ptr_define(b, c)` - creates method definition (for definition of base class use `oop_ptr_define(b, b)`)
* `oop_ptr_template_base_define(b)` - meant to be put in base class declaration if the base class is a template
* `oop_ptr_template_child_define(b, c)` - meant to be put in child class declaration if the base class is a template

## Customization with `#define`s

| define name | created method | defined by default |
| --- | --- | --- |
| `OOP_PTR_PTR_MOVE_CONSTRUCTOR` | `oop_ptr(T* ptr)` | yes |
| `OOP_PTR_OBJECT_COPY_CONSTRUCTOR` | `oop_ptr(const T& obj)` | yes |
| `OOP_PTR_PTR_COPY_CONSTRUCTOR` | `oop_ptr(const T* ptr)` | no |
| `OOP_PTR_OBJECT_COPY_OPERATOR` | `operator=(const T& obj)` | no |
| `OOP_PTR_PTR_COPY_OPERATOR` | `operator=(const T* ptr)` | no |
| `OOP_PTR_PTR_MOVE_OPERATOR` | `operator=(T* ptr)` | no |

> [!WARNING]  
> Do not define both: `OOP_PTR_PTR_COPY_OPERATOR` and `OOP_PTR_PTR_MOVE_OPERATOR` at once.  
> Do not define both: `OOP_PTR_PTR_COPY_CONSTRUCTOR` and `OOP_PTR_PTR_MOVE_CONSTRUCTOR` at once.

It is also possible to change name of the method generated by macros with `oop_ptr_get_copy` `#define`.

``` c++
#define oop_ptr_get_copy awesomelyNamedMethod
```

By default (if `oop_ptr_get_copy` is not defined) macros will generate method named: `get_copy_ptr`

---

> When in doubt look at the source code (it's not complicated).

# Disclamers

Although `oop_ptr` was created to enabe storing objects that inherit from some base class in a C++ containers or as class members, this is not a particularly cache friendly approach, since `oop_ptr` still stores a pointer, that can point anywhere, potentially creating a *cache miss*. Because of this issue i plan to create a seperate utility that would store object directly (without any extra indirection), as a separete project. I will provide a link when it's ready.

# Future

With new [reflection system for C++ 26](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2023/p2996r0.html) there should be a way to do all this without the need to define special functions in class definitions. I have plans for reimplementing oop_ptr using new reflection system as a separate smart pointer (`oop_ptr26`). Existing implementation will remain to be used in projects that don't want to switch to version 26 of C++.

Look into [issues](https://github.com/pwalig/oop_ptr/issues) to see all planned features.

# Adoption

Here is a list of projects that utilize `oop_ptr`:
* [pwalig/mesh-compiler](https://github.com/pwalig/mesh-compiler)
