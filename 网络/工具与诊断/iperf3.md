# 一、 iperf3简介

`iperf3`是一个网络性能测试工具([iperf3下载地址](https://iperf.fr/iperf-download.php))。

iperf3 是一个用于测量网络带宽的开源工具，它是 iperf 工具的第三个版本。它主要用于评估网络的性能，包括带宽、延迟和数据包丢失率等。

iperf3 使用客户端-服务器模型进行带宽测试。用户可以在一台计算机上运行 iperf3 服务器，并在另一台计算机上运行 iperf3 客户端，通过网络连接进行通信。

iperf可以测试TCP和UDP带宽质量。iperf可以测量最大TCP带宽，具有多种参数和UDP特性。iperf可以报告带宽，延迟抖动和数据包丢失。

iperf3 可以在多种操作系统上运行，包括 Linux、Windows、macOS 等。

**测试方法：**

1. 先在服务器端挂起 `iperf3 -s` 服务
2. 然后在客户端测试命令 `iperf3 -c 服务器IP地址`
3. 分析网络带宽、网络延时，判断网络整体性能
4. 默认用 5201 端口，需要对防火墙进行相关的设置

# 二、iperf3测速

Linux 服务器命令：**iperf3 -s**

Windows10 客户端：_**iperf3.exe -c 服务器IP地址 -P 10**_ //客户端10路并行给服务器发送TCP包；

**Windows10：*iperf3.exe -u -c 服务器IP地址 -b 1g*** //客户端给服务器发送udp数据包，最大带宽为1g

# 三、iperf3参数介绍

```bash
-f [k|m|K|M] 分别表示以Kbits, Mbits, KBytes, MBytes显示报告，默认以Mbits为单位,eg:iperf -c 222.35.11.23 -f K
-i sec 以秒为单位显示报告间隔，eg:iperf -c 222.35.11.23 -i 2
-l 缓冲区大小，默认是8KB,eg:iperf -c 222.35.11.23 -l 16
-m 显示tcp最大mtu值
-o 将报告和错误信息输出到文件eg:iperf -c 222.35.11.23 -o c:\iperflog.txt
-p 指定服务器端使用的端口或客户端所连接的端口eg:iperf -s -p 9999;iperf -c 222.35.11.23 -p 9999
-u 使用udp协议
-w 指定TCP窗口大小，默认是8KB
-B 绑定一个主机地址或接口（当主机有多个地址或接口时使用该参数）
-C 兼容旧版本（当server端和client端版本不一样时使用）
-M 设定TCP数据包的最大mtu值
-N 设定TCP不延时
-V 传输ipv6数据包
```

**服务器参数说明**

```bash
-D 以服务方式运行ipserf，eg:iperf -s -D
-R 停止iperf服务，针对-D，eg:iperf -s -R
```

**客户端参数说明**

```bash
-c 在客户端模式下运行
-u 使用UDP而不是TCP ，必须设置最大限制带宽（默认1 Mbit/sec）
--sctp 使用 SCTP 而不是 TCP（Linux、FreeBSD 和 Solaris）
-b UDP模式使用的带宽，单位bits/sec。此选项与-u选项相关。默认值是1 Mbit/sec， 0：不限制
-t 设置传输的总时间。Iperf在指定的时间内，重复的发送指定长度的数据包。默认是10秒钟。
-n 要传输的字节数,通常情况，Iperf按照10秒钟发送数据。-n参数跨越此限制，按照指定次数发送指定长度的数据，而不论该操作耗费多少时间。
-k 要传输的块（数据包）数
-l 要读取或写入的缓冲区的长度。TCP 的默认值为 128 KB，UDP 程序会自动调整，若自动调整失败，则用 1460
-R 反向模式运行，即服务端发送，客户端接收(默认是客户端发送，服务端接收)
-r 分别进行双向测试
-P 大写字母P，要运行的并行客户端流的数量
-d 同时进行双向传输测试
-n 指定传输的字节数，eg:iperf -c 222.35.11.23 -n 100000
-r 单独进行双向传输测试
-t 测试时间，默认10秒,eg:iperf -c 222.35.11.23 -t 5
-F 指定需要传输的文件
-T 指定ttl值
-O n 忽略前n秒
```
