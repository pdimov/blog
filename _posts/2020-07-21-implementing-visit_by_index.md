---
layout: post
title: Implementing visit_by_index
---

When manipulating variants, one often needs to perform different actions
depending on the type in the variant. For example, when a
`std::variant<int, float>` contains an `int x`, we want to perform

```
std::printf( "int: %d\n", x );
```

but in the case it contains a `float x`, we want

```
std::printf( "float: %f\n", x );
```

One way to do this is to define an appropriate function object

```
struct V
{
    void operator()(int x) const
    {
        std::printf( "int: %d\n", x );
    }

    void operator()(float x) const
    {
        std::printf( "float: %f\n", x );
    }
};
```

and then call `std::visit` with it:

```
void f( std::variant<int, float> const & v )
{
    std::visit( V(), v );
}
```

[This works](https://godbolt.org/z/WzTnd8), but having to define
a dedicated function object is not as convenient as just using a lambda.
Lambdas, however, can't have more than one `operator()`.

One solution that has been [suggested](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2018/p0051r3.pdf)
is to have an `overload` function that, when passed several lambdas, creates
a composite function object with an overloaded `operator()` out of its
arguments.

However, this doesn't strike me as very elegant. We combine our
several actions into one, pass it to `std::visit`, which then needs to
undo this composition in order to call the right overload depending on
the containing type.

How about we throw all this composition and decomposition away and
just define a visitation function that takes the lambdas directly, and
invokes the correct one based on the containing type? Or better yet,
based on the variant index, so that we no longer need to depend on
the types being unique?

```
template<class V, class... F> void visit_by_index( V&& v, F&&... f );

// Effects: if v.index() is I, calls the I-th f with get<I>(v).
```

This would allow us to just do

```
void f( std::variant<int, float> const & v )
{
    visit_by_index( v,

        [](int x)
        {
            std::printf( "int: %d\n", x );
        },

        [](float x)
        {
            std::printf( "float: %f\n", x );
        }
    );
}
```

Is `visit_by_index` a one liner using Mp11?

```
template<class V, class... F> void visit_by_index( V&& v, F&&... f )
{
    constexpr auto N = std::variant_size<
        std::remove_reference_t<V> >::value;

    boost::mp11::mp_with_index<N>( v.index(), [&]( auto I )
    {
      std::tuple<F&&...> tp( std::forward<F>(f)... );
      std::get<I>(std::move(tp))( std::get<I>(std::forward<V>(v)) );
    } );
}
```

Almost, at least in its initial iteration. To make this more suitable
for production use, we need to handle the `valueless_by_exception()` case,
and check that the number of function objects matches the number
of variant alternatives:

```
template<class V, class... F> void visit_by_index( V&& v, F&&... f )
{
    if( v.valueless_by_exception() )
    {
        throw std::bad_variant_access();
    }

    constexpr auto N = std::variant_size<
        std::remove_reference_t<V> >::value;

    static_assert( N == sizeof...(F),
        "Incorrect number of function objects" );

    boost::mp11::mp_with_index<N>( v.index(), [&]( auto I )
    {
      std::tuple<F&&...> tp( std::forward<F>(f)... );
      std::get<I>(std::move(tp))( std::get<I>(std::forward<V>(v)) );
    } );
}
```

How does [this work](https://godbolt.org/z/f4c4Y3)?

`mp_with_index<N>(i, f)` (which requires `i` to be between `0` and `N-1`
inclusive) calls `f` with `std::integral_constant<std::size_t, i>`
(by generating a big `switch` over the possible values of `i`.)

`f` in our case is a lambda object of the form `[&](auto I){ ... }`, so `I`
gets to be a variable of type `std::integral_constant<std::size_t, i>`.

Since `std::integral_constant<T, K>` has a `constexpr` conversion operator
to `T` returning `K`, we can use `I` in places where a constant expression
of type `size_t` is expected, such as `std::get<I>`.

`std::tuple<F&&...> tp( std::forward<F>(f)... );` creates a tuple referencing
the function objects passed as arguments. `std::get<I>(std::move(tp))`
returns the `I`-th one of them, properly forwarded.

`std::get<I>(std::forward<V>(v))`, since `I` is `v.index()`, returns a
reference to the type contained into `v`, again properly forwarded. Therefore,
`std::get<I>(std::move(tp))( std::get<I>(std::forward<V>(v)) );` calls
the correct function object with the correct argument. QED.
