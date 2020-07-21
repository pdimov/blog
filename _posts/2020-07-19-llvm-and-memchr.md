---
layout: post
title: LLVM and memchr
hidden: true
---

LLVM, the back end of Clang, does something remarkable with this
ordinary-looking call to `memchr`:

```
bool is_whitespace( char ch )
{
    return std::memchr( " \t\r\n", ch, 4 ) != 0;
}
```

It turns it into [this](https://godbolt.org/z/7M5ad4):

```
is_whitespace(char):
        cmp     dil, 64
        setb    cl
        movabs  rax, 4294977024
        bt      rax, rdi
        setb    al
        and     al, cl
        ret
```

What's going on here? This doesn't bear any resemblance to what we wrote!

Well, the first step of the explanation is that LLVM recognizes this
`memchr(...) != 0` pattern and understands that it's equivalent to

```
bool is_whitespace_2( char ch )
{
    return ch == ' ' || ch == '\t' || ch == '\r' || ch == '\n';
}
```

It so happens that LLVM generates something
[slightly different](https://godbolt.org/z/zodd9z) for `is_whitespace_2`,
because the pattern in `is_whitespace_2` engages a different code path
in the optimizer.

[Nevertheless](https://books.google.bg/books?id=fr5SBAAAQBAJ&pg=PT88&lpg=PT88&dq=nevertheless&source=bl&ots=HnmWRXPRU0&sig=ACfU3U3oRHjPTzabiIt6NMUTw4vcD5ozIg&hl=en&sa=X&ved=2ahUKEwjH5p3l_N7qAhVyAmMBHQSjBqkQ6AEwAXoECAoQAQ#v=onepage&q=nevertheless&f=false),
if we [use MSVC instead](https://godbolt.org/z/YYoKKn),
we can get something very close to the code produced for `is_whitespace`:

```
bool is_whitespace_2(char) PROC
        cmp     cl, 32
        ja      SHORT $LN5@is_whitesp
        mov     rax, 4294977024
        bt      rax, rcx
        jae     SHORT $LN5@is_whitesp
        mov     al, 1
        ret     0
$LN5@is_whitesp:
        xor     al, al
        ret     0
bool is_whitespace_2(char) ENDP
```

The constant 4294977024 here is 100002600 in hex, and has bits 9 (`'\t'`), 10
(`'\r'`), 13 (`'\n'`), and 32 (`' '`) set.

So, basically, what this does is

```
bool is_whitespace_3( unsigned char ch )
{
    uint64_t mask = (1ull << ' ') | (1ull << '\t')
        | (1ull << '\r') | (1ull << '\n');

    return ch <= 32 && ((1ull << ch) & mask);
}
```

instead of doing four comparisons.

Fine, but why does LLVM recognize the above `memchr` use at all? Nobody would
write the `is_whitespace` function this way.

Well, it turns out that the `find_first_not_of` member function of `std::string`
and `std::string_view` is commonly implemented as something along the lines of

```
std::size_t string_view::find_first_not_of( string_view s ) const
{
    char const * p = data();
    std::size_t n = size();

    for( std::size_t i = 0; i < n; ++i )
    {
        if( std::memchr( s.data(), p[i], s.size() ) == 0 ) return i;
    }

    return npos;
}
```

Note the `memchr(...) == 0` pattern in the loop. So when someone compiles

```
std::size_t f( std::string_view s )
{
    return s.find_first_not_of( " \t\r\n" );
}
```

with Clang, the `memchr` optimization kicks in under both
[libstdc++](https://godbolt.org/z/dnsxT5) and [libc++](https://godbolt.org/z/9GGqcr).

Cheaters, all of them.
