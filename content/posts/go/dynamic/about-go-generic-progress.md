---
title: "关于 Go 语言泛型设计的最新进展和一些问题的说明"
date: 2020-08-23T18:12:00+08:00
toc: true
isCJKLanguage: true
tags: 
  - Go
  - 泛型
---

前段时间 Go 官方发布了新的泛型草案，一时间在社区引起了很大的反响，各种关于泛型的文章、讨论涌现出来。8 月 21日 Ian Lance Taylor 在 [golang-nuts](https://groups.google.com/g/golang-nuts/c/iAD0NBz3DYw/m/VcXSK55XAwAJ) 讨论组总结了泛型设计的最新进展和一些问题的说明。

Go Team 在经过多次讨论并阅读了许多评论后，计划对泛型设计进行一些更改并澄清草案的一些问题。

## 1

泛型语法极有可能使用方括号 `[]`（不用 <> 是因为和比较运算符大于、小于冲突，为了保持 Go1 兼容性，所以选择了 []）但考虑删除类型参数中的 `type` 关键字，因为使用方括号足以区分类型参数和普通参数。为了避免与数组声明混淆，将要求所有类型参数都提供一个约束（constraint）。这样做的好处是可以给类型参数列表与普通参数列表使用完全相同的语法（除了括号的区别之外）。为简化类型参数的常见情况，该参数可以无限制，将引入一个新的预先声明的标识符 `any` 作为 `interface{}` 的别名。

所以支持泛型的声明类似这样：

```go
type Vector[T any] []T 
func Print[T any](s []T) { … } 
func Index[T comparable](s []T, e T) { … } 
```

Go Team 认为预定义新标识符 `any` 的成本较低：因为每个常规参数始终都有一个类型，每个类型参数始终具有一个约束（其元类型）。

将 `[type T]` 更改为 `[T any]` 似乎同样易读，并且节省了一个字符。我们将能够简化许多现有的标准库和其他地方的代码，只需替换 `interface {}` 为 `any` 即可。

## 2

将简化类型列表满足的规则。如果类型参数或者类型参数的底层（underlying ）类型和类型列表中的任意类型相同，则类型参数满足约定。调整后的规则意味着，类型列表可以决定是否接受除预先声明的类型外的确切定义的类型，或者是否接受具有匹配底层类型的任何类型。

这是一个微小的变化，预计不会影响任何现有的实验代码。

## 3

需要澄清的是，在考虑允许的操作时，对于类型参数中某个类型的值，将忽略在类型列表中的任何类型方法。一般规则是，泛型函数可以使用类型中每种类型允许的任何操作清单。但是，这仅适用于自定义函数和预声明的函数（例如 `len` 和 `cap`）。它不适用于方法，因为类型列表包含所有定义了方法的类型列表。任何方法都必须在 interface 中单独列出，而不是从类型列表中继承。即泛型函数只能使用类型约束所定义的那些操作。

该规则通常看起来很清晰，并且避免了一些复杂的推理涉及类型列表，其中包括带有嵌入式类型的结构参数。

## 4

允许对具有类型列表的类型参数执行类型开关（type switch）操作。用 (.type) 的语法来阐明类似 `switch v := x.(type)` 的代码。在类型参数上的类型开关不能使用 `:=` 语法，因此 `.(type)` 是不必要的。在具有类型列表的类型参数上执行类型开关操作时，列出的每一个 case 都必须是出现在类型列表中的（当然也允许使用 `default`）。如果和类型参数匹配，则该 case 被选中。如上面讨论的那样，可能该 case 不是完全匹配的类型参数而是类型参数的底层类型，该 case 也会被选中。

为了使该规则更明确。没有类型列表的类型参数不允许使用类型开关。但这种情况是允许的：对于一个没有类型列表的类型参数值 `x`，可以写这样的代码，`switch (interface{})(x).(type)`，根据上面的说明，以后应该写成这样 `switch any(x).(type)`。这个结构不是最简单的，但它仅使用了语言已有的特性。

## 接下来

这些更改将很快在以下实验设计中实现：dev.generics 分支，并在 go2go Playground 可用。以上有些已经可以完成了。同时 Go Team 会相应地更新设计草案。