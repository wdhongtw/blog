---
title: Python 的 type 不是 type？
date: 2024-06-05 17:38:00
categories: [notes]
tags: [python, generic, typing]
---

## Intro

近期因為一些原因，想自己寫一個 Python 用的 DI library。寫是寫完了，不含 test 基本上不到 50 行，
也 release 到 [luckydep · PyPI](https://pypi.org/project/luckydep/) 了。
不過在寫的過程中發現了一些問題。

與 Golang 不同。 在 Python 中，DI container 拿取特定 instance 的介面 (invoke/bind/get 等，下稱 invoke)
需要明確傳遞想拿取的 instance 的 type。

```golang
func Invoke[T any](c *Container) T {
    var _ T // can construct T event if T is a interface
}

// although we need to specify the type parameter, the type parameter
// is not passed during runtime
var instance = Invoke[SomeType](c)
```

```Python
class Container:
    def invoke(t): # search and build a instance of type t

c = Container()
instance = c.invoke(SomeType) # need to pass type as a parameter
```

其中的根本差異是，Golang 類型的 static type language，generic function 會真的根據不同型別，產生對應的
function 出來，這些 function 的 byte code/machine code 自然知道當下在處理的型別。
而 Python 這類語言，靠 static type checker 建立 generic function，實際上到 runtime 時還是只有一個
function，自然會需要傳遞 type 給 invoke 介面。

自從 Python 3.6 開始我們有 type hint，所以我們可以 annotate function/method 來幫助
IDE/type checker 來推論正確的型別。

```Python
class Container:
    def invoke(t: type[T]) -> T: # search and build a instance of type t

c = Container()
instance: SomeType = c.invoke(SomeType) # ok, we can infer instance is SomeType
```

這邊 `type[T]` (or `typing.Type[T]`, the old way) 用來表示我們正在用 `t` 來傳遞傳遞某個 type `T`，
而非 type 為 `T` 的某個 instance。

From `typing` document:

> A variable annotated with `C` may accept a value of type `C`. In contrast, a variable annotated with
> `type[C]` (or `typing.Type[C]`) may accept values that are classes themselves

## The Problem

OK，我們有 `type[T]` 可以用。 DI library 開發者可以用型別為 `type[T]` 的 `t` 來做 indexing，
library 使用者可以享受到 static type checker 帶來的 type safety。

於是我們拿這個 library 來用在真實情境.. 沒想到一下子就碰上問題了。
當我們定義 interface type，並透過 DI container 對該 interface 拿取對應的 implementation instance 時。
因為 interface 通常是個 abstract class (or protocol)，`mypy` type checker 會報錯
([mypy: type-abstract](https://mypy.readthedocs.io/en/stable/error_code_list.html#safe-handling-of-abstract-type-object-types-type-abstract))。

```python
class SomeInterface(Protocol):
    def hello(self): ...

class SomeImplementation:
    def hello(self): ...

c.invoke(SomeInterface) # trigger mypy error [type-abstract]
```

不會吧... 這不是我們需要 DI 的最重要原因嗎？
我們定義 interface 並另外提供 implementation，來達到隔離不同 class 職責的效果。
結果當 user 要用這個 library 的時候卻卡在型別檢查...

## The History

翻閱文件，第一時間以為這是 `mypy` 的設計問題。

> Mypy always allows instantiating (calling) type objects typed as `Type[t]`

沒想到翻了 `mypy` issue [#4717 · python/mypy](https://github.com/python/mypy/issues/4717)
後，發現這是已經寫在 PEP 544 內的規格。

> Variables and parameters annotated with `Type[Proto]` accept only concrete (non-protocol) subtypes of Proto.
> The main reason for this is to allow instantiation of parameters with such type. For example:
> 
> ```python
> def fun(cls: Type[Proto]) -> int:
>     return cls().meth() # OK
> ```

`mypy` 允許 construct 一個不知道 constructor 長什麼樣子的 interface type，
所以該標示 `Type[Proto]` 的 parameter 只能傳遞 concrete type... 嗯？

繼續往下追，想不到一開始會有這個檢查，是因為 **Guido 本人** 在 2016 年開的 [#1843 · python/mypy](https://github.com/python/mypy/issues/1843)，
認為應該允許這種使用方法。

於是 `mypy` 加入了這個檢查，後來 2017 年的 PEP 544 也明確定義了這個使用規則。

## The Controversy

這個 `t: type[T]` 的設計引起很多爭議，從 [#4717 · python/mypy](https://github.com/python/mypy/issues/4717)
來看，不少人認為: 為了允許 construct `t()` 而限制只能傳遞 concrete class 會大幅限制這個 `type[T]` 的使用情境。

也有人認為這個檢查根本就不合理，因為沒有人能保證這個 protocol type 底下的 concrete class 的 constructor
到底要吃什麼東西。 即使 static type check 檢查過了，`t()` 在 runtime 噴掉一點也不奇怪。
更何況根本沒看過有人在 protocol type 上面定義 `__init__` method，這個 `t()` 一開始到底要怎麼檢查也不知道。

如果看相其他語言的開發經驗...
Golang 生態系 constructor 是 plain function，定義 interface type 時自然不會包含 constructor。
寫 C++ 的人應該也沒聽過什麼 abstract constructor，只有 destructor 會掛 `abstract` keyword。
回到 Python 自身，`mypy` 和 `pyright` 兩大工具也都允許 `__init__` 的 signature 在繼承鍊中被修改。
(see: [python/typing · Discussion #1305](https://github.com/python/typing/discussions/1305))

至於 `typing.Type` 的文件，寫得很模糊，我想有一定程度的人看到反而更容易誤會。

> `type[C]` ... may accept values that are classes themselves ...

就算捨棄掉 protocol，限制都只能用 concrete class 來定義 interface。
這個只能允許 concrete class 的規則還造成了另一個問題: 使用者該如何傳遞 function type？

```python
c.register(Callable[[int, int], int], lambda a, b: a + b) # ????
```

說好的 function as first-class citizen 呢？ 怎麼到了要傳遞型別時就不行了？

在翻閱 issue 的過程中，發現其他 DI framework 的 repo 也遇上同樣的問題 [#143 · python-injector/injector](https://github.com/python-injector/injector/issues/143)，
頓時覺得自己不孤單。

## The Future

由於 PEP 544 自從 2017 年就已經完成，`mypy` 預設執行這個檢查也行之有年，
現在再來改這個行為或許已經來不及了。

於是為了解決這個問題，2020 有人在開了新 issue [9773 · python/mypy](https://github.com/python/mypy/issues/9773)
想要定義新的 annotation `TypeForm[T]`/`TypeExpr[T]` 來達成要表達任意 type 的 type 的需求。
到目前 (2024-06)，對應的 [PEP 747 draft](https://github.com/python/peps/pull/3798) 也已經被提出了。

若一切順利，以後我們就會用 `TypeExpr[T]` 來表達這類 generic function

```python
class Container:
    def invoke(t: TypeExpr[T]) -> T: # search and build a instance of type t

class SomeType(Protocol): ...

c = Container()
instance = c.invoke(SomeType) # ok, we find a object for type SomeType for you!
operator = c.invoke(Callable[[int], bool]) # you need a (int -> bool)? no problem!
```

至於目前嘛.. library user 在使用到這類 library 的檔案加入下面這行即可。
我想要修改的範圍和造成的影響應該都還可以接受。

```python
# mypy: disable-error-code="type-abstract"
```

期許 Python typing system 完好的那天到來。

## Timeline

- 2016-07: [#1843 · python/mypy](https://github.com/python/mypy/issues/1843) Guido 提出要 instantiate 的需求
- 2017-05: [PEP 544](https://peps.python.org/pep-0544/) standardized and published
- 2018-05: [#4717 · python/mypy](https://github.com/python/mypy/issues/4717) first discussion against `type[T]` design
- 2020-04: [#143 · python-injector/injector](https://github.com/python-injector/injector/issues/143)
- 2020-10: [#9773 · python/mypy](https://github.com/python/mypy/issues/9773) propose idea of `TypeFrom[T]`
- 2024-06: [PEP 747](https://github.com/python/peps/pull/3798) draft created
