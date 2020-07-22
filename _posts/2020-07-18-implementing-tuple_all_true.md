---
layout: post
title: Implementing tuple_all_true
---

Suppose you need a function `tuple_all_true` that, when invoked on a tuple,
returns whether all of its elements are `true` when interpreted as booleans.

Why? Well, you might have a
`std::tuple<std::optional<X>, std::optional<Y>, std::optional<Z>>` and want
to check whether all the optionals are engaged.

Can you do that with Mp11? Yes, you can.

```
template<class... T> bool tuple_all_true( std::tuple<T...> const& tp )
{
    bool r = true;
    boost::mp11::tuple_for_each( tp, [&](auto&& e){ r = r && e; } );
    return r;
}
```

`mp11::tuple_for_each` invokes the supplied function object on each
element of the tuple, so we use a lambda that checks whether the
element evaluates as `true`.

If we could use a `for` loop here, we would probably have used an
early `return`, something like (using a
[hypothetical](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2019/p1306r1.pdf)
`for...` syntax)

```
    for...(auto e: tp)
    {
        if( !e ) return false;
    }

    return true;
```

Unfortunately, if we use `return` in our lambda, it would just return
from the lambda, not the containing function, so we need to use the less
intuitive approach of accumulating the result into a variable `r`.

This may look inefficient -- the loop will visit every `e` instead of
stopping early -- but today's compilers are smart enough to figure out
that `r && e` does not evaluate `e` when `r` is `false`, and
generate [reasonably efficient code](https://godbolt.org/z/18oh5z).

A tip when using Compiler Explorer: the MSVC output is significantly
cleaner when you pass the `-Zc:inline` option, as the compiler doesn't
then emit the unreferenced inline functions.
