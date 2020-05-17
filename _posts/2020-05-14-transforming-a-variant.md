---
layout: post
title: Transforming a Variant
hidden: true
---

Q: How do I transform a `variant<X, Y, Z>` `V` into `variant<vector<X>, vector<Y>, vector<Z>>`?

A: [Use](https://godbolt.org/z/B52eaF) `mp_transform<std::vector, V>`.

Q: And what if I want to use my own allocator `A`, that is, want `variant<vector<X, A>, vector<Y, A>, vector<Z, A>>`?

A: [Use](https://godbolt.org/z/Ag58uX) `mp_transform_q<mp_bind_back<std::vector, A>, V>`.

Q: But my allocator is a template. How about `variant<vector<X, A<X>>, vector<Y, A<Y>>, vector<Z, A<Z>>>`?

A: [Use](https://godbolt.org/z/YXXMbL) `mp_transform<std::vector, V, mp_transform<A, V>>`.

Q: Does [Mp11](https://boost.org/libs/mp11) answer every question with a cryptic one liner?

A: Yes.
