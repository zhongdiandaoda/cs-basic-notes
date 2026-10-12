# 包含判断

- 前后缀包含

```
func main() {
	//前后缀包含
	var str string = "This is a fat cat"
	var str1 string = "Hello Boss!"
	var str2 string = "Lucy is name My"
	fmt.Printf("%t\n", strings.HasPrefix(str, "is"))		//false
	fmt.Printf("%t\n", strings.HasPrefix(str1, "He"))		//true
	fmt.Printf("%t\n,", strings.HasSuffix(str2, "My"))     //true    
}
```

- 子字符串包含关系

```
func main() {

	//使用ContainsAny方法查询字符串是否包含多个字符，
	fmt.Println(strings.ContainsAny("team", "e"))     //true
	fmt.Println(strings.ContainsAny("chain", ""))     //false
	fmt.Println(strings.ContainsAny("name", "n & m")) //true

	//使用Contains方法查找某个字符串是否存在这个字符串中，若存在，返回true
	fmt.Println(strings.Contains("team", "t & a")) //false
	fmt.Println(strings.Contains("chain", ""))     //true
	fmt.Println(strings.Contains("name", "n & m")) //false
}
```

# 索引

```
func main() {

	str := "Hello, Bye Bye"
	str1 := "前尘往事，莫再提及"

	fmt.Printf("%d\n", strings.Index(str, "Bye"))     //7
	fmt.Printf("%d\n", strings.LastIndex(str, "Bye")) //11
	fmt.Printf("%d\n", strings.Index(str, "Tiger"))   //不存在相应的子串，返回-1

	//适用于非ASCII编码的字符，使用strings.IndexRune()对字符进行定位
	fmt.Printf("%d\n", strings.IndexRune(str1, '事')) //一个中文占用3字节长度，返回9。
}
```

# Replace

```
func main() {
	str := "七秒钟的记忆" //原字符串
	r := "亿年"       //替换的新内容
	old := "秒钟"     //被替换的字符串
	n := 1          //表示替换到第n个old
	// n = -1代表全部替换
	fmt.Println(strings.Replace(str, old, r, n)) //使用Replace函数替换字符串
	// 或者
	fmt.Println(strings.ReplaceAll(str, old, r)) //使用Replace函数替换字符串
}
```

# Count

统计子串出现频率

```
func main() {
	str := "what a mess, this room"
	str1 := "统计子串出现的频率，频率，频率"

	//使用strings.Count()统计的字符串出现频率
	fmt.Printf("%d\n", strings.Count(str, "ss")) //1
	fmt.Printf("%d\n", strings.Count(str1, "频率")) //3
}
```

统计字符串中的字符数量：

```
func main() {
    str := "星辰之力"
    fmt.Printf("%d\n", len([]rune(str)))     //4
    fmt.Println(utf8.RuneCountInString(str)) //4
}
```

# 大小写转换

```
func main() {
	str := "Hello World!\n"
	fmt.Printf(strings.ToLower(str)) //hello world!
	fmt.Printf(strings.ToUpper(str)) //HELLO WORLD!
}
```

# Split

```
func main() {
	str := "a,b,c,d,e,f,g 6"
	fmt.Println(str)
	fmt.Printf("%q\n", strings.Split(str, "d")) //["a,b,c," ",e,f,g 6"]
	fmt.Printf("%q\n", strings.Split(str, " ")) //["a,b,c,d,e,f,g" "6"]
	fmt.Printf("%q\n", strings.Split(str, ",")) //["a" "b" "c" "d" "e" "f" "g 6"]
}
```

实现

```
func genSplit(s, sep string, sepSave, n int) []string {
	if n == 0 {
		return nil
	}
	if sep == "" {
		return explode(s, n)
	}
	if n < 0 {
		n = Count(s, sep) + 1
	}

	a := make([]string, n)
	n--
	i := 0
	for i < n {
		m := Index(s, sep)
		if m < 0 {
			break
		}
		a[i] = s[:m+sepSave]
		s = s[m+len(sep):]
		i++
	}
	a[i] = s
	return a[:i+1]
}
```

# Trim

```
//输出：love
str := strings.TrimSpace("   love   ")
//输出：love
str := strings.Trim("@love@", "@")
//输出：love@
str := strings.TrimLeft("@love@", "@")
//输出：@love
str := strings.TrimRight("@love@", "@")
```

# Repeat

```
func main() {
	str := "uuid-" //原字符串
	str = strings.Repeat(str, 10)
	fmt.Println(str)
}
// uuid-uuid-uuid-uuid-uuid-uuid-uuid-uuid-uuid-uuid-
```

# Join

```
func main() {
	list := []string{"a", "b", "c"}
	str := strings.Join(list, "==========")
	fmt.Println(str)
}
// a==========b==========c
```

# 整数转换

```
/*输出：string*/
str := strconv.Itoa(22)
fmt.Printf("%T", str)

/*输出：int*/
i, _ := strconv.Atoi("22")
fmt.Printf("%T", i)
```
