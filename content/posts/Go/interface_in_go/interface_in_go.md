---
title: "interface in go"
subtitle: ""
date: 2021-06-17T20:53:34+08:00
description: ""
# #上次修改时间
# lastmod: ""
#draft: true
# author: ""
# authorLink: ""
# license: ""

tags: ["Go","interface"]
categories: ["Go"]

# resources:
# - name: "resource-name"
#   src: "resource-path"
---
<!-- 此处内容将作为摘要，若为空，则将description变量的内容作为摘要 -->
<!--more-->

go语言的接口表示一个确定的方法集，并且是隐式实现的，也就是说无需像java那样显式的说明该类实现了哪个接口，只要某个非接口类型的具体值实现了某个接口里的全部方法，就可以说该值实现了某个接口类型，该接口类型的变量就能存储它。

一个典型的例子是io.Reader和io.Writer，即io包中的Reader和Writer类型：
```go
// Reader is the interface that wraps the basic Read method.
type Reader interface {
    Read(p []byte) (n int, err error)
}

// Writer is the interface that wraps the basic Write method.
type Writer interface {
    Write(p []byte) (n int, err error)
}
```
任何正确实现了Read/Write方法（指方法的参数和返回值要对应）的类型，同时也就实现了io.Reader/io.Writer接口。如果某个值的类型实现了Read方法，io.Reader类型的变量就能保存它：
```go
var r io.Reader
r = os.Stdin
r = bufio.NewReader(r)
r = new(bytes.Buffer)
// and so on
```

无论r保存了什么具体的值，r的类型总是io.Reader：Go是静态类型的，而r的静态类型就是io.Reader


## 接口的使用

首先定义接口类型：
```go
type ReadCloser interface {
    Read(b []byte) (n int, err os.Error)
    Close()
}
```
然后定义一个函数，需要传入一个ReadCloser类型作为参数：
```go
func ReadAndClose(r ReadCloser, buf []byte) (n int, err os.Error) {
    for len(buf) > 0 && err == nil {
        var nr int
        nr, err = r.Read(buf)
        n += nr
        buf = buf[nr:]
    }
    r.Close()
    return
}
```
该函数会循环读取buf中的数据然后调用close

调用该函数的代码需要传入一个值，该值的类型必须正确的实现Read和Close方法。不同于Python，如果你传入了一个错误类型的值，会在编译时报错而不是运行时

接口并不是只能静态检查，你也可以动态地检查特定的接口值是否有确定的方法：
```go
type Stringer interface {
    String() string
}

func ToString(any interface{}) string {
    if v, ok := any.(Stringer); ok {
        return v.String()
    }
    switch v := any.(type) {
    case int:
        return strconv.Itoa(v)
    case float:
        return strconv.Ftoa(v, 'g', -1)
    }
    return "???"
}
```
ToString方法传入的参数any是interface{}类型，也就是空接口类型，它表示空方法集，因为任何值都有零个或多个方法，因此任何值都满足空接口，也就是说空接口可以包括任何值。if语句表示将any转为Stringer类型的值是否可行。如果可行，调用String()方法，否则执行switch语句判断是否是其他类型。这个过程类似于fmt包的执行过程

为64位的integer类型实现一个Stringer()方法：
```go
type Binary uint64

func (i Binary) String() string {
    return strconv.Uitob64(i.Get(), 2)
}

func (i Binary) Get() uint64 {
    return uint64(i)
}
```
这样一个Binary类型的值就可以作为参数传入ToString方法中，然后就可以调用String方法将其值格式化，即使Binary并没有显式的声明它实现了Stringer接口，甚至写Binary的人不知道Stringer接口的存在

上述例子表明即使在编译时完成了隐式转换的检查，显式的接口间的转换会在运行时检查方法集，[Effective Go](https://golang.org/doc/effective_go#interfaces)中有更详细的解释

## 接口值

{{<image src="/images/image1.png">}}