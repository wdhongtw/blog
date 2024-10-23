---
title: There is No Unit Type in C++
date: 2024-10-23 08:15:00
categories: [notes]
tags: [cpp, typing]
---

There is NO unit type in C++. (Not in core language spec, at least.)

## A Story

Assuming we are define a abstract interface for storage to get name / set name for some user.

```cpp
class Storage {
public:
    virtual ~Storage() = default;

    virtual std::string GetName(int id) const = 0;
    virtual void SetName(int id, std::string name) const = 0;
};
```

Simple and straightforward.

But it's intended to be a storage accessed through network, so any operation on it is
inherently going to fail at some time.
Also, We are 2024 now, loving FP, preferring expression over statement, monadic operation
being so cool.
Thus we decide to wrap all return type in `std::optional` to indicate these actions may fail.

```cpp
class Storage {
public:
    virtual ~Storage() = default;

    virtual std::optional<std::string> GetName(int id) const = 0;
    virtual std::optional<void> SetName(int id, std::string name) const = 0;
};
```

Looks good! But now it fails to be compiled.

```
...\include\optional(100,26): error C2182: '_Value': this use of 'void' is not valid
```

Well. template stuff.

## What Happened?

The problem is that `void` is an incomplete type in C/C++, and always to be treat specially
when we are trying to use them.

By incomplete in C/C++, we mean a type that the size of which is not (yet) known.

For example, if we forward declare a struct type, and later define it's member. The struct type
is incomplete before the definition.

```cpp
struct Item;

Item item; // <- invalid usage, since that the size of Item is unknown yet.

struct Item {
    int price;
};

Item item; // <- valid usage here.
```

And `void` is a type that is impossible to be complete by specification.

But we can have a function that return `void`?
Well, we return *nothing*

```cpp
void foo() { }

void foo() { return; } // Or explicit return, both equivalent.
```

BTW, C before C23 prefer putting a `void` in parameter list to indicate that a function takes
*nothing*, e.g. `int bar(void)`, but is's kinda broken design here.

Since that we can not evaluate `bar(foo())`. There is no such thing that is a `void` and exists.

```cpp
void foo() { }

int bar(void) { return 0; }

int main() {
    return bar(foo()); // <- invalid expression here.
}
```

So back to our problem here `std::optional<void>`

Conceptually, `std::optional<T>` is just a some T with additional information of *value-existence*.

```cpp
template <typename T>
struct MyOptional {
    T value;
    bool hasValue;
};
```

Because that there is impossible to have a member `void value`,
`std::optional<void>` is not going to be a valid type at first place.

(Well, we can make a specialization for `void`, but that's another story.)

## So, How can We Fix?

The problem here is that there is no a valid value for `void` in C/C++.
At some level, program can be though of a bunch of expressions. An running a program
is just the evaluation of these expressions. (Also the side effects, for real products / services)

The atom of expression is value. If there is a concept that's not possible to be express
as a value, we are kicking ourselves.

Take Python for example, if we have a function that return nothing, then the function actually
**returns** `None` when it exits.

```python
def foo():
    pass

assert(foo() is None) # check pass here
```

```python
def foo(arg: None) -> None:
    pass

def bar(arg: None) -> None:
    pass

bar(foo(None)) # well.. if we really want to chain them together
```

So *nothing* itself is a thing, in Python, we call it `None`.
Every expression now is evaluated to some value that exists, the the type system build up that
is complete at any case.

The concept of *nothing* itself here is call **unit type** in type theory.
It's a type that has one and only one value.
In Python, the value is `None`, in JavaScript it's `null`, in Golang ... maybe `struct{}{}` is a
good choice, although not standardized by the language.

## Unit Type in C++

Now is the time for C++. As we already see, `void` is not a good choice for unit type because we
can not have a value for it. Are there other choices here?

Just define a empty struct and use it probably not a good choice, since that
now our custom unit type is not compatible with unit type from other code in the product code base.

How about `nullptr`, that's the only one value for `std::nullptr_t`.
(So the type is `std::optional<std::nullptr_t>`).
It's a feasible choice, but looks weird since that pointer implies indirect access semantic,
but it's not the case when using with `std::optional<T>` here.

How about using `std::nullopt_t`? It's also a unit type but it's now more confusing.
What's does it mean by `std::optional<std::nullopt_t>`? A optional with empty option?
There is a static assert in `std::optional<T>` template that forbid this usage directly,
probably because it's too confusing.

Maybe `std::tuple<>`? A tuple with zero element, so it have only one value, the empty tuple.
That seems to be a good choice because the canonical unit type in Haskell is `()` the empty tuple.
So it looks natural for people came from Haskell.
But personally I don't like this either since that now the type has nested angle bracket
as `std::optional<std::tuple<>>`.

There is a type called `std::monostate`, arrived at the same time as `std::optional` in C++17.
This candidate do not have additional implication by it's type or it's name.
It's *monostate*! Just a little wordy.

`std::monostate` is originally designed to solve the problem for a `std::variant<...>` to be
default initialized with any value. But it's name and it's characteristic are all fit our
requirement here. Thus a good choice for wrapping a function that may fail but
return nothing.

Now the interface looks like

```cpp
class Storage {
public:
    virtual ~Storage() = default;

    virtual std::optional<std::string> GetName(int id) const = 0;
    virtual std::optional<std::monostate> SetName(int id, std::string name) const = 0;
};
```

Hmm... `std::optional<std::monostate>`, which takes 29 characters.
C++ is not easy. Just like we use `std::shared_ptr<T>` all over the places.

Maybe the C++ Standards Committee should specialize `std::optional<void>`,
just like `std::expected<void>` in C++23.

---

Wish someday `void` can be a REAL unit type in C/C++. :D
