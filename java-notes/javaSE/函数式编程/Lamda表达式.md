# Overview

在Java程序中，我们经常遇到一大堆单方法接口，即一个接口只定义了一个方法：

- Comparator
- Runnable
- Callable

以`Comparator`为例，我们想要调用`Arrays.sort()`时，可以传入一个`Comparator`实例，以匿名类方式编写如下：

```
String[] array = ...
Arrays.sort(array, new Comparator<String>() {
    public int compare(String s1, String s2) {
        return s1.compareTo(s2);
    }
});
```

Lambda表达式的作用是代替匿名内部类的烦琐语法。

当使用Lambda表达式代替匿名内部类创建对象时，Lambda表达式的代码块将会代替实现抽象方法的方法体。

Lambda表达式主要由三个部分组成：

- 形参列表。形参列表允许省略形参类型。如果形参列表只有一个参数，那么括号也可以省略。
- 箭头(->)。
- 代码块：如果代码块只有一行代码，可以省略花括号。

Lambda表达式和匿名内部类都可以直接访问外部类的成员变量，但它们会被自动当作final成员来进行处理。

**匿名内部类实现的抽象方法的方法体允许调用接口中定义的default方法，而Lambda表达式不可以。**

# Lambda表达式与函数式接口

Lambda表达式的类型，也被称为“目标类型”，必须是函数式接口，即只包含一个抽象方法的接口。函数式接口可以包含多个default方法、类方法，但只能声明一个抽象方法。

Runnable、ActionListener等接口都是函数式接口。

程序中，可以使用Lambda表达式对对象进行赋值。

```java
Runnable r = () -> {
    for(int i = 0; i < 10; i++)
        System.out.println(i);
}
```

`Java.util.fuction`包下包含了以下4类典型的函数式接口：

- `XxxFunction`：通常包含一个apply方法，该方法对参数进行处理、转换，然后返回一个新的值。通常用作对特定数据进行转换处理。
- `XxxConsumer`：通常包含一个accept方法，负责对参数进行处理，但不会返回结果。
- `XxxPredicate`：通常包含一个test方法，用于对参数进行某种判断，然后返回一个`boolean`值。
- `XxxSupplier`：通常包含一个`getAsXxx()`抽象方法，该方法不需要输入参数，该方法会按照某种算法返回一个数据。

demo：

使用Lambda表达式实现忽略大小写排序

```java
Arrays.sort(array, String::compareToIgnoreCase);
// or
Arrays.sort(array, (o1,o2) -> o1.compareToIgnoreCase(o2));
// or
Arrays.sort(array, (o1, o2)-> o1.toLowerCase().compareTo(o2.toLowerCase()));
```

当Lambda表达式的代码块只有一行代码时，程序可以省略表达式中代码块的花括号，还能在代码块使用方法引用和构造器引用。

Lambda表达式支持的引用方式如下：

| 种类                   | 示例               | 说明                                                         |
| ---------------------- | ------------------ | ------------------------------------------------------------ |
| 引用类方法             | 类名::类方法       | 接口被实现方法的全部参数传给该类方法作为参数                 |
| 引用特定对象的实例方法 | 特定对象::实例方法 | 接口被实现方法的全部参数传给该方法作为参数                   |
| 引用某类对象的实例方法 | 类名::实例方法     | 接口被实现方法的第一个参数作为调用者，后面的参数传给该方法作为参数 |
| 引用构造器             | 类名::new          | 接口被实现方法的全部参数传给该构造器作为参数                 |

## 类的静态方法

例如，定义了以下函数式接口：

```java
@FunctionalInterface
interface Conventer {
    Integer convert(String from);
}
```

该接口的方法负责把字符串转为Integer对象，使用Lambda表达式创建对象的语法：

```java
Converter c = from -> Integer.valueOf(from);
```

上面的代码可以使用以下方法引用进行替换：

```java
//函数式接口的所有参数都传递给该方法
Converter c = Integer::valueOf();
```

## 类的实例方法

定义接口：

```java
@FunctionalInterface
interface MyTest {
    String test(String a, int b, int c);
}
```

创建实例：

```java
MyTest myTest = (a, b, c) -> a.substring(b, c);
```

方法引用：

```java
MyTest myTest = String::substring;
```

**类名::方法名，如果方法为静态方法，则将接口方法的全部参数传递给该静态方法；方法为实例方法是，将接口方法的第一个参数作为调用者，其他参数全部传递给该实例方法。**

## 特定对象的实例方法

使用基本lambda表达式创建一个Converter对象：

```java
String a = "999999994444445";
Conventer c = from -> a.indexOf(from);
```

调用：

```java
Integer v = c.convert();
```

可替换为：

```java
Converter c = a::indexOf;
```

# 构造器

如果要把一个`List<String>`转换为`List<Person>`，应该怎么办？

```
class Person {
    String name;
    public Person(String name) {
        this.name = name;
    }
}

List<String> names = List.of("Bob", "Alice", "Tim");
List<Person> persons = ???
```

传统的做法是先定义一个`ArrayList<Person>`，然后用`for`循环填充这个`List`：

```
List<String> names = List.of("Bob", "Alice", "Tim");
List<Person> persons = new ArrayList<>();
for (String name : names) {
    persons.add(new Person(name));
}
```

要更简单地实现`String`到`Person`的转换，我们可以引用`Person`的构造方法：

```java
public class Main {
    public static void main(String[] args) {
        List<String> names = List.of("Bob", "Alice", "Tim");
        List<Person> persons = names.stream().map(Person::new).collect(Collectors.toList());
        System.out.println(persons);
    }
}

class Person {
    String name;
    public Person(String name) {
        this.name = name;
    }
    public String toString() {
        return "Person:" + this.name;
    }
}
```

stream 的`map()`需要传入的 FunctionalInterface 的定义是：

```
@FunctionalInterface
public interface Function<T, R> {
    R apply(T t);
}
```

把泛型对应上就是方法签名`Person apply(String)`，即传入参数`String`，返回类型`Person`。而`Person`类的构造方法恰好满足这个条件，因为构造方法的参数是`String`，而构造方法虽然没有`return`语句，但它会隐式地返回`this`实例，类型就是`Person`，因此，此处可以引用构造方法。构造方法的引用写法是`类名::new`，因此，此处传入`Person::new`。

# Stream

Java 8引入了一个全新的流式API：Stream API。它位于`java.util.stream`包中。

这个`Stream`不同于`java.io`的`InputStream`和`OutputStream`，它代表的是任意Java对象的序列。两者对比如下：

|      | java.io                  | java.util.stream           |
| :--- | :----------------------- | :------------------------- |
| 存储 | 顺序读写的`byte`或`char` | 顺序输出的任意Java对象实例 |
| 用途 | 序列化至文件或网络       | 内存计算／业务逻辑         |

`List`的用途是操作一组已存在的Java对象，而`Stream`实现的是惰性计算，两者对比如下：

|      | java.util.List           | java.util.stream     |
| :--- | :----------------------- | :------------------- |
| 元素 | 已分配并存储在内存       | 可能未分配，实时计算 |
| 用途 | 操作一组已存在的Java对象 | 惰性计算             |

使用 Stream 存储全体自然数：

```java
Stream<BigInteger> naturals = createNaturalStream(); // 全体自然数
```

Stream 可以转换成不同的 Stream：

```java
Stream<BigInteger> naturals = createNaturalStream(); // 不计算
Stream<BigInteger> s2 = naturals.map(BigInteger::multiply); // 不计算
Stream<BigInteger> s3 = s2.limit(100); // 不计算
s3.forEach(System.out::println); // 计算
```

基本用法：

```java
int result = createNaturalStream() // 创建Stream
             .filter(n -> n % 2 == 0) // 任意个转换
             .map(n -> n * n) // 任意个转换
             .limit(100) // 任意个转换
             .sum(); // 最终计算结果
```

Stream API的特点是：

- Stream API提供了一套新的流式处理的抽象序列；
- Stream API支持函数式编程和链式操作；
- Stream可以表示无限序列，并且大多数情况下是惰性求值的。

## 创建 Stream

### Stream.of()

创建`Stream`最简单的方式是直接用`Stream.of()`静态方法，传入可变参数即创建了一个能输出确定元素的`Stream`：

```java
public class Main {
    public static void main(String[] args) {
        Stream<String> stream = Stream.of("A", "B", "C", "D");
        // forEach()方法相当于内部循环调用，
        // 可传入符合Consumer接口的void accept(T t)的方法引用：
        stream.forEach(System.out::println);
    }
}
```

### 基于数组或Collection

第二种创建`Stream`的方法是基于一个数组或者`Collection`，这样该`Stream`输出的元素就是数组或者`Collection`持有的元素：

```java
public class Main {
    public static void main(String[] args) {
        Stream<String> stream1 = Arrays.stream(new String[] { "A", "B", "C" });
        Stream<String> stream2 = List.of("X", "Y", "Z").stream();
        stream1.forEach(System.out::println);
        stream2.forEach(System.out::println);
    }
}
```

把数组变成`Stream`使用`Arrays.stream()`方法。对于`Collection`（`List`、`Set`、`Queue`等），直接调用`stream()`方法就可以获得`Stream`。

### 基于Supplier

创建`Stream`还可以通过`Stream.generate()`方法，它需要传入一个`Supplier`对象：

```
Stream<String> s = Stream.generate(Supplier<String> sp);
```

基于`Supplier`创建的`Stream`会不断调用`Supplier.get()`方法来不断产生下一个元素，这种`Stream`保存的不是元素，而是算法，它可以用来表示无限序列。

例如，我们编写一个能不断生成自然数的`Supplier`，它的代码非常简单，每次调用`get()`方法，就生成下一个自然数：

```java
public class Main {
    public static void main(String[] args) {
        Stream<Integer> natual = Stream.generate(new NatualSupplier());
        // 注意：无限序列必须先变成有限序列再打印:
        natual.limit(20).forEach(System.out::println);
    }
}

class NatualSupplier implements Supplier<Integer> {
    int n = 0;
    public Integer get() {
        n++;
        return n;
    }
}
```

对于无限序列，如果直接调用`forEach()`或者`count()`这些最终求值操作，会进入死循环，因为永远无法计算完这个序列，所以正确的方法是先把无限序列变成有限序列，例如，用`limit()`方法可以截取前面若干个元素，这样就变成了一个有限序列，对这个有限序列调用`forEach()`或者`count()`操作就没有问题。

## 使用 map

`Stream.map()`是`Stream`最常用的一个转换方法，它把一个`Stream`转换为另一个`Stream`。

所谓`map`操作，就是把一种操作运算，映射到一个序列的每一个元素上。例如，对`x`计算它的平方，可以使用函数`f(x) = x * x`。我们把这个函数映射到一个序列1，2，3，4，5上，就得到了另一个序列1，4，9，16，25：

```java
Stream<Integer> s = Stream.of(1, 2, 3, 4, 5);
Stream<Integer> s2 = s.map(n -> n * n);
```

`map()`的声明：`<R> Stream<R> map(Function<? super T, ? extends R> mapper);`

其中，`Function`的定义是：

```java
@FunctionalInterface
public interface Function<T, R> {
    // 将T类型转换为R:
    R apply(T t);
}
```

demo：

```java
public class Main {
    public static void main(String[] args) {
        List.of("  Apple ", " pear ", " ORANGE", " BaNaNa ")
                .stream()
                .map(String::trim) // 去空格
                .map(String::toLowerCase) // 变小写
                .forEach(System.out::println); // 打印
    }
}
```

## 使用 filter

`Stream.filter()`是`Stream`的另一个常用转换方法。

所谓`filter()`操作，就是对一个`Stream`的所有元素一一进行测试，不满足条件的就被“滤掉”了，剩下的满足条件的元素就构成了一个新的`Stream`。

例如，我们对1，2，3，4，5这个`Stream`调用`filter()`，传入的测试函数`f(x) = x % 2 != 0`用来判断元素是否是奇数，这样就过滤掉偶数，只剩下奇数，因此我们得到了另一个序列1，3，5：

```java
public class Main {
    public static void main(String[] args) {
        IntStream.of(1, 2, 3, 4, 5, 6, 7, 8, 9)
                .filter(n -> n % 2 != 0)
                .forEach(System.out::println);
    }
}
```

`filter` 方法接收一个 Predicate 接口对象：

```java
@FunctionalInterface
public interface Predicate<T> {
    // 判断元素t是否符合条件:
    boolean test(T t);
}
```

`filter()`除了常用于数值外，也可应用于任何Java对象。例如，从一组给定的`LocalDate`中过滤掉工作日，以便得到休息日：

```java
public class Main {
    public static void main(String[] args) {
        Stream.generate(new LocalDateSupplier())
                .limit(31)
                .filter(ldt -> ldt.getDayOfWeek() == DayOfWeek.SATURDAY || ldt.getDayOfWeek() == DayOfWeek.SUNDAY)
                .forEach(System.out::println);
    }
}

class LocalDateSupplier implements Supplier<LocalDate> {
    LocalDate start = LocalDate.of(2020, 1, 1);
    int n = -1;
    public LocalDate get() {
        n++;
        return start.plusDays(n);
    }
}
```

## 使用 reduce

`map()`和`filter()`都是`Stream`的转换方法，而`Stream.reduce()`则是`Stream`的一个聚合方法，它可以把一个`Stream`的所有元素按照聚合函数聚合成一个结果。

```java
public class Main {
    public static void main(String[] args) {
        // 初始化结果为0
        int sum = Stream.of(1, 2, 3, 4, 5, 6, 7, 8, 9).reduce(0, (acc, n) -> acc + n);
        System.out.println(sum); // 45
    }
}
```

`reduce()`方法传入的对象是`BinaryOperator`接口，它定义了一个`apply()`方法，负责把上次累加的结果和本次的元素 进行运算，并返回累加的结果：

```
@FunctionalInterface
public interface BinaryOperator<T> {
    // Bi操作：两个输入，一个输出
    T apply(T t, T u);
}
```

除了可以对数值进行累积计算外，灵活运用`reduce()`也可以对Java对象进行操作。下面的代码演示了如何将配置文件的每一行配置通过`map()`和`reduce()`操作聚合成一个`Map<String, String>`：

```java
public class Main {
    public static void main(String[] args) {
        // 按行读取配置文件:
        List<String> props = List.of("profile=native", "debug=true", "logging=warn", "interval=500");
        Map<String, String> map = props.stream()
                // 把k=v转换为Map[k]=v:
                .map(kv -> {
                    String[] ss = kv.split("\\=", 2);
                    return Map.of(ss[0], ss[1]);
                })
                // 把所有Map聚合到一个Map:
                .reduce(new HashMap<String, String>(), (m, kv) -> {
                    m.putAll(kv);
                    return m;
                });
        // 打印结果:
        map.forEach((k, v) -> {
            System.out.println(k + " = " + v);
        });
    }
}
```

`reduce()`方法将一个`Stream`的每个元素依次作用于`BinaryOperator`，并将结果合并。

`reduce()`是聚合方法，聚合方法会立刻对`Stream`进行计算。

## 输出为集合

`reduce()`只是一种聚合操作，如果我们希望把`Stream`的元素保存到集合，例如`List`，因为`List`的元素是确定的Java对象，因此，把`Stream`变为`List`不是一个转换操作，而是一个聚合操作，它会强制`Stream`输出每个元素。

下面的代码演示了如何将一组`String`先过滤掉空字符串，然后把非空字符串保存到`List`中：

```java
public class Main {
    public static void main(String[] args) {
        Stream<String> stream = Stream.of("Apple", "", null, "Pear", "  ", "Orange");
        List<String> list = stream.filter(s -> s != null).collect(Collectors.toList());
        System.out.println(list);
    }
}
```

把`Stream`的每个元素收集到`List`的方法是调用`collect()`并传入`Collectors.toList()`对象，它实际上是一个`Collector`实例，通过类似`reduce()`的操作，把每个元素添加到一个收集器中（实际上是`ArrayList`）。

类似的，`collect(Collectors.toSet())`可以把`Stream`的每个元素收集到`Set`中。

### 输出为数组

把Stream的元素输出为数组和输出为List类似，我们只需要调用`toArray()`方法，并传入数组的“构造方法”：

```
List<String> list = List.of("Apple", "Banana", "Orange");
String[] array = list.stream().toArray(String[]::new);
```

注意到传入的“构造方法”是`String[]::new`，它的签名实际上是`IntFunction<String[]>`定义的`String[] apply(int)`，即传入`int`参数，获得`String[]`数组的返回值。

### 输出为Map

如果我们要把Stream的元素收集到Map中，就稍微麻烦一点。因为对于每个元素，添加到Map时需要key和value，因此，我们要指定两个映射函数，分别把元素映射为key和value：

```java
public class Main {
    public static void main(String[] args) {
        Stream<String> stream = Stream.of("APPL:Apple", "MSFT:Microsoft");
        Map<String, String> map = stream
                .collect(Collectors.toMap(
                        // 把元素s映射为key:
                        s -> s.substring(0, s.indexOf(':')),
                        // 把元素s映射为value:
                        s -> s.substring(s.indexOf(':') + 1)));
        System.out.println(map);
    }
}
```

## 其他操作

### 排序

对`Stream`的元素进行排序十分简单，只需调用`sorted()`方法：

此方法要求`Stream`的每个元素必须实现`Comparable`接口。如果要自定义排序，传入指定的`Comparator`即可：

```java
public class Main {
    public static void main(String[] args) {
        List<String> list = List.of("Orange", "apple", "Banana")
            .stream()
            .sorted()
            .collect(Collectors.toList());
        System.out.println(list);
    }
}
```

注意`sorted()`只是一个转换操作，它会返回一个新的`Stream`。

### 去重

对一个`Stream`的元素进行去重，没必要先转换为`Set`，可以直接用`distinct()`：

```
List.of("A", "B", "A", "C", "B", "D")
    .stream()
    .distinct()
    .collect(Collectors.toList()); // [A, B, C, D]
```

### 截取

截取操作常用于把一个无限的`Stream`转换成有限的`Stream`，`skip()`用于跳过当前`Stream`的前N个元素，`limit()`用于截取当前`Stream`最多前N个元素：

```
List.of("A", "B", "C", "D", "E", "F")
    .stream()
    .skip(2) // 跳过A, B
    .limit(3) // 截取C, D, E
    .collect(Collectors.toList()); // [C, D, E]
```

截取操作也是一个转换操作，将返回新的`Stream`。

### 合并

将两个`Stream`合并为一个`Stream`可以使用`Stream`的静态方法`concat()`：

```
Stream<String> s1 = List.of("A", "B", "C").stream();
Stream<String> s2 = List.of("D", "E").stream();
// 合并:
Stream<String> s = Stream.concat(s1, s2);
System.out.println(s.collect(Collectors.toList())); // [A, B, C, D, E]
```

### flatMap

如果`Stream`的元素是集合：

```
Stream<List<Integer>> s = Stream.of(
        Arrays.asList(1, 2, 3),
        Arrays.asList(4, 5, 6),
        Arrays.asList(7, 8, 9));
```

而我们希望把上述`Stream`转换为`Stream<Integer>`，就可以使用`flatMap()`：

```
Stream<Integer> i = s.flatMap(list -> list.stream());
```

因此，所谓`flatMap()`，是指把`Stream`的每个元素（这里是`List`）映射为`Stream`，然后合并成一个新的`Stream`：

```ascii
┌─────────────┬─────────────┬─────────────┐
│┌───┬───┬───┐│┌───┬───┬───┐│┌───┬───┬───┐│
││ 1 │ 2 │ 3 │││ 4 │ 5 │ 6 │││ 7 │ 8 │ 9 ││
│└───┴───┴───┘│└───┴───┴───┘│└───┴───┴───┘│
└─────────────┴─────────────┴─────────────┘
                     │
                     │flatMap(List -> Stream)
                     │
                     │
                     ▼
   ┌───┬───┬───┬───┬───┬───┬───┬───┬───┐
   │ 1 │ 2 │ 3 │ 4 │ 5 │ 6 │ 7 │ 8 │ 9 │
   └───┴───┴───┴───┴───┴───┴───┴───┴───┘
```

### 并行

通常情况下，对`Stream`的元素进行处理是单线程的，即一个一个元素进行处理。但是很多时候，我们希望可以并行处理`Stream`的元素，因为在元素数量非常大的情况，并行处理可以大大加快处理速度。

把一个普通`Stream`转换为可以并行处理的`Stream`非常简单，只需要用`parallel()`进行转换：

```
Stream<String> s = ...
String[] result = s.parallel() // 变成一个可以并行处理的Stream
                   .sorted() // 可以进行并行排序
                   .toArray(String[]::new);
```

经过`parallel()`转换后的`Stream`只要可能，就会对后续操作进行并行处理。我们不需要编写任何多线程代码就可以享受到并行处理带来的执行效率的提升。

### 其他聚合方法

除了`reduce()`和`collect()`外，`Stream`还有一些常用的聚合方法：

- `count()`：用于返回元素个数；
- `max(Comparator<? super T> cp)`：找出最大元素；
- `min(Comparator<? super T> cp)`：找出最小元素。

针对`IntStream`、`LongStream`和`DoubleStream`，还额外提供了以下聚合方法：

- `sum()`：对所有元素求和；
- `average()`：对所有元素求平均数。

还有一些方法，用来测试`Stream`的元素是否满足以下条件：

- `boolean allMatch(Predicate<? super T>)`：测试是否所有元素均满足测试条件；
- `boolean anyMatch(Predicate<? super T>)`：测试是否至少有一个元素满足测试条件。

最后一个常用的方法是`forEach()`，它可以循环处理`Stream`的每个元素，我们经常传入`System.out::println`来打印`Stream`的元素：

```
Stream<String> s = ...
s.forEach(str -> {
    System.out.println("Hello, " + str);
});
```

### 小结

`Stream`提供的常用操作有：

转换操作：`map()`，`filter()`，`sorted()`，`distinct()`；

合并操作：`concat()`，`flatMap()`；

并行处理：`parallel()`；

聚合操作：`reduce()`，`collect()`，`count()`，`max()`，`min()`，`sum()`，`average()`；

其他操作：`allMatch()`, `anyMatch()`, `forEach()`。

# 数据与List互转

```java
public void test5(){
    int[] array = {1, 2, 5, 5, 5, 5, 6, 6, 7, 2, 9, 2};

    /*int[]转list*/
    //方法一：需要导入apache commons-lang3  jar
    List<Integer> list = Arrays.asList(ArrayUtils.toObject(array));
    //方法二：java8及以上版本
    List<Integer> list1 = Arrays.stream(array).boxed().collect(Collectors.toList());

    /*list转int[]*/
    //方法一：
    Integer[] intArr =  list.toArray(new Integer[list.size()]);
    //方法二：java8及以上版本
    int[] intArr1 =  list.stream().mapToInt(Integer::valueOf).toArray();
}
```

mapToInt 将一个 Stream 转换为 IntStream，也可以ToDouble，ToLong等。
