---
title: Easier Iteration with Generator
date: 2025-02-26 09:05:00
categories: [notes]
tags: [cpp]
---

In Python, we can make a container type iterable by implement `__iter__` method.

For example, if we want to provide all value in a range `[lo, hi)` ...

```python
class TwoTillSix:

    def __iter__(self):
        for val in range(2, 6):
            yield val

list(TwoTillSix())  # [2, 3, 4, 5]
```

With the help of this generator function, we don't need a separate iterator type.
Just keep the state during iteration directly in some local variable.

In this post, let's see how we can do this in C++, with the help of `std::generator<T>`.

A shorter version of this post can be found in Gist and Compiler Explorer.

- <https://gist.github.com/wdhongtw/aa61b9b4dd132ffeae64d9189e9ba2cc>
- <https://godbolt.org/z/K57Enfhcr>

## Coroutine and Concurrent Programming

What happens in the example above is that Python interpreter binds the execution state of
that function to a generator object. Every time we want a value from that generator,
the execution resumes until the function yields the next value (or returns).

This is doable because Python provide a builtin support for **coroutine, an interruptible/resumable
function**, so the control flow can interleave between callee and caller functions. And that's the
primitive building block of concurrent programming.

```python
values = TwoTillSix()

for val in values:
    print(val)
    if val == 3:
        break
# Print 2 and 3. The body of `TwoTillSix.__iter__` is only executed twice.
```

As we can see here, the generator makes it possible that some values are evaluated lazily,
and we can even stop the evaluation any time when we lost the interest.
All of this is possible because the support of coroutine.


## C++ Coroutine and `std::generator`

Fortunately, C++ has language level support for coroutine since C++20.
When compiler see a function using new keywords `co_await` / `co_yield` and `co_return`,
it will generate corresponding code to provide the ability of partial execution, and the storage
of state for that function when it's interrupted.

Also, starting from C++23, `std::generator` is included in standard library. As the
coroutine (promise-)type that capture the common usage of... yes, generator!!
(Actually I'm surprised that there is no any high-level coroutine type in standard library
when coroutine is accepted as a language feature when C++20 is finalized.)

```cpp
std::generator<int> fibonacci() {
    int a = 1, b = 0;
    while (true) {
        co_yield a;
        b = std::exchange(a, a + b);
    }
}

int main() {
    for (int num : fibonacci()) {
        if (num > 10)
            break;
        std::println("{}", num); // prints 1 1 2 3 5 8
    }
    return 0;
}
```

When `fibonacci` is called here, an interruptible function is created, and wrapped in the
`std::generator` return value. Every time we take next value out in the for-loop, the function
resumes and yield next value (and is interrupted again).

So now we have the ability to interleave control flow, just like the example in Python earlier!!
But... that's a free function, usually what we encounter is a container class.
A class must has `begin`/`end` method, and corresponding increment/dereference operator on
the returned iterator type, in order to be iterable.
(It's modeled as named requirements *Container* in standard.)
If a class need another method call to be iterated on, it just looks... not like C++.

```cpp
struct Factorial { // a virtual container for factorial series
    std::generator<int> generate() {
        int product = 1;
        int val = 1;
        while (true) {
            co_yield product;
            product *= val++;
        }
    }
};

int main() {
    Factorial fac {};
    for (int num : fac.generate()) { // well.. can we iterate on fac directly?
        if (num > 10)
            break;
        std::println("{}", num);
    }
}
```

So... can we keep the implementation simple (using `std::generator` coroutine type),
while keeping the container being iterated in usual way?

## CRTP: Curiously Recurring Template Pattern

Now our goal become clear:

> How can we make a class be used as before, if the class only provides a method
> that return a `std::generator`?"

That's the place where we adopt CRTP to solve problem.

```cpp
/// @brief HasGenerator identifies the customization point of CRTP base AutoContainer.
/// @tparam C the child class type.
/// @tparam T the value type.
template <typename C, typename T>
concept HasGenerator = requires(C c) {
    { c.generate() } -> std::same_as<std::generator<T>>;
};

/// @brief A CRTP base type that allow easy iteration implementation.
/// @tparam T the value type of element being iterated. Not child class type.
/// @details Child class must provide std::generator<T> generate() member function.
template <std::semiregular T>
struct AutoContainer {

    class Iter {
        std::generator<T> gen_;
        std::ranges::iterator_t<std::generator<T>> iter_;

    public:
        Iter(std::generator<T>&& gen)
            : gen_ { std::move(gen) } // ensure generator being alive
            , iter_ { gen_.begin() } { }

        bool operator==(const std::default_sentinel_t&) const { return iter_ == gen_.end(); }
        Iter& operator++() { return ++iter_, *this; }
        T operator*() const { return *iter_; }
    };

    // deducing-this so that we can do CRTP.
    // Also use concept to give better error message when doing mistake.
    Iter begin(this HasGenerator<T> auto&& self) {
        return Iter(self.generate());
    }

    // deducing-this not required, just be symmetric with begin.
    std::default_sentinel_t end(this auto&& self) {
        return std::default_sentinel;
    };
};
```

`AutoContainer` can inherited by any class that want the all the implementation for iteration.
The base class here provides an iterator type to capture the generator, and forwards the
increment/dereference operation to it. With this iterator type, the remain
effort is on the `begin` method, just construct the iterator from `self.generate()`.
Here `self` is typed by deducing-this so `generate()` can be provided later by child class.
And that's the place where the base class connects to it's child.

Also at the `begin` method, we add concept constrain `HasGenerator`.
This way we can avoid error waterfall when using templated library.
And the interesting thing here is: "The thing to be templated so that we achieve CRTP,
is the templated `begin` method, not the `AutoContainer` class itself.
If there are two child class both  inherit from `AuthContainer<int>`,
only one `AuthContainer` type is instantiated, but two different `begin` are instantiated.

(See also: [Curiously Recurring Template Pattern - cppreference.com](https://en.cppreference.com/w/cpp/language/crtp))

```cpp
int main() {

    /// @brief Factorial sequence generator.
    struct Factorial : public AutoContainer<int> {

        // Implement `generate` so this class can be iterated like a container.
        std::generator<int> generate() const {
            // The mutable state is the local variable of this coroutine,
            // so the method receiver can be const.
            int product = 1;
            int val = 1;
            while (true) {
                co_yield product;
                product *= val++;
            }
        }
    };

    const Factorial fac {};
    for (int value : fac) { // noticed that we don't call generate() here
        if (value > 10)
            break;
        std::println("{}", value); // prints 1, 1, 2, 6
    }
}
```

Now we can use the `AutoContainer` by inherit it in any child class.
In the example above, `Factorial` inherit from `AutoContainer<int>` (since it generates integers).
All we need to do is implement the `generate` method. Being a coroutine, the logic is clean.

Now everything works and **looks nice**!! :D

To make this class recognized as a `std::ranges::input_range`, that is, to make this class
works with `<ranges>` library, the base `AutoContainer` actually need more stuffs, like
`value_type`/`difference_type` on it's iterator, support of post-increment operator, and so on...
But that's probably too much in a tutorial, so let's stop here.

## More Standard Library Support, Maybe?

We describe the minimal concept about coroutine, one important application of coroutine as
Generator (`std::generator`), and how to use it in a beautiful way.
But there is another important application of coroutine: Promise (with Event Loop).

JS developers already get used to the `Promise` type (or maybe the `async`/`await` keyword).
Python also support this kind of usage since Python 3.5.
But unfortunately, there is still no such thing in C++ standard library, even if the coroutine
is standardized in C++20.

The good news is that there is already a proposal P2300 and a reference implementation `NVIDIA/stdexec`.
(They don't use the term Promise, probably because `promise_type` already have other meaning
in the context of C++ coroutine)

Hope we can see more usage about these in the future.
