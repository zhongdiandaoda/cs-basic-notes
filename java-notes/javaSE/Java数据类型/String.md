# `String`

String是不可变类是典型的 Immutable 类，被声明成为 final class，所有属性也都是 final 的。也由于它的不可变性，类似拼接、裁剪字符串等动作，都会产生新的 String 对象。由于字符串操作的普遍性，所以相关操作的效率往往对应用性能有明显影响。

`StringBuffer` 是为解决上面提到拼接产生太多中间对象的问题而提供的一个类，我们可以用 append 或者 add 方法，把字符串添加到已有序列的末尾或者指定位置。`StringBuffer` 本质是一个线程安全的可修改字符序列，它保证了线程安全，也随之带来了额外的性能开销，所以除非有线程安全的需要，不然还是推荐使用它的后继者，也就是 `StringBuilder`。

`StringBuilder` 是 Java 1.5 中新增的，在能力上和 `StringBuffer` 没有本质区别，但是它去掉了线程安全的部分，有效减小了开销，是绝大部分情况下进行字符串拼接的首选。

String类的构造方法：

- `String()`
- `String(byte[] bytes, Charset charset)`
- `String(byte[] bytes, String charsetName)`
- `String(byte[] bytes, int offset, int length)`
- `String(byte[] bytes, int offset, int length, String charsetName)`
- `String(char[] value)`
- `String(char[] value, int offset, int count)`
- `String(StringBuffer buffer)`
- `String(StringBuilder builder)`

常用方法：

- `char charAt(int index)`
- `int compareTo(String anotherString)`：根据字典序比较两个字符串。
- `String concat(String str)`：与字符串拼接符'+'相同。
- `boolean contentEquals(StringBuffer sb)`：比较`String`与`StringBuffer`。
- `static String copyValueOf(char[] data)`：将字符数组转换成字符串并返回。
- `static String copyValueOf(char[] data, int offset, int count)`
- `boolean endsWith(String suffix)`：判断该`String`对象是否以`suffix`结尾。
- `boolean equals(Object anObject)`：比较两个字符串，字符序列相同时返回true。
- `boolean equalsIgnoreCase(String str)`：忽略大小写比较字符串。
- `byte[] getBytes()`：将String转为byte数组。
- `void getChars(int srcBegin, int srcEnd, char[] dst, int dstBegin)`
- `int indexOf(int ch)`
- `int indexOf(int ch, int fromIndex)`：找出`fromIndex`后`ch`的第一次出现位置。
- `int indexOf(String str)`
- `int indexOf(String str, int fromIndex)`
- `int lastIndexOf(int ch)`
- `int lastIndexOf(int ch, int fromIndex)`
- `int lastIndexOf(String str)`
- `int lastIndexOf(String str, int fromIndex)`
- `int length()`
- `String replace(char oldChar, char newChar)`：将字符串第一个`oldChar`替换为`newChar`。
- `String replaceFirst(String regex, String replacement)`：替换匹配的第一个子串。
- `String replaceAll(String  regex, String replacement)`
- `boolean startsWith(String prefix)`
- `boolean startsWith(String prefix, int offset)`：offset位置开始是否以prefix开头。（左闭）
- `String substring(int beginIndex)`
- `String substring(int beginIndex, int endIndex)`
- `char[] toCharArray()`
- `String toLowerCase()`
- `String toUpperCase()`
- `static String valueOf(X x)`：X为基本类型。
- `String[] split(String regex, int limit)`
- `String trim()`：删除字符串的头尾空白符，包括`\t,\r,\n`。

String类是不可变的，因此会产生很多临时变量。

Join方法：

```java
String[] names = {"Bob", "Alice", "Grace"};
String s = String.join(", ", names);
```

# `StringBuffer`

`StringBuffer`对象代表一个字符序列可变的字符串。`StringBuffer`对象创建后可通过`append`、`insert`、`reverse`、`setCharAt`、`setLength`等方法改变字符序列，最后调用`toString`方法转换回字符串。

并发安全性由 synchronized 方法保证。

`StringBuffer`的默认长度是初始字符串长度+16。

`StringBuffer`的方法：

- `StringBuffer append(char[]/String/char/int/boolean/long/...)`
- `StringBuffer delete(int start, int end)`：左闭右开
- `StringBuffer deleteCharAt(int index)`
- `StringBuffer replace(int start, int end, String str)`
- `StringBuffer reverse()`
- `StringBuffer insert(int offset, obj o)`：append 方法等价于 `sb.insert(sb.length(), x)`。
- `String substring(int start, int end)`：左闭右开
- `int indexOf(String str)`：如果包含子串，返回子串的开始位置
- `int indexOf(String str, int fromIndex)`
- `int lastIndexOf(String str)`
- `int lastIndexOf(String str, int fromIndex)`

# `StringBuilder`

`StringBuilder`时Jdk1.5新增的类，类似`StringBuffer`。但`StringBuffer`是线程安全的，而`StringBuilder`没有实现线程安全功能，因此性能略强。通常情况下，优先考虑使用`StringBuilder`类。

# String的基本特性

- String的创建方式：
  - `String s1 = "abc";	//字面量的定义方式，“abc”将被存储在常量池`
  - `String s2 = new String("efg"); //如果常量池没有"efg",现在常量池创建"eft"，然后在堆中创建String对象，其中value数组引用常量池中的value数组；否则，堆中直接创建String对象，value指向常量池中的value数组`

- String的权限是final，不可以被继承。
- String实现了Serializable接口，支持序列化。
- String实现了Comparable接口，可以按照字典序排序。
- 在JDK8及以前，String内部定义了final char[] value数组用于存储数据，JDK9时改为byte[]数组。
- 使用new关键字创建字符串时，由于内容在编译期已确定，编译器会进行代码优化，在堆中创建对象后，还会在常量池中创建该对象，然后返回堆中的地址引用。

# 字符串拼接

- 常量和常量的拼接结果会放在常量池，原理是编译期优化。
- 对于final字段，在编译时会进行常量替换。
- 只要一个字符串是变量，就会调用`StringBuilder`的`append`方法在堆中创建对象。
- 常量池中相同的字符串是唯一的，堆中可以存在多个相同的字符串。

demo:

```java
String x = "p" + 2 + "p";

//编译后：
String x = new StringBuilder(String.valueOf("p")).append(2).append("p").toString();
```

`StringBuilder`的`toString`方法：

```java
@Override
public String toString() {
    // Create a copy, don't share the array
    return new String(value, 0, count);
}
```

demo：

```java
@Test
public void test1(){
    String s1 = "a" + "b" + "c";//编译期优化，没有创建StringBuilder进行拼接，等同于"abc"
    String s2 = "abc"; //"abc"一定是放在字符串常量池中，将此地址赋给s2
    /*
         * 最终.java编译成.class,再执行.class
         * String s1 = "abc";
         * String s2 = "abc"
         */
    System.out.println(s1 == s2); //true
    System.out.println(s1.equals(s2)); //true
}

@Test
public void test2(){
    String s1 = "javaEE";
    String s2 = "hadoop";

    String s3 = "javaEEhadoop";
    String s4 = "javaEE" + "hadoop";//编译期优化
    //如果拼接符号的前后出现了变量，则相当于在堆空间中new String()，具体的内容为拼接的结果：javaEEhadoop
    String s5 = s1 + "hadoop";
    String s6 = "javaEE" + s2;
    String s7 = s1 + s2; //调用了new String方法

    System.out.println(s3 == s4);//true
    System.out.println(s3 == s5);//false
    System.out.println(s3 == s6);//false

    System.out.println(s3 == s7);//false

    System.out.println(s5 == s6);//false
    System.out.println(s5 == s7);//false
    System.out.println(s6 == s7);//false
    //intern():判断字符串常量池中是否存在javaEEhadoop值，如果存在，则返回常量池中javaEEhadoop的地址；
    String s8 = s6.intern();
    System.out.println(s3 == s8);//true
}
```

# intern方法

根据统计，把常见应用进行堆转储（Dump Heap），然后分析对象组成，会发现平均 25% 的对象是字符串，并且其中约半数是重复的。如果能避免创建重复字符串，可以有效降低内存消耗和对象创建开销。

对于位于堆中的字符串，使用intern方法后，会首先在常量池查询在字符串是否存在，如果存在，返回其常量池中的索引。

否则，JDK1.6以前，会在常量池中创建该字符串，并返回其索引（存在的问题：被缓存的字符串被存在永久代，很少被垃圾回收，很容易导致 OOM）；JDK7及之后，会将堆中字符串对象的索引复制到常量池之中。

在后续版本中，这个缓存被放置在堆中，这样就极大避免了永久代占满的问题，甚至永久代在 JDK 8 中被 `MetaSpace`（元数据区）替代了。而且，默认缓存大小也在不断地扩大中，从最初的 1009，到 7u40 以后被修改为 60013。你可以使用下面的参数直接打印具体数字，可以拿自己的 JDK 立刻试验一下。

```
-XX:+PrintStringTableStatistics
```

你也可以使用下面的 JVM 参数手动调整大小，但是绝大部分情况下并不需要调整，除非你确定它的大小已经影响了操作效率。

```
-XX:StringTableSize=N
```

Intern 是一种**显式的排重机制**，但是它也有一定的副作用，因为需要开发者写代码时明确调用，一是不方便，每一个都显式调用是非常麻烦的；另外就是我们很难保证效率，应用开发阶段很难清楚地预计字符串的重复情况，有人认为这是一种污染代码的实践。

幸好在 Oracle JDK 8u20 之后，推出了一个新的特性，也就是 G1 GC 下的字符串排重。它是通过将相同数据的字符串指向同一份数据来做到的，是 JVM 底层的改变，并不需要 Java 类库做什么修改。

注意这个功能目前是默认关闭的，你需要使用下面参数开启，并且记得指定使用 G1 GC：

```
-XX:+UseStringDeduplication
```

demo1：

```java
String str2 = new String("str") + new String("01"); //常量池中存在"str"和"01"，不存在"str01"
str2.intern();	//将堆中"str01"的索引复制到常量池
String str1 = "str01"; //str1指向堆中的字符串对象
System.out.println(str2==str1); //true
//在JDK1.6之前，由于会把字符串复制到常量池，因此返回false
```

将中间两行顺序调换后：

```java
String str2 = new String("str")+new String("01");
String str1 = "str01"; //堆中没有“str01”，因此在堆中新建该字符串并返回其索引
str2.intern();	//什么都不做
System.out.println(str2==str1); //false
```

demo2：

```java
public static void main(String[] args) {
    String x = new String("state");
    x.intern();
    String y = "state";
    System.out.println(x == y);  //false
}
//new String("state")编译后，会将“state”字符串往常量池也添加一份，因此x.intern()什么都没做
//因此y指向常量池的对象，x指向堆中的对象
```

# new String创建几个对象

`String str = new String(“xyz”)`在运行时调用的构造方法是`String(String original)`

因此，Javac在编译时，上述过程可等价于：

```java
String tmp = "xyz"; //常量池中不存在"xyz"时，创建"xyz"字符串，否则直接返回其地址
str = new String(tmp);//在堆中新建对象
```

因此，new String可能会创建1个或2个对象。

使用StringBuilder的toString()方法新建字符串时，利用的是以char数组作为参数的String的构造方法，因此不会向常量池中添加字符串。

# **创建字符串分析**

1 直接使用双引号创建字符串

判断这个常量是否存在于常量池
  如果存在
       判断这个常量是堆中对象的引用还是常量
            如果是引用，返回引用地址指向的堆空间对象，
            如果是常量，则直接返回常量池常量，
  如果不存在
            在常量池中创建该常量，并返回此常量

2 使用new String创建字符串

在堆上创建对象(无论堆上是否存在相同字面量的对象)
判断常量池中是否存在该字符串
  如果不存在
       在常量池上创建常量
  如果存在
       不做任何操作
