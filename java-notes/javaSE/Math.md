```java
public static void main(String[] args) {
    /**
     * 常量
     */
    System.out.println("------>" + Math.E);//2.718281828459045
    System.out.println("------>" + Math.PI);//3.141592653589793

    /**
     * Math.abs()计算绝对值
     */
    System.out.println("------>" + Math.abs(-3));//3

    /**
     * 三角函数与反三角函数
     * cos求余弦
     * sin求正弦
     * tan求正切
     * acos求反余弦
     * asin求反正弦
     * atan求反正切
     * atan2(y,x)求向量(x,y)与x轴夹角
     * cosh计算值的双曲余弦
     * sinh计算双曲正弦
     * tanh计算双曲正切
     */
    System.out.println("------>" + Math.acos(1));
    System.out.println("------>" + Math.acos(-1));

    /**
     * Math.cbrt()开立方根
     */
    System.out.println("------>" + Math.cbrt(-1));//-1.0
    System.out.println("------>" + Math.cbrt(1));//1.0
    System.out.println("------>" + Math.cbrt(0.5));//0.7937005259840998
    System.out.println("------>" + Math.cbrt(5));//1.709975946676697

    /**
     * Math.sqrt(x)开平方
     */
    System.out.println("------>" + Math.sqrt(4.0));//2.0

    /**
     * Math.hypot(x,y)求sqrt(x*x+y*y)在求两点间距离时有用sqrt((x1-x2)^2+(y1-y2)^2)
     */
    System.out.println("------>" + Math.hypot(3.0, 4.0));//5.0

    /**
     * ceil天花板，向上取整，返回大的值
     */
    System.out.println("------1>" + Math.ceil(7.2));//8.0
    System.out.println("------2>" + Math.ceil(7.5));//8.0
    System.out.println("------3>" + Math.ceil(7.6));//8.0
    System.out.println("------4>" + Math.ceil(-7.2));//-7.0
    System.out.println("------5>" + Math.ceil(-7.5));//-7.0
    System.out.println("------6>" + Math.ceil(-7.6));//-7.0

    /**
     * floor地板，向下取整，返回小的值
     */
    System.out.println("------1>" + Math.floor(7.2));//7.0
    System.out.println("------2>" + Math.floor(7.5));//7.0
    System.out.println("------3>" + Math.floor(7.6));//7.0
    System.out.println("------4>" + Math.floor(-7.2));//-8.0
    System.out.println("------5>" + Math.floor(-7.5));//-8.0
    System.out.println("------6>" + Math.floor(-7.6));//-8.0

    /**
     * Math.cosh()返回 double 值的双曲线余弦。x 的双曲线余弦的定义是 (ex + e-x)/2，其中 e 是欧拉数
     */
    System.out.println("------>" + Math.cosh(1));//1.543080634815244
    System.out.println("------>" + Math.cosh(0));//1.0

    /**
     * exp(x) 返回e^x的值
     * expm1(x) 返回e^x - 1的值
     * pow(x,y) 返回x^y的值
     * 这里可用的数据类型也只有double型
     */
    System.out.println("------>" + Math.exp(2));//7.38905609893065
    System.out.println("------>" + Math.expm1(2));//6.38905609893065
    System.out.println("------>" + Math.pow(2.0, 3.0));//8.0


    /**
     * 对数
     * Math.log(a) a的自然对数(底数是e)
     * Math.log10(a) a 的底数为10的对数
     * Math.log1p(a) a+1的自然对数
     * 值得注意的是，前面其他函数都有重载，对数运算的函数只能传double型数据并返回double型数据
     */
    System.out.println("------1>" + Math.log(Math.E));//1.0
    System.out.println("------2>" + Math.log10(10));//1.0
    System.out.println("------3>" + Math.log1p(Math.E - 1.0));//1.0

    /**
     * Math.max()求最大值
     * Math.min()求最小值
     */
    System.out.println("------1>" + Math.max(1, 2));//2
    System.out.println("------2>" + Math.min(1, -2));//-2


    /**
     * Math.nextAfter()返回与第二个参数方向相邻的第一个参数的浮点数。
     */
    System.out.println("------1>" + Math.nextAfter(-1, 2));//-0.99999994
    System.out.println("------2>" + Math.nextAfter(1, 2));//1.0000001

    /**
     * Math.nextUp()返回与正无穷大方向相邻的 d的浮点值。
     */
    System.out.println("------>" + Math.nextUp(1));//1.0000001
    System.out.println("------>" + Math.nextUp(-1));//-0.99999994

    /**
     * Math.Random()函数能够返回带正号的double值，该值大于等于0.0且小于1.0，即取值范围是[0.0,1.0)的左闭右开区间，
     * 返回值是一个伪随机选择的数，在该范围内（近似）均匀分布
     */
    System.out.println("------>" + Math.random());//取值范围是[0.0,1.0)的随机数

    /**
     * Math.rint(x)：x取整为它最接近的整数，如果x与两个整数的距离相等，则返回其中为偶数的那一个。
     */
    System.out.println("------>" + Math.rint(3.5));//4.0
    System.out.println("------>" + Math.rint(4.5));//4.0

    System.out.println("------>" + Math.rint(3.1));//3.0
    System.out.println("------>" + Math.rint(4.1));//4.0

    System.out.println("------>" + Math.rint(3.7));//4.0
    System.out.println("------>" + Math.rint(4.7));//5.0
    /**
     * Math.round(x)：返回Math.floor(x+0.5)，即“四舍五入”值。
     */
    System.out.println("------>" + Math.round(3.5));//4
    System.out.println("------>" + Math.round(4.5));//5

    System.out.println("------>" + Math.round(3.1));//3
    System.out.println("------>" + Math.round(4.7));//5


    /**
     * Math.scalb(double d, int scaleFactor),返回 f × 2scaleFactor，其舍入方式如同将一个正确舍入的浮点值乘以 float
     * 值集合中的一个值
     * 返回 d 和正无穷大之间与 d 相邻的浮点值 tatic double   nextUp(double d)
     */
    System.out.println("------1>" + Math.scalb(1.5d, 6));//96

    /**
     * Math.scalb(float f, int scaleFactor)
     * 返回 f 和正无穷大之间与 f 相邻的浮点值 tatic float    nextUp(float f)
     * 返回第一个参数和第二个参数之间与第一个参数相邻的浮点数
     */
    System.out.println("------2>" + Math.scalb(1.5f, 6));//96

    /**
     * Math.signum方法返回指定int值的符号函数（如果指定值为负，则返回-1；如果指定值为零，则返回 0；如果指定值为正，则返回1）。
     */
    System.out.println("------1>" + Math.signum(10));//1.0
    System.out.println("------2>" + Math.signum(-10));//-1.0
    System.out.println("------3>" + Math.signum(0));//0.0

    /**
     * Math.toDegrees()将弧度转换角度
     */
    System.out.println("------3>" + Math.toDegrees(1.57));//89.95437383553926

    /**
     * Math.toRadians()将角度转换为弧度
     */
    System.out.println("------3>" + Math.toRadians(90));//1.5707963267948966
}
```
