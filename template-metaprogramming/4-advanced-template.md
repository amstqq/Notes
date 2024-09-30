## Name binding and dependent names
Dependent names are names that depend on the type or value of a template parameter that can be a type, non-type, or template parameter. The name lookup is performed differently for dependent and non-dependent names:
• For dependent names, it is performed at the point of template instantiation.
• For non-dependent names, it is performed at the point of the template definition.

### Name lookup for non-dependent names
```cpp
template <typename T>
struct processor; // [1] template declaration

void handle(double value) // [2] handle(double) definition
{
    std::cout << "processing a double: " << value << '\n';
}

template <typename T>
struct parser // [3] template definition
{
    void parse()
    {
        handle(42); // [4] non-dependent name
        /*
        For non-dependent name, name lookup and binding happens right here at
        template definition.

        Although handle(int) is defined below, it is ignored since naming binding
        happened here.
        */
    }
};

void handle(int value) // [5] handle(int) definition
{
    std::cout << "processing an int: " << value << '\n';
}

int main()
{
    parser<int> p; // [6] template instantiation
    p.parse(); // output: processing a double
}
```

### Name lookup for dependent names

```cpp
template <typename T>
struct handler // [1] template definition
{
    void handle(T value)
    {
        std::cout << "handler<T>: " << value << '\n';
    }
};

template <typename T>
struct parser // [2] template definition
{
    void parse(T arg)
    {
        arg.handle(42); // [3] dependent name
        /**
        handle is not bound at this point because arg is a dependent name.
        */
    };
};

template <>
struct handler<int> // [4] template specialization
{
    void handle(int value)
    {
        std::cout << "handler<int>: " << value << '\n';
    }
};

int main()
{
    handler<int> h; // [5] template instantiation
    parser<handler<int>> p; // [6] template instantiation
    /*
    Dependent name is bounded at point of template instantiation.
    Dependent name 'arg' at point [3] is bounded to handler<int> at point [4], since
    it is a better match for arg.handle(42)
    */

    p.parse(h);
}
```

## Two-phase name lookup
Instantiation of a template happens in two phases:
• The first phase occurs at the point of the definition when the template syntax is checked and names are categorized as dependent or non-dependent.
• The second phase occurs at the point of instantiation when the template arguments are substituted for the template parameters. Name binding for dependent names happens at this point.

Example:
```cpp
template <typename T>
struct base_parser
{
    void init()
    {
        std::cout << "init\n";
    }
};

template <typename T>
struct parser : base_parser<T>
{
    void parse()
    {
        init(); // error: identifier not found
        std::cout << "parse\n";
    }
};

int main()
{
    parser<int> p;
    p.parse();
}

```
Although the intention is to call base class method `init`, compiler will issue an error because `init` is a non-dependent name. This means `init` MUST be known at point of definition of `parser`. Compiler cannot assume `base_parser<T>::init` is what we want to call because template `base_parser` can be later
specialized and init can be defined as something else (such as a type, or a variable, or another function, or it may be missing entirely). For example:
```cpp
template<>
struct base_parser<int>
{
    using init = std::vector<int>;
    // now init() call in parser is ambiguous
};
```
This problem can be fixed by making init a dependent name. This can be done either by prefixing with `this->` or with `base_parser<T>::`. By turning init into a dependent name, its name binding is moved from the point of template definition to the point of template instantiation.

## Dependent type names

```cpp
template <typename T>
struct base_parser
{
    using value_type = T;
};

template <typename T>
struct parser : base_parser<T>
{
    void parse()
    {
        value_type v{}; // [1] error
        // or
        base_parser<T>::value_type v{}; // [2] error
        std::cout << "parse\n";
    }
};
```
- `value_type` does not work because it’s a **non-dependent** name and therefore it will not be looked up in the base class, only in the enclosing scope.
- `base_parser<T>::value_` type does not work either because the compiler cannot assume this is actually a type. A specialization of `base_parser` may follow and `value_type` could be defined as something else than a type.


we need to tell the compiler the name refers to a type. This is done with the typename keyword, at the point of definition, shown as follows:
```cpp
template <typename T>
struct parser : base_parser<T>
{
    void parse()
    {
        typename base_parser<T>::value_type v{}; // [3] OK
        std::cout << "parse\n";
    }
};
```

There are two exceptions to this rule:
- When specifying a base class
- When initializing class members

```cpp
struct dictionary_traits
{
    using key_type = int;
    using map_type = std::map<key_type, std::string>;
    static constexpr int identity = 1;
};

template <typename T>
struct dictionary : T::map_type // [1]: typename not required when specifying base class
{
    int start_key { T::identity }; // [2]: typename not required when initializing class members
    typename T::key_type next_key; // [3]
};
```

### Dependent template names
when the dependent name is a template
```cpp
template <typename T>
struct base_parser
{
    template <typename U>
    struct token {};
};

template <typename T>
struct parser : base_parser<T>
{
    void parse()
    {
        using token_type = base_parser<T>::template token<int>; 

        token_type t1{};

        typename base_parser<T>::template token<int> t2{};
        std::cout << "parse\n";
    }
};
```

## decltype(auto)
Suppose we have the following template:
```cpp
template <typename T, typename U>
??? minimum(T&& a, U&& b)
{
   return a < b ? a : b;
} 
```
what is the return type of this function?

C++11:
```cpp
template <typename T, typename U>
auto minimum(T&& a, U&& b) -> decltype(a < b ? a : b)
{
    return a < b ? a : b;
}
```

c++14:
```cpp
template <typename T, typename U>
decltype(auto) minimum(T&& a, U&& b)
{
    return a < b ? a : b;
}
```
Decltype is a type specifier used to deduce the type of an expression.

## std::declval type operator
let’s consider a class template that does the composition of two values of different types, and we want to create a type alias for the result of applying
the plus operator to two values of these types. How could such a type alias be defined?
Let’s start with the following form:
```cpp
template <typename T, typename U>
struct composition
{
    using result_type = decltype(???);
};
```
We can use the decltype specifier but we need to provide an expression. We cannot say decltype(T + U) because these are types, not values. We could invoke the default constructor and, therefore, use the expression `decltype(T{} + U{})`. This can work fine for built-in types such as int and double, as shown in the following snippet:
```cpp
template <typename T, typename U>
struct composition
{
    using result_type = decltype(T{} + U{})
};

static_assert(
    std::is_same_v<double,
    composition<int, double>::result_type>);
```
What about for classes that dont have a default constructor? This is where `declval` comes in. It has the following declaration:
```cpp
template<typename T>
T&& declval() noexcept;
```

`declval` exists to be used in what C++ calls an "unevaluated context". This is in a place where **an expression will be parsed, the types used worked out, but the expression will never actually be evaluated**. Some examples of unevaluated context are `sizeof`, `decltype`, etc.

Even though declval has no implementation, it is a function with a well-defined return type. So even though you can't actually execute it, the compiler does know what the type of declval<T>() will be. And therefore, the compiler can examine the expression containing it.

```cpp
template <typename T, typename U>
struct composition
{
    using result_type = decltype(std::declval<T>() +
        std::declval<U>());
};
```
This is saying: assuming I have a **object** of T and a **object** of U, what is the return type of applying the `+` operator to a T object and a U object?


