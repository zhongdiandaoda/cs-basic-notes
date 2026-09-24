# Date类

Date类从JDK1.0就存在了，目前存在很多问题。

目前没有被弃用的构造器如下：

- `Date()`：生成一个代表当前日期的Date对象。
- `Date(long date)`：date为自GMT1970年1月1日00:00:00到date经历的毫秒数。

没有被弃用的方法包括：

- `boolean after(Date date)`
- `boolean before(Date date)`
- `long getTime()`：返回以毫秒为单位与GMT1970年1月1日00:00:00的时间差。
- `long setTime(long date)`

Java官方推荐尽量不要使用Date类。

# Calendar类

Calendar类是一个抽象类，用于表示日历。Calendar类本身不能实例化，Java提供了GregorianCalendar类，一个代表公历的子类。

Calendar提供了几个静态getInstance方法来获取Calendar对象，这些方法根据TimeZone、Locale来获取特定的Calendar，如果不指定，则使用默认的TimeZone和locale。

示例：

```java
public class Test {
    public static void main(String[] args) {
        Calendar calendar = Calendar.getInstance();
        System.out.println(calendar.getTime());
        Date date = calendar.getTime();
        Calendar calendar1 = calendar.getInstance();
        calendar1.setTime(date);
        System.out.println(calendar1.getTime());
    }
}
```

输出：

```
Thu Dec 17 10:43:57 CST 2020
Thu Dec 17 10:43:57 CST 2020

Process finished with exit code 0
```

Calendar类提供了大量访问、修改日期时间的方法：

- `void add(int field, int amount)`：**根据日历规则**，为field字段加上或减去指定的时间量。

- `int get(int field)`
- `int getActualMaximum(int field)`：返回日历字段可能拥有的最大值，月为11（从0开始）。
- `int getActualMinimum(int field)`

- `void roll(int field, int amount)`：与add方法类似，只是字段超过最大范围时不会进位。
- `void set(int field, int value)`
- `void set(int year, int month, int day, int hourOfDay, int minute, int second)`
- `void set(int year, int month, int day)`

field是一个Calendar的类变量，例如`Calendar.YEAR`、`Calendar.DAY_OF_MONTH`等，Calendar中月是从0开始的，而不是1。

示例：

```java
public class Test {
    public static void main(String[] args) {
        Calendar calendar = Calendar.getInstance();
        System.out.println(calendar.get(Calendar.YEAR));
        System.out.println(calendar.get(Calendar.MONTH));
        System.out.println(calendar.get(Calendar.DAY_OF_MONTH));
        calendar.set(2018, 10, 1, 5, 59, 54);
        calendar.roll(Calendar.YEAR, -5);
        calendar.add(Calendar.DAY_OF_MONTH, -3);
        System.out.println(calendar.getTime());
    }
}
```

Calendar类的set方法传入一个不合法的参数时，会自动进位，但是将lenient方法设置为false时，它将执行严格的参数检查，在接收到不合法参数时抛出异常。

```java
public class Test {
    public static void main(String[] args) {
        Calendar calendar = Calendar.getInstance();
        calendar.set(Calendar.MONTH, 14);
        System.out.println(calendar.getTime());
        calendar.setLenient(false);
        //引发异常
        calendar.set(Calendar.MINUTE, 66);
        System.out.println(calendar.getTime());
    }
}
```

```
Wed Mar 17 12:17:18 CST 2021
Exception in thread "main" java.lang.IllegalArgumentException: MINUTE
	at java.util.GregorianCalendar.computeTime(GregorianCalendar.java:2648)
	at java.util.Calendar.updateTime(Calendar.java:3395)
	at java.util.Calendar.getTimeInMillis(Calendar.java:1782)
	at java.util.Calendar.getTime(Calendar.java:1755)
	at com.lq.crazyJava.Test.main(Test.java:17)

Process finished with exit code 1
```

**set()方法的延迟修改：**

`set(field, value)`方法将日历字段field修改为value，其对应的Calendar对象不会立即修改，直到下次调用`get(),getTime(),add(),roll(),getTimeMillis()`才会重新计算日历的时间，称为`set()`方法的延迟修改。

```java
public class Test {
    public static void main(String[] args) {
        Calendar calendar = Calendar.getInstance();
        calendar.set(2018, 7, 31);
        //修改为9月，9月没有31日，系统会将calendar调整为10月1日，但由于延迟修改机制，目前系统还没有调整
        calendar.set(Calendar.MONTH, 8);
        calendar.set(Calendar.DAY_OF_MONTH, 5);
        //output: Wed Sep 05 12:27:01 CST 2018
        System.out.println(calendar.getTime());
    }
}
```

# Java8新增的日期、时间包

Java8新增了一个java.Time包，包括以下常用的类：
