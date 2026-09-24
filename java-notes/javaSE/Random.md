Random类用于生成一个伪随机数，它有两个构造器：一个构造器使用默认的种子（当前时间），另一个构造器需要显式传入一个long型整数的种子。

`ThreadLocalRandom`类是Java7新增的一个类，是Random的增强版。在并发访问的环境下，使用`ThreadLocalRandom`对象来代替Random可以减少多线程资源竞争，最终保证系统具有更好的线程安全性。

# Random

示例：

```java
public class Test {
    public static void main(String[] args) throws Exception {
        Random random = new Random();
        System.out.println("random.nextBoolean=" + random.nextBoolean());
        byte[] buffer = new byte[16];
        random.nextBytes(buffer);
        System.out.println(Arrays.toString(buffer));
        System.out.println("0.0-1.0的随机小数：" + random.nextDouble());
        //生成均值为0，标准差为1的伪高斯数
        System.out.println(random.nextGaussian());
        //生成0-99范围内的伪随机整数
        System.out.println(random.nextInt(100));
        //生成伪随机整数
        System.out.println(random.nextInt());
    }
}
```

Random类会使用一个48位的种子，当两个Random对象种子相同时，它们会产生相同的随机数序列。

# `ThreadLocalRandom`

多线程环境下，`ThreadLocalRandom`的使用方法为：

```java
public class Test {
    public static void main(String[] args) throws Exception {
        ThreadLocalRandom threadLocalRandom = ThreadLocalRandom.current();
        System.out.println(threadLocalRandom.nextInt(20));
        System.out.println(threadLocalRandom.nextBoolean());
    }
}
```
