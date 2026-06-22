# 06 Concurrency & Network 复习提纲

## 考试定位

CON&NET 题重点不是 signal，也不是 system-level I/O。复习课明确说 signal 期末完全不考，system-level I/O 大概率也不会考。这里真正要优先准备的是 socket API、SNAT、bounded buffer / semaphore、prethreaded server。重点参考 `courseware/2-20-network_*.pdf`、`2-21-con_*.pdf`、`2-22-sync_*.pdf`、`2-23-locks_*.pdf`、`2-24-cv_*.pdf`，以及 `EXE/concurrent&network.pdf`。

高频题型：

- `connect / bind / listen / accept` 解释。
- `listenfd` 和 `connfd` 区别。
- SNAT 地址转换表。
- prethreaded server 为什么快。
- bounded buffer 的 semaphore 初值和 P/V 顺序。

低优先级背景：

- descriptor table / open file table / v-node table、`dup2` 只作为背景理解即可。
- signal handler safety、`sigsuspend`、EXE8 I/O tracing 不再作为期末核心准备。

## Network 基础

客户端-服务器模型（client-server model）：

1. client sends request。
2. server handles request。
3. server sends response。
4. client handles response。

Socket address 通常是：

```text
IP address : port
```

TCP connection 由 socket pair 标识：

```text
(client IP, client port, server IP, server port)
```

## Socket API 高频解释

`connect(clientfd, server_addr, addrlen)`：

客户端调用，请求内核与 server socket address 建立 TCP connection。成功后，`clientfd` 成为客户端连接端点。

`bind(listenfd, server_addr, addrlen)`：

服务器调用，把 socket descriptor 绑定到本地 IP:port。

`listen(listenfd, LISTENQ)`：

把已绑定的 socket 变成 listening socket，告诉内核它用于接收连接请求。

`accept(listenfd, client_addr, client_len)`：

服务器调用，阻塞直到有连接请求，完成连接后返回新的 connected descriptor。

`listenfd` vs `connfd`：

- `listenfd`：监听描述符，只负责接收连接请求，server 生命周期内通常一直存在。
- `connfd`：已连接描述符，每个 client connection 一个，用来 read/write，服务结束后 close。

## SNAT 题型

SNAT（Source NAT）会把内网 client 的源地址改写成 router 的公网地址和端口。

典型设定：

- Client: `192.168.1.10:10086`
- Router public: `1.2.3.4:40000`
- Server: `5.6.7.8:80`

填表：

1. Client -> Router, before SNAT

```text
src = 192.168.1.10:10086
dst = 5.6.7.8:80
```

2. Router -> Server, after SNAT

```text
src = 1.2.3.4:40000
dst = 5.6.7.8:80
```

3. Server -> Router, before reverse translation

```text
src = 5.6.7.8:80
dst = 1.2.3.4:40000
```

4. Router -> Client, after reverse translation

```text
src = 5.6.7.8:80
dst = 192.168.1.10:10086
```

为什么 server 看不到 `192.168.1.10:10086`？

标准回答：这是私有地址（private LAN address），不能在公网直接路由；SNAT router 把源地址替换为自己的公网 IP:port，并维护 NAT mapping，server 看到的是 router 的地址。

## Concurrent Server 三种模型

Iterative server：

- 一次只服务一个 client。
- 实现简单。
- 慢 client 会阻塞后面的 client。

Thread-per-connection：

- 每来一个连接创建一个 thread。
- 并发性好。
- thread 创建/销毁开销大，连接很多时资源压力大。

Prethreaded server：

- 启动时预先创建固定数量 worker threads。
- master thread 只 `accept`，把 `connfd` 放入 bounded buffer。
- workers 从 buffer 取 `connfd`，调用 `echo(connfd)`，然后 `close(connfd)`。

优点：

- 避免每个连接反复创建线程。
- master 能较快回到 `accept`。
- worker 数量受控，减少资源爆炸。

## Bounded Buffer Semaphore 题型

典型 bounded descriptor buffer：

```text
buf[n]
front, rear
```

需要三个同步量：

- `mutex = 1`：保护 buffer 结构的互斥锁。
- `slots = n`：空槽数量。
- `items = 0`：已有 item 数量。

插入 `sbuf_insert(connfd)`：

```c
P(&slots);
P(&mutex);
insert connfd into buffer;
V(&mutex);
V(&items);
```

删除 `sbuf_remove()`：

```c
P(&items);
P(&mutex);
remove connfd from buffer;
V(&mutex);
V(&slots);
```

为什么 P 顺序重要？

必须先等资源计数（`slots/items`），再拿 `mutex`。如果先拿 `mutex` 再等 `slots/items`，线程可能拿着锁睡眠，导致其他线程无法进入 buffer 改变条件，形成 deadlock。

简答句式：counting semaphores express availability; mutex protects the critical section。

## Threads 与共享变量

线程（threads）共享进程的：

- code segment。
- data segment。
- heap。
- shared libraries。
- open files。
- installed signal handlers。

每个线程独有：

- thread ID。
- stack。
- stack pointer。
- program counter。
- registers。
- condition codes。

并发程序的典型问题：

- data race：多个线程并发访问共享变量，至少一个写，且没有同步。
- deadlock：互相等待资源。
- livelock/starvation：一直活动但没有进展 / 长期得不到资源。

## Locks 与 condition variables

Mutex lock：

```c
pthread_mutex_lock(&lock);
/* critical section */
pthread_mutex_unlock(&lock);
```

评价 lock：

- mutual exclusion：互斥。
- fairness：公平性。
- performance：性能。

Condition variable：

用于“条件不满足时睡眠，条件满足后被唤醒”。

标准模式：

```c
pthread_mutex_lock(&mutex);
while (!condition)
    pthread_cond_wait(&cond, &mutex);
/* condition holds */
pthread_mutex_unlock(&mutex);
```

必须用 `while` 而不是 `if`，因为可能 spurious wakeup，或者被唤醒后条件已被其他线程改变。

## Signal 安全题型

复习课修正：signal 部分已经确定期末不考。本节只保留给你理解 QUIZ/EXE 的背景，不建议作为临考重点。

异步信号安全（async-signal safety）常见规则：

- G0：handler 中只做最少工作。
- G1：handler 中只调用 async-signal-safe 函数，例如 `write` / `sio_puts` / `sio_putl`；不要用 `printf`、`sprintf`、`malloc`、`exit`。
- G2：进入 handler 保存 `errno`，退出前恢复 `errno`。

安全 handler 模板：

```c
void chld_handler(int sig) {
    int saved_errno = errno;
    pid_t pid;
    while ((pid = waitpid(-1, NULL, WNOHANG)) > 0) {
        sio_puts("reaped pid ");
        sio_putl((long)pid);
        sio_puts("\n");
        reaped++;
    }
    errno = saved_errno;
}
```

为什么要用 `while waitpid(-1, ..., WNOHANG)`？

多个 child 的 SIGCHLD 可能合并；一次 handler 调用要尽量 reap 所有已退出 child，避免 zombie。

## Pause race 与 sigsuspend

错误模式：

```c
while (reaped < 3)
    Pause();
```

race：主程序检查 `reaped < 3` 后、调用 `Pause()` 前，如果 SIGCHLD 到达并被处理，之后 `Pause()` 可能永远睡眠，因为没有新的 signal 来唤醒。

正确思路：先 block SIGCHLD，fork children；然后在循环中用 `sigsuspend(&prev)` 原子地临时恢复 mask 并睡眠。

`sigsuspend` 关键句式：

它 atomically replaces the process signal mask and suspends the process until a signal handler runs; when it returns, the old mask is restored. 因此关闭了 test-and-wait gap。

为什么 `usleep(100)` 更差？

它是轮询，浪费 CPU，增加延迟，而且仍然依赖时间猜测，不是事件驱动的正确同步。

## I/O Tracing: fork + dup2 + stdio

复习课修正：system-level I/O 大概率不考。本节只保留三表模型和 `dup2` 背景；临考时间紧时优先复习 socket、SNAT、bounded buffer、prethreaded server。

三表模型：

- Descriptor table：每个进程一张，fd -> open file table entry。
- Open file table：系统共享，包含 file position、refcnt、flags。
- V-node table：系统共享，表示实际文件元数据。

`fork` 后：

- child 复制 parent 的 descriptor table。
- parent 和 child 的 fd 指向同一个 open file table entry。
- 因此共享 file offset。

`dup2(oldfd, newfd)`：

- 让 `newfd` 指向 `oldfd` 指向的 open file table entry。
- 若 `newfd` 原来打开，会先关闭。

`O_APPEND`：

- 每次 write 前，内核把文件 offset 移到文件末尾。
- 单个 `write` 的追加是原子的，不会和另一个 `write` 的 bytes 在中间交错。

Stdio buffer 陷阱：

- `printf` 写到 user-space stdio buffer，不一定立刻进文件。
- `fork` 会复制用户态 buffer。
- child `exit(0)` 会 flush 自己复制的 stdio buffer。
- 所以 fork 前未 flush 的 `printf("p1")` 可能在 child exit 时和 child 的 `printf("c1")` 一起写出。

EXE8 的核心套路：按每一行 `Read/Write/printf/Fork/Dup2/exit` 维护：

- 每个 fd 指向哪个 open file entry。
- 每个 open file entry 的 offset。
- stdio buffer 中有什么。
- 文件最终内容。

## Robust I/O / Socket I/O

网络 socket 建立后，`connfd` 可以像普通文件描述符一样用 Unix I/O `read` / `write`。

RIO（Robust I/O）用于处理 short counts：

- `rio_readn`：尽量读满 n bytes，除非 EOF。
- `rio_writen`：尽量写满 n bytes。
- `rio_readlineb`：带缓冲读一行，适合文本协议。

不要在 socket 上混用不合适的 standard I/O；标准库缓冲可能和网络 I/O 语义产生麻烦。

## 高频英文术语

- Client-server model：客户端-服务器模型。
- Socket：套接字。
- Socket pair：连接四元组。
- `listenfd`：监听描述符。
- `connfd`：已连接描述符。
- SNAT: Source Network Address Translation。
- Private IP address：私有 IP 地址。
- Concurrent server：并发服务器。
- Prethreaded server：预线程化服务器。
- Bounded buffer：有界缓冲区。
- Semaphore：信号量。
- Mutex：互斥锁。
- Critical section：临界区。
- Producer-consumer problem：生产者-消费者问题。
- Thread：线程。
- Data race：数据竞争。
- Deadlock：死锁。
- Condition variable：条件变量。
- Async-signal-safe：异步信号安全。
- `sigsuspend`：原子替换信号 mask 并睡眠。
- Descriptor table / open file table / v-node table：描述符表 / 打开文件表 / v-node 表。
- O_APPEND：追加写标志。

## 考前速记

- `listenfd` 接连接，`connfd` 做读写。
- SNAT 后 server 看到 router public IP:port。
- bounded buffer：insert 先 `P(slots)` 再 `P(mutex)`；remove 先 `P(items)` 再 `P(mutex)`。
- handler：少做事、用 `sio_*`、保存恢复 `errno`。
- `sigsuspend` 修复检查条件和睡眠之间的 race。
- fork 共享 open file entry，但复制 stdio buffer。
