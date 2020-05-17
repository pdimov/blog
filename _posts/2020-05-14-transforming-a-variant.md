---
layout: post
title: Transforming a Variant
hidden: true
---

Q: How do I transform a `variant<X, Y, Z>` `V` into `variant<vector<X>, vector<Y>, vector<Z>>`?

A: [Use](https://godbolt.org/z/B52eaF) `mp_transform<std::vector, V>`.
