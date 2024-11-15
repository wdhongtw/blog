---
title: The True Placeholder Symbol in C++
date: 2024-11-26 06:25:00
categories: [news]
tags: [cpp]
---

In many programming language, the common way to indicate that the symbol is not important,
is to use `_` for the symbol.

It was just a convention in C++, but it will become a language feature start from C++26.

Well... what's the difference?

---

We use `_` when there is some declaration but we do not care the name / have no good name for
the variable.

For example, a common trick to preserve the life time of RAII lock is

```cpp
void doJob() {
    static std::mutex mutex;
    std::lock_guard _(mutex); // give it a name so it won't unlock immediately
    // some jobs ...
}
```

Or in structure-binding statement.

```cpp
template <std::regular T>
std::tuple<T, bool> someJob() {
    return { {}, true };
}

void foo() {
    auto [_, done] = someJob<int>();
}
```

The problem is... in C++, this style is just a convention, `_` is still a regular variable.
So if we want to ignore two value with different type, it does not work since the type mismatch.

```cpp
void foo() {
    auto [_1, done1] = someJob<int>();
    auto [_2, done2] = someJob<std::string>();
    // we need to separate _2 from _1
}
```

That's frustrating, especially for people with experience of pattern-matching expression in
other languages.

So in C++26 (proposed by P2169), now we can new way to interpret the semantic of `_`.

The rule is simple.

> If there is only one declaration of `_` in some scope, everything is same as before.

A we can reference it later if we wan't, although it's probably a bad smell to use `_` in this case.

> If there are more declarations of `_`, they all refer to different objects respectively.

In this case, they can only be assigned to. Try to use them is a compiling error.

And we can finally write something that looks more natural.

```cpp
void foo() {
    auto [_, done1] = someJob<int>();
    auto [_, done2] = someJob<std::string>();
}
```

---

Golang has this feature from the beginning, is called *blank identifier*.
For Python, although being a dynamic-type language, there is no problem to do use `_` for different
type value. `_` is defined as a wildcard when pattern-matching is introduced to Python (PEP 634).

It's happy to see this came to C++ now. :D
