**结构体是值类型**

# 结构体的定义

使用type和struct关键字来定义结构体，具体代码格式如下：

```
type 类型名 struct {
    字段名 字段类型
    字段名 字段类型
    …
}
```

demo：

```
type Employee struct {
    ID                int
    Name,address      string
    DoB               time.Time
    Position  		  string
    Salary    		  int
    ManagerID 	 	  int
}
```

# 结构体实例化

结构体本身也是一种类型，我们可以像声明内置类型一样使用var关键字声明结构体类型。

```
var 结构体实例 结构体类型
```

demo：

```
type Employee struct {
	ID            int
	Name, Address string
	DoB           time.Time
	Position      string
	Salary        int
	ManagerID     int
}

// 方法 （属于一个结构体）
// 结构体为值类型，要修改需要传指针
func (e *Employee) increSalary() {
	e.Salary += 1000
}

func main() {
	var e Employee
	e.increSalary()
	fmt.Printf("%+v", e)
}
//{ID:0 Name: Address: DoB:0001-01-01 00:00:00 +0000 UTC Position: Salary:1000 ManagerID:0}
```

## 匿名结构体

```
func main() {
	var user struct {
		Name string
		Age  int
	}
	user.Name = "alice"
	user.Age = 18
	fmt.Printf("%#v\n", user)
}
// struct { Name string; Age int }{Name:"alice", Age:18}
```

## 指针类型结构体

可以通过使用new关键字对结构体进行实例化，得到的是结构体的地址。 格式如下：

```
var e = new(Employee)
//等价于
//var e *Employee = &Employee{}
fmt.Printf("%T\n", e)    
fmt.Printf("e=%#v\n", e)
// *main.Employee
// e=&main.Employee{ID:0, Name:"", Address:"", DoB:time.Time{wall:0x0, ext:0, loc:(*time.Location)(nil)}, Position:"", Salary:0, ManagerID:0}
```

点操作符也可以和指向结构体的指针一起工作：

```
func main() {
	var mike *Employee = new(Employee)
	mike.Salary += 500
	(*mike).Salary += 500
	fmt.Println("salary: ", mike.Salary)  //1000
}
```

取结构体的地址实例化：

```
p := &Employee{}
```

如果结构体成员名字是以大写字母开头的，那么该成员就是导出的；这是Go语言导出规则决定的。一个结构体可能同时包含导出和未导出的成员。

结构体成员首字母大写时才能在包外访问：

```
package p
type T struct{ a, b int } // a and b are not exported

package q
import "p"
var _ = p.T{a: 1, b: 2} // compile error: can't reference a, b
var _ = p.T{1, 2}       // compile error: can't reference a, b
```

## 结构体初始化

没有初始化的结构体，其成员变量都是对应其类型的零值。

```
type Point struct{ X, Y int }

// 两种初始化方法

// 使用值列表初始化（必须初始化所有字段，必须按照声明顺序初始化）
p := Point{1, 2}

// 使用键值对初始化（可省略字段）
q := Point{X:1, Y:2}
```

# 结构体内存

结构体中的数据占用一块连续的内存。

空结构体是不占用空间的。

```
var v struct{}
fmt.Println(unsafe.Sizeof(v))  // 0
```

# 结构体方法

Go语言中的方法（Method）是一种作用于特定类型变量的函数。这种特定类型变量叫做接收者（Receiver）。接收者的概念就类似于其他语言中的this或者 self。

对于方法，用指针类型接收的方法可以修改结构体中的值

而值类型的接收者，由于struct是值类型，修改结构体的数据时，在拷贝的数据上进行操作，不会对原来的结构体产生修改。

指针类型接收者

等价于：`SetAge(p *Person, newAge int8)`

```
// SetAge 设置p的年龄
// 使用指针接收者

func (p *Person) SetAge(newAge int8) {
	p.age = newAge
}
func main() {
	p1 := NewPerson("阿布", 25)
	fmt.Println(p1.age) // 25
	p1.SetAge(30)
	fmt.Println(p1.age) // 30
}
```

值类型接收者

等价于：`SetAge(p Person, newAge int8)`

```
// SetAge2 设置p的年龄
// 使用值接收者
func (p Person) SetAge2(newAge int8) {
	p.age = newAge
}
func main() {
	p1 := NewPerson("阿布", 25)
	fmt.Println(p1.age) // 25
	p1.SetAge2(30) // (*p1).SetAge2(30)
	fmt.Println(p1.age) // 25
}
```

# 结构体匿名字段

结构体允许其成员字段在声明时没有字段名而只有类型，这种没有名字的字段就称为匿名字段。

```
//Person 结构体Person类型
type Person struct {
	string
	int
}

func main() {
	p1 := Person{
		"bob",
		18,
	}
	fmt.Printf("%#v\n", p1)        //main.Person{string:"北京", int:18}
	fmt.Println(p1.string, p1.int) //北京 18
}
```

匿名字段的说法并不代表没有字段名，而是默认会采用类型名作为字段名，结构体要求字段名称必须唯一，因此一个结构体中同种类型的匿名字段只能有一个。

嵌套结构体的匿名字段

```
//Address 地址结构体
type Address struct {
	Province string
	City     string
}

//User 用户结构体
type User struct {
	Name    string
	Gender  string
	Address //匿名字段
}

func main() {
	var user2 User
	user2.Name = "bob"
	user2.Gender = "男"
	user2.Address.Province = "西藏" // 匿名字段默认使用类型名作为字段名
	user2.City = "拉萨"             // 匿名字段可以省略
	fmt.Printf("user2=%#v\n", user2)
}
```

虽然go会为匿名结构体成员设置类型名作为字段名，但是如果自己添加类型名作为字段名，则访问嵌套结构体的成员时不能省略字段。

上面的例子，struct定义改成Address Address时，就不能通过user2.city访问city成员。

当访问结构体成员时会先在结构体中查找该字段，找不到再去嵌套的匿名字段中查找。

匿名嵌套结构体的字段名冲突

发生冲突时，会以结构体中的字段为准。

# 结构体的继承

Go语言中使用结构体也可以实现其他编程语言中面向对象的继承。

```
//Animal 动物
type Animal struct {
	name string
}

func (a *Animal) move() {
	fmt.Printf("%s会动！\n", a.name)
}

//Dog 狗
type Dog struct {
	Feet    int8
	*Animal //通过嵌套匿名结构体实现继承
}

func (d *Dog) wang() {
	fmt.Printf("%s会犬吠~\n", d.name)
}

func main() {
	d1 := &Dog{
		Feet: 4,
		Animal: &Animal{ //注意嵌套的是结构体指针
			name: "alice",
		},
	}
	d1.wang()
	d1.move()
}
// alice会犬吠~
// alice会动！
```

# 结构体标签

Tag是结构体的元信息，可以在运行的时候通过反射的机制读取出来。 Tag在结构体字段的后方定义，由一对**反引号**包裹起来，具体的格式如下：

```
`key1:"value1" key2:"value2"`
```

结构体tag由一个或多个键值对组成。键与值使用冒号分隔，值用双引号括起来。同一个结构体字段可以设置多个键值对tag，不同的键值对之间使用空格分隔。

**注意事项：** 为结构体编写Tag时，必须严格遵守键值对的规则。结构体标签的解析代码的容错能力很差，一旦格式写错，编译和运行时都不会提示任何错误，通过反射也无法正确取值。例如不要在key和value之间添加空格。

```
//Student 学生
type Student struct {
	ID     int    `json:"id"` //通过指定tag实现json序列化该字段时的key
	Gender string //json序列化是默认使用字段名作为key
	name   string //私有不能被json包访问
}

func main() {
	s1 := Student{
		ID:     1,
		Gender: "男",
		name:   "Bob",
	}
	data, err := json.Marshal(s1)
	if err != nil {
		fmt.Println("json marshal failed!")
		return
	}
	fmt.Printf("json string:%s\n", data) //json str:{"id":1,"Gender":"男"}
}
```
