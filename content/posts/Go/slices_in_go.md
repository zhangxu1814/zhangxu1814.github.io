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

{{<admonition>}}
本文翻译自[https://blog.golang.org/slices](https://blog.golang.org/slices)
{{</admonition>}}

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

## Arrays

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

## Slices: The slice header

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

现在可以把切片看作一个拥有两个元素的数据结构：长度以及指向数组中某个元素的指针。可以把切片想象成这样的一个结构体：
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
你可能经常会听到老手程序猿谈论关于“切片头”的问题，因为那确实是存储在切片变量中的内容，也就是说，一个切片实际上就是上面所描述的一个切片头结构体。例如，当你调用一个以切片作为参数的函数时，比如[bytes.IndexRune](https://golang.org/pkg/bytes/#IndexRune)，就会给这个函数传递一个切片头结构体。在下面这条语句中，
```go
slashPos := bytes.IndexRune(slice, '/')
```
传给 `IndexRune` 的参数 `slice` ，其实是一个切片头结构体

切片头结构体中还有一个元素，我们将会在下面讨论，但是首先我们先来看一下当我们使用切片时，切片头的存在意味着什么。

## Passing slices to functions

即使切片包含一个指针，切片本身也是一个值，理解这点很重要。实际上，它是一个**结构体值**，包含一个指针和一个长度值。它**不是**一个指向结构体的指针。

👆上面这点很重要👆

上一个例子中当我们调用 `IndexRune` 时，它实际上接收的是一个切片头结构体的**副本**。这种行为有很重大的影响。

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
运行结果为：
```go
before [0 1 2 3 4 5 6 7 8 9]
after [1 2 3 4 5 6 7 8 9 10]
```
尽管切片是以切片头结构体**值传递**的方式传入的，但是切片头结构体包含了一个指向底层数组的指针，所以不管是原始的切片还是作为参数传递的切片副本都包含了指向相同的底层数组的指针。因此，当上面的函数返回时，我们可以通过原始切片观察到元素的改动。

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
我们可以看到当切片作为参数传入时，函数可以修改它包含的内容，也就是它指向的底层数组中的值，但是无法修改它的切片头结构体中元素的值。存储在 `slice` 中的长度值并没有被修改，上面的函数只是修改了传入的切片头结构体的**副本**，而不是其本体。这样如果我们想写一个函数修改切片头的话，我们就需要将修改后的副本作为结果返回。在上面的例子中， `slice` 并没有改变，但是返回值 `newSlice` 的长度被修改了。

## Pointers to slices: Method receivers

另一种在函数中修改切片头的方法是将切片的指针作为参数传入。下面是上一个例子的变体：
```go
func PtrSubtractOneFromLength(slicePtr *[]byte) {
    slice := *slicePtr
    *slicePtr = slice[0 : len(slice)-1]
}

func main() {
    fmt.Println("Before: len(slice) =", len(slice))
    PtrSubtractOneFromLength(&slice)
    fmt.Println("After:  len(slice) =", len(slice))
}
```
它的运行结果为：
```go
Before: len(slice) = 50
After:  len(slice) = 49
```
这个例子看起来很麻烦，需要处理间接引用（例子里面引入了一个临时变量来帮助处理），但是有一种常见的情况下你可以看到切片指针，即，通常使用指针作为修改切片的方法的接收者。

我们假设一个切片，这个切片拥有一个在最后一个斜杠处截断它的方法。我们可以这样写：
```go
type path []byte

func (p *path) TruncateAtFinalSlash() {
    i := bytes.LastIndex(*p, []byte("/"))
    if i >= 0 {
        *p = (*p)[0:i]
    }
}

func main() {
    pathName := path("/usr/bin/tso") // Conversion from string to path.
    pathName.TruncateAtFinalSlash()
    fmt.Printf("%s\n", pathName)
}
```
其结果为：
```go
/usr/bin
```
如果我们将这个例子里的接收者类型由指针改为值呢？
```go
type path []byte

func (p path) TruncateAtFinalSlash() {
    i := bytes.LastIndex(p, []byte("/"))
    if i >= 0 {
        p = (p)[0:i]
    }
}

func main() {
    pathName := path("/usr/bin/tso") // Conversion from string to path.
    pathName.TruncateAtFinalSlash()
    fmt.Printf("%s\n", pathName)
}
```
此时运行结果为：
```go
/usr/bin/tso
```
方法是一类带有特殊的接收者**参数**的函数，接收者也是个参数，因此，如果使用值接收者，方法会对原始值的副本进行操作。

但是，当接收者是切片类型时，那么不管是使用值接收者还是指针接收者，我们都可以对底层数组进行修改，因为二者都包含了指向同一个底层数组的指针

看这样一个例子，将 `path` 中的小写字母转换为大写字母：
```go
type path []byte

func (p path) ToUpper() {
    for i, b := range p {
        if 'a' <= b && b <= 'z' {
            p[i] = b + 'A' - 'a'
        }
    }
}

func main() {
    pathName := path("/usr/bin/tso")
    pathName.ToUpper()
    fmt.Printf("%s\n", pathName)
}
```
使用值接收者，依然可以完成修改操作：
```go
/USR/BIN/TSO
```
使用指针接收者也会得到同样的结果：
```go
func (p *path) ToUpper() {
    for i, b := range *p {
        if 'a' <= b && b <= 'z' {
            (*p)[i] = b + 'A' - 'a'
        }
    }
}
```

## Capacity

下面这个函数将它参数中的切片扩展了一个指定的元素：
```go
func Extend(slice []int, element int) []int {
    n := len(slice)
    slice = slice[0 : n+1]
    slice[n] = element
    return slice
}
```
（为什么需要返回修改后的切片呢？）现在我们运行它：
```go
func main() {
    var iBuffer [10]int
    slice := iBuffer[0:0]
    for i := 0; i < 20; i++ {
        slice = Extend(slice, i)
        fmt.Println(slice)
    }
}
```
会得到这样的结果：
```
[0]
[0 1]
[0 1 2]
[0 1 2 3]
[0 1 2 3 4]
[0 1 2 3 4 5]
[0 1 2 3 4 5 6]
[0 1 2 3 4 5 6 7]
[0 1 2 3 4 5 6 7 8]
[0 1 2 3 4 5 6 7 8 9]
panic: runtime error: slice bounds out of range [:11] with capacity 10

goroutine 1 [running]:
main.Extend(...)
	/tmp/sandbox070713439/prog.go:16
main.main()
	/tmp/sandbox070713439/prog.go:25 +0x105
```
现在我们需要讨论一下之前提到的切片头结构体的第三个元素了：它的**容量**（capacity）。除了数组指针和长度，切片头结构体中还存储了切片的容量值：
```go
type sliceHeader struct {
    Length        int
    Capacity      int
    ZerothElement *byte
}
```
容量字段记录了底层数组实际拥有多少空间，它的值就是长度字段可以达到的最大值。当我们试图扩展切片超过它的容量值时，就会造成数组的越界访问，从而导致程序报错（就像上面这个例子一样）。

上面这个例子中我们创建了这样一个切片：
```go
slice := iBuffer[0:0]
```
它的切片头结构体看起来是这样的：
```go
slice := sliceHeader{
    Length:        0,
    Capacity:      10,
    ZerothElement: &iBuffer[0],
}
```
它的容量值等于它指向的底层数组 `iBuffer` 的长度减去切片指向的第一个元素的索引值（在本例中这个值为0）。可以使用内置函数 `cap()` 来获得切片的容量：
```go
if cap(slice) == len(slice) {
    fmt.Println("slice is full!")
}
```

## Make

根据定义，切片的容量值限制了切片可扩展的长度，我们无法扩展一个切片超过它的容量。但是我们可以使用这样的操作来实现这一目的：分配一个新数组，将原数组数据复制过来，然后修改切片使其指向新的数组。

下面我们来尝试一下。我们可以使用 `new()` 函数来分配一个更大的数组然后切片，但是使用 `make()` 函数来代替上述操作更加简单。他一次性完成了分配了一个新的数组并且在数组之上创建了一个切片的操作。 `make()` 函数接受三个参数：切片类型，它的初始长度，和它的容量，也就是 `make` 创建的新数组的长度。下面的语句创建了长度为10，容量为15的切片：
```go
slice := make([]int, 10, 15)
fmt.Printf("len: %d, cap: %d\n", len(slice), cap(slice))
```
运行结果为：
```go
len: 10, cap: 15
```
下面这段代码使切片的容量翻倍，长度不变：
```go
slice := make([]int, 10, 15)
fmt.Printf("len: %d, cap: %d\n", len(slice), cap(slice))
newSlice := make([]int, len(slice), 2*cap(slice))
for i := range slice {
    newSlice[i] = slice[i]
}
slice = newSlice
fmt.Printf("len: %d, cap: %d\n", len(slice), cap(slice))
```
运行结果为：
```go
len: 10, cap: 15
len: 10, cap: 30
```
当创建切片时，长度和容量值通常是相同的。 `make` 函数对于这种情况有一个简写：
```go
gophers := make([]Gopher, 10)
```
切片 `gophere` 的长度和容量均为10

## Copy

上一节的例子中我们使用了一个循环来将数据复制到新的切片中。 `Go` 提供了一个函数 `copy` 来使复制过程更加简单。它的参数是两个切片，将数据从第二个切片中复制到第一个切片中：
```go
newSlice := make([]int, len(slice), 2*cap(slice))
copy(newSlice, slice)
```
`copy` 函数只复制它能够复制的值，也就是说，它复制的元素数量是两个切片的长度中的的最小值。它返回值一个 `int` 值，代表它复制的元素的数量。

当 `copy` 的两个参数是同一个值时，他依然可以正确的操作，也就是说我们可以使用 `copy` 在一个切片中移动元素。下面这个例子展示了如何使用 `copy` 在切片指定位置插入一个值：
```go
// Insert inserts the value into the slice at the specified index,
// which must be in range.
// The slice must have room for the new element.
func Insert(slice []int, index, value int) []int {
    // Grow the slice by one element.
    slice = slice[0 : len(slice)+1]
    // Use copy to move the upper part of the slice out of the way and open a hole.
    copy(slice[index+1:], slice[index:])
    // Store the new value.
    slice[index] = value
    // Return the result.
    return slice
}
```
关于这个函数有几件事需要注意：首先，它必须返回一个新的切片，因为切片的长度值已经被修改了；其次，它使用了简写 `slice[i:]` ，等同于 `slice[i:len(slice)]` ，当然我们也可以忽略这个表达式的第一个值，它默认为0，也就是 `slice[:]` ，它代表这个切片本身。当我们创建一个拥有数组全部元素的切片时，我们就可以这么写：`array[:]` 。

好了，现在我们来运行一下上面这个例子：
```go
slice := make([]int, 10, 20) // Note capacity > length: room to add element.
for i := range slice {
    slice[i] = i
}
fmt.Println(slice)
slice = Insert(slice, 5, 99)
fmt.Println(slice)
```
运行结果为：
```go
[0 1 2 3 4 5 6 7 8 9]
[0 1 2 3 4 99 5 6 7 8 9]
```

## Append: An example

在[Capacity](#capacity)那一节中，我们写了个 `Extend` 函数来为切片扩展一个元素。程序在运行时会报错，因为我们设定的切片的容量太小，导致底层数组越界访问造成程序崩溃。现在我们已经解决了这一问题，可以编写一个健壮的 `Extend` 的实现：
```go
func Extend(slice []int, element int) []int {
    n := len(slice)
    if n == cap(slice) {
        // Slice is full; must grow.
        // We double its size and add 1, so if the size is zero we still grow.
        newSlice := make([]int, len(slice), 2*len(slice)+1)
        copy(newSlice, slice)
        slice = newSlice
    }
    slice = slice[0 : n+1]
    slice[n] = element
    return slice
}
```
这个例子中我们返回了一个切片，因为与原切片相比，返回的切片实际上指向了我们重新分配的底层数组，与原切片完全不同。下面这段代码展示了切片在填充过程中发生了什么：
```go
slice := make([]int, 0, 5)
for i := 0; i < 10; i++ {
    slice = Extend(slice, i)
    fmt.Printf("len=%d cap=%d slice=%v\n", len(slice), cap(slice), slice)
    fmt.Println("address of 0th element:", &slice[0])
}
```
其结果为：
```
len=1 cap=5 slice=[0]
address of 0th element: 0xc00007a030
len=2 cap=5 slice=[0 1]
address of 0th element: 0xc00007a030
len=3 cap=5 slice=[0 1 2]
address of 0th element: 0xc00007a030
len=4 cap=5 slice=[0 1 2 3]
address of 0th element: 0xc00007a030
len=5 cap=5 slice=[0 1 2 3 4]
address of 0th element: 0xc00007a030
len=6 cap=11 slice=[0 1 2 3 4 5]
address of 0th element: 0xc000062060
len=7 cap=11 slice=[0 1 2 3 4 5 6]
address of 0th element: 0xc000062060
len=8 cap=11 slice=[0 1 2 3 4 5 6 7]
address of 0th element: 0xc000062060
len=9 cap=11 slice=[0 1 2 3 4 5 6 7 8]
address of 0th element: 0xc000062060
len=10 cap=11 slice=[0 1 2 3 4 5 6 7 8 9]
address of 0th element: 0xc000062060
```
原始数组容量只有5，当原始数组被填满时，重新分配了一个容量为11的数组，此时第0个元素的地址也改变了。

根据这个例子的思路，我们可以写一个更好的函数 `Append` 来为切片一次性扩充多个元素。这我们会用到 `Go` 语言的可变参数函数，在声明可变参数函数时，需要在参数列表的最后一个参数类型之前加上省略符号“...”，这表示该函数会接收任意数量的该类型参数。

作为最初的实现，我们可以将对 `Extend` 进行多次调用。 `Append` 的函数签名如下：
```go
func Append(slice []int, items ...int) []int
```
`Append` 函数接收一个切片以及任意多个 `int` 类型的元素，这些元素实际上被当成一个切片来进行处理：
```go
// Append appends the items to the slice.
// First version: just loop calling Extend.
func Append(slice []int, items ...int) []int {
    for _, item := range items {
        slice = Extend(slice, item)
    }
    return slice
}
```
执行以下语句：
```go
slice := []int{0, 1, 2, 3, 4}
fmt.Println(slice)
slice = Append(slice, 5, 6, 7, 8)
fmt.Println(slice)
```
其结果为：
```
[0 1 2 3 4]
[0 1 2 3 4 5 6 7 8]
```
注意到这里我们同时声明并初始化了一个切片：
```go
slice := []int{0, 1, 2, 3, 4}
```
我们不仅可以传入任意多个参数，还可以以切片的形式传入参数，不过必须在切片后面加上 `...` ：
```go
slice1 := []int{0, 1, 2, 3, 4}
slice2 := []int{55, 66, 77}
fmt.Println(slice1)
slice1 = Append(slice1, slice2...) // The '...' is essential!
fmt.Println(slice1
```
其结果为：
```
[0 1 2 3 4]
[0 1 2 3 4 55 66 77]
```
我们可以改写一下重分配数组的条件，使其更有效率：
```go
// Append appends the elements to the slice.
// Efficient version.
func Append(slice []int, elements ...int) []int {
    n := len(slice)
    total := len(slice) + len(elements)
    if total > cap(slice) {
        // Reallocate. Grow to 1.5 times the new size, so we can still grow.
        newSize := total*3/2 + 1
        newSlice := make([]int, total, newSize)
        copy(newSlice, slice)
        slice = newSlice
    }
    slice = slice[:total]
    copy(slice[n:], elements)
    return slice
}
```
这里我们使用了两次 `copy` ，一次用于将数据复制进新的数组，一次用于将新的数据复制到数组的末尾。这和最初的函数具有同样的效力。

## Append: The build-in function

到此我们已经明白了内置函数 `append` 的设计动机。它和我们的 `Append` 相似，效率上也差不多，但是它适用于任何切片类型。

`Go` 语言的一个缺点是任何泛型操作必须由运行时提供。以后也许会改变，但是现在，为了使切片操作更加简单， `Go` 提供了内置的泛型 `append` 函数。它适用于任何类型。

由于切片头结构体经常会被 `append` 修改，你需要保存返回的切片值。事实上，编译器不允许你在不保存结果的情况下调用 `append` 。

下面是一些使用 `append` 的语句：
```go
// Create a couple of starter slices.
slice := []int{1, 2, 3}
slice2 := []int{55, 66, 77}
fmt.Println("Start slice: ", slice)
fmt.Println("Start slice2:", slice2)

// Add an item to a slice.
slice = append(slice, 4)
fmt.Println("Add one item:", slice)

// Add one slice to another.
slice = append(slice, slice2...)
fmt.Println("Add one slice:", slice)

// Make a copy of a slice (of int).
slice3 := append([]int(nil), slice...)
fmt.Println("Copy a slice:", slice3)

// Copy a slice to the end of itself.
fmt.Println("Before append to self:", slice)
slice = append(slice, slice...)
fmt.Println("After append to self:", slice)
```
运行结果为：
```
Start slice:  [1 2 3]
Start slice2: [55 66 77]
Add one item: [1 2 3 4]
Add one slice: [1 2 3 4 55 66 77]
Copy a slice: [1 2 3 4 55 66 77]
Before append to self: [1 2 3 4 55 66 77]
After append to self: [1 2 3 4 55 66 77 1 2 3 4 55 66 77]
```
有必要花点时间思考一下最后一行的示例来理解切片的设计如何使这个操作正确的执行

在["Slice Tricks"](https://github.com/golang/go/wiki/SliceTricks)有更多的关于 `append` 、`copy` 以及切片的其他用法的例子。

## Nil

现在我们可以来看一下 `nil` 切片代表着什么。自然的，它代表着切片头结构体的零值：
```go
sliceHeader{
    Length:        0,
    Capacity:      0,
    ZerothElement: nil,
}
```
或者是：
```go
sliceHeader{}
```
注意此时指针的值也是 `nil` 。而由 `array[0:0]` 创建的切片长度为零，但是它的指针值不是 `nil` ，因为它确实指向了一个存在的数组，所以它不是一个 `nil` 切片。

要知道，一个空的切片（假设它的容量不为0）是可以扩充的，但是一个 `nil` 切片由于没有可以填充元素的底层数组，所以是无法扩充的。

`nil` 可以通过重分配数组的方式来追加元素，正如上一节所讲的。

## Strings

这一节用于简短的介绍一下 `Go` 语言中在切片背景下的字符串。

字符串很简单：它是只读的字节切片，外加一点来自语言的额外的语法支持。

因为字符串是只读的，所以它们并不需要容量（你不能扩充他们），但是大多数情况下你可以把它们视为只读的字节切片。

我们可以通过索引访问单个字节：
```go
slash := "/usr/ken"[0] // yields the byte value '/'.
```
可以对字符串进行切片来获取子串：
```go
usr := "/usr/ken"[0:4] // yields the string "/usr"
```
当我们对字符串进行切片时，幕后发生了什么你应该已经了解了。

我们也可以用一个普通的字节切片来创建字符串：
```go
str := string(slice)
```
或者反过来操作：
```go
slice := []byte(usr)
```
字符串的底层数组对于我们来说是不可见的，除了通过字符串本身，我们无法访问到它的内容。也就是说在进行上面的转换时，必须复制数组，不过 `Go` 语言帮我们处理了。完成转换之后，对于字节切片底层数组的修改并不会影响对应的字符串，因为他们指向的是不同的底层数组。

这种类切片的字符串设计使得创建子字符串非常容易，我们只需要创建一个字符串头即可，由于字符串是只读的，因此子串可以和原字符串安全的共享相同的底层数组。

这篇[博客](https://blog.golang.org/strings)更加深入的介绍了有关字符串的内容。

## Conclusion

了解切片的工作原理有助于了解切片的实现方式。一个关键的数据结构，即切片头结构体，它与切片变量相关联，并且它描述了底层数组的一部分。当我们传递切片值时，将传递切片头结构体的副本，但它始终指向同一个底层数组。

一旦你了解了切片的工作原理，你会发现切片不仅便于使用，而且非常强大，特别是在多个内置函数的加持下。

## More reading

除了前面提到的["Slice Tricks"](https://github.com/golang/go/wiki/SliceTricks)，[Go Slices](https://blog.go-zh.org/go-slices-usage-and-internals)这篇博客用图表描述了内存布局的细节。Russ Cox写的[Go Data Structures](https://research.swtch.com/godata)这篇文章包括了对切片的讨论以及其他 `Go` 语言内置的数据结构。

还有很多可用的资料，但是最好的学习方法还是在coding中去使用它们。