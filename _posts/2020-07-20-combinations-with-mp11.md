---
layout: post
title: Combinations with Mp11
---

I was asked how can Mp11 be used to implement a `combinations`
metafunction, one that, given a list of types and a number,
produces all sublists of the given length.

For instance, `combinations<2, std::tuple<int, void, float>>` needs
to produce `std::tuple<int, void>`, `std::tuple<int, float>`, and
`std::tuple<void, float>`.

This is not a one liner. Far from it, in fact:

```
struct Q1;
struct Q2;
struct Q3;

template<std::size_t N, class L> using combinations =
    mp_invoke_q<
        mp_cond<
            mp_bool<N == 0>, Q1,
            mp_bool<N == mp_size<L>::value>, Q2,
            mp_true, Q3>,
        mp_size_t<N>, L>;

// N == 0
struct Q1
{
    template<class N, class L> using fn = mp_list<mp_clear<L>>;
};

// N == mp_size<L>
struct Q2
{
    template<class N, class L> using fn = mp_list<L>;
};

// else
struct Q3
{
    template<class N, class L, class R = mp_pop_front<L>> using fn =
        mp_append<
            mp_transform_q<
                mp_bind_back<mp_push_front, mp_front<L>>,
                combinations<N::value - 1, R>
            >,
            combinations<N::value, R>
        >;
};
```

The general idea is to "simply" take all the combinations of length `N`
that contain the first element, by recursively taking all combinations
of length `N-1` from the elements excluding the first, then add to that
all combinations not containing the first element, by recursively taking
all combinations of length `N` from the elements excluding the first.

However, some metaprogramming limitations make the code more complicated
than it needs to be. First, template aliases can't be forward-declared,
and can't be used in their own definitions, so it's not possible to
define a recursive alias template in one go.

Second, the eager nature of template aliases makes it hard to write
"short-circuiting" metafunctions. We want to break the recursion when
`N` is 0 or when `N` is equal to the length of the list, and in these two
special cases, we don't want to perform the above recursive steps.

What we need to do in order to sidestep these limitations is to use
qualified metafunctions (`Q1`, `Q2`, `Q3` above.) Since qualified
metafunctions are ordinary structs, we can forward-declare them, and the
act of selecting one of them using `mp_cond` doesn't evaluate the ones
that remain unselected.
