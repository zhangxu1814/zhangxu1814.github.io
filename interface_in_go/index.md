# interface in go

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

一个Binary类型的值是一个64位的Integer，占两个字长。这里声明一个变量b为一个值为200的Binary类型，如下图所示：

![image1](image1.png)

一个接口值代表了了一对内容：一个指针指向存储在该接口中的具体的底层类型描述符，另一个指针指向该具体类型的具体的值。这里声明一个变量s，s是一个Stringer接口类型，其实际存储的是上面声明的变量b：

![image2](image2.png)

上图中的指针图标是灰色的，用以强调它们是隐式的，不直接暴露给go程序

接口值中的第一个字段指向接口表，也就是上图中的itable，itable中包含了存储的底层类型的元数据和函数指针的列表。这里itable对应的是具体类型，而不是接口类型。在这个例子里面，Stringer的itable保存的是Binary类型，以及满足Stringer接口的方法Sting()，Binary的另一个方法Get()并没有被包含在itable中

接口值中的第二个字段指向实际的值，在这个例子里面是变量b的一个**副本**。赋值语句 `var s Stringer = b` 创造了一个b的副本，而不是直接指向b，同样的，`var c uint64 = b` 也创建了一个b的副本，如果b的值变化了，s和c会维持原来的值，而不会变为新的值。接口中保存的值可能是任意大小，但是接口实际上只是保存相应的指针，因此上述的赋值语句会在堆上分配一块内存来记录实际的值

在给接口赋值时，编译器会根据itable中的信息检查类型是否正确，如果类型匹配，则会复制一份副本存储。调用接口函数时，编译器会根据itable中的函数指针列表调用对应的函数，同时传入指向实际值的指针。通常接口在调用函数时不知道传入的这个指针字段的意义是什么，也不知道这个指针指向什么值，只是itable中的函数指针需要该指针字段作为参数。因此，在这个例子里面，函数指针是 `(*Binary).String` 而不是 `Binary.String` 。

## 生成Itable

现在我们知道itables长什么样了，但是它是怎么生成的呢？编译器为每个具体类型，例如`Binary`，`int`，`func(map[int]string)`等创建类型描述符，包含了元数据，方法列表等等信息，然后为每个结构体常见类似的类型描述符，接口在运行时根据自己的方法表去查找具体类型的类型描述符中对应的方法列表来生成itable

在我们的例子中，Stringer拥有1个方法，Binary有两个方法。假设接口类型有*ni*个方法，具体类型有*nt*个方法，寻找每个接口方法与具体类型中对应的方法的映射的复杂度为*O(ni\*nt)*，但是可以优化至*O(ni+nt)*的复杂度

## 参考

https://research.swtch.com/

https://blog.go-zh.org/laws-of-reflection

https://blog.golang.org/laws-of-reflection
