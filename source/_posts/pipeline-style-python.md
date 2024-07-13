---
title: Pipeline Style Map-Reduce in Python
date: 2024-07-13 11:50:00
categories: [tips]
tags: [python, development]
---

Since C++20, C++ provide a new style of data processing, and the ability of
lazy evaluation by chaining the iterator to another iterator.

```cpp
#include <ranges>
#include <vector>

int main() {
    std::vector<int> input = {0, 1, 2, 3, 4, 5, 6, 7, 8, 9, 10};
    auto output = input | std::views::filter([](const int n) {return n % 3 == 0;})
                        | std::views::transform([](const int n) {return n * n;});
    // now output is [0, 9, 36, 81], conceptually
}
```

Can we use this style in Python? Yes! :D

## Evaluation of Operators

In Python, all expression evolves a arithmetic operator, e.g. `a + b`, is evaluate by follow rule

- If the (forward) special method, e.g. `__add__` exists on left operand
  - It's invoked on left operand, e.g. `a.__add__(b)`
  - If the invocation return some meaningful value other than `NotImplemented`, done!
- If the (forward) special method does not exist, or the invocation returns `NotImplemented`, then
- If the **reverse special method**, e.g. `__radd__` exists on right operand
  - It's invoked on the right operator, e.g. `b.__radd__(a)`
  - If the invocation return some meaningful value other than `NotImplemented`, done!
- Otherwise, `TypeError` is raised

So it seems possible here... Let's make a quick experiment

```python
class Adder:
    def __init__(self, rhs: int) -> None:
        self._rhs = rhs

    def __ror__(self, lhs: int) -> int:
        return lhs + self._rhs


assert 5 == 2 | Adder(3)  # Is 5 equals to 2 + 3 ? Yes!!
```

This works because the `|` operator of integer `2` check the type of `Adder(3)` and found that
is not something it recognized, so it returns `NotImplemented` and our reverse magic method goes.

In C++, the `|` operator is overloaded(?) on range adaptors to accept ranges.
So maybe we can make something similar, having some object implements `__ror__` that accept
an iterable and return another value (probably a iterator).

## Pipe-able Higher Order Function

So back to our motivation, Python already have something like `filter` `map` `reduce`,
and also the powerful generator expression to filter and/or map without explicit function call.

```python
values = filter(lambda v: v % 2 == 0, range(10))
```

```python
values = (v for v in range(10) if v % 2 == 0)
```

But it's just hard to chain multiple operations together while preserving readability.

So let's make a filter object that support `|` operator

```python
class Filter:
    def __init__(self, predicate: Callable[[int], bool]) -> None:
        self._predicate = predicate

    def __ror__(self, values: Iterable[int]) -> Iterator[int]:
        for value in values:
            if self._predicate(value):
                yield value


selected = range(10) | Filter(lambda val: val % 2 == 0)
assert [0, 2, 4, 6, 8] == list(selected)
```

How about map?

```python
class Mapper:
    def __init__(self, transform: Callable[[int], int]) -> None:
        self._transform = transform

    def __ror__(self, values: Iterable[int]) -> Iterator[int]:
        for value in values:
            yield self._transform(value)


processed = range(3) | Mapper(lambda val: val * 2)
assert [0, 2, 4] == list(processed)
```

Works well, we are great again!!

It just take some time for we to write the class representation for `filter`, `map`, `reduce`,
`take`, `any` ... and any higher function you may think useful.

Wait, it looks so tedious. Python should be a powerful language, isn't it?

## Piper and Decorators

The function capturing and `__ror__` implementation can be so annoying for all high order function.
If we can make sure `__ror__` only take left operand, and return the return value of the captured
function, than we can extract a common `Piper` class. We just need another function to produce a
function that already capture the required logic.


```python
class Piper(Generic[_T, _U]):
    def __init__(self, func: Callable[[_T], _U]) -> None:
        self._func = func

    def __ror__(self, lhs: _T) -> _U:
        return self._func(lhs)


def filter_wrapped(predicate: Callable[[_T], bool]):
    def apply(items: Iterable[_T]) -> Iterator[_T]:
        for item in items:
            if predicate(item):
                yield item

    return Piper(apply)


selected = range(10) | filter_wrapped(lambda val: val % 2 == 0)
assert [0, 2, 4, 6, 8] == list(selected)
```

Now it looks a little nicer ... but we still need to implement all wrapper functions for all
kinds of operations?

Again, the only difference between these wrapped functions is the logic inside apply function,
so we can extract this part again, with a decorator!! :D

```python
def on(func: Callable[Concatenate[_T, _P], _R]) -> Callable[_P, Piper[_T, _R]]:
    def wrapped(*args: _P.args, **kwargs: _P.kwargs) -> Piper[_T, _R]:
        def apply(head: _T) -> _R:
            return func(head, *args, **kwargs)

        return Piper(apply)

    return wrapped


@on
def filter(items: Iterable[_T], predicate: Callable[[_T], bool]) -> Iterator[_T]:
    for item in items:
        if predicate(item):
            yield item


selected = range(10) | filter(lambda val: val % 2 == 0)
assert [0, 2, 4, 6, 8] == list(selected)
```

The `on` decorator accept some function `func`, and return a function that first take the
tail arguments of `func` and return a function that accept head argument of `func` through
pipe operator.

So now we can express our thoughts in our codebase using pipeline style code,
just with one helper class and one helper decorator! :D

```python
values = range(10)
result = (
    values
    | filter(lambda val: val % 2 == 0)
    | map(str)
    | on(lambda chunks: "".join(chunks))() # create pipe-able object on the fly
)
assert result == "02468"
```

or

```python
for val in range(10) | filter(lambda val: val % 2 == 0):
    print(val)
```

## Appendix

Complete type-safe code here

```python
"""
pipe is a module that make it easy to write higher-order pipeline function
"""

from collections.abc import Callable, Iterable, Iterator
from typing import Generic, TypeVar, ParamSpec, Concatenate

_R = TypeVar("_R")
_T = TypeVar("_T")
_P = ParamSpec("_P")


class Piper(Generic[_T, _R]):
    """
    Piper[T, R] is a function that accept T and return R

    call the piper with "value_t | piper_t_r"
    """

    def __init__(self, func: Callable[[_T], _R]) -> None:
        self._func = func

    def __ror__(self, items: _T) -> _R:
        return self._func(items)


def on(func: Callable[Concatenate[_T, _P], _R]) -> Callable[_P, Piper[_T, _R]]:
    """
    "on" decorates a func into pipe-style function.

    The result function first takes the arguments, excluding first,
    and returns an object that takes the first argument through "|" operator.
    """

    def wrapped(*args: _P.args, **kwargs: _P.kwargs) -> Piper[_T, _R]:
        def apply(head: _T) -> _R:
            return func(head, *args, **kwargs)

        return Piper(apply)

    return wrapped
```
