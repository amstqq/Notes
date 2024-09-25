## Non-type template parameters
Parameters supplied with compile-time expressions are called non-type template parameters. This category of parameters can only have a structural type. The following are the structural types:
- Integral types
- Floating-point types, as of C++20
- Enumerations
- Pointer types (either to objects or functions)
```cpp
struct device
{
    virtual void output() = 0;
    virtual ~device() {}
};

template <void (*action)()>
struct smart_device : device
{
    void output() override
    {
        (*action)();
    }
};
```
- Pointer to member types (either to member objects or member functions)
- Lvalue reference types (either to objects or functions)
- A literal class type that meets the following requirements:
- All base classes are public and non-mutable.
- All non-static data members are public and non-mutable.
- The types of all base classes and non-static data members are also structural types or arrays thereof.

## Template instantiation

### Implicit instantiation

Implicit instantiation occurs when the compiler generates definitions based on the use of templates and when no explicit instantiation is present.

```cpp
template <typename T>
struct foo
{
    void f() {}
    void g() {}
};

int main()
{
    foo<int>* p; // foo<int> not instantiated when only pointer to template is declared
    foo<int> x; // foo<int> instantiated
    foo<double>* q; // foo<double> not instantiated when only pointer to template is declared

    x.f(); // foo<int>::f() instantiated
    q->g(); // foo<double> AND foo<double>::g() instantiated
}
```

### Explicit instantiation definition

```cpp
namespace ns
{
    template <typename T>
    struct wrapper
    {
        T value;
    };
    template struct wrapper<int>; // [1]
}

template struct ns::wrapper<double>; // [2]
int main() {}
```

### Explicit instantiation declaration


An explicit instantiation declaration (available with C++11) is the way you can tell the compiler that the definition of a template instantiation is found in a different translation unit and that a new definition should not be generated.

```cpp
// wrapper.h
template <typename T>
struct wrapper
{
    T data;
};
extern template wrapper<int>; // explicit instantiation declaration

// source1.cpp
#include "wrapper.h"
#include <iostream>
template wrapper<int>; // explicit instantiation definition

// source2.cpp
#include "wrapper.h"
#include <iostream>
void g()
{
    wrapper<int> a{ 100 }; // definition in source1.cpp also visiable here, so no need to define again
    std::cout << a.data << '\n';
}
```

## Alias template

Alias template can neither be fully nor partially specialized. To overcome this limitation, we create a class template with a type alias member and specialize the class. 

Suppose we want `list_t` to be alias template for `std::vector<T>` provided the size of
the collection is greater than 1. If there is a single element, then `list_t` should
be an alias for the type template parameter T.

Although the following is not valid C++ code, it represents the end goal I want to achieve, had the specialization of alias templates been possible:
```cpp
template <typename T, size_t S>
using list_t = std::vector<T>;

template <typename T>
using list_t<T, 1> = T;
```

Implemented by specializing class template
```cpp
template <typename T, size_t S>
struct list
{
    using type = std::vector<T>;
};

template <typename T>
struct list<T, 1>
{
    using type = T;
};

template <typename T, size_t S>
using list_t = typename list<T, S>::type;
```
