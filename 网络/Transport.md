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

- 流媒体应用（电话，视频会议等）
- 速率敏感
- DNS：域名服务
- SNMP：简单网络管理协议
- TFTP：简单文件传输协议
- NTP：`Network Time Protocol`

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

Timeout(n)事件：重传序列号大于等于n的所有分组。

发送方：

![image-20210702151148903](Transport.assets/image-20210702151148903.png)

接收方：

![image-20210702151521485](Transport.assets/image-20210702151521485.png)

只需要记住唯一的`expectednum`。

对于乱序到达的分组，直接丢弃掉（接收方没有缓存），然后重新确认按序到达的序列号最大的分组。

demo：

![image-20210702151821153](Transport.assets/image-20210702151821153.png)

对于n位的序列号，要求发送窗口的大小<=2^n^-1。

例如，对于3bit的序列号，发送窗口大小为8时：

发送窗口：0 1 2 3 4 5 6 7

情况一：所有确认帧都到达了发送端，随后发送方继续发送0 1 2 3 4 5 6 7

情况二：所有确认帧都丢失了，超时后发送方重发0 1 2 3 4 5 6 7，此时接收方无法判断这是旧的8个数据包还是新的数据包。

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

一般发送窗口的大小等于接收窗口的大小。

解决：

对于选择重传协议，序列号为m比特时，窗口大小<=2^(m-1)^，首先，发送窗口不能比接收窗口大，不然接收窗口可能会溢出。


其次，要最大化发送窗口的流水线分组，但是要保证不能产生二义性。假设序号最大为7即0,1,2,3,4,5,6,7，发送窗口大小为5，当发送窗口发送0,1,2,3,4后，假设接收窗口全部收到，则接收窗口向前移动到5次，接受窗口期望接收5,6,7,0,1.若发送窗口并没接收到任何ACK，所以发送窗口重发0，1，2,3,4此时接收窗口会以为重发的0,1是新的分组。

因为发送窗口<=接收窗口。要最大化发送窗口，则发送窗口=接收窗口。假设发送窗口为m,则接收窗口也为m.发送窗口发送m个分组时，接收窗口向前移动m，接收窗口为m+1,m+2,...2m.要避免二义性,必须满足2m<=序号最大值.
# TCP

TCP是面向连接的、全双工的点对点通信协议，保证可靠的、按序的字节流。

基于TCP的应用层协议：

- HTTP
- FTP
- SMTP：简单邮件传输协议
- TELNET：远程登陆
- SSH：`Secure Shell`

## TCP段结构

![image-20210709150540250](Transport.assets/image-20210709150540250.png)

序列号和ACK的编号不是端的编码，而是利用数据的字节数来计数。

Receive window用于流量控制，指愿意接受的字节数。

序列号：

- 序列号指的是segment中第一个字节的编号，而不是segment的编号；
- 建立TCP连接时，双方随机选择序列号。

ACK number：

- 希望接收到的下一个字节的序列号；
- 累积确认：该序列号之前的所有字节均已被正确收到。

demo：

![image-20210709151033205](Transport.assets/image-20210709151033205.png)

A -> B：发送含有一个字节的段，序列号为42，ACK=79，表示希望收到的下一个字节的编号为79。

B -> A：发送含有一个字节的段，序列号为79，ACK=43，表示序列号43之前的段都被收到了，且希望收到的下一个字节的编号为43。

A -> B：序列号为43，ACK为80（79及之前的字节已被正确收到）。

## 可靠数据传输

TCP使用了流水线机制以提高性能。

TCP使用累积确认机制，使用单一重传定时器。

触发重传的事件：

- 超时
- 收到重复ACK

RTT(Round Trip Time)的设置：

过短：不必要的重传。

过长：对段丢失反应慢。

`SampleRTT`：测试从段发出到收到ACK的时间，是一个随网络情况变化的值。

`EstimatedRTT`：测量多次`sampleRTT`形成的平均值。

`EstimatedRTT = (1 - α) * EstimatedRTT + α * SampleRTT`，α通常为0.125

定时器超时时间的设置：`EstimatedRTT + “安全边界”`

测量RTT的变化值：`SampleRTT`和`EstimatedRTT`的差值，作为定时器的安全边界。

TCP发送端伪代码：

```c
NextSeqNum = InitialSeqNum
SendBase = InitialSeqNum
loop(forever) {
    switch(event) {
    	event: data received from application above
            create TCP segment with sequence number NextSeqNum
            if(timer currently not running)
                start timer
            pass segment to IP
            NextSeqNum += length(data)
                
       event: timer timeout
           retransmit not-yet-acknowledged segment with smallest sequence number
           start timer
           
       event: ACK received, with ACK field value of y
           if(y > SendBase) {
               SendBase = y
               if(there are currently not-yet-acknowledged segments) {
                   start timer
               }
           }
    }
}
```

demo：

<img src="Transport.assets/image-20210709155508215.png" alt="image-20210709155508215" style="zoom:80%;" />

接收端的ACK生成：

| 事件                                                         | 接收方行为                                    |
| ------------------------------------------------------------ | --------------------------------------------- |
| 一个序列号为x的段按序到达，且x之前的段均已被ACK              | 延迟ACK。等待500ms，若没有下一个段，则发送ACK |
| 一个序列号为y的段按序到达，且之前的一个段有正在延迟发送的ACK | 立即发送一个累积确认的ACK                     |
| 一个乱序的段到达，期待的序列号为x，到达的段序列号大于x，产生一个gap | 立即发送一个重复ACK，确认最后一个正确收到的段 |
| 一个可以完全或局部填充gap的段到达                            | 立即发送ACK进行确认                           |

## 快速重传机制

TCP的实现中，发生一次超时之后，超时时间间隔会增大。

通过重复ACK，可以检测出分组丢失。

当某个分组丢失后，接收方会多次回复相同的ACK，当sender收到对同一数据的3个ACK，则假定该数据之后的段已经丢失，在计时器没有超时的前提下直接重传。

<img src="Transport.assets/image-20210709160918621.png" alt="image-20210709160918621" style="zoom:67%;" />

## TCP流量控制

目的：解决发送方发送数据过快或发送数据过多，以至于淹没接收方（buffer溢出）的问题。

- Buffer中的可用空间：`RcvWindow = RcvBuffer - [LastByteRcvd - LastByteRead]`

- Receiver通过在Segment的头部字段将`RcvWindow`的尺寸设置为Buffer中的可用空间大小

- Sender限制自己已经发送的但还未收到ACK的数据不超过接收方的`RcvWindow`大小。

当`RcvWindow`的大小为0时，发送方会启动一个特殊的计时器，每隔一段时间发送一个1字节的探测报文段，以便动态获取接收方新的`RcvWindow`的大小。

## TCP连接管理

**1 建立连接**

生成初始序列号有很多方法，初始序列号会随时间的改变而改变。

三次握手：

Step1：客户端发送一个SYN报文段，指定客户端的初始序列号#，不包含数据。

Step2：服务器接收SYN后，回复一个SYN/ACK报文段。同时，为客户端分配buffer并指定服务器的初始序列号#。

Step3：客户端收到SYN/ACK报文后，回复ACK报文段（可以包含数据）。

三次握手的原因：

- 同步双方初始序列号：两次握手只能保证一方的序列号被对方接收。

- 为了防止两次握手情况下，已失效的连接请求报文段突然又传送到服务器端而导致错误。例如，客户端A向服务器B发出建立连接请求，但该请求在网络中某个节点长时间滞留，超时后A发出另一个请求，B收到后建立连接，数据传输结束后断开连接。而此时，第一个连接请求到达了服务端B，如果使用三次握手，B向A回复ACK，A收到后不做处理，建立连接失败；如果使用两次握手，B会认为连接已经建立，为A分配资源并等待数据，但A并不会发送数据，导致服务器资源的浪费。



TCP对旧的SYN报文段的处理：

接收方无法辨别SYN是不是旧的，返回一个SYN/ACK，发送方发现ACK中的序列号不是期望的序列号时，发送RST报文段重置连接。

## 拥塞控制

太多发送主机发送了太多数据或发送速度太快，以至于网络无法处理。

拥塞的表现：

- 分组丢失（路由器缓存溢出）
- 分组延迟过大（路由器缓存中排队）