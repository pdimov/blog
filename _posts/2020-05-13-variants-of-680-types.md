---
layout: post
title: Variants of 680 Types
hidden: true
---

[Richard Hodges](https://cppalliance.org/people/richard) asked me whether
[Variant2](https://boost.org/libs/variant2) supports 680 types.

You might be tempted to respond to that with "why would any sane person
need a variant of 680 types??", but we here at Variant2 Enterprises, LLC
ask our customers no such questions, particularly not using more than one
question mark in the process.

The answer turned out to be a qualified "yes", the qualification being a
fast machine with 128GB RAM.

I had never tried putting as many types into `variant2::variant`, just kind
of assumed that it would of course work. It turned out to hit two problems.

First, due to the way `std::variant` is specified to support `constexpr`
operations, a specification which `variant2` follows, the necessary underlying
structure is a recursive union, something like

```
template<class T1, class... T> union storage
{
    T1 first_;
    storage<T...> rest_;
};
```

This meant that 680 types caused 680 template instantiations of this
`storage` type. Template instantiations are expensive, both in time and
in space. Compiler Explorer's time limit (and memory quota) were
[being exceeded](https://godbolt.org/z/2zov2c) for 200 types, let alone
680.

I [added a specialization](https://github.com/boostorg/variant2/commit/465e5bac3d8db05ae98cf4c7ea794ec90ed16610)
that cut down the storage instantiations by a factor of 10, which enabled
[the above example](https://godbolt.org/z/EaFccF) to compile on CE.

The second problem was the implementation of
[`mp11::mp_with_index`](https://www.boost.org/doc/libs/1_73_0/libs/mp11/doc/html/mp11.html#mp_with_indexni_f),
which powers `variant2::visit`. It generates a switch of however many
alternatives are requested (680 in our case), but did so 16 alternatives
at a time, which meant 40+ switch statements. This, again for 200 types, often
[exceeded the ability](https://godbolt.org/z/u68FSf) of CE's version of g++.

I [changed](https://github.com/boostorg/mp11/commit/13c36a793c397c1fc75c4e4c5be10e1338680622)
`mp_with_index` to divide the alternatives in half, and it
[conquered](https://godbolt.org/z/aniMjG).

So now you no longer need 128GB RAM to use variants of 680 types, although
they still don't set any compilation speed records.
