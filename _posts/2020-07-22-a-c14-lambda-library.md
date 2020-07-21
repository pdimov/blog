---
layout: post
title: A C++14 Lambda Library
hidden: true
---

In C++11 and later, we have language support for lambda expressions,
so we can write this:

```
int count_even( int const * first, int const * last )
{
    return std::count_if( first, last, []( int x )
        { return x % 2 == 0; } );
}
```

and this:

```
char const * find_whitespace( char const * first, char const * last )
{
    return std::find_if( first, last, []( char ch )
        { return ch == ' ' || ch == '\t'
            || ch == '\r' || ch == '\n'; } );
}
```

but people
[aren't quite happy](https://brevzin.github.io/c++/2020/06/18/lambda-lambda-lambda/).
Compared to other languages, that's a verbose way to express a simple inline
function object, they say.

And you know what? They are right. It is. Here's how these looked like
in the dark ages of C++03, when we had to use primitive tools such as
[Boost.Lambda](https://www.boost.org/libs/lambda):

```
int count_even( int const * first, int const * last )
{
    return std::count_if( first, last, _1 % 2 == 0 );
}

char const * find_whitespace( char const * first, char const * last )
{
    return std::find_if( first, last,
        _1 == ' ' || _1 == '\t' || _1 == '\r' || _1 == '\n' );
}
```

We kind of went backwards here. How about we recreate the Boost.Lambda
experience in C++14, using _modern technology_; how many lines would
that take?

I asked a highly representative sample of C++ developers (two people
chosen at random) and they answered "3,000" and "5,000", respectively.

They couldn't have been more wrong if they tried. The correct answer is **52**.

Yes, implementing a lambda library that makes the above two examples
compile and work literally fits into a blog post, as I shall demonstrate:

```
#include <functional>
#include <type_traits>

template<class T, class T2 = std::remove_reference_t<T>>
    using is_lambda_expression = std::integral_constant<bool,
        std::is_placeholder<T2>::value ||
        std::is_bind_expression<T2>::value>;

template<class A> using enable_unary_lambda =
    std::enable_if_t<is_lambda_expression<A>::value>;

#define UNARY_LAMBDA(op, fn) \
    template<class A, class = enable_unary_lambda<A>> \
    auto operator op( A&& a ) \
    { \
        return std::bind( std::fn<>(), std::forward<A>(a) ); \
    }

template<class A, class B> using enable_binary_lambda =
    std::enable_if_t<is_lambda_expression<A>::value ||
        is_lambda_expression<B>::value>;

#define BINARY_LAMBDA(op, fn) \
    template<class A, class B, class = enable_binary_lambda<A, B>> \
    auto operator op( A&& a, B&& b ) \
    { \
        return std::bind( std::fn<>(), std::forward<A>(a), \
            std::forward<B>(b) ); \
    }

BINARY_LAMBDA(+, plus)
BINARY_LAMBDA(-, minus)
BINARY_LAMBDA(*, multiplies)
BINARY_LAMBDA(/, divides)
BINARY_LAMBDA(%, modulus)
UNARY_LAMBDA(-, negate)

BINARY_LAMBDA(==, equal_to)
BINARY_LAMBDA(!=, not_equal_to)
BINARY_LAMBDA(>, greater)
BINARY_LAMBDA(<, less)
BINARY_LAMBDA(>=, greater_equal)
BINARY_LAMBDA(<=, less_equal)

BINARY_LAMBDA(&&, logical_and)
BINARY_LAMBDA(||, logical_or)
UNARY_LAMBDA(!, logical_not)

BINARY_LAMBDA(&, bit_and)
BINARY_LAMBDA(|, bit_or)
BINARY_LAMBDA(^, bit_xor)
UNARY_LAMBDA(~, bit_not)
```

This even generates [reasonably](https://godbolt.org/z/jj1fb3)
[efficient](https://godbolt.org/z/xcEEx7) [code](https://godbolt.org/z/s1qT94).

What's the trick here?

The trick is that the author of `std::bind` included, quite consciously,
a few foundational bits that make implementing the above lambda library
trivial, as long as we have function objects corresponding to all the
operators we want to support; and then, in C++14 a different unrelated proposal
[added](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2012/n3421.htm)
the exact function objects we need.

In the corporate circles this is known as "synergy".
