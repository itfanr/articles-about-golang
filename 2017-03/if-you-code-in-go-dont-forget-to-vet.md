如果你用Go，不要忘记了vet

## go tool vet是你的好朋友，不要忽视它。


vet是一个优雅的工具，每个Go开发者都要知道并会使用它。它会做代码静态检查发现可能的bug或者可疑的构造。vet是Go tool套件的一部分，我们会在以后的文章中详细描述tool套件。它和go编译器一起发布，这意味着它不需要额外的依赖，可以很方便地通过以下的命令调用：

```
$ go tool vet <directory|files>
```

本文中所有的go代码段可以正常编译。这使得go vet有价值：它可以发现bug，在编译阶段和运行阶段。

同时也注意，本文中的大多数代码都是故意写的很难看，不要使用。

## 在go vet和go tool vet之间选择

go vet和go tool vet实际上是两个分开的命令。

go vet，只在一个单独的包内可用，不能使用flag 选项（来激活某些指定的检测）。

go tool vet更加完整，它可用用于文件和目录。目录被递归遍历来找到包。go tool vet也可以按照检测的分组激活选项。

你可以打开一个终端，并比较go vet --help 和go tool vet --help两个命令的不同。

## Print-format 错误

尽管go是强类型的，但是printf-format错误不会在编译阶段检测。C开发者可能使用默认激活的gcc的-Wformat选项。如果参数不能匹配格式它可以给出一个很好的警告：

```
warning: format ‘%s’ expects argument of type ‘char *’, but argument 2 has type ‘int’ [-Wformat=]
```

不幸的是，在Go里面编译器没有任何输出。这是vet发挥作用的时候了。考虑下面的例子：

```
package main

import "fmt"

func main() {
    str := "hello world!"
    fmt.Printf("%d\n", str)
}
```

这是一个典型的错误，一个坏的printf 格式。因为str是一个字符串，所以format应该用%s，而不是%d。

这个代码编译后运行，打印出`%!d(string=hello world!)`，不够友好。你可以点击源码下面的“run”链接来自己检查。现在，我们开始运行vet。


```
$ go tool vet ex1.go
ex1.go:7: arg str for printf verb %d of wrong type: string
```

当一个指针被使用时，vet也可以检测:

```
package main

import "fmt"

func main() {
    str := "hello world!"
    fmt.Printf("%s\n", &str)
}
```

```
$ go tool vet ex2.go
ex2.go:7: arg &str for printf verb %s of wrong type: *string
```

vet也可以找到所有的Printf()家族函数（Printf(), Sprintf(), Fprintf(), Errorf(), Fatalf(), Logf(), Panicf()等）格式错误。但是如果你要实现一个函数，接收和printf类似的参数，你可以使用-printfuncs选项使得vet来检测。

```
package main

import "fmt"

func customLogf(str string, args ...interface{}) {
    fmt.Printf(str, args...)
}

func main() {
    i := 42
    customLogf("the answer is %s\n", i)
}
```

```
$ go tool vet custom-printf-func.go
$ go tool vet -printfuncs customLogf custom-printf-func.go
custom-printf-func.go:11: arg i for printf verb %s of wrong type: int
```

你可以看到如果没有e -printfuncs选项，vet没有任何输出。

## Boolean 错误

vet可以检查一直为true、false或者冗余的表达式。

```
package main

import "fmt"

func main() {
    var i int

    // always true
    fmt.Println(i != 0 || i != 1)

    // always false
    fmt.Println(i == 0 && i == 1)

    // redundant check
    fmt.Println(i == 0 && i == 0)
}
```

```
$ go vet bool-expr.go
bool-expr.go:9: suspect or: i != 0 || i != 1
bool-expr.go:12: suspect and: i == 0 && i == 1
bool-expr.go:15: redundant and: i == 0 && i == 0
```

这种类型的警告常常是非常危险的，可以引起讨厌的bug。大多数情况下是由于排版错误引起的。

## Range 循环

当读取变量的时候，在range块内的go协程可能是有问题的。在这些场景下，vet可以检测到它们：

```
package main

import "fmt"

func main() {
    words := []string{"foo", "bar", "baz"}

    for _, word := range words {
        go func() {
            fmt.Println(word)
        }()
    }
}
```

注意，这个代码包含竞态，可能不输出任何东西。事实上，main函数可能在所有的协程执行前已经结束，这导致进程退出。

```
$ go tool vet range.go
range.go:10: range variable word captured by func literal
```

## Unreachable的代码

下面的例子包含3个函数，带有不能到达的代码，每个函数使用了不同的方式。

```
package main

import "fmt"

func add(a int, b int) int {
    return a + b

    fmt.Println("unreachable")
    return 0
}

func div(a int, b int) int {
    if b == 0 {
        panic("division by 0")
    } else {
        return a / b
    }

    fmt.Println("unreachable")
    return 0
}

func fibonnaci(n int) int {
    switch n {
    case 0:
        return 1
    case 1:
        return 1
    default:
        return fibonnaci(n-1) + fibonnaci(n-2)
    }

    fmt.Println("unreachable")
    return 0
}

func main() {
    fmt.Println(add(1, 2))
    fmt.Println(div(10, 2))
    fmt.Println(fibonnaci(5))
}
```

```
$ go vet unreachable.go
unreachable.go:8: unreachable code
unreachable.go:19: unreachable code
unreachable.go:33: unreachable code

```

## 混杂的错误

这里是一个代码段，包含了其他的几个vet可以检测的混杂的错误：

```
package main
import (
    "fmt"
    "log"
    "net/http"
)

func f() {}

func main() {
    // Self assignment
    i := 42
    i = i

    // a declared function cannot be nil
    fmt.Println(f == nil)

    // shift too long
    fmt.Println(i >> 32)

    // res used before checking err
    res, err := http.Get("https://www.spreadsheetdb.io/")
    defer res.Body.Close()
    if err != nil {
        log.Fatal(err)
    }
}
```

```
$ go tool vet misc.go
misc.go:14: self-assignment of i to i
misc.go:17: comparison of function f == nil is always false
misc.go:20: i might be too small for shift of 32
misc.go:24: using res before checking for errors
```

## 假正和假负

有时，vet可能忽略了错误，并警告可疑代码，这些代码实际上是正确的。下面的例子：

```
package main

import "fmt"

func main() {
    rate := 42

    // this condition can never be true
    if rate > 60 && rate < 40 {
        fmt.Println("rate %:", rate)
    }
}
```

```
$ go tool vet false.go
false.go:10: possible formatting directive in Println call
```

这种情况很明显永远都不是true，但是并不会检测出来。然而，vet警告了一种可能的错误（使用Println()而不是Printf()），这里Println()非常好。

总的来说，假正和假负足够竞争，使得go tool vet的输出广泛相关。

##性能

vet的README描述了，只是可能的错误是值得检测的。这种方式保证了vet不会变慢。

此时，Docker包含了23M的Go代码（包含依赖）。在Core i5机器上，vet花费了21.6秒来分析它。这是1MB/s的数量级。

可能你期待有一天，可以看到这些“不可能的检查”包含在vet里面。默认不激活他们可能是一个好办法来满足所有人。如果检查在技术上是可行的,并且在现实生活中可以找到实际的缺陷项目,把它作为一个选项是有价值的。

## vet和build比较

虽然vet是不完美的，但是它仍然是一个非常有价值的朋友，它应该在所有的Go项目中定期使用。它是那么有价值，以至于它甚至可以让我们怀疑是不是有些检测不应该被编译器检测到。为什么有人想编译一个检测到有prrintf格式错误的代码呢？