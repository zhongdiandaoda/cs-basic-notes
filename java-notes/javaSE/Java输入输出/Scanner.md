# args

使用命令行执行java程序时可以为main函数提供参数。

示例：

```java
public class ArgTest {
    public static void main(String[] args) {
        System.out.println(args.length);
        for(String str : args) {
            System.out.println(str);
        }
    }
}
```

`java ArgTest who are you`

字符串中含有空格时需要用双引号括起来。

`java ArgTest "this is a line."`

# 使用Scanner获取输入

Scanner是一个基于正则表达式的处理流，可以将不同的流作为数据源。

Scanner主要提供了两个方法：

- `hasNextXxx()`：是否还有下一个输入项，Xxx可以是int，long等基本类型的字符串。如果判断是否还有下一个字符串，则直接使用`hasNext()`。
- `nextXxx()`：获取下一个输入项。

默认情况下，Scanner使用空白（空格、Tab、回车）作为多个输入的分隔符。

可以使用`useDelimiter(String pattern)`方法为`Scanner`设置自定义的分隔符，pattern是一个正则表达式。

Scanner提供了两个方法来处理行：

- `hasNextLine()`
- `nextLine()`

向Scanner传入一个文件对象时，Scanner就可以从文件中读取数据。

```java
public class ScannerTest {
    public static void main(String[] args) throws Exception {
        Scanner scanner = new Scanner(new File("a.ini"));
        while (scanner.hasNextLine())
            System.out.println(scanner.nextLine());
    }
}
/*
output:
#comment line
#Mon Nov 09 14:27:32 CST 2020
user=root
password=123

Process finished with exit code 0
*/
```
