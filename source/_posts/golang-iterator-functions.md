---
title: Golang 1.23 Iterator Functions
date: 2024-08-22 17:05:34
categories: [notes]
tags: [golang, functional]
---

For a long long long time, Golang have no standard way to represent a iterable sequence.

C++ has range adaptor and iterator (although not strictly typed, only by concept), Python has
iterable/iterator by `__iter__`/`__next__`, JavaScript has standardized `for-of` and
`Symbol.iterator` since ES6.

Now it's time for Golang. Starting from Golang 1.23 Aug., we have **iterator functions**.

## How It Works.

Sample code explains faster.

```golang
func Iota() func(yield func(idx int) bool) {
	return func(yield func(idx int) bool) {
		idx := 0
		for {
			if !yield(idx) {
				return
			}
			idx += 1
		}
	}
}

func main() {
	for idx := range Iota() {
		if idx == 3 {
			break
		}
		fmt.Println(idx) // print 0 1 2
	}
}
```

According to [Go 1.23 Release Notes](https://go.dev/doc/go1.23)
Now the `range` keyword accept three kinds of functions, for which takes another `yield` function
that yield zero/one/two values.

```golang
func(func() bool)
func(func(K) bool)
func(func(K, V) bool)
```

The loop control variable and the body of the for-loop is translated into the `yield` function by
language definition. So we can still write imperative-style loop structure even though we are
actually doing some functional-style function composition here.

## Why Do We Need This?

Standardize the iterable/iterator interface is a important pre-condition for lazy evaluation.
For example, how should we do when we need to iterates through all non-negative integer,
and doing some map/filter/reduce on them?  It waste space to allocate a list for all these
integers (if possible).

Someone may say "we already have channel types".
Well, but that requires a separate coroutine instance. We probably don't want such heavy
cost every time we are doing some iterate operations.

Also a separate coroutine means additional synchronization and lifecycle control.
For example, how can we terminate the `Count` coroutine when we need early break in loop?

```golang
func Count(start int) chan int {
	output := make(chan int)
	go func() {
		idx := start
		for {
			output <- idx
			idx += 1
		}
	}()
	return output
}

func main() {
	for idx := range Count(0) {
		if idx == 10 {
			break
		}
		fmt.Println("Loop: ", idx)
	}
}
```

We need some mechanism like context object or another channel right?
That's a burden for such easy task here.

On the other hand, **iterator functions** are just ordinary function that accept another function
to yield/output the iterated values, so it's much lightweight than a separate coroutine.
We want fast program, right? :D

## The Stop Design

For languages like Python and JavaScript, the iterator function (or generator in Python terms)
is paused and the control is transfer back to the function that iterates the values.
When `break`/`return` happens and no more value are required, the iterator function just
got collected by the runtime since that there are no more references to the function object.

But how do we early break the iteration process, if the control is transfer
into the iterator function? Let's look at the function signature again.
(Take one value iterator function for example).

```golang
func(yield func(idx int) bool)
```

The `yield` function returns a `bool` to indicate that whether the loop body does reach the
end, or encounter a `break` statement. So in normal case, we continue to next possible value
after `yield` return, but if we got `false` from `yield`, our iterator function
can return immediately. 

## Ecosystem around Iterator

The beauty of iterator only appears if the ecosystem, or we say, the common operations around
iterator are already implemented in standard library. That means:

- Conversion from and to standard container types, like `slice` `map` and `chan`
- Operations and compositions of iterators, e.g. `map`/`filter`/`reduce`/`chain`/`take` ...

In Python, there are generator expressions, which evolves implicit `map/filter`.
`reduce` is a function at global scope, also there are many useful functions in `itertools` package,
e.g. `pairwise`, `batched`, `chain`. Most builtin container types takes iterable as first argument
in it's constructor.

In Golang, the first part is mostly done along the release of Golang 1.23.
For example, to convert slice from and to iterator, we can use `slices.Collect` and `slices.Values`.

For second part, there is a plan to add `x/exp/xiter` package under `golang.org` namespace.
There should be at least `Concat`, `Map`, `Filter`, `Reduce`, `Zip` ... once it's released.
But unfortunately it's not compete yet.

See: [iter: new package for iterators · Issue #61897 · golang/go](https://github.com/golang/go/issues/61897)

Also I create a toy package [github.com/wdhongtw/mice/flow](https://pkg.go.dev/github.com/wdhongtw/mice/flow)
to address some important building wheel around iterators

- `Empty`/`Pack` return a iterator for zero/one value
- `Any`/`All` short-circuit lazy evaluation of a predicate on a sequence of values
- `Forward`/`Backward` absorb input and iterate in reversed order.

For example, if we want to define a iterator function for a binary tree in recursive manner,
we can use `Empty` and `Pack` together with `Chain` to implement this easily.

```golang
type Node struct {
    val   int
    left  *Node
    right *Node
}

func Traverse(node *Node) iter.Seq[int] {
    // Empty is useful as base case during recursive generator chaining.
    if node == nil {
        return Empty[int]()
    }
    // Pack is useful to promote a single value into a iterable for chaining.
    return Chain(
        Traverse(node.left),
        Pack(node.val),
        Traverse(node.right),
    )
}
```

Looks cool, doesn't it? :D
