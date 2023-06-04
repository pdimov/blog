---
layout: page
title: About
---

I authored and maintain the following [Boost](https://www.boost.org/) libraries:

* [Assert](https://boost.org/libs/assert)
* [Bind](https://boost.org/libs/bind)
* [Describe](https://boost.org/libs/describe)
* [Lambda2](https://boost.org/libs/lambda2)
* [Mp11](https://boost.org/libs/mp11)
* [SmartPtr](https://boost.org/libs/smart_ptr) (with Glen Fernandes)
* [ThrowException](https://boost.org/libs/throw_exception) (with Emil Dotchevski)
* [Variant2](https://boost.org/libs/variant2)

and additionally maintain and develop

* [ContainerHash](https://boost.org/libs/container_hash)
* [Core](https://boost.org/libs/core) (with Andrey Semashev and Glen Fernandes)
* [Endian](https://boost.org/libs/endian)
* [Function](https://boost.org/libs/function)
* [System](https://boost.org/libs/system)
* [Timer](https://boost.org/libs/timer)

and contribute to

* [Unordered](https://boost.org/libs/unordered) (with Joaquin M Lopez Munhoz and Christian Mazakas)

I am responsible for the following components of the C++11 standard:

* `shared_ptr`
* `weak_ptr`
* `enable_shared_from_this`
* `bind`
* `mem_fn`
* `exception_ptr`
* `atomic_thread_fence`, `atomic_signal_fence`

I occasionally use [Boostdep](https://boost.org/tools/boostdep)
to generate the [Boost Dependency Report](https://pdimov.github.io/boostdep-report).

I wrote and maintain the
[part of the Boost installation](https://github.com/boostorg/boost_install)
that generates CMake configuration files for the Boost libraries. This enables
`find_package(Boost)` to work without relying on an up-to-date `FindBoost.cmake`.

I wrote and maintain the
[Boost CMake build infrastructure](https://github.com/boostorg/cmake).
