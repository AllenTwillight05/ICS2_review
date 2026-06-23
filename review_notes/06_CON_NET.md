# 06 Concurrency & Network 开卷速查

## 一分钟速查

CON&NET 题的核心不是 signal，也不是 system-level I/O。复习课明确说 signal 期末完全不考，system-level I/O 大概率不考。真正要准备的是：socket API、SNAT、bounded buffer / semaphore、prethreaded server。

来源优先级：

- `EXE/concurrent&network.pdf`：最核心样题，SNAT and Prethreaded Server。
- `EXE/concurrent&network_sol.pdf`：对应答案。
- `courseware/2-20-network_*.pdf`：network / socket / HTTP 基础。
- `courseware/2-21-con_*.pdf`、`2-22-sync_*.pdf`、`2-23-locks_*.pdf`、`2-24-cv_*.pdf`：concurrency / semaphore / locks / condition variables。

低优先级背景：

- `QUIZ/ics2-quiz.pdf` Problem 2：Signal & I/O。
- `EXE/EXE5.pdf`、`EXE/EXE6.pdf`、`EXE/EXE7.pdf` Problem 1/2、`EXE/EXE8.pdf`：主要是 signal / process / system-level I/O。
- 这些不建议作为期末核心题复习。

题型对应来源：

| 题型 | 主要来源 |
|---|---|
| `connect / bind / listen / accept`、`listenfd` vs `connfd` | `EXE/concurrent&network.pdf` Part 1 |
| SNAT 地址转换表、server 为什么看不到 private IP | `EXE/concurrent&network.pdf` Part 2 |
| bounded buffer、semaphore 初值、P/V 顺序、deadlock 解释 | `EXE/concurrent&network.pdf` Part 3 |
| prethreaded server 相比 iterative / thread-per-connection 的优势 | `EXE/concurrent&network.pdf` Part 4 |
| worker 数量、CPU-bound vs I/O-bound、thread pool 大小权衡 | `EXE/concurrent&network.pdf` Part 5 |
| FIFO buffer、公平性、starvation、semaphore wakeup 是否 FIFO | `EXE/concurrent&network.pdf` Part 6 |

核心句：

> 这题基本就是“网络连接怎么建立 + NAT 地址怎么改 + 多线程服务器如何安全地把 connfd 分发给 worker”。

## Part 1：Socket API

来源：`EXE/concurrent&network.pdf` Part 1。

### `connect`

客户端调用：

```c
connect(clientfd, server_addr, server_addrlen)
```

作用：请求内核和 server socket address 建立 TCP connection。成功后，`clientfd` 成为客户端连接端点。

### `bind`

服务器调用：

```c
bind(listenfd, server_addr, server_addrlen)
```

作用：把 socket descriptor 绑定到本地 IP:port。

### `listen`

服务器调用：

```c
listen(listenfd, LISTENQ)
```

作用：把 socket 变成 listening socket，用来接收连接请求。

### `accept`

服务器调用：

```c
accept(listenfd, client_addr, client_len)
```

作用：阻塞等待连接请求；连接建立后返回新的 connected descriptor。

### `listenfd` vs `connfd`

| 描述符 | 含义 |
|---|---|
| `listenfd` | 监听描述符，长期存在，只负责接收连接 |
| `connfd` | 已连接描述符，每个 client connection 一个，用来 `read/write`，服务结束后关闭 |

开卷提示：

> `accept` 不会用 `listenfd` 直接和 client 通信，而是返回新的 `connfd`。

## Part 2：SNAT

来源：`EXE/concurrent&network.pdf` Part 2。

SNAT: Source Network Address Translation。路由器把内网 client 的 source IP:port 改成路由器自己的公网 IP:port，并维护 NAT mapping。

典型设定：

```text
Client private: 192.168.1.10:10086
Router public:  1.2.3.4:40000
Server:         5.6.7.8:80
```

四步填表：

| 阶段 | src | dst |
|---|---|---|
| Client -> Router, before SNAT | `192.168.1.10:10086` | `5.6.7.8:80` |
| Router -> Server, after SNAT | `1.2.3.4:40000` | `5.6.7.8:80` |
| Server -> Router, before reverse translation | `5.6.7.8:80` | `1.2.3.4:40000` |
| Router -> Client, after reverse translation | `5.6.7.8:80` | `192.168.1.10:10086` |

为什么 server 看不到 `192.168.1.10:10086`？

> `192.168.x.x` 是 private IP，不是公网全局唯一地址，不能在公网直接路由。SNAT router 用自己的公网 IP:port 替换 source address，所以 server 看到的是 router public address。

## Part 3：Bounded Buffer Synchronization

来源：`EXE/concurrent&network.pdf` Part 3。

场景：

- master thread 调 `accept()` 得到 `connfd`。
- master 把 `connfd` 插入 bounded buffer。
- worker threads 从 buffer 取 `connfd`，调用 `echo(connfd)`，然后 `close(connfd)`。

需要三个同步量：

| semaphore | 初值 | 含义 |
|---|---:|---|
| `mutex` | 1 | 保护 buffer 数据结构 |
| `slots` | `n` | 空槽数量 |
| `items` | 0 | 已有 descriptor 数量 |

插入：

```c
P(&slots);
P(&mutex);
insert connfd into buffer;
V(&mutex);
V(&items);
```

删除：

```c
P(&items);
P(&mutex);
remove connfd from buffer;
V(&mutex);
V(&slots);
```

为什么 P 顺序重要？

> 必须先等资源计数 `slots/items`，再拿 `mutex`。如果先拿 `mutex` 再等资源，线程可能拿着锁睡眠，其他线程无法进入 buffer 改变条件，导致 deadlock。

一句话：

> counting semaphore 表示资源数量；mutex 保护 critical section。

## Part 4：Prethreaded Server

来源：`EXE/concurrent&network.pdf` Part 4。

三种 server 模型：

| 模型 | 特点 | 问题 |
|---|---|---|
| Iterative server | 一次服务一个 client | 慢 client 阻塞后面所有 client |
| Thread-per-connection | 每来一个 connection 创建一个 thread | 创建/销毁开销大，连接多时资源压力大 |
| Prethreaded server | 启动时预创建 worker pool | 需要 bounded buffer 和同步 |

Prethreaded server 优势：

- 避免每个连接都动态创建 thread。
- master 能快速回到 `accept`。
- worker 数量可控，不会无限创建线程。
- 对 I/O-bound 服务，多个 workers 能覆盖等待 I/O 的时间。

复习课提醒：prethreaded server 期末肯定会考。

## Part 5：Performance Trade-offs

来源：`EXE/concurrent&network.pdf` Part 5。

worker 数不是越多越好。

如果任务 CPU-bound：

- worker 数接近 CPU cores 通常比较合适。
- 太多 threads 会增加 context switch、调度和同步开销。

如果任务 I/O-bound：

- worker 数可以适当大于 CPU cores。
- 因为很多 worker 可能阻塞在 network/disk I/O 上。

答题句式：

> For CPU-bound workloads, too many workers mainly add scheduling overhead. For I/O-bound workloads, more workers can help overlap waiting time, but an excessive number still wastes memory and causes contention.

## Part 6：Fairness and Starvation

来源：`EXE/concurrent&network.pdf` Part 6。

关键区分：

- bounded buffer 可以保证已经入队的 descriptors 按 FIFO 顺序被取出。
- semaphore / scheduler 不一定保证等待的 worker 按 FIFO 顺序被唤醒。

所以：

> FIFO buffer gives FIFO service order among queued descriptors, but it does not guarantee strict FIFO wakeup order among worker threads.

可能问法：

- 是否保证 client 公平？
- 是否保证 worker 公平？
- 是否可能 starvation？

答题思路：

- queued descriptors 的顺序由 buffer 决定。
- blocked threads 的唤醒顺序由 semaphore implementation / OS scheduler 决定，不一定 FIFO。

## Threads / Locks 背景

线程共享进程的：

- code
- data
- heap
- shared libraries
- open files
- installed handlers

线程独有：

- thread ID
- stack
- registers
- PC
- condition codes

锁和同步：

```c
pthread_mutex_lock(&lock);
/* critical section */
pthread_mutex_unlock(&lock);
```

condition variable 标准模式：

```c
pthread_mutex_lock(&mutex);
while (!condition)
    pthread_cond_wait(&cond, &mutex);
pthread_mutex_unlock(&mutex);
```

为什么用 `while`：

- 可能 spurious wakeup。
- 被唤醒后条件可能已被其他线程改变。

## 低优先级背景：Signal / I/O

复习课修正：

- signal 已确定期末不考。
- system-level I/O 大概率不考。

因此这些只作为背景：

- async-signal-safe
- `sigsuspend`
- `fork + dup2 + stdio buffer + O_APPEND`
- descriptor table / open file table / v-node table

如果时间紧，跳过这部分，优先看 Part 1-6。

## 考场检查清单

1. socket API 题：先区分 client 端 `connect` 和 server 端 `bind/listen/accept`。
2. `listenfd` 只监听；`connfd` 才读写。
3. SNAT 题画四步：出去前、出去后、回来前、回来后。
4. private IP 不能公网路由，server 看到 router public IP。
5. bounded buffer：insert 先 `P(slots)`，remove 先 `P(items)`。
6. 拿 `mutex` 前先等资源数量，避免拿锁睡眠。
7. prethreaded server：重点比较 iterative 和 thread-per-connection。
8. performance：CPU-bound 看 cores，I/O-bound 可以更多 workers。
9. fairness：FIFO buffer 不等于 FIFO semaphore wakeup。

## 高频术语

- Client-server model：客户端-服务器模型。
- Socket：套接字。
- Socket pair：连接四元组。
- `listenfd`：监听描述符。
- `connfd`：已连接描述符。
- SNAT: Source Network Address Translation。
- Private IP address：私有 IP 地址。
- Concurrent server：并发服务器。
- Prethreaded server：预线程化服务器。
- Thread pool：线程池。
- Bounded buffer：有界缓冲区。
- Semaphore：信号量。
- Mutex：互斥锁。
- Critical section：临界区。
- Producer-consumer problem：生产者-消费者问题。
- Data race：数据竞争。
- Deadlock：死锁。
- Starvation：饥饿。
- Condition variable：条件变量。
