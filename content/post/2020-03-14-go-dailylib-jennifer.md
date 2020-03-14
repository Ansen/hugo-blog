---
layout:    post
title:    "Go 每日一库之 jennifer"
subtitle: 	"每天学习一个 Go 库"
date:		2020-03-14T12:32:23
author:		"darjun"
image:	"img/post-bg-2015.jpg"
tags:
    - Go 每日一库
URL: "2020/03/14/godailylib/jennifer"
categories: [
	"Go"
]
---

## 简介

今天我们介绍一个 Go 代码生成库[`jennifer`](https://github.com/dave/jennifer)。`jennifer`支持所有的 Go 语法和特性，可以用它来生成任何 Go 语言代码。

感谢[kiyonlin](https://github.com/kiyonlin)的推荐！

## 快速使用

先安装：

```golang
$ go get github.com/dave/jennifer
```

今天我们换个思路来介绍`jennifer`这个库。既然，它是用来生成 Go 语言代码的。我们就先写出想要生成的程序，然后看看如何使用`jennifer`来生成。先从第一个程序`Hello World`开始：

```golang
package main

import "fmt"

func main() {
  fmt.Println("Hello World")
}
```

我们如何用`jennifer`来生成上面的程序代码呢：

```golang
package main

import (
  "fmt"

  . "github.com/dave/jennifer/jen"
)

func main() {
  f := NewFile("main")
  f.Func().Id("main").Params().Block(
    Qual("fmt", "Println").Call(Lit("Hello, world")),
  )
  fmt.Printf("%#v", f)
}
```

Go 程序的基本组织单位是文件，每个文件属于一个包，可执行程序必须有一个`main`包，`main`包中又必须有一个`main`函数。所以，我们使用`jennifer`生成代码的大体步骤也是差不多的：

* 先使用`NewFile`定义一个包文件对象，参数即为包名；
* 然后调用这个包文件对象的`Func()`定义函数；
* 函数可以使用`Params()`定义参数，通过`Block()`定义函数体；
* 函数体是由一条一条语句组成的。语句的内容比较多样，后面会详细介绍。

上面代码中，我们首先定义一个`main`包文件对象。然后调用其`Func()`定义一个函数，`Id()`为函数命名为`main`，`Params()`传入空参数，`Block()`中传入函数体。函数体中，我们使用`Qual("fmt", "Println")`来表示引用`fmt.Println`这个函数。使用`Call()`表示函数调用，然后`Lit()`将字符串**"Hello World"**字面量作为参数传给`fmt.Println()`。

`Qual`函数这里需要特意讲一下，我们不需要显示导入包，`Qual`函数的第一个参数就是包路径。如果是标准库，直接就是包名，例如这里的`fmt`。如果是第三方库，需要加上包路径限定，例如`github.com/spf13/viper`。这也是`Qual`名字的由来（`Qualified`，想到**full qualified class name**了吗😄）。`jennifer`在生成程序时会汇总所有用到的包，统一导入。

运行程序，我们最终输出了一开始想要生成的程序代码！

实际上，大多数编程语言的语法都有相通之处。Go 语言的语法比较简单：

* 基本的概念：变量、函数、结构等，它们都有一个名字，又称为标识符。直接写在程序中的数字、字符串等被称为字面量，如上面的**"Hello World"**；
* 流程控制：条件（`if`）、循环（`for`）；
* 函数和方法；
* 并发相关：goroutine 和 channel。

有几点注意：

* 我们在导入`jennifer`包的时候在包路径前面加了一个`.`，使用这种方式导入，在后面使用该库的相关函数和变量时不需要添加`jen.`限定。一般是不建议这样做的。但是`jennifer`的函数比较多，如果不这样的话，每次都需要加上`jen.`比较繁琐。
* `jennifer`的大部分方法都是可以**链式调用**的，每个方法处理完成之后都会返回当前对象，便于代码的编写。

下面我们从上面几个部分依次来介绍。

## 变量定义与运算

其实从语法层面来讲，变量就是标识符 + 类型。上面我们直接使用了字面量，这次我们想先定义一个变量存储欢迎信息：

```golang
func main() {
  var greeting = "Hello World"
  fmt.Println(greeting)
}
```

变量定义的方式有好几种，`jennifer`可以比较直观的表达我们的意图。例如，上面的`var greeting = "Hello World"`，我们基本上可以逐字翻译：

* `var`是变量定义，`jennifer`中有对应的函数`Var()`；
* `greeting`实际上是一个标识符，我们可以用`Id()`来定义；
* `=`是赋值操作符，我们使用`Op("=")`来表示；
* **"Hello World"**是一个字符串字面量，最开始的例子中已经介绍过了，可以使用`Lit()`定义。

所以，这条语句翻译过来就是：

```golang
Var().Id("greeting").Op("=").Lit("Hello World")
```

同样的，我们可以试试另一种变量定义方式`greeting := "Hello World"`：

```golang
Id("greeting").Op(":=").Lit("Hello World")
```

是不是很简单。整个程序如下（省略包名和导入，下同）：

```golang
func main() {
  f := NewFile("main")
  f.Func().Id("main").Params().Block(
    // Var().Id("greeting").Op("=").Lit("Hello World"),
    Id("greeting").Op(":=").Lit("Hello World"),
    Qual("fmt", "Println").Call(Id("greeting")),
  )
  fmt.Printf("%#v\n", f)
}
```

接下来，我们用变量做一些运算。假设，我们要生成下面这段程序：

```golang
package main

import "fmt"

func main() {
  var a = 10
  var b = 2
  fmt.Printf("%d + %d = %d\n", a, b, a+b)
  fmt.Printf("%d + %d = %d\n", a, b, a-b)
  fmt.Printf("%d + %d = %d\n", a, b, a*b)
  fmt.Printf("%d + %d = %d\n", a, b, a/b)
}
```

变量定义这里不再赘述了，方法和函数调用实际上[快速开始](#快速开始)部分也介绍过。首先用`Qual("fmt", "Printf")`表示取包`fmt`中的`Printf`函数这一概念。然后使用`Call()`表示函数调用，参数第一个是字符串字面量，用`Lit()`表示。第二个和第三个都是一个标识符，用`Id("a")`和`Id("b")`即可表示。最后一个参数是两个标识符之间的运算，运算用`Op()`表示，所以最终就是生成程序：

```golang
func main() {
  f := NewFile("main")
  f.Func().Id("main").Params().Block(
    Var().Id("a").Op("=").Lit(10),
    Var().Id("b").Op("=").Lit(2),
    Qual("fmt", "Printf").Call(Lit("%d + %d = %d\n"), Id("a"), Id("b"), Id("a").Op("+").Id("b")),
    Qual("fmt", "Printf").Call(Lit("%d + %d = %d\n"), Id("a"), Id("b"), Id("a").Op("-").Id("b")),
    Qual("fmt", "Printf").Call(Lit("%d + %d = %d\n"), Id("a"), Id("b"), Id("a").Op("*").Id("b")),
    Qual("fmt", "Printf").Call(Lit("%d + %d = %d\n"), Id("a"), Id("b"), Id("a").Op("/").Id("b")),
  )
  fmt.Printf("%#v\n", f)
}
```

逻辑运算是类似的。

## 条件和循环

假设我们要生成下面的程序：

```golang
func main() {
  score := 70

  if score >= 90 {
    fmt.Println("优秀")
  } else if score >= 80 {
    fmt.Println("良好")
  } else if score >= 60 {
    fmt.Println("及格")
  } else {
    fmt.Println("不及格")
  }
}
```

依然采取我们的**逐字翻译**大法：

* `if`关键字用`If()`来表示，条件语句是基本的标识符和常量操作。条件语句块与函数体一样，都使用`Block()`；
* `else`关键字用`Else()`来表示，`else if`就是`Else()`后面再调用`If()`即可。

完整的代码如下：

```golang
func main() {
  f := NewFile("main")

  f.Func().Id("main").Params().Block(
    Id("score").Op(":=").Lit(70),

    If(Id("score").Op(">=").Lit(90)).Block(
      Qual("fmt", "Println").Call(Lit("优秀")),
    ).Else().If(Id("score").Op(">=").Lit(80)).Block(
      Qual("fmt", "Println").Call(Lit("良好")),
    ).Else().If(Id("score").Op(">=").Lit(60)).Block(
      Qual("fmt", "Println").Call(Lit("及格")),
    ).Else().Block(
      Qual("fmt", "Println").Call(Lit("不及格")),
    ),
  )

  fmt.Printf("%#v\n", f)
}
```

对于`for`循环也是类似的，如果我们要生成下面的程序：

```golang
package main

import "fmt"

func main() {
  var sum int
  for i := 1; i <= 100; i++ {
    sum += i
  }

  fmt.Println(sum)
}
```

我们需要编写下面的程序：

```golang
func main() {
  f := NewFile("main")

  f.Func().Id("main").Params().Block(
    Var().Id("sum").Int(),

    For(
      Id("i").Op(":=").Lit(0),
      Id("i").Op("<=").Lit(100),
      Id("i").Op("++"),
    ).Block(
      Id("sum").Op("+=").Id("i"),
    ),

    Qual("fmt", "Println").Call(Id("sum")),
  )

  fmt.Printf("%#v\n", f)
}
```

`For()`里面的 3 条语句对应实际`for`语句中的 3 个部分。

## 函数

函数是每个编程语言的必要元素。函数的核心要素是名字（标识符）、参数列表、返回值，最关键的就是函数体。我们之前编写`main`函数的时候大概介绍过。假设我们要编写一个计算两个数的和的函数：

```golang
func add(a, b int) int {
  return a + b
}
```

* 函数我们使用`Func()`表示，参数用`Params()`表示，返回值使用`Int()`表示；
* 函数体用`Block()`；
* `return`语句使用`Return()`函数表示，其他都是一样的。

看下面的完整代码：

```golang
func main() {
  f := NewFile("main")

  f.Func().Id("add").Params(Id("a"), Id("b").Int()).Int().Block(
    Return(Id("a").Op("+").Id("b")),
  )

  f.Func().Id("main").Params().Block(
    Id("a").Op(":=").Lit(1),
    Id("b").Op(":=").Lit(2),
    Qual("fmt", "Println").Call(Id("add").Call(Id("a"), Id("b"))),
  )

  fmt.Printf("%#v\n", f)
}
```

**一定要注意，即使没有参数，`Params()`也一定要调用，否则生成的代码有语法错误**。

## 结构和方法

下面我们看看结构和方法如何生成，假设我们想生成下面的程序：

```golang
package main

import "fmt"

type User struct {
  Name string
  Age  int
}

func (u *User) Greeting() {
  fmt.Printf("Hello %s", u.Name)
}
func main() {
  u := User{Name: "dj", Age: 18}
  u.Greeting()
}
```

需要用到的新函数：

* 结构体是一个类型，所以需要用到类型定义函数`Type()`，然后结构体的字段在`Struct()`内通过`Id()`+类型定义；
* 方法其实也是一个函数，只不过多了一个接收器，我们还是使用`Func()`定义，接收者也可以用定义参数的`Params()`函数来指定，其它与函数没什么不同；
* 然后结构体初始化，在`Values()`中给字段赋值;
* 方法先用`Dot("方法名")`找到方法，然后`Call()`调用。

最后的程序：

```golang
func main() {
  f := NewFile("main")

  f.Type().Id("User").Struct(
    Id("Name").String(),
    Id("Age").Int(),
  )

  f.Func().Params(Id("u").Id("*User")).Id("Greeting").Params().Block(
    Qual("fmt", "Printf").Call(Lit("Hello %s"), Id("u").Dot("Name")),
  )

  f.Func().Id("main").Params().Block(
    Id("u").Op(":=").Id("User").Values(
      Id("Name").Op(":").Lit("dj"),
      Id("Age").Op(":").Lit(18),
    ),
    Id("u").Dot("Greeting").Call(),
  )

  fmt.Printf("%#v\n", f)
}
```

## 并发支持

还是一样，假设我想生成下面的程序：

```golang
package main

import "fmt"

func generate() chan int {
  out := make(chan int)
  go func() {
    for i := 1; i <= 100; i++ {
      out <- i
    }
    close(out)
  }()

  return out
}

func double(in <-chan int) chan int {
  out := make(chan int)

  go func() {
    for i := range in {
      out <- i * 2
    }
    close(out)
  }()

  return out
}

func main() {
  for i := range double(generate()) {
    fmt.Println(i)
  }
}
```

需要用到的新函数：

* 首先是`make`一个`chan`，用`Make(Chan().Int())`；
* 然后启动一个 goroutine，用`Go()`；
* 关闭`chan`，用`Close()`；
* `for ... range`对应使用`Range()`。

拼在一起就是这样：

```golang
func main() {
  f := NewFile("main")
  f.Func().Id("generate").Params().Chan().Int().Block(
    Id("out").Op(":=").Make(Chan().Int()),
    Go().Func().Params().Block(
      For(
        Id("i").Op(":=").Lit(1),
        Id("i").Op("<=").Lit(100),
        Id("i").Op("++"),
      ).Block(Id("out").Op("<-").Id("i")),
      Close(Id("out")),
    ).Call(),
    Return().Id("out"),
  )

  f.Func().Id("double").Params(Id("in").Op("<-").Chan().Int()).Chan().Int().Block(
    Id("out").Op(":=").Make(Chan().Int()),
    Go().Func().Params().Block(
      For().Id("i").Op(":=").Range().Id("in").Block(Id("out").Op("<-").Id("i").Op("*").Lit(2)),
      Close(Id("out")),
    ).Call(),
    Return().Id("out"),
  )

  f.Func().Id("main").Params().Block(
    For(
      Id("i").Op(":=").Range().Id("double").Call(Id("generate").Call()),
    ).Block(
      Qual("fmt", "Println").Call(Id("i")),
    ),
  )

  fmt.Printf("%#v\n", f)
}
```

## 保存代码

上面的程序中，我们生成代码后直接输出了。在实际应用中，肯定是需要保存到文件中，然后编译运行的。`jennifer`也提供了保存到文件的方法`File.Save()`，直接传入文件名即可，这个`File`就是我们上面调用`NewFile()`生成的对象：

```golang
func main() {
  f := NewFile("main")
  f.Func().Id("main").Params().Block(
    Qual("fmt", "Println").Call(Lit("Hello, world")),
  )

  _, err := os.Stat("./generated")
  if os.IsNotExist(err) {
    os.Mkdir("./generated", 0666)
  }

  err = f.Save("./generated/main.go")
  if err != nil {
    log.Fatal(err)
  }
}

```

这种方式必须要保证`generated`目录存在。所以，我们使用`os`库在目录不存在时创建一个。

## 常见问题

`jennifer`在生成代码后会调用`go fmt`对代码进行格式化，如果代码存在语法错误，这时候会输出错误信息。我遇到最多的问题就是最后生成的程序代码以`})`结尾，这明显不符合语法。查了半天发现`Func()`后忘记加`Params()`。即使是空参数，这个`Params()`也不能省略！

## 总结

`jennifer`支持的远不止我上面介绍的那些，实际上 Go 的语法和特性它都支持，如`select/goto/panic/recover/continue/break`，还有类型断言`b, ok := i.(bool)`、注释、cgo 等等等等。感兴趣可以自己去探索。虽然`jennifer`函数众多，但是按照我们的**逐字翻译**大法，实际上用起来很简单。说实话，我在写上次的测试程序时，基本上没看文档，先按照自己的理解去写，结果大部分都是对！只有一些有问题的地方再去看文档。

说了这么多，`jennifer`的用途是什么呢？答曰，根据配置生成代码，编写生成代码的工具。另外`jennifer`的 GitHub 中有一个`genjen`目录，实际上我们用到的很多函数都是通过它来生成的，是不是很棒😄。

大家如果发现好玩、好用的 Go 语言库，欢迎到 Go 每日一库 GitHub 上提交 issue😄

## 参考

1. jennifer GitHub：[https://github.com/dave/jennifer](https://github.com/dave/jennifer)
2. Go 每日一库 GitHub：[https://github.com/darjun/go-daily-lib](https://github.com/darjun/go-daily-lib)

## 我

[我的博客](https://darjun.github.io)

欢迎关注我的微信公众号【GoUpUp】，共同学习，一起进步~

![](/img/wxgzh8.jpg#center)