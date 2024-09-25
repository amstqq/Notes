## Parameter packs expansion

### Template parameter list
Specify parameters for a template
```cpp
template <typename... T> // template parameter list
struct outer
{
    template <T... args>
    struct inner {};
};
outer<int, double, char[5]> a;
```

### Template argument list
specify arguments for a template
```cpp
template <typename... T>
struct tag {};
template <typename T, typename U, typename ... Args>
void tagger()
{
    tag<T, U, Args...> t1; // template argument list
    tag<T, Args..., U> t2;
    tag<Args..., T, U> t3;
    tag<U, T, Args...> t4;
}
```

### Function parameter list and argument list
specify parameters and arguments for a function template:
```cpp
template <typename... Args> // function parameter list
void make_it(Args... args) // function argument list
{
}

make_it(42);
make_it(42, 'a');
```

### Fold expression
Discussed in detail later. A fold expression applies a binary operator (like +, *, &&, etc.) across all elements in a parameter pack.
```cpp
template<typename... Args>
auto sum(Args... args) {
    return (... + args);
}
```

### Using declarations
introduce names from the base classes into the definition of the derived class

```cpp
template<typename... Bases>
struct X : public Bases...
{
    X(Bases const & ... args) : Bases(args)...
    {}
    using Bases::execute...;
};

A a;
B b;
C c;
X x(a, b, c);

x.A::execute();
x.B::execute();
```

### Parameter pack followed by another parameter pack
A parameter pack can be followed by another parameter pack:
```cpp
template <typename... Ts, typename... Us>
constexpr auto multipacks(Ts... args1, Us... args2)
{
    std::cout << sizeof...(args1) << ',' 
              << sizeof...(args2) << '\n';
};

multipacks<int>(1, 2, 3, 4, 5, 6);
// 1,5
multipacks<int, int, int>(1, 2, 3, 4, 5, 6);
// 3,3
multipacks<int, int, int, int>(1, 2, 3, 4, 5, 6);
// 4,2
multipacks<int, int, int, int, int, int>(1, 2, 3, 4, 5, 6);
// 6,0
```

## Fold expression
A fold expression is an expression involving a parameter pack that folds (or reduces) the elements of the parameter pack over a binary operator.
- ( pack op ... ) --> unary right fold
    - (arg1 op (... op (argN-1 op argN)))
- ( ... op pack ) --> Unary left fold
    - ((arg1 op arg2) op ...) op argN)
- ( pack op ... op init ) --> Binary right fold
    - (arg1 op (... op (argN op init)))
- ( init op ... op pack ) --> Binary left fold
    - (((init op arg1) op ...) op argN)

Note:
- `pack` is an expression that contains an unexpanded parameter pack, and arg1,
arg2, argN-1, and argN are the arguments contained in this pack.
- op is one of the following binary operators: + - * / % ^ & | = < > << >> += -= *= /= %= ^= &= |= <<= >>= == != <= >= && || , .* ->*.
- init is an expression that does not contain an unexpanded parameter pack.

In a unary fold, if the pack does not contain any elements, only some operators are allowed. These are listed in the following table, along with the value of the empty pack:
- && --> true
- || --> false
- , --> void()

Example
```cpp
template <typename... Ts>
int sum(Ts... args)
{
    return (args + ...);
    // ((arg0 + arg1) + arg2) + ...
}

int s1 = sum(); // error because unary fold expressions over '+' operator must have non-empty expressions

template <typename... Ts>
int sumFromZero(Ts... args)
{
    return (0 + ... + args);
    // (0 + arg1) + arg2 + ... 
}

int s2 = sumFromZero(); // valid

// unary fold expression over the comma operator
template <typename... T>
void printl(T... args)
{
    ((std::cout << args), ...) << '\n';
    // (((std::cout << arg1), std::cout << arg2), ...)
}

// another example
template <typename... T, typename... Args>
void push_back(std::vector<T>& vec, Args... args)
{
    (vec.push_back(args), ...);
}
```

Benefits of fold expression:
- Less and simpler code to write.
- Fewer template instantiations, which leads to faster compile times.
- Potentially faster code since multiple function calls are replaced with a single
expression. However, this point may not be true in practice, at least not when
optimizations are enabled. We have already seen that the compilers optimize code by removing these function calls.

## Variadic class template example: Tuple
### Tuple
```cpp
template<typename T, typename... Ts>
class tuple
{
  public:
    tuple(const T& t, const Ts&... ts)
      : value(t)
      , rest(ts...)
      {}
  

  	T value;
  	tuple<Ts...> rest;
};

template<typename T>
class tuple<T>
{
  public:
    tuple(const T& t)
      : value(t)
      {}
   
   	T value;
};
```

### Nth type helper
Define variadic class template that determines what the type of the element is at the N index of the tuple.
```cpp
template<size_t N, typename... Ts>
struct nth_type : nth_type<N-1, Ts...>
{
};

template<typename T, typename... Ts>
struct nth_type<0, T, Ts...>
{
  using value_type = T;
};
```

Flattened template from cppinsights.io
```cpp
template<>
struct nth_type<2, int, double, char> : public nth_type<1, double, char>
{ };

template<>
struct nth_type<1, double, char> : public nth_type<0, char>
{ };

template<>
struct nth_type<0, char>
{
    using value_type = char;
};
```
Note that type of nth element in tuple is determined at compile time.

### Nth value helper

```cpp
template<size_t N>
struct getter
{
   template <typename... Ts>
   static typename nth_type<N, Ts...>::value_type get(const tuple<Ts...>& t)
   {
     return getter<N-1>::get(t.rest);
   }
};

template<>
struct getter<0>
{
	template <typename T, typename... Ts>
    static const T& get(const tuple<T, Ts...>& t)
    {
    	return t.value;
	}
};
```

Flattened:
```cpp
template<>
struct getter<2>
{
    template<>
    static inline typename
    nth_type<2UL, int, double, char>::value_type& 
    get<int, double, char>(tuple<int, double, char> & t)
    {
        return getter<1>::get(t.rest);
    }
};

template<>
struct getter<1>
{
    template<>
    static inline typename nth_type<1UL, double, char>::value_type&
    get<double, char>(tuple<double, char> & t)
    {
        return getter<0>::get(t.rest);
    }
};

template<>
struct getter<0>
{
    template<>
    static inline char & get<char>(tuple<char> & t)
    {
        return t.value;
    }
};
```

Finally, the get function:
```cpp
template<size_t N, typename... Ts>
typename nth_type<N, Ts...>::value_type get(const tuple<Ts...>& t)
{
  return getter<N>::get(t);
}
```