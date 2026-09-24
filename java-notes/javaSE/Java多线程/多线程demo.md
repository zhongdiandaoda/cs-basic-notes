# 交替打印奇数偶数

继承Thread的方式

```java
package com.lq.leetcode;


/**
 * @author liuqi
 */
public class Main {
    public static void main(String[] args) {
        Object obj = new Object();
        Odd odd = new Odd(obj);
        Even even = new Even(obj);
        odd.setName("奇数线程");
        even.setName("偶数线程");
        even.start();
        odd.start();
    }
}
class Odd extends Thread {
    int base = 1;
    private Object obj;
    public Odd(Object obj) {
        this.obj = obj;
    }
    @Override
    public void run() {
        while(base <= 100) {
            synchronized (obj) {
                System.out.println(currentThread().getName() + ",value: " + base);
                base += 2;
                obj.notify();
                if(base < 100) {
                    try {
                        obj.wait();
                    }catch (Exception e) {
                        e.printStackTrace();
                    }
                }
            }
        }
    }
}
class Even extends Thread {
    int base = 0;
    private Object obj;
    public Even(Object obj) {
        this.obj = obj;
    }
    @Override
    public void run() {
        while(base <= 100) {
            synchronized (obj) {
                System.out.println(currentThread().getName() + ",value: " + base);
                base += 2;
                obj.notify();
                if(base < 100) {
                    try {
                        obj.wait();
                    }catch (Exception e) {
                        e.printStackTrace();
                    }
                }
            }
        }
    }
}
```

实现Runnable的方式

```
package com.lq.leetcode;


/**
 * @author liuqi
 */
public class Main {
    public static void main(String[] args) {
        new Thread(new Printer(), "奇数线程").start();
        new Thread(new Printer(), "偶数线程").start();
    }
}
class Printer implements Runnable {
    private static int count = 0;
    private static Object obj = new Object();
    private int limit = 100;

    @Override
    public void run() {
        while(count <= limit) {
            synchronized (obj) {
                System.out.println(Thread.currentThread().getName() + "打印：" + count++);
                obj.notify();
                if (count < limit) {
                    try {
                        obj.wait();
                    }catch (Exception e) {
                        e.printStackTrace();
                    }
                }
            }
        }
    }
}
```
