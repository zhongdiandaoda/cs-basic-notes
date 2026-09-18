# shell参数

# 测试磁盘

```bash
# 测试磁盘写
time dd if=/dev/zero of=/testw.dbf bs=4k count=100000
rm -f /testw.dbf
# 测试磁盘读
time dd if=/dev/sdb of=/dev/null bs=4k
# 读写
time dd if=/dev/sdb of=/testrw.dbf bs=4k
rm -f /testrw.dbf
```

# 查看端口占用情况

```bash
# lsof
lsof -i:8086

# netstat
## tcp
netstat -nlpt | grep xxx
## udp
netstat -nlpu | grep xxx

-n  显示ip，不解析为域名
-l  显示listening状态的socket
-p  现实PID和程序名
-u	只显示udp
-t  只显示tcp
```

# 查找最近修改过的文件

```bash
# 查找最近24小时内的etc目录下修改过的文件：
find /etc -mtime 0
# 查找前48-24小时修改过的文件：
find /etc -mtime 1

# 查找最近24小时内修改过的当前目录的文件：
find . -mtime -1
# 查找最近48小时内修改过的当前目录的文件：
find . -mtime -2
# 查找最近100天内修改过的当前目录的文件：
sudo find . -mtime -100
# 查找100前修改过的当前目录的文件：
sudo find . -mtime +100

# 查找最近60分钟修改过的文件：
sudo find . -mmin -60
```

# mkdir -p

```bash

# 加上-p，递归创建路径，以下语句先创建files目录，再到files目录里创建input目录，再在input里创建group1目录
mkdir -p files/input/group1

```

# 添加执行权限

Linux 文件权限由读、写、执行三类权限组成：

| 权限 | 数字 | 含义 |
| --- | ---: | --- |
| `r` | 4 | 读取文件内容，或列出目录内容 |
| `w` | 2 | 修改文件内容，或在目录中创建、删除文件 |
| `x` | 1 | 执行文件，或进入目录 |

常见写法：

```bash
# 给所有用户添加执行权限
chmod +x a.sh

# 只给文件所有者添加执行权限
chmod u+x a.sh

# 设置为所有者可读、可写、可执行，其他人无权限
chmod 700 a.txt

# 设置为所有者可读写执行，组用户和其他用户可读可执行
chmod 755 a.sh
```

符号模式适合做局部增删权限：

| 命令                      | 作用               |
| ----------------------- | ---------------- |
| `chmod u+x a.sh`        | 给所有者添加执行权限       |
| `chmod g-w a.txt`       | 去掉所属组写权限         |
| `chmod o-r a.txt`       | 去掉其他用户读权限        |
| `chmod a+x a.sh`        | 给所有用户添加执行权限      |
| `chmod u=rw,go=r a.txt` | 所有者读写，组用户和其他用户只读 |

数字模式适合一次性指定完整权限：

| 数字 | 权限 |
| ---: | --- |
| `7` | `rwx` |
| `6` | `rw-` |
| `5` | `r-x` |
| `4` | `r--` |
| `0` | `---` |

脚本能否直接执行，取决于两个条件：

1. 文件有执行权限。
2. 文件第一行有正确的解释器声明，也就是 shebang。

```bash
#!/bin/bash

echo "hello"
```

执行方式区别：

```bash
# 需要执行权限，并使用 shebang 指定的解释器
./a.sh

# 不需要执行权限，由 bash 读取脚本内容执行
bash a.sh

# 在当前 shell 中执行，会影响当前 shell 的变量、cd 路径等状态
source a.sh
. a.sh
```

如果执行时报错 `Permission denied`，优先检查执行权限：

```bash
ls -l a.sh
chmod u+x a.sh
./a.sh
```

# scp

本地到远程

```bash
scp local_file remote_username@remote_ip:remote_folder
或者
scp local_file remote_username@remote_ip:remote_file
或者
scp local_file remote_ip:remote_folder
或者
scp local_file remote_ip:remote_file
```

远程到本地

```bash
scp root@www.runoob.com:/home/root/1.mp3 .
scp -r www.runoob.com:/home/root/others/ .
```

# grep

-i：忽略大小写

-w：匹配整个单词

-n：列出行并显示行号

-c：显示匹配到的行数

demo文件

```bash
running syntax check ...
checked 100 lines，2 have errors
error： nothing matched ！
error： cannot open the target ！
syntax check finished！
```

1. 搜索出包含demo的行

```bash
grep  error  log.txt
>>
checked 100 lines，2 have errors
error： nothing matched ！
error： cannot open the target ！
```

2. 搜索以demo开头的行

```bash
grep  "^error"  log.txt
>>
error： nothing matched ！
error： cannot open the target ！
```

3. 统计文件 log.txt 中字符 error 出现的次数

```bash
grep  -c  error  log.txt
>>
3
```

4. 搜索不包含字符串 error 的那些行

```bash
grep  -v  error  log.txt
>>
running syntax check ...
syntax check finished！
```

5. 在某路径的所有文件中搜索

```bash
grep  -r  error  log
```

# wc

统计文件行数

```bash
wc -l a.txt
```

统计单词数

```bash
wc -w a.txt

# 仅输出数字
wc -w a.txt | awk '{print $1}'
6
```

# 查看socket信息

```bash
# -n:不显示名字，以数字形式显示ip+端口
# -l:只显示listen状态的socket
# -p:显示进程信息
# -t:只显示TCP
liuqi.lq67@n248-187-151:~$ ss -nlpt
State     Recv-Q    Send-Q       Local Address:Port        Peer Address:Port
LISTEN    0         16384            127.0.0.1:1988             0.0.0.0:*
...

# socket统计信息
liuqi.lq67@n248-187-151:~$ ss -s
Total: 471
TCP:   259 (estab 57, closed 175, orphaned 0, timewait 13)

Transport Total     IP        IPv6
RAW	  1         1         0
UDP	  83        12        71
TCP	  84        30        54
INET	  168       43        125
FRAG	  0         0         0
```

# 查看网络状态

```bash
# -n numeric 使用IP
# -l listening
# -p program
# -t tcp
netstat -nltp
```

# 查找文件

```bash
find [搜索范围][选项][条件]
find / -name install.log  #在根目录下查找名为install.log文件
find / -type f "*.txt" # 在根目录下查找文件名后缀为.txt的文件
find / -type d dirname # 在根目录下查找目录名为缀为test的文件
```

# 复制文件夹

```bash
假设复制源目录 为 dir1 ,目标目录为dir2。怎样才能将dir1下所有文件复制到dir2下了
如果dir2目录不存在，则可以直接使用
cp -r dir1 dir2
即可。
如果dir2目录已存在，则需要使用
cp -r dir1/. dir2
如果这时使用cp -r dir1 dir2,则也会将dir1目录复制到dir2中，明显不符合要求。
```

# sed

```bash
// 若干个空格+一个#开头
sed -i '/^\s*#/d' vpn.conf
// 空行
sed -i '/^$/d' vpn.conf
```

假设文本fin.txt如下：

```bash
hello Jobs
hello Pony
hello Jack,  hi Jack
```

1. 把每一行的jack替换成mark

```bash
sed 's/Jack/Mark/' fin.txt
>>
hello Jobs
hello Pony
hello Mark, hi Jack
```

开头的s表示替换，

需要注意的是，在默认情况下，sed 只会替换每行中匹配到的第一个字符串，所以上面例子中最后一行的第二个 Jack 没有被替换，如果希望替换每一行中所有匹配到的字符串，需加在命令末尾上选项 g，比如：

```bash
sed  's/Jack/Mark/g'  fin.txt
>>
hello Jobs
hello Pony
hello Mark, hi Mark
```

注意这条命令并不会修改文件 fin.txt 的内容，只是将文件中的每一行读入缓存，执行替换，然后输出到屏幕，文件内容并没有发生改变。

如果希望直接修改文件内容，可加上选项 “ -i ”

`sed -i 's/Jack/Mark/g' fin.txt`

2. 将2-3行的 hello 替换成 hey

```bash
sed  -i '2,3s/hello/hey/g'  fin.txt
>>
hello Jobs
hey   Pony
hey   Jack,  hi Jack
```

3. 找出包含字符 Pony 的那些行，将这些行中的 hello 替换成 hey

```bash
sed  -i '/Pony/s/hello/hey/g'  fin.txt
>>
hello Jobs
hey   Pony
hello Jack, hi Jack
```

4. 删除2-3行

```bash
sed -i ‘2,3/d' fin.txt
```

5. 删除包含字符串Pony的行

```bash
sed -i '/Pony/d' fin.txt
```

6. 删除空白行

```bash
//^匹配开头  $匹配结尾
sed -i '/^$/d' fin.txt
//如果有空格，需要改成下面的样子，\s*匹配若干个空格
sed -i '/^\s*$/d' fin.txt
```

7. 删除不包含Pony字符串的行

```bash
sed -i '/Pony/\!d' fin.txt
```

8. 在某一行的前面或后面添加一行

```bash
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

9. 在指定行的前面或后面添加一行

```bash
sed -i  '/Pony/a\welcome'  fin.txt
>>
hello Jobs
hello Pony
welcome
hello Jack,  hi Jack

sed -i  '/Pony/i\welcome'  fin.txt
```

# awk常见用法

awk 的工作原理是将文件内容逐行读入，然后以每一行中的空格为分隔符将每行数据切分成几列，再对每列的元素进行各种分析处理。

NF：分割后元素数量

假设form.txt内容如下：

```bash
Num  Name  Company  Product
1    Jobs  Apple    iPhone
2    Jack  Alibaba  taobao
3    Pony  Tencent  wechat
```

1. 打印第2列和第3列

```bash
awk  '{print $2"\t"$3}'  form.txt
>>
Name	Company
Jobs	Apple
Jack	Alibaba
Pony	Tencent
```

2. 打印出第 2，3，4行的第二列和第三列，以制表符分隔

```bash
awk  '/^[0-9]/{print $2"\t"$3}'  form.txt
>>
Jobs    Apple
Jack    Alibaba
Pony    Tencent
```

取出以数字开头的行，并打印其第二列和第三列

3. 打印出前三行的第二列和第三列

```bash
awk  '{ if (NR<=3)  {print $2"\t"$3} }'  form.txt
>>
Name    Company
Jobs    Apple
Jack    Alibaba
```

4. 使用非空格字符切分列

```bash
Num,Name,Company,Product
1,Jobs,Apple,iPhone
2,Jack,Alibaba,taobao
3,Pony,Tencent,wechat
```

```bash
awk  -F","  '{print $2"\t"$3  }'  form.txt
>>
Name    Company
Jobs    Apple
Jack    Alibaba
Pony    Tencent
```

# 解压

```bash
tar cvf jpg.tar *.jpg #将目录里所有jpg文件打包成tar.jpg
tar xvf jpg.tar   #解压jpg.tar
zip jpg.zip *.jpg #zip格式的压缩
unzip jpg.zip     #解压jpg.zip

# 指定文件夹
tar -zxvf /bbs.tar.zip -C /zzz/bbs
```

**实例1：将文件全部打包成tar包**

**命令：**

`tar -cvf log.tar log2012.log`

`tar -zcvf log.tar.gz log2012.log`

`tar -jcvf log.tar.bz2 log2012.log`

**输出：**

```bash
[root@localhost test]# ls -al log2012.log

---xrw-r-- 1 root root 302108 11-13 06:03 log2012.log

[root@localhost test]# tar -cvf log.tar log2012.log

log2012.log

[root@localhost test]# tar -zcvf log.tar.gz log2012.log

log2012.log

[root@localhost test]# tar -jcvf log.tar.bz2 log2012.log

log2012.log

[root@localhost test]# ls -al *.tar*

-rw-r--r-- 1 root root 307200 11-29 17:54 log.tar

-rw-r--r-- 1 root root   1413 11-29 17:55 log.tar.bz2

-rw-r--r-- 1 root root   1413 11-29 17:54 log.tar.gz
```

**说明：**

`tar -cvf log.tar log2012.log` 仅打包，不压缩！

`tar -zcvf log.tar.gz log2012.log` 打包后，以 gzip 压缩

`tar -zcvf log.tar.bz2 log2012.log` 打包后，以 bzip2 压缩

在参数 f 之后的文件档名是自己取的，我们习惯上都用 .tar 来作为辨识。 如果加 z 参数，则以 .tar.gz 或 .tgz 来代表 gzip 压缩过的 tar包； 如果加 j 参数，则以 .tar.bz2 来作为tar包名。

**实例3：将tar 包解压缩**

**命令：**

`tar -zxvf /opt/soft/test/log.tar.gz`

**输出：**

```bash
[root@localhost test3]# ll

总计 0[root@localhost test3]# tar -zxvf /opt/soft/test/log.tar.gz

log2012.log

[root@localhost test3]# ls

log2012.log
```

**说明：**

在预设的情况下，我们可以将压缩档在任何地方解开的

# 单引号和双引号的区别

Shell 会先对命令行做变量展开、命令替换、通配符展开、单词分割，然后才执行命令。引号的核心作用是控制这些展开行为。

| 写法 | 是否展开变量 | 是否执行命令替换 | 是否保留空格为一个参数 | 典型用途 |
| --- | --- | --- | --- | --- |
| 不加引号 | 是 | 是 | 否 | 简单命令、明确不含空格的参数 |
| 单引号 `'...'` | 否 | 否 | 是 | 完全按字面量传递 |
| 双引号 `"..."` | 是 | 是 | 是 | 保留变量值整体，避免空格拆分 |

变量展开：

```bash
name="hello world"

echo $name
# hello world

echo "$name"
# hello world

echo '$name'
# $name
```

`echo $name` 和 `echo "$name"` 的输出看起来一样，是因为 `echo` 会把收到的多个参数用空格拼起来打印。真正的区别在于命令实际收到的参数个数：

```bash
name="hello world"

# 不加双引号，变量展开后会按空格拆成两个参数：hello 和 world
for arg in $name; do
  echo "<$arg>"
done
# <hello>
# <world>

# 加双引号，变量展开后仍然是一个整体参数：hello world
for arg in "$name"; do
  echo "<$arg>"
done
# <hello world>
```

所以日常写脚本时，只要变量值可能包含空格、换行或通配符，引用变量就应该加双引号。

命令替换：

```bash
today=$(date +%Y%m%d)
tar -czf "log-${today}.tar.gz" /var/log
```

推荐使用 `$()`，可读性比反引号更好，也更容易嵌套。

```bash
# 不推荐
tar -czf log-`date +%Y%m%d`.tar.gz /var/log

# 推荐
tar -czf "log-$(date +%Y%m%d).tar.gz" /var/log
```

通配符展开：

```bash
# 展开为当前目录下所有 .log 文件
ls *.log

# 不展开，查找名字就叫 *.log 的文件
ls '*.log'
```

常用经验：

- 引用变量时默认使用双引号：`"$var"`。
- 需要完全禁止展开时使用单引号：`'$var'`。
- 需要在单引号字符串中拼接变量时，拆开写：`'prefix-'"$var"'-suffix'`。
- 写 JSON、正则、awk、sed 脚本时，经常用单引号保护特殊字符。

# 后台运行程序

默认情况下，前台命令会占住当前终端，shell 会等待它结束后才继续接收下一条命令：

```bash
python server.py
```

在命令后面加 `&`，可以让进程在后台运行：

```bash
python server.py &

# 输出最近一个后台任务的 PID
echo $!
```

这种方式只是让当前 shell 不阻塞。进程仍然属于当前终端会话，退出登录或关闭终端时，进程通常会收到 `SIGHUP` 信号并退出。

查看和切换当前 shell 管理的后台任务：

```bash
# 查看后台任务
jobs -l

# 将最近一个后台任务切回前台
fg

# 将编号为 1 的任务切回前台
fg %1

# 让暂停的任务继续在后台运行
bg %1
```

如果前台程序正在运行，可以先按 `Ctrl-Z` 暂停，再用 `bg` 放到后台继续运行：

```bash
# Ctrl-Z 之后
bg %1
```

希望退出终端后进程继续运行，常用 `nohup`：

```bash
nohup python server.py > server.log 2>&1 &
```

这条命令包含三部分：

| 片段 | 含义 |
| --- | --- |
| `nohup` | 忽略挂断信号，终端退出时尽量不终止进程 |
| `> server.log` | 将标准输出写入日志文件 |
| `2>&1` | 将标准错误也写入同一个日志文件 |
| `&` | 放到后台运行 |

如果命令已经用 `&` 启动，也可以用 `disown` 把任务从当前 shell 的作业表中移除：

```bash
python server.py > server.log 2>&1 &
disown %1
```

查看和停止后台进程：

```bash
# 按命令名查进程
pgrep -af server.py

# 优雅终止
kill <pid>

# 强制终止，只有普通 kill 无效时再使用
kill -9 <pid>
```

长期服务不建议只依赖 `nohup`。更稳妥的方式是使用 `systemd` 管理，或者在临时运维场景中使用 `tmux`、`screen` 保持会话：

```bash
# 新建 tmux 会话
tmux new -s work

# 断开会话但不中止程序：Ctrl-B 后按 D

# 重新进入会话
tmux attach -t work
```

常见选择：

| 场景 | 推荐方式 |
| --- | --- |
| 临时跑一个耗时命令 | `cmd &` |
| 退出终端后继续跑一次性任务 | `nohup cmd > log 2>&1 &` |
| 已经后台启动，需要脱离当前 shell | `disown` |
| 需要随时重新进入交互现场 | `tmux` 或 `screen` |
| 生产长期服务 | `systemd` |
