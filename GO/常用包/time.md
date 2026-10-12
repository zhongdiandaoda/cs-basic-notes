# type Duration

time.Duration是一个int64整数，单位为ns。

1、ParseDuration将一个字符串转换为Duration类型，例如“300ms”，“2h45m”，“5.9665m”，“2.1ms”等

```
import (
	"fmt"
	"time"
)

func main() {
	hours, _ := time.ParseDuration("10h")
	complex, _ := time.ParseDuration("1h10m10s")
	micro, _ := time.ParseDuration("1µs")
	// The package also accepts the incorrect but common prefix u for micro.
	micro2, _ := time.ParseDuration("1us")

	fmt.Println(hours)
	fmt.Println(complex)
	fmt.Printf("There are %.0f seconds in %v.\n", complex.Seconds(), complex)
	fmt.Printf("There are %d nanoseconds in %v.\n", micro.Nanoseconds(), micro)
	fmt.Printf("There are %6.2e seconds in %v.\n", micro2.Seconds(), micro)
}

Output:

10h0m0s
1h10m10s
There are 4210 seconds in 1h10m10s.
There are 1000 nanoseconds in 1µs.
There are 1.00e-06 seconds in 1µs.
```

2、Round函数

以某个时间单位为基准，进行四舍五入。

```
package main

import (
	"fmt"
	"time"
)

func main() {
	d, err := time.ParseDuration("1h15m30.918273645s")
	if err != nil {
		panic(err)
	}

	round := []time.Duration{
		time.Nanosecond,
		time.Microsecond,
		time.Millisecond,
		time.Second,
		2 * time.Second,
		time.Minute,
		10 * time.Minute,
		time.Hour,
	}

	for _, r := range round {
		fmt.Printf("d.Round(%6s) = %s\n", r, d.Round(r).String())
	}
}
Output:

d.Round(   1ns) = 1h15m30.918273645s
d.Round(   1µs) = 1h15m30.918274s
d.Round(   1ms) = 1h15m30.918s
d.Round(    1s) = 1h15m31s
d.Round(    2s) = 1h15m30s
d.Round(  1m0s) = 1h16m0s
d.Round( 10m0s) = 1h20m0s
d.Round(1h0m0s) = 1h0m0s
```

3、truncate

类似Round，但直接去尾，不四舍五入。

4、转换

```
// 返回将Duration转换为指定单位的值
Hours()
Minutes()
Seconds()
Microseconds()
Milliseconds()
Nanoseconds()
```

# select结合time.After使用

```
import "time"
import "fmt"
func main() {
    c1 := make(chan string, 1)
    go func() {
        time.Sleep(time.Second * 2)
        c1 <- "result 1"
    }()
    select {
    case res := <-c1:
        fmt.Println(res)
    case <-time.After(time.Second * 1):
        fmt.Println("timeout 1")
    }
}
```

它利用的是time.After方法，它返回一个类型为<-chan Time的单向的channel，在指定的时间发送一个当前时间给返回的channel中。

# NewTimer

timer是一个定时器，代表未来的一个单一事件，你可以告诉timer你要等待多长时间，它提供一个Channel，在将来的那个时间那个Channel提供了一个时间值。下面的例子中第二行会阻塞2秒钟左右的时间，直到时间到了才会继续执行。

```
timer1 := time.NewTimer(time.Second * 2)
<-timer1.C
fmt.Println("Timer 1 expired")
```

可以使用timer.stop手动停止timer

```
timer2 := time.NewTimer(time.Second)
go func() {
    <-timer2.C
    fmt.Println("Timer 2 expired")
}()
stop2 := timer2.Stop()
if stop2 {
    fmt.Println("Timer 2 stopped")
}
```

# NewTicker

ticker是一个定时触发的计时器，它会以一个间隔(interval)往Channel发送一个事件(当前时间)，而Channel的接收者可以以固定的时间间隔从Channel中读取事件。下面的例子中ticker每500毫秒触发一次，你可以观察输出的时间。

```
ticker := time.NewTicker(time.Millisecond * 500)
go func() {
    for t := range ticker.C {
        fmt.Println("Tick at", t)
    }
}()
```

类似timer, ticker也可以通过Stop方法来停止。一旦它停止，接收者不再会从channel中接收数据了。

# Tick

Tick方法可以达成同样的效果

```
package main

import (
	"fmt"
	"time"
)

func statusUpdate() string { return "" }

func main() {
	c := time.Tick(5 * time.Second)
	for next := range c {
		fmt.Printf("%v %s\n", next, statusUpdate())
	}
}
```

源码：

```
func Tick(d Duration) <-chan Time {
    if d <= 0 {
        return nil
    }
    return NewTicker(d).C
}
```

**Tick是NewTicker的封装，只提供对ticking频道的访问。虽然Tick对于不需要关闭Ticker的客户端很有用，但要注意，如果没有关闭Ticker的方法，垃圾收集器将无法恢复底层的Ticker；将会造成内存泄露。与NewTicker不同，如果d<=0，Tick将返回nil。**
