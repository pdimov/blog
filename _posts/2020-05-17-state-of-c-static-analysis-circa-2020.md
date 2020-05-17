---
layout: post
title: State of C++ Static Analysis circa 2020
hidden: true
---

Take the following code:

```
#include <string_view>

int * f1()
{
    int x = 5;
    return &x;
}

struct V
{
    int * p;
};

V f2()
{
    int x = 5;
    return { &x };
}

std::string_view f3()
{
    char tmp[] = "tmp";
    return tmp;
}
```

All three functions _obviously_ return dangling pointers to local
stack variables. Let's see what a few major compilers have to say
on the matter.

`g++ 10.1 -O2 -std=c++2a -fanalyzer -Wall -Wextra` ([link](https://godbolt.org/z/NU6vUc)):

```
f1():
        xor     eax, eax
        ret
f2():
        lea     rax, [rsp-4]
        ret
f3():
        mov     eax, 3
        lea     rdx, [rsp-4]
        ret
```
```
<source>: In function 'int* f1()':
<source>:6:12: warning: address of local variable 'x' returned [-Wreturn-local-addr]
    6 |     return &x;
      |            ^~
```

In addition to the warning in `f1`, it even zapped the pointer to `nullptr`. An interesting
choice, with which not everyone agrees, but in my opinion returning a null pointer is
much better than returning a dangling pointer to just-deallocated stack memory... which is
exactly what happens in `f2` and `f3`.

Let's try Microsoft `cl.exe 19.24 /O2 /std:c++latest /W4 /analyze` ([link](https://godbolt.org/z/OSPCjC)):

```
<source>(6) : warning C4172: returning address of local variable or temporary: x
<source>(17) : warning C4172: returning address of local variable or temporary: x
```

That's better, but not better enough. `std::string_view` is a rather important type, and
a potential rich source of lifetime mistakes.

Maybe Intel `icc 19.0.1 -O2 -std=c++17 -Wall -Wextra` ([link](https://godbolt.org/z/-2jd3J)) will fare better?

```
<source>(6): warning #1251: returning pointer to local variable
      return &x;
             ^
```

Sadly, not really. `clang++ 10.0.0 -O2 -std=c++2a -Wall -Wextra` ([link](https://godbolt.org/z/Bj6Ke_)) is our last hope.

```
<source>:6:13: warning: address of stack memory associated with local variable 'x' returned [-Wreturn-stack-address]
    return &x;
            ^
<source>:17:15: warning: address of stack memory associated with local variable 'x' returned [-Wreturn-stack-address]
    return { &x };
              ^
```

Good but not still good enough.

Everything is lost, then? We'll never have compilers that catch obvious lifetime mistakes?

Maybe not. Let's try our real last hope, the experimental `-Wlifetime` build of `clang` ([link](https://godbolt.org/z/QjvkX6)):

```
<source>:6:13: warning: address of stack memory associated with local variable 'x' returned [-Wreturn-stack-address]
    return &x;
            ^
<source>:6:5: warning: returning a dangling pointer [-Wlifetime]
    return &x;
    ^~~~~~~~~
<source>:6:5: note: pointee 'x' left the scope here
    return &x;
    ^~~~~~~~~
<source>:17:15: warning: address of stack memory associated with local variable 'x' returned [-Wreturn-stack-address]
    return { &x };
              ^
<source>:23:12: warning: address of stack memory associated with local variable 'tmp' returned [-Wreturn-stack-address]
    return tmp;
           ^~~
<source>:23:5: warning: returning a dangling pointer [-Wlifetime]
    return tmp;
    ^~~~~~~~~~
<source>:23:5: note: pointee 'tmp' left the scope here
    return tmp;
    ^~~~~~~~~~
```

Interesting. Not only did `-Wlifetime` catch `f1` and `f3` (but not `f2` for some reason!), the
normal `-Wreturn-stack-address` warning caught `f3` this time as well, in addition to `f1` and `f2`.

(Herb Sutter has [an interesting post about the experimental
`-Wlifetime` compiler](https://herbsutter.com/2018/09/20/lifetime-profile-v1-0-posted/). It can't arrive soon enough
if you ask me.)
