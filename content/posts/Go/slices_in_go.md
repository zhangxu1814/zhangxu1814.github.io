---
title: "slice in Go"
subtitle: ""
date: 2021-05-05T19:18:25+08:00
description: "本文翻译自https://blog.golang.org/slices"
# 上次修改时间
# lastmod: ""
# draft: false
# author: ""
# authorLink: ""
# license: ""

tags: ["Go","切片"]
categories: ["Go"]

# resources:
# - name: "resource-name"
#   src: "resource-path"
---

<!-- 此处内容将作为摘要，若为空，则将description变量的内容作为摘要 -->
<!--more-->

过程式编程语言最常见的特性之一便是数组的概念。数组看起来很简单，但是当把它加进一门语言里时，需要考虑很多问题，例如：

* 固定大小 or 可变大小？
* 数组的大小要作为数组类型的一部分吗？
* 多维数组看起来是什么样的？
* 空数组有意义吗？
* ...

数组是语言的一个特性还是其设计的关键部分将取决于这些问题的答案。

在go的早期开发过程中，花了一年的时间来决定这些问题的答案。其中关键的一步便是引入了切片，它构建在固定大小的数组上，提供了灵活、可扩展的数据结构。然而，直到今天，收到来自于其他编程语言的经验的影响，刚接触go的程序员往往会掉进切片工作方式这个大坑里。

这篇文章将会解释 `slice` 以及内置函数 `append` 是如何工作的，以及它们为什么会被设计为这样工作。

### Arrays

数组是go语言中一个重要的构建模块，但是如建筑物的地基一样，它通常隐藏在更多的可见组件之下。在讨论切片的其他更有趣，更强大，更突出的概念之前，我们必须先简要的讨论一下数组。

数组在go语言中并不常见，因为数组的大小是它的类型的一部分，这限制了数组的表达能力。

```go
var buffer [256]byte
```
上面的表达式声明了一个变量 `buffer` ，它包含256个字节。 `buffer` 的类型声明包含了它的大小，[256]byte。而一个包含512个字节的数组变量类型是[512]byte。

包含在一个数组中的数据在内存中看起来是这样的：
```
buffer: byte byte byte ...... byte byte byte
```
我们可以通过索引来访问 `buffer` 中的元素，例如 `buffer[0]` ， `buffer[1]` ，直到 `buffer[255]` 。越界访问buffer将导致程序崩溃。

内置函数 `len()` 可以返回数组或者切片或者其他某些数据类型中包含的元素的数量， `len(buffer)` 返回固定值256

数组有它自己的用处，比如数组很好的表示了转换矩阵，但是在go语言中他们更多的被用来为切片保留存储空间

### Slices: The slice header

想要更好的使用切片，我们必须明白切片是什么以及切片可以做什么

切片是一种数据结构，它描述了一个数组中的一段连续的数据，数组与切片并不占用同一块存储空间。一个切片并不是一个数组。切片描述了数组的一个片段。

继续使用上一节中的数组变量 `buffer` ，我们可以通过切分（slicing）这个数组来创造一个描述 `buffer` 中的第100-150个元素（不包括第150个元素）的切片：
```go
var slice []byte = buffer[100:150]
```
变量 `slice` 的类型为 `[]byte` ，读作"slice of bytes"。 `slice` 通过切分数组 `buffer` 的第100到第150个元素来完成初始化。更常用的语法是去掉类型部分，根据初始化表达式设置类型：
```go
var slice = buffer[100:150]
```
在函数内部，我们可以使用短变量声明：
```go
slice := buffer[100:150]
```
注意只能在函数内部使用短变量声明，因为函数外的每条语句都必须以关键字开始（var，func等）。

现在可以把切片看作一个拥有两个元素的数据结构：长度以及指向数组中某个元素的指针。可以把切片当作这样的一个结构：
```go
type sliceHeader struct {
    Length        int
    ZerothElement *byte
}

slice := sliceHeader{
    Length:        50,
    ZerothElement: &buffer[100],
}
```
当然，这只是一个为了便于理解做出的假设示例。尽管上面这段代码表明 `sliceHeader` 类型对程序猿是不可见的，并且指针的类型取决于它所指向的元素的类型，但是这给了我们关于切片机制的大体思路。

目前为止我们已经在数组上进行了切片操作，但是我们还可以在切片上进行切片操作，就像这样：
```go
slice2 := slice[5:10]
```
就像之前的切片操作一样，上面的语句创建了一个新的切片， `slice2` 包括了 `slice` 中的第5-9（不含）个元素，也就是数组 `buffer` 中的第105-109（不含）个元素。 `slice2` 底层的 `sliceHeader` 看起来是这样的：
```go
slice2 := sliceHeader{
    Length:        5,
    ZerothElement: &buffer[105],
}
```
注意此时 `slice2` 的 `sliceHeader` 仍然指向相同的底层数组，也就是 `buffer` 。

我们也可以重切片（*reslice*），也就是对一个切片进行切片操作，并将结果存储在原始的切片结构中：
```go
slice = slice[5:10]
```
此时 `slice` 的 `sliceHeader` 和上面的 `slice2` 的 `sliceHeader` 看起来是一样的。重切片操作应用的更多，例如截断一个切片。下面这条语句删除了切片的第一个和最后一个元素：
```go
slice = slice[1:len(slice)-1]
```
您可能经常会听到老练的go程序猿谈论关于“切片头”的问题，因为那确实是存储在切片变量中的内容。例如，当你调用一个以切片作为参数的函数时，比如[bytes.IndexRune](https://golang.org/pkg/bytes/#IndexRune)，就会给这个函数传递一个切片头。在下面这条语句中，
```go
slashPos := bytes.IndexRune(slice, '/')
```
传给 `IndexRune` 的参数 `slice` ，其实是一个切片头

切片头中还有一个元素，我们将会在下面讨论，但是首先我们先来看一下当我们使用切片时，切片头的存在意味着什么。

### Passing slices to functions

即使切片包含一个指针，切片本身也是一个值，理解这点很重要。实际上，它是一个结构体**值**，包含一个指针和一个长度值。它**不是**一个指向结构体的指针。

👆上面这点很重要👆

上一个例子中当我们调用 `IndexRune` 时，它实际上接收的是一个切片头的**副本**。这种行为有很重大的影响。

考虑这样一个简单的函数：
```go
func AddOneToEachElement(slice []byte) {
    for i := range slice {
        slice[i]++
    }
}
```
就像他的函数名一样，这个函数会遍历整个切片，并将切片中的每个元素加一。

```go
func main() {
    slice := buffer[10:20]
    for i := 0; i < len(slice); i++ {
        slice[i] = byte(i)
    }
    fmt.Println("before", slice)
    AddOneToEachElement(slice)
    fmt.Println("after", slice)
}
```
```go
before [0 1 2 3 4 5 6 7 8 9]
after [1 2 3 4 5 6 7 8 9 10]
```
尽管切片头是以值传递的方式传入的，但是切片头包含了一个指向底层数组的指针，所以不管是原始的切片还是作为参数传递的切片都指向了相同的底层数组。因此，当上面的函数返回时，我们可以通过原始切片观察到元素的改动。

下面这个例子展示了传递给函数的参数是副本：
```go
func SubtractOneFromLength(slice []byte) []byte {
    slice = slice[0 : len(slice)-1]
    return slice
}

func main() {
    fmt.Println("Before: len(slice) =", len(slice))
    newSlice := SubtractOneFromLength(slice)
    fmt.Println("After:  len(slice) =", len(slice))
    fmt.Println("After:  len(newSlice) =", len(newSlice))
}
```
```go
Before: len(slice) = 50
After:  len(slice) = 50
After:  len(newSlice) = 49
```
我们可以看到当切片作为参数传入时，函数可以修改它包含的内容，但是无法修改它的切片头。存储在 `slice` 中的长度值并没有被修改，上面的函数只是修改了传入的副本的切片头，而不是本体的切片头。这样如果我们想写一个函数修改切片头的话，我们就需要将修改后的副本作为结果返回。在上面的例子中， `slice` 并没有改变，但是返回值 `newSlice` 的长度被修改了。