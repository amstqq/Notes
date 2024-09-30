## std::enable_if
There are two categories of type traits:
- Type traits that enable us to query properties of types at compile-time.
- Type traits that enable us to perform type transformations at compile-time (such
as adding or removing the const qualifier, or adding or removing pointer or reference from a type). These type traits are also called metafunctions.

One metafunction is `std::enable_if`. This is used to enable SFINAE and remove candidates from a function’s overload set. A possible implementation is the following:
```cpp
template<bool B, typename T = void>
struct enable_if {};

template<typename T>
struct enable_if<true, T> { using type = T; };
```

The enable_if metafunction can be used in several scenarios:
- To define a template parameter that has a default argument
```cpp
template <
    typename T,
    typename=typename std::enable_if_t<
                        std::is_integral_v<T>>>
struct integral_wrapper
{
    T value;
};
```
- To define a function parameter that has a default argument
- To specify the return type of a function
```cpp
template <typename T>
typename std::enable_if_t<!uses_write_v<T>> 
serialize(
    std::ostream& os, T const& value
)
{
    os << value;
}
```

