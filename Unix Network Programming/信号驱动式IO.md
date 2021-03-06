# 信号驱动式 I/O

信号驱动式 I/O 是指进程预先告诉内核，使得当某个描述符上发生某事件时，内核使用信号通知相关进程。

## 信号驱动式 I/O 的使用

要使用信号驱动 I/O，程序要按照以下步骤执行：
1. 建立 SIGIO 信号的信号处理函数。
2. 设置文件描述符的属主，即当文件描述符上可执行 I/O 时会接收到通知信号的进程或进程组。通常让调用进程成为属主。可以使用 fcntl 的 F_SETOWN 完成：

    ```
    fcntl(fd, F_SETOWN, pid);
    ```
3. 设定 O_NONBLOCK 标志使之成为非阻塞 I/O。
4. 打开 O_ASYNC 标志开启信号驱动式 I/O，可以和上一步合并为一个操作：

    ```
    flags = fcntl(fd, F_GETFL);
    fcntl(fd, F_SETFL, flags | O_ASYNC | O_NONBLOCK);
    ``` 
5. 调用进程就可以执行其他任务。当 I/O 操作就绪，内核为进程发送一个信号，然后调用第一步的信号处理函数。
6. 信号驱动式 I/O 提供的是边缘触发通知。因此进程一旦被通知 I/O 就绪，就应该尽可能多地执行 I/O。如果文件描述符是非阻塞的，表明需要在循环中执行 I/O 系统调用直到失败为止，错误码为 EAGIAN 或 EWOULDBLOCK。

## 何时发送 "I/O" 就绪信号

### 终端和伪终端

对于终端和伪终端，当产生新的输入时会生出一个信号，即使之前的输入还没有被读取。如果终端上出现文件结尾的情况，也会发送 “输入就绪” 的信号（伪终端则不会）。

对于终端来说没有 “输出就绪” 信号。当终端断开连接时也不会发出信号。

当伪终端主设备侧读取了输入后就会产生 “输出就绪” 的信号。

### 管道和 FIFO

对于管道和 FIFO 的读端，信号产生的条件：
- 数据写入到管道中（即使已经有未读取的输入存在）
- 管道的写端关闭

对于写端：
- 对管道的读操作增加了管道中的空余空间大小，因此现在可以写入 PIPE_BUF 个字节而不被阻塞
- 管道的读端关闭

### 套接字

#### TCP 套接字

信号驱动式 I/O 对于 TCP 套接字近乎无用，因为该信号产生的太过频繁，而且其出现并没有告诉我们发生了什么事件。

对于 TCP 套接字，SIGIO 信号产生的条件为：
- 监听套接字上某个连接请求已经完成
- 某个断连请求已发起
- 某个断连请求已完成
- 某个连接已经半关闭
- 数据到达套接字
- 数据已经从套接字发送走
- 发生某个异步错误

可以看到如果一个进程既读套接字又写套接字，信号处理函数无法区分这两种情况。我们应当考虑只对监听 TCP 套接字使用 SIGIO，其产生 SIGIO 的唯一条件是某个新连接的完成。

#### UDP 套接字

对于 UDP 套接字，SIGIO 信号在发生以下事件时产生：
- 数据报到达套接字
- 套接字上发生异步错误

信号驱动式 I/O 对于套接字的现实用途一般是 UDP 的 NTP 服务器程序。

示例： [使用 SIGIO 的 UDP 回设服务器程序](https://github.com/Rjerk/snippets/blob/master/unp/sigio/udpechoserv_sigio.cpp)

### inotify 文件描述符

当 inotify 文件描述符成为可读状态时会产生一个信号 —— 也就是由 inotify 文件描述符监视的其中一个文件上有时间发生时。

## 信号驱动式 I/O 的优化

在需要同时检查大量文件描述符的应用程序中，同 select 和 poll 相比，信号驱动式 I/O 能提供显著的性能优势。因为内核可以记住这些要检查的文件描述符，且仅当 I/O 事件实际发生在这些文件文件描述符时才向相应进程发送信号，所以用信号驱动式 I/O 的程序性能可以根据发生的 I/O 事件的数量来扩展，而和被检查的文件描述符的数量无关。

### 指定实时信号

如果需要在多个文件描述符上检查大量 I/O 事件，为利用信号驱动式 I/O 的全部优点，必须执行这两个步骤：
1. 通过 fcntl 的 F_SETSIG 来指定一个实时信号，当文件描述符上 I/O 就绪时，这个实时信号取代 SIGIO 被发送。
2. 使用 sigaction 安装信号处理函数时，为前一步的实时信号指定 SA_SIGINFO 标记。

这样做有两个理由：
1. 默认的 SIGIO 信号是标准的非排队的信号之一，如果有多个 I/O 事件发送了信号，而 SIGIO 被阻塞了，那么除了第一个通知外，其他后续通知都会丢失。如果通过指定实时信号作为通知信号，多个通知就能排队处理。
2. 使用 sigaction 并指定 SA_SIGINFO 标记，那么结构体 siginfo_t 会作为第二个参数传递给信号处理函数。这个结构体包含的字段标识出了在哪个文件描述符上发生了事件，以及事件的类型。

```
int sigaction(int signum, const struct sigaction* act,
              struct sigaction* oldact);

struct sigaction {
    void     (*sa_handler)(int);
    void     (*sa_sigaction)(int, siginfo_t*, void*);
    sigset_t sa_mask;
    int      sa_flags;
    void     (*sa_restorer)(void);
};
```

对于 I/O 就绪事件，传递给 siginfo_t 中与之相关的字段有：
- si_signo: 引发信号处理函数得到调用的信号值。
- si_fd: 发生 I/O 事件的文件描述符。
- si_code: 表示发生事件类型的掩码。
- si_band: band 事件，一个位掩码。

### 信号队列溢出的处理

可排队的实时信号的数量有限，达到上限后内核对于 I/O 就绪的通知将恢复为默认的 SIGIO 信号，出现这种现象表明信号队列溢出。

可以通过增加可排队的实时信号数量的限制来减小信号队列溢出的可能性，但不能完全消除溢出的可能性。

一个设计良好的采用 F_SETSIG 来建立实时信号作为 I/O 就绪通知的程序也要为 SIGIO 安装信号处理函数。如果发送了 SIGIO 信号，那么进程可以先通过 sigwaitinfo 将队列中的实时信号获取，然后临时切换到 select 或 poll，通过它们获取剩余的发生 I/O 事件的文件描述符列表。

### 在多线程程序中使用信号驱动式 I/O

Linux 提供了两个非标准的 fcntl，F_SETOWN_EX 和 F_GETOWN_EX。

F_SETOWN_EX 类似 F_SETOWN，除了允许指定进程或进程组作为接受信号的目标外，还可以指定一个线程作为 I/O 就绪信号的目标。对此操作，第三个操作为指向如下结构体的指针：

```
struct f_owner_ex {
    int type;
    pid_t pid;
};
```

F_GETOWN_EX 使用 f_owner_ex 返回之前由 F_SETOWN_EX 所定义的操作。
