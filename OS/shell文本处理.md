# 正则表达式

## grep

grep命令的格式：

`grep [options] regx [file...]`

regx是指一个正则表达式。

grep的常用选项如下：

| 选项 | 描述                                                         |
| ---- | ------------------------------------------------------------ |
| -i   | 忽略大小写。                                                 |
| -v   | 不匹配。通常，grep程序打印出包含匹配项的文本行，加上该选项后，grep程序会打印出不包含匹配项的文本行。 |
| -c   | 打印匹配的数量。                                             |
| -l   | 打印包含匹配项的文件名，而不是文本行本身。                   |
| -L   | 打印不包含匹配项的文件名。                                   |
| -n   | 在每个匹配行之前打印出其位于文件中的相应行号。               |
| -h   | 用于多文件搜索，不输出文件名。                               |
| -E   | 识别一个ERE（扩展的正则表达式）                              |

示例：

```shell
[root@docker ~]# ls /bin > dirlist-bin.txt
[root@docker ~]# ls /usr/sbin > dirlist-usr-sbin.txt
[root@docker ~]# ls /sbin > dirlist-sbin.txt
[root@docker ~]# ls /usr/bin > dirlist-usr-bin.txt
[root@docker ~]# ls dir*.txt
dirlist-bin.txt  dirlist-sbin.txt  dirlist-usr-bin.txt  dirlist-usr-sbin.txt
# 列出含有name字符串的文件名
[root@docker ~]# grep -l name dirlist*.txt
dirlist-bin.txt
dirlist-sbin.txt
dirlist-usr-bin.txt
dirlist-usr-sbin.txt
# 列出不含有name字符串的文件名
[root@docker ~]# grep -L lvm dirlist*.txt
dirlist-bin.txt
dirlist-usr-bin.txt
```

正则表达式元字符：

| 元字符       | 含义                                                         |
| ------------ | ------------------------------------------------------------ |
| 基本元字符： |                                                              |
| `.`          | 匹配任意单个字符                                             |
| `^`          | 只有在文本行的开始匹配到正则表达式时才认为发生一次匹配；     |
| `$`          | 只有在文本行的结尾匹配到正则表达式时才认为发生一次匹配       |
| `[ ]`        | 中括号表达式匹配中括号中的集合中匹配任意单个字符             |
| `[^...]`     | 在中括号里的^字符表示否定，匹配不在中括号字符集合里的任意单个字符 |
| `[-]`        | 表示范围，例如0-9，a-z                                       |
| *            | 匹配任意长度字符串                                           |
| 扩展元字符： |                                                              |
| ?            | 匹配0个或1个元素                                             |
| +            | 匹配一个或多个元素                                           |
| \|           | 或，分割字符串                                               |
| ( )          | 优先级                                                       |
| { }          | 匹配指定个数的元素                                           |

需要在正则表达式中包含一个连字符时，要把连字符放在第一个字符的位置：`grep -h '[-AZ]' dirlist*.txt`

锚点示例：

```shell
[me@linuxbox ~]$ grep -h '^zip' dirlist*.txt
zip
zipcloak
zipgrep
zipinfo
zipnote
zipsplit
[me@linuxbox ~]$ grep -h 'zip$' dirlist*.txt
gunzip
gzip
funzip
gpg-zip
preunzip
prezip
unzip
zip
[me@linuxbox ~]$ grep -h '^zip$' dirlist*.txt
zip
```

字符区域：

```shell
[root@docker ~]# grep -h '^[A-Z]' dirlist*.txt
VGAuthService
NetworkManager
VGAuthService
NetworkManager
[root@docker ~]# grep -h '^[A-Za-z0-9]*ve$' dirlist*.txt
ifenslave
logsave
lvremove
pvmove
pvremove
vgremove
ifenslave
logsave
lvremove
pvmove
pvremove
vgremove
```

## Posix字符集

字符区域不总是工作，在ls命令中，它会产生一些错误的结果。

![image-20210202201934989](/Users/bytedance/Documents/cs_basic_notes/OS/shell文本处理.assets/image-20210202201934989.png)

![image-20210202202119627](/Users/bytedance/Documents/cs_basic_notes/OS/shell文本处理.assets/image-20210202202119627.png)

原因：

ASCII排序规则的字符集：

`ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz`

locale不是US时，系统可能使用其他排序规则的字符集，例如：

`aAbBcCdDeEfFgGhHiIjJkKlLmMnNoOpPqQrRsStTuUvVwWxXyYzZ`

此时，[a-z]会同时包含大小写字母。

解决方式为使用POSIX字符集：

| 类           | 描述                                            |
| ------------ | ----------------------------------------------- |
| `[:alnum:]`  | 任意字母和数字，同`A-Za-z0-9`                   |
| `[:alpha:]`  | 任意字母                                        |
| `[:blank:]`  | 空格和制表符，同\\\t                            |
| `[:cntrl:]`  | ASCII控制字符，0-31和127                        |
| `[:digit:]`  | 任意数字                                        |
| `[:print:]`  | 任意可打印字符                                  |
| `[:graph:]`  | 不包括空格的任意可打印字符，包括33-126          |
| `[:punct:]`  | 不在`cntrl`和`alnum`中的任意字符                |
| `[:space:]`  | 包括空格在内的任意空白字符，同`\\f\\n\\r\\t\\v` |
| `[:upper:]`  | 任意大写字母                                    |
| `[:lower:]`  | 任意小写字母                                    |
| `[:xdigit:]` | 任意十六进制数                                  |
| `[:word:]`   | alnum加上下划线                                 |

POSIX将正则表达式分成了基本正则表达式(BRE)和扩展正则表达式(ERE)。

BRE可以辨别以下元字符：

`^ $ . [ ] *`

ERE添加了以下元字符：

`( ) { } ? + |`

ERE的第一个特性是交替：

BRE示例：

```shell
[root@docker ~]# echo "AAA" | grep AAA
AAA
[root@docker ~]# echo "BBB" | grep AAA
[root@docker ~]# 
```

ERE示例：

```shell
# 将|用单引号引起来防止shell将其解释成管道
[root@docker ~]# echo "AAA" | grep -E 'AAA|BBB'
AAA
[root@docker ~]# echo "BBB" | grep -E 'AAA|BBB'
BBB
[root@docker ~]# echo "CCC" | grep -E 'AAA|BBB'
[root@docker ~]# 
```

限定符（指定一个元素被匹配的次数）

eg. 匹配以下格式的电话号码：

```
(nnn) nnn-nnnn
nnn nnn-nnnn
```

可以构造这样的正则表达式：

`^\(?[0-9][0-9][0-9]\)? [0-9][0-9][0-9]-[0-9][0-9][0-9][0-9]$`

需要对圆括号添加\进行转义，圆括号之后的？代表他们可以出现也可以不出现。

{ }可以用来匹配特定个数的元素，它可以通过四种方法来指定：

| 限定符 | 含义                                                    |
| ------ | ------------------------------------------------------- |
| n      | 匹配前面的元素，如果它确切地出现了 n 次。               |
| n,m    | 匹配前面的元素，如果它至少出现了 n 次，但是不多于 m次。 |
| n,     | 匹配前面的元素，如果它出现了 n 次或多于 n 次。          |
| ,m     | 匹配前面的元素，如果它出现的次数不多于 m 次。           |

之前匹配电话号码的正则表达式可优化为：

`^\(?[0-9]{3}\)? [0-9]{3}-[0-9]{4}$`



# grep



-i：忽略大小写

-w：匹配整个单词

-n：列出行并显示行号

-c：显示匹配到的行数

demo文件

```go
running syntax check ...
checked 100 lines，2 have errors
error： nothing matched ！
error： cannot open the target ！
syntax check finished！
```

1. 搜索出包含demo的行

```go
grep  error  log.txt
>>
checked 100 lines，2 have errors
error： nothing matched ！
error： cannot open the target ！
```

1. 搜索以demo开头的行

```go
grep  "^error"  log.txt
>>
error： nothing matched ！
error： cannot open the target ！
```

1. 统计文件 log.txt 中字符 error 出现的次数

```go
grep  -c  error  log.txt
>>
3
```

1. 搜索不包含字符串 error 的那些行

```go
grep  -v  error  log.txt
>>
running syntax check ...
syntax check finished！
```

1. 在某路径的所有文件中搜索

```go
grep  -r  error  log
```



# wc

统计文件行数

```go
wc -l a.txt
```

统计单词数

```go
wc -w a.txt

# 仅输出数字
wc -w a.txt | awk '{print $1}'
6
```

# sed

```go
// 若干个空格+一个#开头
sed -i '/^\s*#/d' vpn.conf
// 空行
sed -i '/^$/d' vpn.conf
```

假设文本fin.txt如下：

```go
hello Jobs
hello Pony
hello Jack,  hi Jack
```

1. 把每一行的jack替换成mark

```go
sed 's/Jack/Mark/' fin.txt
>>
hello Jobs
hello Pony
hello Mark, hi Jack
```

开头的s表示替换，

需要注意的是，在默认情况下，sed 只会替换每行中匹配到的第一个字符串，所以上面例子中最后一行的第二个 Jack 没有被替换，如果希望替换每一行中所有匹配到的字符串，需加在命令末尾上选项 g，比如：

```go
sed  's/Jack/Mark/g'  fin.txt
>>
hello Jobs
hello Pony
hello Mark, hi Mark
```

注意这条命令并不会修改文件 fin.txt 的内容，只是将文件中的每一行读入缓存，执行替换，然后输出到屏幕，文件内容并没有发生改变。

如果希望直接修改文件内容，可加上选项 “ -i ”

``sed -i 's/Jack/Mark/g' fin.txt

1. 将2-3行的 hello 替换成 hey

```go
sed  -i '2,3s/hello/hey/g'  fin.txt
>>
hello Jobs
hey   Pony
hey   Jack,  hi Jack
```

1. 找出包含字符 Pony 的那些行，将这些行中的 hello 替换成 hey

```go
sed  -i '/Pony/s/hello/hey/g'  fin.txt
>>
hello Jobs
hey   Pony
hello Jack, hi Jack
```

1. 删除2-3行

```go
sed -i ‘2,3/d' fin.txt
```

1. 删除包含字符串Pony的行

```go
sed -i '/Pony/d' fin.txt
```

1. 删除空白行

```go
//^匹配开头  $匹配结尾
sed -i '/^$/d' fin.txt
//如果有空格，需要改成下面的样子，\s*匹配若干个空格
sed -i '/^\s*$/d' fin.txt   
```

1. 删除不包含Pony字符串的行

```go
sed -i '/Pony/\!d' fin.txt
```

1. 在某一行的前面或后面添加一行

```go
# 前面 i   1代表第一行
sed -i '1i\welcome'  fin.txt
>>
welcome
hello Jobs
hello Pony
hello Jack,  hi Jack

# 后面 a 
sed -i  '1a\welcome'  fin.txt
```

1. 在制定行的前面或后面添加一行

```go
sed -i  '/Pony/a\welcome'  fin.txt
>>
hello Jobs
hello Pony
welcome
hello Jack,  hi Jack

sed -i  '/Pony/i\welcome'  fin.txt
```

# awk

awk 的工作原理是将文件内容逐行读入，然后以每一行中的空格为分隔符将每行数据切分成几列，再对每列的元素进行各种分析处理。

NF：分割后元素数量

假设form.txt内容如下：

```go
Num  Name  Company  Product
1    Jobs  Apple    iPhone
2    Jack  Alibaba  taobao
3    Pony  Tencent  wechat
```

1. 打印第2列和第3列

```go
awk  '{print $2"\t"$3}'  form.txt
>>
Name	Company
Jobs	Apple 
Jack	Alibaba 
Pony	Tencent 
```

1. 打印出第 2，3，4行的第二列和第三列，以制表符分隔

```go
awk  '/^[0-9]/{print $2"\t"$3}'  form.txt
>>
Jobs    Apple
Jack    Alibaba
Pony    Tencent
```

取出以数字开头的行，并打印其第二列和第三列

1. 打印出前三行的第二列和第三列

```go
awk  '{ if (NR<=3)  {print $2"\t"$3} }'  form.txt
>>
Name    Company
Jobs    Apple
Jack    Alibaba
```



1. 使用非空格字符切分列

```go
Num,Name,Company,Product
1,Jobs,Apple,iPhone
2,Jack,Alibaba,taobao
3,Pony,Tencent,wechat
```



```go
awk  -F","  '{print $2"\t"$3  }'  form.txt
>>
Name    Company
Jobs    Apple
Jack    Alibaba
Pony    Tencent
```