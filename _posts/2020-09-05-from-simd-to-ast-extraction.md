---
layout: post
title: From SIMD to AST Extraction
hidden: true
---

Suppose we have the functions

```
float f( float x )
{
    return x * 2.0f + 1.0f;
}

float g( float x, float y )
{
    return f( x ) * 0.3f + f( y ) * 0.7f;
}
```

and we need to apply `g` to the two arrays `x` and `y`,
storing the result in the array `z`:

```
void h( float const * x, float const * y, float * z, std::size_t n )
{
    for( std::size_t i = 0; i < n; ++i )
    {
        z[ i ] = g( x[ i ], y[ i ] );
    }
}
```

Nowadays, all major compilers
[automatically vectorize](https://godbolt.org/z/q148se)
this code and generate SIMD instructions for it -- at most,
we need to pass [-O3](https://godbolt.org/z/c71vo5) instead
of [-O2](https://godbolt.org/z/63hqG4) to GCC. But let's
suppose, for the sake of discussion, that it's 2008, the
compilers don't autovectorize, and we still want to employ SIMD.

One elegant technique that allows us to keep our functions
mostly as-is is to convert them to templates:

```
template<class T> T f( T x )
{
    return x * 2.0f + 1.0f;
}

template<class T> T g( T x, T y )
{
    return f( x ) * 0.3f + f( y ) * 0.7f;
}
```

This still lets us to call them with `float` as before, but
it also enables us calling them with a
[SIMD pack of four floats](https://gcc.gnu.org/onlinedocs/gcc/Vector-Extensions.html):

```
using m128 = __attribute__(( vector_size( 4*sizeof(float) ) )) float;
```

so that we can now rewrite `h` to
[work at four elements at a time](https://godbolt.org/z/Y8xqd9):

```
void h( float const * x, float const * y, float * z, std::size_t n )
{
    std::size_t i = 0;

    for( ; i + 3 < n; i += 4 )
    {
        m128 xi;
        std::memcpy( &xi, x + i, sizeof( m128 ) );

        m128 yi;
        std::memcpy( &yi, y + i, sizeof( m128 ) );

        m128 zi = g( xi, yi );

        std::memcpy( z + i, &zi, sizeof( m128 ) );
    }

    for( ; i < n; ++i )
    {
        z[ i ] = g( x[ i ], y[ i ] );
    }
}
```

OK, but what's the point of all this in 2020?

Well, it turns out that templatizing our functions enables more
than vectorization. We can pass other things to them. In particular,
we can define a type that instead of doing calculations when operators
such as `+` and `*` are applied to it, builds an abstract syntax tree
instead.

This means that when we call `g` with this type, instead of the value
`g(x)` at some point `x`, we can get a symbolic representation of the body
of `g`.

To illustrate that, I will define a simple type `Q` that for reasons of
brevity will build a string representation of the function body, instead
of a proper syntax tree:

```
struct Q
{
    std::string s_;

    Q( std::string const & s ): s_( s ) {}
    Q( float x ): s_( std::to_string( x ) ) {}
};

Q operator+( Q const& q1, Q const& q2 )
{
    return { "(" + q1.s_ + " + " + q2.s_ + ")" };
}

Q operator*( Q const& q1, Q const& q2 )
{
    return { "(" + q1.s_ + " * " + q2.s_ + ")" };
}

std::ostream& operator<<( std::ostream& os, Q const& q )
{
    return os << q.s_;
}
```

Now, when I pass this type to our function `g`:

```
    std::cout << g( Q{"x"}, Q{"y"} ) << std::endl;
```

I [get](https://godbolt.org/z/jonEWT)

```
((((x * 2.000000) + 1.000000) * 0.300000) + (((y * 2.000000) + 1.000000) * 0.700000))
```

which is exactly what `g` does.
