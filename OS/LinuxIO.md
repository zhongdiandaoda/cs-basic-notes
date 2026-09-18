# Linux 网络 I/O

本文讨论 Linux 网络编程中的 I/O 模型、并发模型和事件驱动机制，重点是非阻塞 socket、`select/poll/epoll` 和 Reactor。普通文件的 Page Cache、Direct I/O 和磁盘异步 I/O 不在本文展开。

理解网络 I/O 时，要先分清两组概念：

- **阻塞 / 非阻塞**：描述调用线程在操作不能立即完成时，是等待，还是立即返回。
- **同步 / 异步**：描述真正的 I/O 操作由谁完成。同步 I/O 需要应用线程自己调用 `read/write` 完成数据拷贝；异步 I/O 是应用提交请求后，由内核完成 I/O，再通知应用。

这两个维度经常被混在一起。比如 I/O 多路复用可以配合非阻塞 fd 使用，但它本身仍属于同步 I/O，因为数据最终还是应用线程调用 `read/write` 读写。

本文按“概念定义 -> 代码模型 -> 适用场景”的方式组织。每一节尽量独立说明，不依赖图片或隐含上下文；如果只想查某个模型，可以直接跳到对应章节。

阅读约定：

- 本文里的 `fd` 都表示 Linux 文件描述符，socket、文件、管道都可以用 fd 表示。
- 本文里的 `read/write` 泛指一次同步系统调用，不限定只能读写普通文件。
- 示例代码主要用于说明模型，省略了完整错误处理、资源释放和协议解析细节。
- “就绪”只表示系统调用现在大概率不会阻塞，不表示一定能读到完整业务消息，也不表示一定能一次写完全部数据。

全文结构：

```text
基础概念
  -> 并发模型的演进
  -> select、poll、epoll
  -> Reactor 与 io_uring
  -> 常见误区和排障
```

## 1. 网络 I/O 基础

### 1.1 I/O 模型总览

以网络读取为例，一次读操作通常分成两个阶段：

1. 等待数据到达内核 socket 接收缓冲区；
2. 把数据从内核 socket 缓冲区拷贝到用户缓冲区。

不同 I/O 模型的差异主要就在这两个阶段。

| 模型 | 等待数据阶段 | 数据拷贝阶段 | 调用线程是否阻塞 | 典型接口 |
|---|---|---|---|---|
| 阻塞 I/O | 阻塞等待 | 阻塞拷贝 | 是 | 默认 `read` |
| 非阻塞 I/O | 立即返回，应用轮询 | 有数据时仍由 `read` 拷贝 | 等待阶段不阻塞 | `O_NONBLOCK` |
| I/O 多路复用 | 阻塞在复用器上等待多个 fd | 就绪后由 `read/write` 拷贝 | 阻塞在 `select/poll/epoll_wait` | `select`、`poll`、`epoll` |
| 信号驱动 I/O | 内核发信号通知 fd 就绪 | 应用收到信号后调用 `read/write` | 等待阶段不阻塞 | `SIGIO` |
| 异步 I/O | 内核等待 | 内核完成拷贝 | 提交后不阻塞 | Linux AIO、`io_uring` 部分模式 |

要点：

- **阻塞 I/O、非阻塞 I/O、I/O 多路复用、信号驱动 I/O 都属于同步 I/O**，因为最终的数据读写仍由应用线程调用 `read/write` 完成。
- **异步 I/O** 的关键是内核完成整个 I/O 操作后通知应用，而不是只通知“可以读了”。

用伪代码看这几个模型的差异：

```text
阻塞 I/O:
    read(fd)                 # 没有数据时，线程睡眠
    handle(data)

非阻塞 I/O:
    while true:
        n = read(fd)
        if n < 0 and errno == EAGAIN:
            do_other_work()  # 没有数据时，应用自己决定稍后重试
            continue
        handle(data)

I/O 多路复用:
    ready_fds = wait_many(fds)
    for fd in ready_fds:
        data = read(fd)      # 就绪后仍由应用线程读取数据
        handle(data)

异步 I/O:
    submit_read(fd, buf)
    do_other_work()
    event = wait_completion()
    handle(event.result)     # I/O 已由内核完成
```

如果只记一个判断标准，可以这样区分：

```text
就绪通知：内核告诉应用“现在可以读/写了”，应用还要自己 read/write。
完成通知：内核告诉应用“I/O 已经完成了”，应用直接处理完成结果。
```

#### 阻塞 I/O

默认情况下，socket 是阻塞的。

```c
ssize_t n = read(fd, buf, sizeof(buf));
```

如果此时 socket 接收缓冲区没有数据，调用线程会睡眠，直到数据到达、连接关闭或发生错误。

流程：

```text
应用线程调用 read
  -> 内核发现数据未就绪
  -> 线程睡眠，让出 CPU
  -> 数据到达，内核唤醒线程
  -> 内核把数据拷贝到用户缓冲区
  -> read 返回
```

阻塞 I/O 的优点是编程模型简单。缺点是一个线程在等待一个连接时，无法同时处理其他连接。

典型阻塞式服务端伪代码：

```c
listen_fd = socket();
bind(listen_fd, addr);
listen(listen_fd);

for (;;) {
    conn_fd = accept(listen_fd);     // 没有新连接时阻塞
    read(conn_fd, buf, sizeof(buf)); // 没有请求数据时阻塞
    handle_request(buf);
    write(conn_fd, response, len);   // 发送缓冲区满时可能阻塞
    close(conn_fd);
}
```

这个模型一次只能处理一个连接。只要某个客户端迟迟不发数据，整个服务端主循环就会卡住。

#### 非阻塞 I/O

可以通过 `fcntl` 把 fd 设置为非阻塞：

```c
int flags = fcntl(fd, F_GETFL, 0);
fcntl(fd, F_SETFL, flags | O_NONBLOCK);
```

非阻塞读：

```c
ssize_t n = read(fd, buf, sizeof(buf));
if (n < 0 && (errno == EAGAIN || errno == EWOULDBLOCK)) {
    // 当前没有数据可读，稍后再试
}
```

非阻塞 I/O 的特点：

- 没有数据时，`read` 不会睡眠，而是立刻返回 `EAGAIN` 或 `EWOULDBLOCK`；
- 应用需要自己决定什么时候重试；
- 如果简单地 while 循环轮询，会浪费 CPU。

所以非阻塞 I/O 通常会和 I/O 多路复用配合使用。

单独使用非阻塞 I/O 时，容易写成忙轮询：

```c
set_nonblocking(fd);

for (;;) {
    n = read(fd, buf, sizeof(buf));
    if (n > 0) {
        handle_data(buf, n);
        continue;
    }

    if (n < 0 && errno == EAGAIN) {
        // 没有数据，但循环马上重试，会浪费 CPU
        continue;
    }
}
```

这个例子能说明：非阻塞只是让系统调用不睡眠，不等于程序天然高效。高并发场景通常需要用 `select/poll/epoll` 来告诉应用“什么时候值得重试”。

#### I/O 多路复用

I/O 多路复用解决的问题是：一个线程如何同时等待多个 fd 的 I/O 事件。

基本模式：

```text
把多个 fd 注册给复用器
  -> 调用 select/poll/epoll_wait 等待事件
  -> 返回一批就绪 fd
  -> 应用遍历就绪事件
  -> 对就绪 fd 调用非阻塞 read/write
```

它的优势不是让单个 I/O 更快，而是让一个线程可以管理大量连接，避免“一个连接一个线程”的资源成本。

多路复用事件循环的抽象伪代码：

```c
for (;;) {
    ready_list = wait_io_events(all_connections);

    for each event in ready_list {
        if (event.type == READABLE) {
            read_until_not_ready(event.fd);
        }

        if (event.type == WRITABLE) {
            write_until_not_ready(event.fd);
        }
    }
}
```

这个模型要求业务处理不能长期占用事件循环。耗时业务通常要交给工作线程、协程或异步任务，否则会阻塞其他连接的 I/O 事件处理。

#### 信号驱动 I/O

信号驱动 I/O 指应用先告诉内核：当某个 fd 可读或可写时，向进程发送 `SIGIO` 信号。应用收到信号后，再主动调用 `read/write`。

它的关键点是：信号只是通知 fd 就绪，不负责把数据拷贝到用户缓冲区。所以信号驱动 I/O 仍然属于同步 I/O。

信号处理函数本身只能安全调用少量异步信号安全函数，因此不应该直接执行业务逻辑。更稳妥的做法是让信号处理函数只设置标志或唤醒正常事件循环：

```c
set_owner(fd, getpid());      // 告诉内核 SIGIO 发给哪个进程
enable_async_signal(fd);      // 设置 O_ASYNC
set_nonblocking(fd);          // 收到信号后仍建议非阻塞读写

on_SIGIO() {
    io_ready = true;          // 实际代码可使用 volatile sig_atomic_t
    wakeup_event_loop();      // 例如写入 self-pipe
}

event_loop() {
    wait_for_event();

    for (;;) {
        n = read(fd, buf, sizeof(buf));
        if (n > 0) {
            handle_data(buf, n);
            continue;
        }
        if (n < 0 && errno == EAGAIN) {
            break;
        }
        close(fd);
        break;
    }
}
```

这个模型在网络服务中不如 `epoll` 常见，原因是信号处理逻辑复杂，且信号本身不适合承载大量精细事件分发。

#### 异步 I/O

异步 I/O 指应用提交 I/O 请求后，不需要自己等待 fd 就绪，也不需要自己在就绪后完成数据拷贝；内核或内核协作机制完成 I/O 后，再把完成结果通知给应用。

抽象流程：

```text
应用提交 read 请求
  -> 内核或 I/O 框架执行等待和拷贝
  -> 完成后写入完成队列或触发完成通知
  -> 应用读取完成结果
```

伪代码：

```c
request = {
    .op = READ,
    .fd = fd,
    .buf = user_buf,
    .len = sizeof(user_buf),
};

submit_async_io(request);     // 提交后立即返回

while (do_other_work()) {
    // 应用线程可以继续处理其他任务
}

completion = wait_completion();
if (completion.result > 0) {
    handle_data(user_buf, completion.result);
}
```

需要注意：不同接口的“异步程度”并不完全相同。比如 `io_uring` 支持提交队列和完成队列，但某个具体操作是否真正由内核异步完成，还取决于操作类型、文件系统、内核版本和使用的 flag。

### 1.2 Socket 服务端基础流程

TCP 服务端通常经过以下步骤：

```text
socket
  -> bind
  -> listen
  -> accept
  -> read/write
  -> close
```

含义：

- `socket()`：创建 socket 文件描述符。
- `bind()`：绑定本地 IP 和端口。
- `listen()`：让 socket 进入监听状态。
- `accept()`：从全连接队列中取出一个已完成握手的连接。
- `read/write()`：在已连接 socket 上收发数据。

监听 socket 和已连接 socket 是不同的 fd：

- 监听 socket 负责接收新连接；
- `accept()` 返回的已连接 socket 负责和某个客户端通信。

一个监听 socket 可以不断 `accept` 多个连接。阻塞模型的问题不是 TCP 只能一对一，而是单线程阻塞写法一次只能服务一个连接。

阻塞 `accept` 的模型：

```c
for (;;) {
    int conn_fd = accept(listen_fd, NULL, NULL);
    if (conn_fd < 0) {
        continue;
    }

    handle_connection(conn_fd); // 如果这里阻塞，主循环就不能继续 accept
}
```

非阻塞事件循环里，监听 socket 也应该设置为非阻塞，并且一次性 `accept` 到 `EAGAIN`：

```c
for (;;) {
    int conn_fd = accept(listen_fd, NULL, NULL);
    if (conn_fd >= 0) {
        set_nonblocking(conn_fd);
        add_to_epoll(conn_fd, EPOLLIN);
        continue;
    }

    if (errno == EAGAIN || errno == EWOULDBLOCK) {
        break; // 当前没有更多已完成握手的连接
    }

    if (errno == EINTR) {
        continue; // 被信号中断，重试 accept
    }

    log_accept_error(errno);
    break;
}
```

### 1.3 文件描述符与 socket

在 Linux 中，socket 也通过文件描述符访问。

```text
进程 fd 表
  -> struct file
  -> socket
  -> sock
  -> 接收队列 / 发送队列
```

socket 收到的数据会组织为 `sk_buff`。`sk_buff` 是 Linux 网络协议栈里的核心数据结构，可以在链路层、IP 层、TCP 层之间传递，避免每一层都重新复制一份完整数据。

## 2. 并发模型的演进

本章从 C10K 问题出发，说明服务端为什么会从“一连接一执行单元”逐步演进到事件驱动模型。

### 2.1 C10K 问题

C10K 指单机同时处理一万个客户端连接的问题。

理论上，TCP 连接由四元组唯一标识：

```text
本地 IP + 本地端口 + 对端 IP + 对端端口
```

服务端监听端口固定时，不同客户端仍然可以通过不同的对端 IP 和对端端口建立大量连接。但实际并发能力会受这些因素限制：

- 单进程文件描述符上限；
- 系统总文件描述符上限；
- 每个连接占用的内核内存；
- 应用线程数、上下文切换成本；
- 网络带宽和协议栈处理能力；
- 应用层业务处理耗时。

解决 C10K 的核心不是“能不能建立连接”，而是“能不能以较低资源成本管理大量连接”。

### 2.2 BIO 多进程模型

最朴素的并发模型是：主进程负责 `accept`，每接受一个连接就 `fork` 一个子进程处理。

```text
主进程
  -> accept 得到 conn_fd
  -> fork 子进程
  -> 子进程处理 conn_fd
  -> 主进程继续 accept
```

优点：

- 编程简单；
- 进程之间地址空间隔离，单个子进程崩溃影响较小。

缺点：

- 进程创建和销毁成本高；
- 每个进程都有独立地址空间和内核数据结构；
- 进程切换要保存/恢复 CPU 上下文，并切换地址空间相关上下文；
- 子进程退出后需要父进程 `wait` 回收，否则会产生僵尸进程。

注意：进程上下文切换不会复制用户空间的虚拟内存、栈、全局变量。它主要切换寄存器、程序计数器、内核栈、调度状态、地址空间页表上下文等。

`fork` 后父子进程都会持有继承来的 fd。父进程应关闭本次连接的 `conn_fd`，子进程不再接收新连接时应关闭继承的 `listen_fd`：

```c
conn_fd = accept(listen_fd, ...);
pid = fork();

if (pid == 0) {
    close(listen_fd);          // 子进程只处理当前连接
    handle_connection(conn_fd);
    close(conn_fd);
    exit(0);
}

close(conn_fd);                // 父进程继续 accept
```

### 2.3 BIO 多线程模型

多线程模型通常是：主线程负责 `accept`，每个连接交给一个工作线程处理。

```text
主线程
  -> accept 得到 conn_fd
  -> 创建线程或交给线程池
  -> 工作线程阻塞 read/write
```

同一进程内的线程共享地址空间、文件描述符表、全局变量、堆和共享库。线程切换通常比进程切换轻，但不是没有成本。

Linux 中进程和线程都用 `task_struct` 表示。区别在于线程通过 `clone` 共享地址空间、文件表、信号处理等资源。

多线程模型的问题：

- 线程数量大时，线程栈占用明显；
- 大量线程阻塞和唤醒会增加调度成本；
- 线程池队列需要同步，存在锁竞争；
- 连接数很大时，“一个连接一个线程”仍然不适合。

## 3. I/O 多路复用

`select`、`poll` 和 `epoll` 都用于等待多个 fd 的就绪事件。三者解决的问题相同，主要差异在于关注集合的表达方式、内核处理方式以及返回就绪事件的成本。

### 3.1 select

`select` 的接口核心是 `fd_set`。

```c
int select(
    int nfds,
    fd_set *readfds,
    fd_set *writefds,
    fd_set *exceptfds,
    struct timeval *timeout
);
```

工作方式：

```text
应用准备 fd_set
  -> 调用 select，fd_set 从用户态拷贝到内核态
  -> 内核扫描 fd_set
  -> 如果没有 fd 就绪，线程睡眠
  -> fd 就绪后唤醒线程
  -> 内核修改 fd_set 并拷贝回用户态
  -> 应用遍历 fd_set 找出就绪 fd
```

`select` 的主要问题：

- `fd_set` 通常受 `FD_SETSIZE` 限制，常见值是 1024；
- 每次调用都要重新设置 `fd_set`，因为返回时它会被内核改写；
- 每次调用都要在用户态和内核态之间拷贝 fd 集合；
- 返回后应用还要线性扫描 fd 集合，复杂度是 `O(n)`。

`select` 的典型伪代码：

```c
for (;;) {
    FD_ZERO(&read_set);
    max_fd = listen_fd;

    FD_SET(listen_fd, &read_set);
    for each conn_fd in connections {
        FD_SET(conn_fd, &read_set);
        max_fd = max(max_fd, conn_fd);
    }

    n = select(max_fd + 1, &read_set, NULL, NULL, NULL);

    if (FD_ISSET(listen_fd, &read_set)) {
        accept_new_connection();
    }

    for each conn_fd in connections {
        if (FD_ISSET(conn_fd, &read_set)) {
            read_and_handle(conn_fd);
        }
    }
}
```

这里每轮都要重新构造 `fd_set`，并且返回后要遍历所有连接。

### 3.2 poll

`poll` 使用 `struct pollfd` 数组表示关注的 fd：

```c
struct pollfd {
    int fd;
    short events;
    short revents;
};

int poll(struct pollfd *fds, nfds_t nfds, int timeout);
```

相比 `select`：

- 不再使用固定大小的 bitmap；
- 不受 `FD_SETSIZE` 限制；
- 数量仍受进程最大打开文件数限制；
- 仍然需要把 `pollfd` 数组传入内核；
- 内核和用户态仍需要线性扫描，复杂度仍是 `O(n)`。

所以 `poll` 解决了 `select` 的 fd 数量限制，但没有从根本上解决大量 fd 的扫描成本。

`poll` 的典型伪代码：

```c
for (;;) {
    pollfds[0].fd = listen_fd;
    pollfds[0].events = POLLIN;

    for each conn_fd in connections {
        pollfds[i].fd = conn_fd;
        pollfds[i].events = POLLIN;
    }

    n = poll(pollfds, nfds, -1);

    for i in 0..nfds-1 {
        if (pollfds[i].revents & POLLIN) {
            handle_readable_fd(pollfds[i].fd);
        }
    }
}
```

`poll` 比 `select` 更容易管理大量 fd，但每次仍然要传入数组，并线性检查 `revents`。

### 3.3 epoll

`epoll` 把“维护关注列表”和“等待就绪事件”拆开：

```c
int epfd = epoll_create1(0);
epoll_ctl(epfd, EPOLL_CTL_ADD, fd, &event);
int n = epoll_wait(epfd, events, maxevents, timeout);
```

核心数据结构：

- interest list：应用关心的 fd 集合；
- ready list：已经就绪的事件集合；
- wait queue：等待 `epoll_wait` 返回的进程或线程。

基本流程：

```text
epoll_create1 创建 epoll 实例
  -> epoll_ctl 把 fd 加入 interest list
  -> fd 状态变化时，内核通过回调把事件加入 ready list
  -> epoll_wait 从 ready list 取事件并拷贝到用户提供的 events 数组
```

`epoll_wait` 返回的是就绪事件个数，同时会把具体事件写入用户提供的 `events` 数组。

和 `select/poll` 相比，`epoll` 的优势：

- 不需要每次等待都重新传入完整 fd 集合；
- 应用和内核不需要在每轮等待时扫描完整 fd 集合；
- 只返回就绪事件；
- 更适合大量连接、少量活跃的场景。

`epoll` 并不是没有处理成本。内核仍要执行就绪回调、维护 ready list，并把就绪事件复制到用户空间。它也不是所有场景都一定比 `poll` 快：如果 fd 数很少，或者绝大多数 fd 每次都活跃，优势会变小。

`epoll` 的使用可以拆成两个阶段。

初始化阶段：

```c
epfd = epoll_create1(0);

event.events = EPOLLIN;
event.data.fd = listen_fd;
epoll_ctl(epfd, EPOLL_CTL_ADD, listen_fd, &event);
```

事件循环阶段：

```c
for (;;) {
    n = epoll_wait(epfd, events, MAX_EVENTS, -1);

    for (i = 0; i < n; i++) {
        fd = events[i].data.fd;

        if (fd == listen_fd) {
            accept_new_connections();
        } else if (events[i].events & EPOLLIN) {
            read_and_handle(fd);
        }
    }
}
```

关键差异是：`epoll_ctl` 负责维护关注列表，`epoll_wait` 只负责等待和取出就绪事件。

### 3.4 epoll 事件如何产生

网络包到达时，不是“中断程序直接操作用户进程”。更准确的路径是：

```text
网卡收到数据
  -> 触发硬中断或 NAPI 轮询
  -> 内核软中断处理网络包
  -> 协议栈把数据放入 socket 接收队列
  -> socket 唤醒回调触发
  -> epoll 关注该 socket 时，把事件加入 ready list
  -> 唤醒阻塞在 epoll_wait 上的线程
```

这条路径说明：epoll 是通过内核里的回调和就绪队列减少重复扫描，而不是让应用直接处理硬件中断。

### 3.5 LT 和 ET

`epoll` 支持两种触发模式：

- **LT（Level Triggered，水平触发）**：只要 fd 当前仍然满足条件，就会持续通知。
- **ET（Edge Triggered，边缘触发）**：只有 fd 状态发生变化时通知一次，例如从不可读变为可读。

#### LT

读事件：

```text
接收缓冲区还有数据 -> epoll_wait 会持续返回可读
```

写事件：

```text
发送缓冲区还有空间 -> epoll_wait 会持续返回可写
```

LT 是默认模式，容错性更好。即使一次没有读完，下次 `epoll_wait` 还会继续提醒。

#### ET

ET 只在状态变化时提醒。如果应用收到可读事件后没有把数据读完，后续可能不会再次收到通知。

因此 ET 模式必须遵守两个规则：

1. fd 必须设置为非阻塞；
2. 读操作持续到 `EAGAIN`、EOF 或错误；写操作持续到应用发送缓冲区为空，或者返回 `EAGAIN`。

典型读模板：

```c
for (;;) {
    ssize_t n = read(fd, buf, sizeof(buf));
    if (n > 0) {
        // 处理读取到的数据
        handle_data(buf, n);
        continue;
    }

    if (n == 0) {
        // 对端关闭连接
        close(fd);
        break;
    }

    if (errno == EAGAIN || errno == EWOULDBLOCK) {
        // 当前已经没有更多数据可读
        break;
    }

    if (errno == EINTR) {
        // 被信号中断，继续重试
        continue;
    }

    // 其他错误
    close(fd);
    break;
}
```

典型写模板：

```c
while (has_pending_data(fd)) {
    ssize_t n = write(fd, pending_buf(fd), pending_len(fd));
    if (n > 0) {
        // 移动待发送缓冲区游标
        consume_pending_data(fd, n);
        continue;
    }

    if (n < 0 && (errno == EAGAIN || errno == EWOULDBLOCK)) {
        // 发送缓冲区满，等待下一次 EPOLLOUT
        enable_epollout(fd);
        break;
    }

    if (n < 0 && errno == EINTR) {
        continue;
    }

    close(fd);
    break;
}
```

不建议一直监听 `EPOLLOUT`，因为 socket 大多数时候都是可写的，LT 模式下会导致频繁唤醒。常见做法是：

- 平时只监听 `EPOLLIN`；
- 当应用层发送缓冲区还有数据没写完时，再临时打开 `EPOLLOUT`；
- 数据写完后关闭 `EPOLLOUT`。

### 3.6 epoll 常用事件

| 事件 | 含义 |
|---|---|
| `EPOLLIN` | 可读，普通数据或连接关闭都可能触发 |
| `EPOLLOUT` | 可写，发送缓冲区有空间 |
| `EPOLLET` | 使用边缘触发 |
| `EPOLLONESHOT` | 事件只触发一次，适合多线程处理同一个 fd |
| `EPOLLRDHUP` | 对端关闭写方向，常用于检测半关闭 |
| `EPOLLERR` | fd 发生错误 |
| `EPOLLHUP` | fd 被挂起或关闭 |

`EPOLLERR` 和 `EPOLLHUP` 通常不需要显式注册，也可能被返回。处理事件时应该检查它们。

收到 `EPOLLERR` 时，可以通过 `getsockopt(fd, SOL_SOCKET, SO_ERROR, ...)` 获取 socket 的具体错误。一个事件也可能同时包含 `EPOLLIN`、`EPOLLRDHUP` 等多个标志，不能假设它们互斥。

### 3.7 一个最小 epoll 服务端流程

伪代码：

```c
listen_fd = socket(...);
set_nonblocking(listen_fd);
bind(listen_fd, ...);
listen(listen_fd, SOMAXCONN);

epfd = epoll_create1(0);

listen_event.events = EPOLLIN | EPOLLET;
listen_event.data.fd = listen_fd;
epoll_ctl(epfd, EPOLL_CTL_ADD, listen_fd, &listen_event);

for (;;) {
    n = epoll_wait(epfd, events, MAX_EVENTS, -1);
    for (i = 0; i < n; i++) {
        fd = events[i].data.fd;

        if (fd == listen_fd) {
            for (;;) {
                conn_fd = accept(listen_fd, ...);

                if (conn_fd >= 0) {
                    set_nonblocking(conn_fd);
                    add_to_epoll(epfd, conn_fd, EPOLLIN | EPOLLET | EPOLLRDHUP);
                    continue;
                }

                if (errno == EAGAIN || errno == EWOULDBLOCK) {
                    break;
                }

                if (errno == EINTR) {
                    continue;
                }

                log_accept_error(errno);
                break;
            }
            continue;
        }

        if (events[i].events & EPOLLERR) {
            error = get_socket_error(fd); // 使用 SO_ERROR 获取具体错误
            close_connection(fd, error);
            continue;
        }

        if (events[i].events & EPOLLIN) {
            result = read_until_eagain(fd);
            if (result == EOF || result == ERROR) {
                close(fd);
                continue;
            }
        }

        if (events[i].events & EPOLLRDHUP) {
            // 对端关闭写方向；读完残留数据后，根据协议决定是否关闭
            handle_peer_half_close(fd);
        }

        if (events[i].events & EPOLLHUP) {
            close(fd);
            continue;
        }

        if (events[i].events & EPOLLOUT) {
            write_until_eagain(fd);
        }
    }
}
```

这个流程的重点不是代码细节，而是以下原则：

- 监听 socket 也要设置非阻塞，并且 `accept` 到 `EAGAIN`；
- 普通连接 fd 设置非阻塞；
- ET 模式下读到 `EAGAIN`、EOF 或错误，写到发送缓冲区为空或 `EAGAIN`；
- 一个 epoll 事件可能同时携带多个标志，需要按连接状态组合处理。

## 4. 事件驱动架构

I/O 多路复用解决的是事件等待问题，Reactor 进一步定义如何分发事件和组织业务处理；`io_uring` 则提供基于提交队列和完成队列的另一种接口模型。

### 4.1 Reactor 模型

基于 I/O 多路复用的服务端通常会演化成 Reactor 模型。

核心思想：

```text
Reactor 负责等待事件
  -> 收到事件后分发给 Handler
  -> Handler 执行 accept/read/write/业务处理
```

常见形态：

- 单 Reactor 单线程：一个线程负责监听、读写、业务处理。
- 单 Reactor 多线程：一个线程负责 I/O 事件分发，工作线程处理业务。
- 主从 Reactor 多线程：主 Reactor 负责 accept，从 Reactor 负责连接读写，工作线程负责业务。

Redis 早期主要是单线程事件循环处理网络 I/O 和命令执行；Nginx 常见是多 worker 进程，每个 worker 内部使用事件驱动模型。

### 4.2 和 io_uring 的关系

`epoll` 解决的是“如何高效等待大量 fd 的就绪事件”，本质仍然是同步 I/O 复用。

`io_uring` 是更现代的 Linux I/O 接口，提供提交队列 SQ 和完成队列 CQ：

```text
应用把 I/O 请求写入提交队列
  -> 内核消费请求
  -> I/O 完成后写入完成队列
  -> 应用读取完成事件
```

它可以减少系统调用次数，并支持更多异步化能力。但实际是否绕过 PageCache、是否真正异步，还要看具体操作类型、内核版本和文件系统支持。

学习顺序上，可以先掌握：

```text
阻塞 I/O -> 非阻塞 I/O -> select/poll -> epoll -> io_uring
```

## 5. 常见误区

### 5.1 select、poll、epoll 不是异步 I/O

它们只是等待 fd 就绪。就绪后，应用仍然要自己调用 `read/write`。

### 5.2 非阻塞 I/O 不等于不会失败

非阻塞 I/O 的常态返回值之一就是 `EAGAIN`。这不是异常故障，而是“现在不能继续读/写”。

### 5.3 epoll 返回可读，不代表一次 read 一定能读到完整业务包

TCP 是字节流，没有消息边界。应用层必须自己处理粘包、半包和协议解析。

常见做法是给每个连接维护一个应用层接收缓冲区，每次可读时先把内核数据读到这个缓冲区，再按协议格式解析完整消息。

以“4 字节长度头 + 消息体”的协议为例：

```c
on_readable(fd) {
    read_until_eagain(fd, conn[fd].input_buffer);

    for (;;) {
        if (buffer_size(conn[fd].input_buffer) < 4) {
            break; // 长度头还没收全，等待下次可读事件
        }

        uint32_t body_len = peek_u32(conn[fd].input_buffer);
        if (buffer_size(conn[fd].input_buffer) < 4 + body_len) {
            break; // 消息体还没收全，保留半包
        }

        discard(conn[fd].input_buffer, 4);
        message = read_bytes(conn[fd].input_buffer, body_len);
        handle_message(message);
    }
}
```

这里的重点是：`read` 只负责从内核缓冲区取字节，协议边界必须由应用层缓冲区和解析逻辑维护。

### 5.4 epoll 返回可写，不代表可以无限写

可写只表示发送缓冲区目前有空间。一次 `write` 仍可能只写入部分数据，剩余数据需要放入应用层发送缓冲区，等待下次继续写。

### 5.5 ET 不一定比 LT 更适合所有场景

ET 可以减少重复通知，但对代码正确性要求更高。如果没有严格处理到 `EAGAIN`，很容易出现连接卡死。工程上 LT 更简单，ET 更容易写出高性能但也更容易出错。

## 6. 排障方法

### 6.1 查看 fd 数量限制

```bash
ulimit -n
cat /proc/sys/fs/file-max
```

### 6.2 查看进程打开的 fd

```bash
ls /proc/<pid>/fd | wc -l
lsof -p <pid>
```

### 6.3 查看 socket 状态

```bash
ss -antp
ss -s
```

### 6.4 查看进程是否阻塞在 epoll

```bash
strace -p <pid>
```

如果看到大量：

```text
epoll_wait(...)
```

说明线程正在等待 I/O 事件。

如果看到：

```text
read(...) = -1 EAGAIN
write(...) = -1 EAGAIN
```

说明 fd 是非阻塞的，当前暂时不能继续读写，需要回到事件循环。

### 6.5 查看网络收发队列

```bash
ss -antp
```

关注：

- `Recv-Q`：接收队列中尚未被应用读取的数据。
- `Send-Q`：发送队列中尚未发送或尚未确认的数据。

如果 `Recv-Q` 长期很高，可能是应用读得慢。
如果 `Send-Q` 长期很高，可能是对端读得慢、网络拥塞或发送逻辑堆积。

## 7. 总结

Linux I/O 的核心链路可以概括为：

```text
阻塞 I/O：简单，但一个线程容易卡在一个连接上
非阻塞 I/O：不等待，但单独使用会忙轮询
I/O 多路复用：一个线程等待多个 fd 的就绪事件
epoll：把关注列表放在内核，只返回就绪事件，适合大量连接
异步 I/O：提交请求后由内核完成 I/O，再通知应用
```

实践中，高并发网络服务通常使用：

```text
非阻塞 socket + epoll + 事件循环 + 应用层缓冲区
```

写对这类程序的关键不是只会调用 `epoll_wait`，而是要正确处理：

- 非阻塞 `accept/read/write` 的 `EAGAIN`；
- TCP 字节流的半包和粘包；
- 写缓冲区和部分写；
- `EPOLLERR`、`EPOLLHUP`、`EPOLLRDHUP`；
- ET 模式下读到 `EAGAIN`、EOF 或错误，写到发送缓冲区为空或 `EAGAIN`。
