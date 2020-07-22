---
layout: post
title: Tuple in a Tweet
hidden: true
---

Some time ago [I tweeted](https://twitter.com/pdimov2/status/1222564429016989696)
the following mini-implementation of a `tuple` class template:

```
template<class I, class T> struct tuple_element_base
{
    T t_;
};

template<class... T> struct tuple: mp_apply<mp_inherit,
    mp_transform<tuple_element_base,
        mp_iota_c<sizeof...(T)>, mp_list<T...>>>
{
};

template<class... T> tuple(T...) -> tuple<T...>;
```

This is a functional aggregate tuple. You can
[create one, and pass it around](https://godbolt.org/z/94r7r1):

```
template<class T> void f( T t );

int main()
{
    tuple tp{ 1, 2.1f, "3.14" };
    f( tp );
}
```

It's missing an implementation of the basic tuple primitives
`tuple_size`, `tuple_element_t` and `get`, so you can't do much
else with it yet. But before we add these, let's first figure out
how what we have so far works.

The basic idea is that we want to derive `tuple<T1, T2, T3>` from
`tuple_element_base<0, T1>`, `tuple_element_base<1, T2>`, and
`tuple_element_base<2, T3>`. Each of these base classes will hold
the corresponding tuple element. We also want to keep the index
as a template parameter, both to disambiguate the case when some
of the types are identical, and to be able to look up an element
by index in `get<I>`.

Since Mp11 likes types better than integers, we'll declare
`tuple_element_base` to have two type parameters, and will use
`mp_size_t<I>` instead of just `I` as the first template parameter.

So now, given `tuple<T...>`, we need to somehow turn the parameter
pack `T...` into `tuple_element_base<mp_size_t<I>, T>...`.

First, we prepare type lists holding the two sequences we need,
`mp_size_t<I>...` and `T...`. The second one is trivially
`mp_list<T...>`, the first one is `mp_iota_c<N>`, where N is the 
number of type in `T...`, i.e. `sizeof...(T)`.

Once we have these two lists, let's call them `L1` and `L2`, we need
a way to create a new list such that for every element `A1` from the
first list and `A2` from the second list the result has
`tuple_element_base<A1, A2>`. This is what `mp_transform` does, more
specifically `mp_transform<tuple_element_base, L1, L2>`. Call that
final list `L3`.

Now we need to make `tuple<T...>` derive from each element of `L3`.
Mp11 provides `mp_inherit<T...>`, a type deriving from each element
of the passed parameter pack `T...`. So, given our list `L3`, which
is of the form `mp_list<T...>`, we need to obtain `mp_inherit<T...>`
somehow.

This is done by using `mp_apply<mp_inherit, L3>`, or its equivalent
`mp_rename<L3, mp_inherit>`. Using one or the other is a matter of
personal style. Let's go with `mp_apply` here, and the result is as
above:

```
template<class... T> struct tuple: mp_apply<mp_inherit,
    mp_transform<tuple_element_base,
        mp_iota_c<sizeof...(T)>, mp_list<T...>>>
{
};
```

The odd-looking (if you haven't seen one before) line

```
template<class... T> tuple(T...) -> tuple<T...>;
```

is a C++17 _deduction guide_. It allows us to use the template `tuple`
directly as if it were a type:

```
tuple tp{ 1, 2.1f, "3.14" };
```

without supplying the template parameters (`<int, float, char const*>`
in this case.)

Now we just need to add `tuple_size`:

```
template<class T> using tuple_size = mp_size<std::remove_cv_t<T>>;
```

`tuple_element_t`:

```
template<std::size_t I, class T> using tuple_element_t =
    mp_at_c<std::remove_cv_t<T>, I>;
```

and `get`:

```
template<std::size_t I, class T>
  auto get( tuple_element_base<mp_size_t<I>, T> const & e )
    -> decltype((e.t_))
{
    return e.t_;
}
```

In `get` we take advantage of the fact that the compiler can perform
template argument deduction on some base class when a derived type is
passed. When `get<1>(tp)` is invoked, it sees that the parameter has a
type of `tuple_element_base<mp_size_t<1>, T>`, where `T` is a free template
parameter, and looks into the base classes of `tp` for one that would match.

Since all of the `tuple_element_base` base classes have a distinct first
parameter, only one will match, the one we need. So we just return its member.

Our mini-tuple is now functionally complete, and [we can use it](https://godbolt.org/z/MKd1MW):

```
int main()
{
    tuple tp{ 1, 2.1f, "3.14" };

    mp_for_each<mp_iota<tuple_size<decltype(tp)>>>( [&]( auto I )
    {
        std::cout << get<I>( tp ) << '\n';
    });
}
```
