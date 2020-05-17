---
layout: post
title: Transforming a Variant
hidden: true
---

Q: How do I transform a `variant<X, Y, Z>` `V` into `variant<vector<X>, vector<Y>, vector<Z>>`?

A: [Use](https://godbolt.org/z/B52eaF) `mp_transform<std::vector, V>`.

Q: And what if I want to use my own allocator `A`, that is, want `variant<vector<X, A>, vector<Y, A>, vector<Z, A>>`?

A: [Use](https://godbolt.org/z/Ag58uX) `mp_transform_q<mp_bind_back<std::vector, A>, V>`.
