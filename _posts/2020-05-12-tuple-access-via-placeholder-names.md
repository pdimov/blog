---
layout: post
title: Tuple Access via Placeholder Names
---

The elements of a `std::tuple` `tp` are accessed by `get<0>(tp)`, `get<1>(tp)`,
and so on. A tuple is just an anonymous struct, so it would have been kind of
nice if we could use `tp._1`, `tp._2`, and so on, instead. (Since elements
have no names, we have to synthesize some. The [Circle](https://www.circle-lang.org/)
language does something similar, except it (erroneously) starts from `_0`.)

We can't implement `tp._1` because `operator.` is not overloadable. We can't
implement `tp->_1` because even though `operator->` is overloadable, it doesn't
work for our needs; we can only modify the object on which the member access
occurs, not the field name.

What about `tp.*_1`? Not overloadable, but wait, `->*` is. Can we use it?

We'll need to define `_1`, `_2` and so on... or maybe not. The standard
already defines these for us, in the header `<functional>` and the
namespace `std::placeholders`. These are ordinarily used with `std::bind`, but
nothing stops us from repurposing them.

We just need to define `operator->*` which takes a `std::tuple` on the left
side and a placeholder type on the right side... except that the placeholder
types are _unspecified_.

Fortunately, the person who designed `std::bind` (that would be me) provided a
way to recognize whether a type is a placeholder type: the type trait
`std::is_placeholder`. `std::placeholder<T>::value` is 1 when `T` is `_1`, 2
when `T` is `_2`, 3 when `T` is `_3`... and 0 otherwise.

There we go:

```
#include <functional>
#include <tuple>
#include <type_traits>

template<class T, class U,
  std::size_t I = std::size_t{std::is_placeholder<U>::value - 1},
  std::size_t J = std::tuple_size<std::remove_reference_t<T>>::value>
decltype(auto) operator->*( T&& t, U ) noexcept
{
    return std::get<I>( std::forward<T>( t ) );
}
```

Ignore the details, and this is pretty straightforward -- `t->*_1` returns `std::get<1-1>(t)`.
The details are needed to constrain the operator so that it only applies to things that
are tuples on the left side, and things that are placeholders on the right side.

When `U` is not a placeholder type, `std::is_placeholder<U>::value` is 0, and
`std::size_t{std::is_placeholder<U>::value - 1}` is `std::size_t{-1}`. This is called a
_narrowing conversion_ and is an error, specifically a "soft" error called a
_substitution failure_. The result is that the operator is not considered when `U` is not
a placeholder, which is exactly what we want.

When `T` is not a tuple (which means one of `std::tuple`, `std::pair`, `std::array`),
`std::tuple_size<T>::value` will not be defined, and we again get our desired substitution
failure.

Does this work?

```
#include <iostream>

int main()
{
    using namespace std::placeholders;

    auto t = std::make_tuple( 1, 2.0, "3" );

    std::cout << t->*_1 << std::endl;
    std::cout << t->*_2 << std::endl;
    std::cout << t->*_3 << std::endl;
}
```

[It does.](https://godbolt.org/z/ARZqXc)
