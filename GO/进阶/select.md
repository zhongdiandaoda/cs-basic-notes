Go 语言中的 select语句用于监控并选择一组case语句执行相应的代码。它看起来类似于switch语句，但是select语句中所有case中的表达式都必须是channel的发送或接收操作。一个典型的select使用示例如下：

```
select {
case <-ch1:
	fmt.Println("aaa")
case ch2 <- 1:
	fmt.Println("qqq")
}
```

Go 语言中的 select 关键字也能够让当前 goroutine 同时等待ch1 的可读和ch2的可写，在ch1和ch2状态改变之前，select 会一直阻塞下去，直到其中的一个 channel 转为就绪状态时执行对应case分支的代码。如果多个channel同时就绪的话则随机选择一个case执行。

# 单case

如果select中只包含一个case，那么该select就变成了一个阻塞的channel读/写操作。

```
select {
case <-ch1:
    fmt.Println("abc")
}
```

# 有default语句

如果select中还可以包含default语句，用于当其他case都不满足时执行一些默认操作。

```
select {
case <-ch1:
	fmt.Println("liwenzhou.com")
default:
    fmt.Println("default action")
}
```

上面的代码，当ch1可读时会执行打印操作，否则就执行default语句中的代码，然后退出。

# 总结

1. select 不存在任何的 case：永久阻塞当前 goroutine
2. select 只存在一个 case：阻塞的发送/接收
3. select 存在多个 case：随机选择一个满足条件的case执行，然后退出
4. select 存在 default，其他case都不满足时：执行default语句中的代码，然后退出

如果有一个或多个IO操作可以完成，则Go运行时系统会随机的选择一个执行，否则的话，如果有default分支，则执行default分支语句，如果连default都没有，则select语句会一直阻塞，直到至少有一个IO操作可以进行。执行完这个操作后，select会退出。

带有优先级的select

```
func worker(ch1, ch2 <-chan int, stopCh chan struct{}) {

	for {
		select {
		case <-stopCh:
			return
		case job1 := <-ch1:
			fmt.Println(job1)
		default:
			select {
			case job2 := <-ch2:
				fmt.Println(job2)
			default:
			}
		}
	}
}
```

问题：两个chan都未就绪一直自旋

解决：

```
func worker2(ch1, ch2 <-chan int, stopCh chan struct{}) {
	for {
		select {
		case <-stopCh:
			return
		case job1 := <-ch1:
			fmt.Println(job1)
		case job2 := <-ch2:
		priority:
			for {
				select {
				case job1 := <-ch1:
					fmt.Println(job1)
				default:
					break priority
				}
			}
			fmt.Println(job2)
		}
	}
}
```
