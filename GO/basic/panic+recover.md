- panic 能够改变程序的控制流，调用 panic 后会立刻停止执行当前函数的剩余代码，并在当前 Goroutine 中递归执行调用方的 defer；
- recover 可以中止 panic 造成的程序崩溃。它是一个只能在 defer 中发挥作用的函数，在其他作用域中调用不会发挥作用；

![](../assets/basic/panic+recover/1653043502359-3bf04b12-1c56-430a-85ab-96c57b3b855b.png)

利用recover捕获panic异常：

```
package main

import "fmt"

func stupidCode() {
    n := 0
    fmt.Println(1/ n)
}

func main() {
    defer func() {
        if err := recover(); err != nil {
            fmt.Println(err)
        }
    }()
    stupidCode()
}
```
