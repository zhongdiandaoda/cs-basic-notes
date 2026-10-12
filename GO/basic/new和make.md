Go语言中new和make是内建的两个函数，主要用来分配内存。

Go语言中：

- 值类型分别有：int系列、float系列、bool、string、数组和结构体
- 引用类型有：指针、slice切片、管道channel、接口interface、map、函数等

make和new的区别：

- make 的作用是初始化内置的数据结构，也就是切片、哈希表和 Channel；
- new 的作用是根据传入的类型分配一片内存空间并返回指向这片内存空间的指针；

new是一个内置的函数，它的函数签名如下：

```
func new(Type) *Type
```

其中：

- Type表示类型，new函数只接受一个参数，这个参数是一个类型
- *Type表示类型指针，new函数返回一个指向该类型内存地址的指针。

```
slice := make([]int, 0, 100)
hash := make(map[int]bool, 10)
ch := make(chan int, 5)
```

1. slice 是一个包含 data、cap 和 len 的结构体 reflect.SliceHeader；
2. hash 是一个指向 runtime.hmap 结构体的指针；
3. ch 是一个指向 runtime.hchan 结构体的指针；
