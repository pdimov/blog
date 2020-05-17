---
layout: post
title: It's a Small World
hidden: true
---

I came across [this article](https://www.viva64.com/en/b/0733/),
in which the author (a fan of Rust by the look of it) takes apart
outrageous claims by [Antony Polukhin](http://apolukhin.github.io/en/)
that Rust is not highly superior to C++.

Antony is a
[Boost author and maintainer](http://apolukhin.github.io/en/developer.html),
and active in the C++ standard committee, so I obviously decided to watch
[his video](https://www.youtube.com/watch?v=fT3OALUyuhs) (in Russian).

At one point he said something to the effect of "rigged language
benchmarks often compare their built-in regex performance with
`std::regex`, and this is unfair, because if your regular expression
is known and fixed at compile time, you should use Hana's CTRE."

Hana is [Hana Dusíková](https://twitter.com/hankadusikova), and CTRE
is [her compile-time regular expression library](https://github.com/hanickadot/compile-time-regular-expressions).

So I decided to look at CTRE. It's a pretty interesting library, and
the most interesting part of it is the resuling assembly code. The
regular expression `a*b`, for instance, is matched via
```
.LBB0_25:
        lea     rdx, [rcx + rax]
.LBB0_26:
        xor     esi, esi
.LBB0_27:
        movzx   ebx, byte ptr [rcx + rsi]
        cmp     bl, 97
        jne     .LBB0_28
        add     rsi, 1
        cmp     rax, rsi
        jne     .LBB0_27
        jmp     .LBB0_31
.LBB0_28:
        cmp     bl, 98
        je      .LBB0_29
.LBB0_31:
        add     rcx, 1
        add     rax, -1
        cmp     rcx, rdx
        jne     .LBB0_26
.LBB0_32:
        xor     ebp, ebp
        jmp     .LBB0_33
.LBB0_29:
        mov     bpl, 1
.LBB0_33:
```
which may not be the exact same code you or I would have written,
but it's fairly readable and does the same basic loop and
comparisons as a hand-written matcher would.

This is incidentally many times faster than `std::regex` on the
same task. (And still faster, but significantly fewer times, than
[Boost.Regex](https://boost.org/libs/regex), on which the
specification of `std::regex` is based.)

Regular expressions turned out to be a pretty interesting area. I
was soon reading [Russ Cox](https://swtch.com/~rsc/)'s series of
articles ([1](https://swtch.com/~rsc/regexp/regexp1.html),
[2](https://swtch.com/~rsc/regexp/regexp2.html),
[3](https://swtch.com/~rsc/regexp/regexp3.html),
[4](https://swtch.com/~rsc/regexp/regexp4.html).) At part 3, it
turned out that he's the author of Google's
[RE2](https://github.com/google/re2) regex library.

Another browser tab had in it open [an article](https://blog.burntsushi.net/ripgrep/)
describing how and why a search tool called `ripgrep` and written in Rust
was faster than other competing `grep` tools, and it, too, proved engaging.
I learned that Rust's regex library uses a
[SIMD algorithm](https://github.com/rust-lang-nursery/regex/blob/3de8c44f5357d5b582a80b7282480e38e8b7d50d/src/simd_accel/teddy128.rs)
called "Teddy", invented by [Geoffrey Langdale](https://twitter.com/geofflangdale) for the
[Hyperscan](https://www.hyperscan.io/) regex library. Hyperscan was
apparently a startup that was specializing in scanning streaming
data (such as network packets) against many regular expressions,
and doing so very quickly thanks to insane SIMD optimizations. (It's
been since acquired by Intel, and the library open-sourced.)

I started reading posts by Geoff Langdale, first on the
[Hyperscan blog](https://www.hyperscan.io/2015/10/20/match-regular-expressions/),
then on [his own](https://branchfree.org/). Clever SIMD tricks applied
to regex scanning were certainly present, but also was a mention
of [simdjson](https://simdjson.org/). So that was where all that SIMD
in `simdjson` came from!

I had heard of this library because of my involvement with
[proposed-for-Boost.JSON](https://github.com/CPPAlliance/json) by Vinnie Falco,
a new JSON library that also takes performance seriously (among other things.)

It's a small world.
