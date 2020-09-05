---
layout: post
title: Named Parameters in C++20
hidden: true
---

A programming language supports _Named Parameters_ when one
can call a function supplying the parameters by name, as in
the following hypothetical example (using C++ syntax):

```
void f( int x, int y );

int main()
{
    f( x = 1, y = 2 );
}
```

C++ is obviously not such a language and there have been
numerous proposals to rectify this omission, unfortunately none
of them successful. The latest attempt is Axel Naumann's paper
[Self-explanatory Function Arguments](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2018/p0671r2.html),
which attempts to attack the problem from another angle by just
allowing normal function calls to be tagged with the parameter
name, as in

```
    f( x: 1, y: 2 );
```

enabling compilers to issue helpful warnings when a name doesn't
match, but not allowing one to omit, or reorder, arguments.

Even in this limited form, named parameters would still be immensely
useful, but this is not what this post is about. What this post is
about is that we can already achieve something very close to named
parameters in C++20, by using a C99 feature called _designated
initializers_.

Designated initializers allow one to initialize structures by
member name, as in the following example:

```
struct A
{
    int x;
    int y;
};

A a1 = { .x = 1, .y = 2 };
A a2 = { .x = 3 }; // a2.y == 0
A a3 = { .y = 4 }; // a3.x == 0
A a4 = { .y = 5, .x = 6 }; // valid C, invalid C++ (reorder)
```

C++ introduces a restriction C doesn't have: the initializers
must follow the declaration order, similarly to how class member
initalizers are executed in member declaration order. But in
exchange, it allows us to supply default values:

```
struct A
{
    int x = 0;
    int y = 0;
};

A a3 = { .y = 4 }; // a3.x == 0, no warning
```

You can already see where this is going. Instead of

```
void f( int x, int y );
```

we declare

```
void f( A args );
```

and then call it like this:

```
int main()
{
    f({ .x = 1, .y = 2 });
}
```

This works under [GCC](https://godbolt.org/z/YfWj3W) and
[Clang](https://godbolt.org/z/vbnz4T) even without `-std=c++20` because
they support designated initializers in earlier language modes as
an extension, and it works under [MSVC](https://godbolt.org/z/bKozaW)
with `-std:c++latest`.

For a more realistic example, consider this snippet, taken from real
code, that sets a 10 second timeout on a Boost.Beast websocket:

```
#include <boost/beast/websocket/stream.hpp>
#include <boost/beast/core/tcp_stream.hpp>
#include <chrono>

void f1( boost::beast::websocket::stream<boost::beast::tcp_stream>& ws )
{
    auto opt = boost::beast::websocket::stream_base::timeout();

    opt.keep_alive_pings = true;
    opt.idle_timeout = std::chrono::seconds(10);

    ws.set_option(opt);
}
```

Here's how we can reformulate it by using the above idiom and `<chrono>`
literals:

```
#include <boost/beast/websocket/stream.hpp>
#include <boost/beast/core/tcp_stream.hpp>
#include <chrono>

using namespace std::chrono_literals;

void f2( boost::beast::websocket::stream<boost::beast::tcp_stream>& ws )
{
    ws.set_option({ .idle_timeout = 10s, .keep_alive_pings = true });
}
```

Apart from the slightly awkward `({ ... })` syntax and the need to observe
the right parameter order, that's not that far from the ideal; and it's
considerably better than `f1`.
