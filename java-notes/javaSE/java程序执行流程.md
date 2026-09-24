# 标签
标签是后边带有冒号的标识符，eg：

`label1:`

在JAVA中，break和continue只能作用于最近的一层循环。通过将标签添加到break和continue之后，可结束多层循环。

```java
label1:
outer_iteration {
    inner_iteration {
        //...
        break;
        //...
        continue;
        //...
        continue label1;
        //...
        break label1;
    }
}
```

# Switch
- switch的每个case都以break结束
- 如果没有break，会顺序执行之后的所有case中的代码，即使不满足case的条件。
- switch表达式expression表达式的数据类型仅支持byte、short、char、int和java7新增的String。

# DoWhile

dowhile和while的区别是，dowhile在至少执行一次后才判断是否符合循环条件。

(do while的最后必须有一个分号)

```java
do {

}while();
```
