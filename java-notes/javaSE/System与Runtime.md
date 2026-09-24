# System类

System类代表当前Java的运行平台，它提供了一些类变量和类方法，允许程序通过System类调用这些类方法，访问类变量。

> System类提供了加载文件和动态链接库的方法，主要用于native方法。对于一些特殊的功能，必须借助C语言来实现，实现步骤为：
>
> 1. Java程序中声明native修饰的方法，类似abstract方法，只有方法签名，没有实现。编译该程序生成.class文件。
> 2. 用`javah`编译第一步生成的class文件，并产生一个`.h`文件。
> 3. 写一个`.cpp`文件实现native方法，并include第二步生成的`.h`文件。
> 4. 将`cpp`文件编译成动态链接库文件。
> 5. 在Java中用System类的`loadLibrary`方法或Runtime类的`loadLibrary`方法加载动态链接库文件。

示例：获取系统环境变量

```java
public class SystemTest {
    public static void main(String[] args) {
        System.out.println(System.getenv("TMP"));
        Map<String, String> map = System.getenv();
        map.forEach((k, v) -> System.out.println(k + ": " + v));
    }
}
```

获取系统属性：

```java
public class SystemTest {
    public static void main(String[] args) {
        Properties prop = System.getProperties();
        prop.forEach((k, v) -> System.out.println(k+":"+v));
        System.out.println(System.getProperty("os.name"));
    }
}
```

System类下有两个获取当前时间的方法：`currentTimeMillis()`和`nanoTime()`，它们都返回一个long型整数，与1970年1月1日0点的时间差，前者以`ms`作为单位，后者以`ns`作为单位。当操作系统不支持时，这两个方法将不能返回精确的结果。

System类提供了in、out、err三种流代表标准输入、标准输出与错误输出流，并提供了`setIn`、`setOut`和`setErr`来重定位流。

System类提供了`identityHashCode(Object x)`方法根据对象的地址计算出其精确的`hashCode`值。即使类的`hashCode`方法被重写，其示例的`hashCode`方法不能唯一标识一个对象，通过`identityHashCode`也能确定两个对象是否地址相同。

# Runtime类

Runtime类代表Java的运行时环境，每个Java程序都有一个与之对应的Runtime实例，应用程序通过该对象与其运行时环境相连。应用程序不能创建自己的Runtime实例，但可以通过getRuntime方法获取与之关联的Runtime对象。

与System类似，Runtime类也提供了gc()方法和runFinalization()方法来通知系统进行垃圾回收、清理系统资源，并提供了load(String filename)和loadLibrary(String libname)方法来加载文件和动态链接库。

Runtime类可以访问JVM的相关信息，如处理器数量、内存信息等。Runtime还可以单独启动一个进程给来运行操作系统的命令。

示例：

```java
public class Test {
    public static void main(String[] args) throws Exception {
        Runtime rt = Runtime.getRuntime();
        System.out.println(rt.availableProcessors());
        System.out.println(rt.freeMemory());
        System.out.println(rt.totalMemory());
        System.out.println(rt.maxMemory());
        rt.exec("notepad.exe");
    }
}
```
