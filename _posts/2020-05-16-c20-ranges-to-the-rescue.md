---
layout: post
title: C++20 Ranges to the Rescue
hidden: true
---

Vittorio Romeo [documents](https://vittorioromeo.info/index/blog/gamedev_modern_cpp_thoughts.html)
his collision with gamedev Twitter, and in the process shows the following
snippet:

```
template <typename... Images>
TextureAtlas stitchImages(const Images&... images)
{
    const auto width = (images.width + ...);
    const auto height = std::max({images.height...});

    // ...
```

His point, and he certainly has one, is that pack expansion with fold
expressions leads to much cleaner code than the equivalent that would
operate on a runtime container of images:

```
TextureAtlas stitchImages(const std::vector<Image>& images)
{
    std::size_t width = 0;
    for(const auto& img : images)
    {
        width += img.width;
    }

    std::size_t maxHeight = 0;
    for(const auto& img : images)
    {
        maxHeight = std::max(maxHeight, img.height);
    }

    // ...
```

He then preempts the obvious "use the standard algorithms" retort with

```
TextureAtlas stitchImages(const std::vector<Image>& images)
{
    const auto width = std::accumulate(images.begin(), images.end(), 0
        [](const std::size_t acc, const Image& img)
        {
            return acc + img.width;
        });

    const auto height = std::max_element(images.begin(), images.end(),
        [](const Image& imgA, const Image& imgB)
        {
            return imgA.height < imgB.height;
        })->height;

    // ...
```

A-ha, we say to ourselves. He has missed our non-obvious retort, which,
in keeping with his "modern C++" theme, is "just use the C++20 range
algorithms, which take _projections_."

```
TextureAtlas stitchImages(const std::vector<Image>& images)
{
    const auto width = std::ranges::accumulate( images, 0, {},
        &Image::width );

    const auto height = std::ranges::max( images, {},
        &Image::height );

    // ...
```

Modern C++ wins again!

Or maybe not. The above certainly feels like it ought to work, but it has
two problems. First, when I wrote the above, I certainly expected
`auto const height` to be the maximum image height, but it isn't. Instead,
as written, it actually holds _the image_ with the maximum height, because
`max` returns a reference to the actual element, not the maximum value
after the projection.

That's easily fixed:
```
    const auto height = std::ranges::max( images, {},
        &Image::height ).height;
```
but allows me to offer some entirely unsolicited and unwarranted advice. You
know those people who tell you to "almost always use `auto`"? Don't listen to
them.

The second problem with the above snippet is that... `std::ranges::accumulate`
doesn't exist.

The reason it doesn't exist is that `std::accumulate` is defined in `<numeric>`.
While the algorithms in the appropriately named `<algorithm>` have `ranges::`
versions in C++20, those in `<numeric>` do not.

They do not because C++20 got at once two major features, Ranges and Concepts,
which meant that the standard library algorithms got to be modernized twice,
once to acquire a range-aware version, placed in `std::ranges::`, and once to
be properly constrained via concepts. This, in turn, meant that those concepts
needed to be designed.

There wasn't enough time to properly design the numeric concepts required for
the algorithms in `<numeric>` to be brought up to date, although [work on that
is underway](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2019/p1813r0.pdf).

None of which, sadly, helps the C++20 user. (Don't tell any of this to any game
developers.)
