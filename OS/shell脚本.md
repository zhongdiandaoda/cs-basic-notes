

# overview

编写第一个shell脚本：Hello World

```shell
#!/bin/bash
# This is our first script
echo 'Hello World'
```

每一个shell脚本都需要把`#!/bin/bash`放在第一行，来告诉操作系统执行此脚本需要的解释器的名字。

为了使得脚本可执行，需要修改权限：

```shell
[root@m1node3 tmp]# ls -ll | grep hello
-r-xr-xr-x 1 root root 31 Mar  2 10:33 hello.sh
[root@m1node3 tmp]# chmod 644 hello.sh 
[root@m1node3 tmp]# ls -ll | grep hello
-rw-r--r-- 1 root root 31 Mar  2 10:33 hello.sh
[root@m1node3 tmp]# chmod 744 hello.sh 
[root@m1node3 tmp]# ls -ll | grep hello
-rwxr--r-- 1 root root 31 Mar  2 10:33 hello.sh
[root@m1node3 tmp]# ./hello.sh 
hello world
```

当没有给出可执行程序的明确路径名时，系统会在PATH环境变量中搜索置否存在该可执行程序。

```shell
[root@m1node3 tmp]# echo $PATH
/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/root/bin
```

∼/bin 目录存放个人使用的脚本。

如果我们编写了一个脚本，系统中的每个用户都可以使用它，那么这个脚本的位置应该是/usr/local/bin。

系统管理员使用的脚本经常放到/usr/local/sbin 目录下。

大多数情况下，本地支持的软件，不管是脚本还是编译过的程序，都应该放到/usr/local 目录下，而不是在/bin 或/usr/bin 目录下。这些目录都是由 Linux 文件系统层次结构标准指定，只包含由 Linux 发行商所提供和维护的文件。

方法二：在提示符中用绝对或相对文件路径来引用 shell 脚本文件。

```bash
$ ./test.sh
```



# 行继续符

通过使用行继续符\，可以将复杂的命令写成若干行，提高可读性。

![image-20210302104553825](/Users/bytedance/Documents/cs_basic_notes/OS/shell脚本.assets/image-20210302104553825.png)



双引号中可以包括换行符。

![image-20210302104846598](/Users/bytedance/Documents/cs_basic_notes/OS/shell脚本.assets/image-20210302104846598.png)

上图程序等价于：

![image-20210302104902431](/Users/bytedance/Documents/cs_basic_notes/OS/shell脚本.assets/image-20210302104902431.png)

在命令行中：

![image-20210302104916094](/Users/bytedance/Documents/cs_basic_notes/OS/shell脚本.assets/image-20210302104916094.png)

\>符号是在shell中键入多行语句时的提示符。



# shell脚本的执行方式

**方式一**

直接执行shell脚本的前提是**shell脚本具备可执行权限**，如果shell脚本所在目录不在shell环境变量PATH中，则需要使用绝对路径或者相对路径来执行shell脚本；如果shell脚本所在目录在shell环境变量PATH中，则在任意目录下都可以执行这个shell脚本。

hello world：

```shell
#!/bin/bash
STR="hello world"
echo STR
```

执行：

```shell
给shell脚本helloworld添加可执行权限
[root@rookie_centos shellScript]# chmod +x helloworld.sh
使用相对路径执行
[root@rookie_centos shellScript]# ./helloworld.sh
hello world
[root@rookie_centos shellScript]# pwd  
/root/shellScript
使用绝对路径执行
[root@rookie_centos shellScript]# /root/shellScript/helloworld.sh
hello world
[root@rookie_centos shellScript]# 
```

**方式二**

**直接使用bash来执行**，shell脚本作为输入参数，此时shell脚本不必具备可执行权限，使用bash来执行和直接执行的效果是一样的，我们可以这样来执行shell脚本“helloworld ”：

```shell
[root@rookie_centos shellScript]# bash ./helloworld 
hello world
[root@rookie_centos shellScript]# bash /root/shellScript/helloworld 
hello world
[root@rookie_centos shellScript]# 
```

**方式三**

使用souce执行

不管是直接执行还是使用bash来执行shell脚本，都是使用fork出来的子进程来执行shell脚本，shell脚本中的操作不会对当前用户登录的shell产生任何影响，因为子进程的操作不会影响到父进程。我们可以使用"souce"或者"."来执行shell脚本，且不要求shell脚本需要具备可执行权限，这样shell脚本就是在当前用户登录的shell中执行，而不是fork出一个子进程在子进程中执行，此时shell脚本中的操作就能对当前用户登录的shell产生影响。

```shell
# 直接执行shell脚本，fork出子进程来执行
[root@rookie_centos shellScript]# ./helloworld 
hello world
# 变量STR只存在于子进程中，用户登录的shell父进程中没有STR变量，故输出空串
[root@rookie_centos shellScript]# echo $STR

[root@rookie_centos shellScript]# 
# 使用source执行shell脚本，是在用户登录的shell父进程中执行
[root@rookie_centos shellScript]# source ./helloworld 
hello world
# 变量存在于当前用户登录的shell进程中，故使用echo打印STR变量的内容时能再次输出"hello world"
[root@rookie_centos shellScript]# echo $STR
hello world
[root@rookie_centos shellScript]# 
```

# 变量

## 环境变量

环境变量是存在当前登录shell中的公共数据，在当前shell中执行的任何程序和shell脚本都可以访问。通常来说，环境变量存储的都是系统的一些公用信息，例如可执行程序的搜索路径，动态库的搜索路径，当前用户名等。环境变量是“variable=value”形式的字符串集合，所以说环境变量是存储公共配置信息的地方。

我们可以在shell中执行env命令来查看当前登录shell中的环境变量，通过"export variable"命令和“declare -x variable”命令可以将其他非环境变量设置成环境变量，但新增的环境变量只在当前登录的shell中有效，退出当前登录shell重新登录shell后便无效。

| 环境变量 | 作用                                                         |
| -------- | ------------------------------------------------------------ |
| HOME     | 当前登录用户的home目录，使用“cd ~”或者“cd”可以直接切换到home目录 |
| PATH     | 可执行文件的查找目录集合，目录之间使用冒号“:”分隔            |
| SHELL    | 保存我们当前使用的是那个shell                                |
| HISTSIZE | shell历史命令保存的条数                                      |
| PWD      | 当前的工作目录                                               |

修改bashrc可以添加新的用户环境变量。

使用`source .bashrc`命令使得更改立即生效。

## 自定义变量

除了环境变量之外，在shell脚本中我们还需要定义一些自定义变量来保存诸如文件路径、文件名、用户输入、临时结果、命令执行结果等的信息。shell变量是弱类型变量，在使用时声明，不用区分类型，shell会自动解析。

- 变量设置语法
- 使用“variable=value”来设置变量，variable为变量名，value为变量值，等号左右不能有任何空格，如果value中有空格可以使用单引号或者双引号包含起来。

​		例如：STR="hello world"

- 变量名由英文字母和数字组成，但开头不能是数字。下面的声明就是错误的。

- 变量可以用于保存命令的执行结果，命令放在value部分，使用两个反单引号\` 或者$()包含起来。例如:

```shell
DIR=`pwd`
# OR
DIR=$(pwd)
```

- 使用单引号包含的value中的字符都是纯粹的字符，不会有任何特殊的含义，使用双引号包含的value中的特殊字符保留原来的语义，比如“\”为转义字符，“$”和“${}”为变量引用。
- 使用echo命令输出变量的值，可以使用$variable，或者${variable}里引用变量的值，第二种方式为了防止变量后面跟着其他字符，导致无法识别真正的变量名。

- 清除变量：当变量不再使用想删除变量时，可以使用unset命令来清除变量，例如：“unset USER_NAME”

## 特殊变量

| 特殊变量 | 含义                                                         |
| -------- | ------------------------------------------------------------ |
| $0       | 表示执行的shell脚本名称                                      |
| $n       | n为整数，n大于0，表示shell脚本的第n个输入参数，n大于等于10时需要使用${n}这中方式引用 |
| $#       | 表示shell脚本输入参数的个数，不包括$0                        |
| $*       | 表示全部的输入参数，不包括$0，当使用""将$*包含起来时，所有的参数被当作一个整体 |
| $@       | 表示全部的输入参数，不包括$0，当使用""将$@包含起来时，各个参数是放开的 |
| $?       | 上一条shell命令的执行结果，0表示成功，非0表示失败            |
| $$       | 当前进程的进程id                                             |

# test命令

Demo:

```shell
[root@rookie_centos shellScript]# cat dirCheck 
#!/bin/bash

#read命令从终端获取输入保存到dir变量中，p选项表示先在终端显示提示信息"please input the directory name : "
read -p "please input the directory name : " dir
#判断dir是否存在，如果不存在则提示错误信息并退出
test ! -e $dir && echo "$dir is not exist" && exit 1
#判断dir是否真的为目录，如果不是目录则提示错误信息并退出
test ! -d $dir && echo "$dir is not a directory" && exit 1
#判断当前用户是否有读目录的权限
test -r $dir && echo "$USER can read the $dir"
#判断当前用户是否有写目录的权限
test -w $dir && echo "$USER can write the $dir"
#判断当前用户是否有执行目录的权限
test -x $dir && echo "$USER can execution the $dir"
[root@rookie_centos shellScript]# 
[root@rookie_centos shellScript]# bash dirCheck 
please input the directory name : /usr/bin
root can read the /usr/bin
root can write the /usr/bin
root can execution the /usr/bin
[root@rookie_centos shellScript]#
```

test命令常用的例子和含义如下表：

| 例子                          | 含义                                                         |
| ----------------------------- | ------------------------------------------------------------ |
| test -e file                  | 判断file是否存在                                             |
| test -d file                  | 判断file是否是目录                                           |
| test -f file                  | 判断file是否是文件                                           |
| test -S file                  | 判断file是否是socket文件                                     |
| test -L file                  | 判断file是否是链接文件                                       |
| test -r file                  | 判断当前用户是否有读file的权限                               |
| test -w file                  | 判断当前用户是否有写file的权限                               |
| test -x file                  | 判断当前用户是否有执行file的权限                             |
| test -z str                   | 判断str字符串是否为空                                        |
| test str1 == str2             | 判断str1和str2这两字符串是否相等                             |
| test str2 != str2             | 判断str1和str2这两字符串是否不相等                           |
| test a -eq b                  | 判断整数a和b是否相等                                         |
| test a -ne b                  | 判断整数a和b是否不相等                                       |
| test a -ge b                  | 判断整数a是否大于等于整数b                                   |
| test a -le b                  | 判断整数a是否小于等于整数a                                   |
| test a -gt b                  | 判断整数a是否大于整数b                                       |
| test a -lt b                  | 判断整数a是否小于整数a                                       |
| test ! condition              | 判断condition是否不成立，比如“test ! -e /usr/bin”            |
| test condition1 -a condition2 | 判断condition1和condition2是否同时成立，比如“test -x /usr/bin -a -d /usr/bin” |
| test condition1 -o condition2 | 判断 condition1和condition2至少一个成立，比如“test -x /usr/bin -o -f /usr/bin” |

在shell脚本中的if判断语句中可以使用判断符号“[]”来替换test命令，例如“test -d file”可以使用"[ -d file ]"来替换。需要特别注意的是“[”后和“]”前至少要保留一个空格。判断大部分是使用在if语句中。



# if

demo：

```shell
#!/bin/bash

if [ $# -ne 1 ] ; then
	echo "please input one file"
	exit 1
fi

if [ -e "$1" ] ; then
	echo "$1 is exist"
fi 
[root@rookie_centos shellScript]# 
[root@rookie_centos shellScript]# cat ifDemo2
#!/bin/bash

if [ $# -ne 1 ] ; then
	echo "please input one directory"
	exit 1
fi

#判断用户输入的第一次参数是否是一个目录
if [ -d "$1" ] ; then
	echo "$1 is directory"
else 
	echo "$1 is not directory"
fi
[root@rookie_centos shellScript]# 
[root@rookie_centos shellScript]# cat ifDemo3
#!/bin/bash

if [ $# -ne 1 ] ; then
	echo "please input one interger"
	exit 1
fi

#输入的第一个整数参数是否大于等于10
if [ "$1" -ge 10 ] ; then
	echo "$1 >= 10"
elif [ "$1" -ge 5 ] ; then #输入输入的第一个整数参数是否大于等于5且小于10
	echo "5 <= $1 < 10" 
else
	echo "$1 < 5"
fi        
```



# 循环

打印99乘法表

```shell
#!/bin/bash

i=1    #初始化i的值为1

while [ $i -le 9 ]    #只要i的值小于等于9循环就一直执行
do
	j=1
	while [ $j -le $i ]    #内层循环，只要j小于等于i的值就一直执行
	do
		temp=$(($i*$j))    #计算i*j的值保存到temp变量中，在shell脚本中$(( ))可以做算数运算
		echo -n "$i * $j = $temp  "
		j=$(($j + 1))    #j的值累计加1
	done
	i=$(($i+1))    #i的值累计加1
	echo 
done 
```



for循环：

```shell
#!/bin/bash

files=$(ls /usr/bin/)    #"ls /usr/bin"的执行结果保存到变量files中

for file in $files   #遍历files这个结果集
do
	if [ -L "/usr/bin/$file" ] ; then    #如果某个文件是连接文件，就打印出一条提示信息
		echo "$file is link file"
	fi	
done
[root@rookie_centos shellScript]# 
[root@rookie_centos shellScript]# cat forDemo2
#!/bin/bash 

sum=0
for i in $(seq 1 100)    #seq用于生成1-100的序列，for循环就可以遍历这个序列
do
	sum=$(($i+$sum))    #sum用于累计i的和
done
echo "sum=$sum"
```



```shell
#!/bin/bash

sum=0
for ((i=1;i<=100;i+=1))    #i从1开始，只要i的值小于等于100就一直执行循环语句，i每次递增1
do
	sum=$(($sum+$i))    #sum用于累计i的和
done
echo "sum=$sum"
```

