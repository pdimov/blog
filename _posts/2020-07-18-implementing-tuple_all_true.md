---
layout: post
title: Implementing tuple_all_true
hidden: true
---

Suppose you need a function `tuple_all_true` that, when invoked on a tuple,
returns whether all of its elements are `true` when interpreted as booleans.

Why? Well, you might have `std::tuple<std::optional<X>, std::optional<Y>>`
and want to check whether all the optionals are engaged.

Can you do that with Mp11? Yes, you can.

```
template<class... T> bool tuple_all_true( std::tuple<T...> const& tp )
{
    bool r = true;
    boost::mp11::tuple_for_each( tp, [&](auto&& e){ r = r && e; } );
    return r;
}
```

The generated code [is fine](https://godbolt.org/z/18oh5z).

Speaking of Compiler Explorer, the output when using MSVC is significantly
cleaner when you pass the `-Zc:inline` option, as this doesn't emit the
unreferenced inline functions.
