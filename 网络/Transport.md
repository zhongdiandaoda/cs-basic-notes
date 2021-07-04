# 概述

传输层为运行在不同Host上的进程提供了一种逻辑通信机制。

发送方：将应用递交的消息分成一个或多个Segment，并向下传输给网络层。

接收方：将接收到的Segment重组为消息，上交给应用层。

# 多路复用和多路分用

![image-20210701165019807](Transport.assets/image-20210701165019807.png)

多路分用：根据TCP/UDP的传输层报文段中的源端口和目的端口信息等，将Segment导向不同的Socket。

主机收到UDP段后，检查目的端口号，导向对应的Socket（主机不同，端口号相同的UDP数据报会被同一个Socket接收）。

TCP的Socket用源/目的IP、源/目的端口号这样的四元组来标识，服务器会为每一个客户端建立一个Socket。

# UDP

User Datagram Protocol，它基于IP协议进行了复用/分用，并添加了简单的错误校验。

UDP提供尽力而为的服务，UDP的Segment可能会丢失或非按序到达，需要应用层保证可靠数据传输。

优点：

- 实现简单，无需维护连接状态
- 头部开销少
- 无需建立连接，延迟低
- 没有拥塞控制：应用可以更好的控制发送时间和速率

应用：

- 流媒体应用
- 速率敏感
- DNS
- SNMP

UDP段的格式：

<img src="Transport.assets/image-20210701170456750.png" alt="image-20210701170456750" style="zoom:80%;" />

UDP校验和：

首部和数据全部参与运算，用于检测UDP段在传输时是否发生错误（不一定能准确检测）。

16bit数相加，进位加给最终结果。

# 可靠数据传输

reliable data transfer(RDT)

![image-20210701171416671](Transport.assets/image-20210701171416671.png)

RDT基于停等协议。

## RDT1.0

底层信道完全可靠。

![image-20210701171706648](Transport.assets/image-20210701171706648.png)

## RDT2.0

RDT2.x假设信道只会发生位错误。

引入差错检测机制：利用校验和检测位翻转错误。

确认机制：

- ACK：接收方显式地告知发送方分组已正确接收。
- NAK：接收方显式地告知发送方分组有错误。

重传机制：发生错误时，重传分组。

停等协议：发送数据后，等收到ACK或者NAK后再进行下一次发送。

![image-20210702105218866](Transport.assets/image-20210702105218866.png)

## RDT2.1

RDT2.0的问题：ACK/NAK消息可能会发生错误。解决方式：当收到被破坏的ACK/NAK时，重传分组（为了避免产生重复分组，需要为分组添加序列号）。

RDT2.1同样基于停等机制：

发送方

![image-20210702110033002](Transport.assets/image-20210702110033002.png)

接收方

![image-20210702110452718](Transport.assets/image-20210702110452718.png)

## RDT2.2

取消了NAK，收到重复ACK后重传当前分组。

- 接收方通过ACK告知最后一个被正确接收的分组
- 在ACK消息中显式地加入被确认分组的序列号

## RDT3.0

RDT3.0假设信道既会发生位错误又会发生分组丢失。

加入了定时器。

![image-20210702111252910](Transport.assets/image-20210702111252910.png)

![image-20210702111452911](Transport.assets/image-20210702111452911.png)

![image-20210702111527251](Transport.assets/image-20210702111527251.png)

效率很低。

![image-20210702111859050](Transport.assets/image-20210702111859050.png)

# 滑动窗口协议

Go Back N

为了允许发送方在接收到ACK之前发送多个分组：

- 设置更大的存储空间来缓存分组
- 设置更大的序列号范围

滑动窗口协议：Sliding Window Protocol

窗口：允许使用的序列号范围。窗口尺寸为N，最多有N个未确认的消息。

滑动窗口协议：GBN（回退N帧），SR（选择性重传）。

## GBN

分组首部包含k-bit序列号。

ACK(n)：使用累积确认机制，表示直到序号n的分组均已被正确接收。

Timeout(n)时间：重传序列号大于等于n的所有分组。

发送方：

![image-20210702151148903](Transport.assets/image-20210702151148903.png)

接收方：

![image-20210702151521485](Transport.assets/image-20210702151521485.png)

只需要记住唯一的`expectednum`。

对于乱序到达的分组，直接丢弃掉（接收方没有缓存），然后重新确认按序到达的序列号最大的分组。

demo：

![image-20210702151821153](Transport.assets/image-20210702151821153.png)

## SR

Selective Repeat

SR：接收方对每个分组进行单独确认；发送方只重传没有收到ACK的分组，为每个分组设置Timeout；

发送方/接收方窗口：

![image-20210702152325306](Transport.assets/image-20210702152325306.png)

SR协议：

![image-20210702152413393](Transport.assets/image-20210702152413393.png)

`sender`：`ACK(n)`，只有n是最小的`unACKed`序列号，才增大窗口。

demo：

![image-20210702152738159](Transport.assets/image-20210702152738159.png)

SR协议的问题：

窗口大小太小时，将无法区分第一轮序列号与第二轮序列号。

![image-20210702153446747](Transport.assets/image-20210702153446747.png)

解决：发送方与接收方窗口和要小于等于2^k^，令N~S~+N~R~ <= 2^k^

# TCP

