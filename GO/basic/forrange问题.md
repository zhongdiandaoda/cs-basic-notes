# for range

for range创建了每个元素的副本，而不是直接返回每个元素的引用。

```
ha := a
hv1 := 0
hn := len(ha)
v1 := hv1
//创建v2作为循环使用的变量
v2 := nil
for ; hv1 < hn; hv1++ {
    tmp := ha[hv1]
    v1, v2 = hv1, tmp
    ...
}
```

如果使用该值变量的地址作为指向每个元素的指针，就会导致错误，在迭代时，返回的变量是一个迭代过程中根据切片依次赋值的新变量，所以值的地址总是相同的，导致结果不如预期。

```
package main

import "fmt"

func main() {
    slice := []int{0, 1, 2, 3}
    myMap := make(map[int]*int)
    
    // value每次拷贝slice中的值
    // 最后map所有元素指向value，value存的是slice最后一个值
    for index, value := range slice {
        myMap[index] = &value
    }
    fmt.Println("=====new map=====")
    prtMap(myMap)
}

func prtMap(myMap map[int]*int) {
    for key, value := range myMap {
        fmt.Printf("map[%v]=%v\n", key, *value)
    }
}
```

输出：

```
=====new map=====
map[3]=3
map[0]=3
map[1]=3
map[2]=3
```

修改为：

```
package main

import "fmt"

func main() {
    slice := []int{0, 1, 2, 3}
    myMap := make(map[int]*int)

    for index, value := range slice {
        num := value
        myMap[index] = &num
    }
    fmt.Println("=====new map=====")
    prtMap(myMap)
}

func prtMap(myMap map[int]*int) {
    for key, value := range myMap {
        fmt.Printf("map[%v]=%v\n", key, *value)
    }
}
```

无限循环问题：

```
func main() {
	arr := []int{1, 2, 3}
	for _, v := range arr {
		arr = append(arr, v)
	}
	fmt.Println(arr)
}

$ go run main.go
1 2 3 1 2 3
```

由于range会直接拷贝arr到ha中，并获取了ha的长度，因此只会遍历123，向尾部添加123。
