---
layout: post
title: Compilers Do Static Analysis, They Just Don't Tell You
hidden: true
---

Let's take the following code:

```
int * f()
{
    int x = 2;
    return &x;
}

int g( int * p  )
{
    return 3 + *( p? f(): p );
}
```

It has two errors in it. First, `f` returns a pointer to a local variable,
which goes out of scope. Second, the check in `g` is reversed; in the case
`p` is a null pointer, it dereferences it, instead of doing the opposite --
dereference `p` only when it's not null.

This is what [GCC emits](https://godbolt.org/z/rbn9KP) for `g`:

```
g(int*):
        mov     eax, DWORD PTR ds:0
        ud2
```

That is, it can obviously see that `g` will either dereference a null pointer
(undefined behavior), or an invalid pointer (undefined behavior), so it generates
code that dereferences a null pointer (`mov eax, [0]`), which crashes, and then
emits the guaranteed-invalid instruction `ud2`, which crashes.

(You can't be too sure, is GCC's motto.)

And this is what [Clang emits](https://godbolt.org/z/zW5dzY) for the same function:

```
g(int*):
        ret
```

Presumably, the logic here is that one of the branches leads to undefined behavior,
and the other also leads to undefined behavior, so we might as well remove the whole
thing, it's not going to get called anyway in a correct program. (But if it does, let's
just return, as crashing would be gauche.)

Now, I don't know about you, but I'm left wondering here; if the compilers can clearly
see that all possible code paths through this function lead to undefined behavior,
why don't they _tell us that_? Something like "Warning: function invokes undefined behaviour
on all control paths, you might want to check your code, mate", except less formal.

Who cares about that, some might say, nobody writes such code anyway, this will catch
no bugs. Well, I have obviously oversimplified a bit. Let's take a more realistic example,
such as this one:

```
int slice_sum( std::vector<int> const& v, int i, int n )
{
    int s = 0;

    for( int j = 0; j < n; ++j )
    {
        assert( i+j >= 0 );
        assert( i+j < v.size() );
        
        s += v[ i+j ];
    }

    return s;
}

int f()
{
    std::vector<int> v{ 1, 2, 3, 4 };
    return slice_sum( v, 3, 2 );
}
```

What does [GCC do](https://godbolt.org/z/5YGYYd)?

```
.LC0:
        .string "int slice_sum(const std::vector<int>&, int, int)"
.LC1:
        .string "./example.cpp"
.LC3:
        .string "i+j < v.size()"

f():
        sub     rsp, 8
        mov     ecx, OFFSET FLAT:.LC0
        mov     edx, 11
        mov     esi, OFFSET FLAT:.LC1
        mov     edi, OFFSET FLAT:.LC3
        call    __assert_fail
```

Goes straight to `__assert_fail`, without passing Go and collecting $200.

What does [Clang do](https://godbolt.org/z/aP5qEW)? Same thing.

Again, if the compilers can clearly see that calling `f` produces a guaranteed
assertion failure, why don't they tell us?

Because we are put here on this planet to suffer, that is why.

(Also because `assert` is a macro that the compilers do not even see, and they
don't know that `__assert_fail` is the assertion failure handler, and contracts,
which would have allowed us to write `[[assert: i+j < v.size()]]` , were removed
from C++20 as being too useful, but those are just random second-order
manifestations of the cosmic need for suffering.)
