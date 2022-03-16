---
title: Golang 1.18 Generics 終於來臨
date: 2022-03-16 22:54:00
categories: [notes]
tags: [golang, generic]

---

今天 Golang 1.18 終於正式釋出啦！
我們終於有辦法在 Golang 裡做 generic programming 啦！

Golang 是一個近 10 年快速串紅的程式語言，但在很多面向其實還是非常土炮。
得靠後續社群不斷的討論與貢獻才達到一個比較完善的水準。 像是..

- `context` package in Golang 1.7: 解決 long job cancelling 的問題
- `errors` package in Golang 1.13: 滿足其他語言常見的 error 嵌套需求和提供統一的判斷方式
- Generic support in Golang 1.18: 提供開發者實作各種型無關演算法的機會

一直以來，在 Golang 的標準函式庫中，碰到類型無關的抽象問題時，最後給出來的解法大概就兩種

1. 所有 input / output 參數都定義成 `interface{}`，大家一起把型別檢查往 run-time 丟
2. 同一個 class / function 對於常用的 data type 通通實作一遍

前者最典型的大概就是 `sync` 這個函示庫，後者.. 大家應該都看過那個慘不忍睹的 `sort`..

不過這些都是過去式了，從今天開始，大家都可以寫自己想要的 generic code / library / framework。 :D

## Usage

基本語法很簡單，只要在想做成 generic 的 function / struct 後面多加一個 `[T TypeName]` 即可，
`TypeName` 是用原本就有的 `interface{...}` 語法來表示，可以自己描述這個 generic function / struct
支援的型態必須滿足什麼樣的介面。

以 Python style 的 sort by key 當例子。
我們可以定義一個 generic 的 sort function，並且明定送進來的 list 內的每個元素需要支援一個 `Key` 函示，
作為排序時的根據。

範例 code 如下

```golang
package main

type Keyable interface {
	Key() int
}

func Sort[T Keyable](items []T) []T {
	if len(items) <= 1 {
		return items
	}

	pivot := items[0]
	less, greater := []T{}, []T{}
	for _, item := range items[1:] {
		if item.Key() < pivot.Key() {
			less = append(less, item)
		} else {
			greater = append(greater, item)
		}
	}

	return append(append(less, pivot), greater...)
}

type Person struct {
	Name string
	Age  int
}

func (n Person) Key() int {
	return n.Age
}

func main() {
	persons := []Person{{Name: "alice", Age: 33}, {Name: "bob", Age: 27}}

	persons = Sort(persons)
}
```

在這個範例中，`Sort` 要求送進來的 `[]T` 當中的 `T` 要實作 `Keyable` 介面 (提供 `Key` method)。
當我們想排序一堆 `Person` 時，我們可以在這個 `Person` 物件上定義 `Key` method，取出 `Person` 的年齡。
完成之後，我們就可以依年齡來排序 `[]Person` 了。

期許自己未來可以多加利用這個遲來的功能.. XD

## References

- [Go 1.18 Release Notes](https://tip.golang.org/doc/go1.18)
- [Tutorial: Getting started with generics](https://go.dev/doc/tutorial/generics)
