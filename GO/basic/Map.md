map是引用类型

# 创建Map

```
// 声明（值为nil）
var m map[int]int 
// 声明并初始化
ages := make(map[string]int) 
m := map[string]int{}
// 创建并初始化元素
ages := map[string]int{"alice":31,"charlie": 34}
// 指定map长度为100
ages := make(map[string]int, 100)
```

# 删除元素

`delete(ages, "alice") // remove element ages["alice"]`

如果一个查找失败将返回value类型对应的零值，例如，即使map中不存在“bob”下面的代码也可以正常工作，因为ages["bob"]失败时将返回0。

`ages["bob"] = ages["bob"] + 1`

禁止对map元素取址的原因是map可能随着元素数量的增长而重新分配更大的内存空间，从而可能导致之前的地址无效。

# 遍历Map

```
func main() {
	scoreMap := make(map[string]int)
	scoreMap["张三"] = 90
	scoreMap["小明"] = 100
	scoreMap["娜扎"] = 60
	for k, v := range scoreMap {
		fmt.Println(k, v)
	}
}
```

# 判断Map是否包含某元素

```
func main() {
	scoreMap := make(map[string]int)
	scoreMap["张三"] = 90
	scoreMap["小明"] = 100
	// 如果key存在ok为true,v为对应的值；不存在ok为false,v为值类型的零值
	v, ok := scoreMap["张三"]
	if ok {
		fmt.Println(v)
	} else {
		fmt.Println("查无此人")
	}
}
```

Go语言中并没有提供一个set类型，通过map[string]bool 表示set
