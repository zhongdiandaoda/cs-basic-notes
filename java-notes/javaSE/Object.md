# Object

Object时所有类、数组的父类，java允许把任何类型的对象赋给Object类型的变量。

Object类包含以下几个常用方法：

- `boolean equals(Object obj)`：判断指定对象与该对象是否相等，即两个对象是否是同一个对象。
- `Class<?> getClass()`：返回该对象的运行时类。
- `protected void finalize()`：当系统中没有引用变量引用到该对象时，垃圾回收器会调用该方法。
- `int hashCode()`：返回该对象的`hashCode`值。在默认情况下，Object类的`hashCode()`方法根据该对象的地址来计算（即与`System.identityHashCode(Object x)`方法的计算结果相同）。但很多类都重写了`hashCode`方法。
- `String toString()`

Object类还提供了`wait()`、`notify()`、`notifyAll()`几个方法控制线程的暂停与运行。

Object类提供了protected修饰的clone方法，只能由子类重写或调用。

自定义类实现克隆的步骤如下：

1. 自定义类实现Cloneable接口。
2. 自定义类实现clone方法。
3. 调用Object类的clone方法得到对象的副本并返回该副本。

示例：

```java
class Address {
    String detail;
    public Address(String detail) {
        this.detail = detail;
    }
}
public class User implements Cloneable {
    int age;
    Address address;
    public User(int age) {
        this.age = age;
        address = new Address("moon");
    }
    public User clone() throws CloneNotSupportedException {
        return (User)super.clone();
    }

    public static void main(String[] args) throws CloneNotSupportedException {
        User u = new User(100);
        User uu = u.clone();
        //输出false
        System.out.println(u == uu);
        //输出true
        System.out.println(u.address == uu.address);
    }
}
```

Object类提供的clone方法只是对对象里的实例变量进行简单复制，如果实例变量为引用类型，会复制一个引用指向相同的对象。

# Objects

Objects是Java7新增的工具类，提供了一些方法来操作对象，这些方法大部分是空指针安全的。例如，一个引用变量为null时，调用`toString()`方法会抛出`NullPointerException`，但使用Objects类的`toString`方法时会返回null。

示例：

```java
public class ObjectsTest {
    static ObjectsTest objectsTest;
    public static void main(String[] args) {
        System.out.println(Objects.hashCode(objectsTest));
        System.out.println(Objects.toString(objectsTest));
        //当对象不为null时返回对象本身，否则抛出NullPointerException并打印提示信息
        System.out.println(Objects.requireNonNull(objectsTest, "对象为null"));
    }
}
```
