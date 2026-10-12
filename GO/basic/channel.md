Go语言采用的并发模型是CSP（Communicating Sequential Processes），提倡**通过通信共享内存**而不是**通过共享内存而实现通信**。

如果说 goroutine 是Go程序并发的执行体，channel就是它们之间的连接。channel是可以让一个 goroutine 发送特定值到另一个 goroutine 的通信机制。

Go 语言中的通道（channel）是一种特殊的类型。通道像一个传送带或者队列，总是遵循先入先出（First In First Out）的规则，保证收发数据的顺序。每一个通道都是一个具体类型的管道，也就是声明channel的时候需要为其指定元素类型。

# channel类型

声明：`var 变量名称 chan 元素类型`

如：

```
var ch1 chan int
var ch2 chan string
var ch3 chan []int
```

# chan零值

未初始化的channel变量默认值为nil

# 初始化channel

声明的通道类型变量需要使用内置的make函数初始化之后才能使用。具体格式如下：

`make(chan 元素类型, [缓冲大小])`

demo：

```
ch4 := make(chan int)
ch5 := make(chan int, 8)
```

# channel 操作

通道共有发送，接受，关闭三种操作，发送和接受都使用`<-`操作符。

定义channel：`chan := make(chan int)`

发送：`ch <- 10`

接收：`x := <- ch`或者`<- ch`（忽略返回值）

关闭：`close(ch)`

**注意：**一个通道值是可以被垃圾回收掉的。通道通常由发送方执行关闭操作，并且只有在接收方明确等待通道关闭的信号时才需要执行关闭操作。它和关闭文件不一样，通常在结束操作之后关闭文件是必须要做的，但关闭通道不是必须的。

关闭后的通道有以下特点：

- 对一个关闭的通道再发送值就会导致 panic。
- 对一个关闭的通道进行接收会一直获取值直到通道为空。
- 对一个关闭的并且没有值的通道执行接收操作会得到对应类型的零值。
- 关闭一个已经关闭的通道会导致 panic。

缓冲型通道读写经过buffer，非缓冲型通道的数据交互不经过通道，直接通过内存写传递。

# 无缓冲的通道

无缓冲的通道又称为阻塞的通道

demo：

```
func main() {
	ch := make(chan int)
	ch <- 10
	fmt.Println("发送成功")
}
```

能够编译，但是执行时：

```
➜  concurrent go run main.go
fatal error: all goroutines are asleep - deadlock!

goroutine 1 [chan send]:
main.main()
        /Users/bytedance/example/concurrent/main.go:8 +0x31
exit status 2
```

ch是无缓冲的通道，只有接收方能够接收值时才能发送成功，否则会一直处于等待发送的状态。同理，向一个无缓冲的通道执行接收操作时，没有任何向通道中发送值的操作也会导致阻塞状态。

解决办法：

```
func fetch(ch chan int) {
	<-ch
}
func main() {
	ch := make(chan int)
	ch <- 1
	go fetch(ch)
}
```

输出：

```
➜  concurrent go run main.go               
fatal error: all goroutines are asleep - deadlock!

goroutine 1 [chan send]:
main.main()
        /Users/bytedance/example/concurrent/main.go:8 +0x36
exit status 2
```

修改（先准备好接收的内存，才能往过发）：

```
package main

func fetch(ch chan int) {
	<-ch
}
func main() {
	ch := make(chan int)
	go fetch(ch)
	ch <- 1
}
```

**对于无缓冲的通道，读取和写入是一对操作，两个协程会在c的读取和写入行达成同步。**

# 有缓冲的通道

demo：

```
package main

import "fmt"

func main() {
	ch := make(chan int, 1)
	ch <- 1
	fmt.Println("发送成功")
}
```

写在读之前发生

# 多返回值模式

当向通道中发送完数据时，我们可以通过close函数来关闭通道。当一个通道被关闭后，再往该通道发送值会引发panic，从该通道取值的操作会先取完通道中的值。通道内的值被接收完后再对通道执行接收操作得到的值会一直都是对应元素类型的零值。那我们如何判断一个通道是否被关闭了呢？

对一个通道执行接收操作时支持使用如下多返回值模式。

```
func f2(ch chan int) {
	for {
		v, ok := <-ch
		if !ok {
			fmt.Println("通道已关闭")
			break
		}
		fmt.Printf("v:%#v ok:%#v\n", v, ok)
	}
}

func main() {
	ch := make(chan int, 2)
	ch <- 1
	ch <- 2
	close(ch)
	f2(ch)
}
```

通常我们会选择使用for range循环从通道中接收值，当通道被关闭后，会在通道内的所有值被接收完毕后会自动退出循环。上面那个示例我们使用for range改写后会很简洁。

```
func f3(ch chan int) {
	for v := range ch {
		fmt.Println(v)
	}
}
```

for range会一直迭代直到channel被关闭。

# 单向通道

```
chan<- float64 //可以用于发送float64类型的数据
<-chan int     //可以用于接收int类型的数据
```

# 通道的遍历

对于`for v := range ch {}`，会转换为：

```
ha := a
hv1, hb := <-ha
for ; hb != false; hv1, hb = <-ha {
    v1 := hv1
    hv1 = nil
    ...
}
```
