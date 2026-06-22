# Review Class 摘要

## 总体考纲

复习课明确说期末暂定 6 道大题，顺序大致是：

1. GPU
2. Cache
3. Schedule
4. Virtual Memory
5. Agent
6. Concurrency & Network

老师反复强调：题目主要来源/参考是 `QUIZ` 和 `EXE`，期末很可能是在这些样题基础上改数据、改场景、换问法。

## GPU

GPU 部分不会考太深，主要考概念和简单 CUDA/SIMT 代码分析。

重点：

- CPU vs GPU：CPU 更擅长复杂控制流，GPU 有大量 ALU，更适合并行计算。
- NVIDIA GPU 由多个 SM（Streaming Multiprocessor）组成。
- 一个 SM 内有 CUDA cores、L1 cache/shared memory、register file、warp scheduler 等。
- SIMT（Single Instruction Multiple Threads）模型：写一个 thread 的逻辑，GPU 并行执行很多 threads。
- Warp 通常是 32 threads，所以 block/thread 维度常取 32 的倍数。
- Kernel launch 时指定 grid/block/thread 维度。
- 需要用 `blockIdx`、`threadIdx`、`blockDim` 计算当前 thread 的全局 index。

可能题型：

- 给一段 CUDA kernel，解释它的逻辑。
- 写非常短的 CUDA kernel。
- 计算总 threads 数、真正写 `out` 的 threads 数。
- 计算某个 `blockIdx/threadIdx` 的全局 index。
- 判断最后一个 block 中哪些 threads 因 `idx < n` 不工作。
- 分析 warp divergence：例如偶数/奇数分支导致同一 warp 内 lanes 分裂。
- 把 CUDA kernel 改写成传统单线程 C/C++ for 循环。
- 计算读写字节数、FLOPs、operational intensity，判断 compute-bound 还是 bandwidth-bound。

复习课最后详细讲的 GPU 样题基本就是 `EXE/gpu_exe.pdf` 的套路，期末 GPU 题大概率只是“数据改改”。

## Cache

Cache 题“肯定会考”，而且会给 cache 结构图/表格让大家判断。

重点：

- Memory hierarchy：register、L1/L2/L3 cache、main memory、disk。
- Cache 有效的根本原因是 locality。
- Cache 由 sets 组成，每个 set 有若干 cache lines，每个 line 存一个 block。
- Physical address 切成三段：tag / set index / block offset。
- 要和 VM 里的 PPN/PPO 划分对比，尤其服务于 VIPT。
- Direct mapped 是每个 set 只有一条 line。
- 做题一定看 valid bit：tag 对上但 valid=0 仍然 miss。
- 按访问顺序更新 cache，前面访问可能已经把某条 line evict 掉。

可能题型：

- 计算 cache 总大小。
- 计算 tag / set index / block offset 位宽。
- 给地址和 cache 表，判断 hit/miss，命中时返回对应 byte。
- 根据实际程序执行过程计算 miss rate。
- 问一些扩展：为什么 miss、如何优化、硬件参数如何影响 miss rate。

常见坑：

- 忘看 valid bit。
- 忘记 LRU/访问顺序导致某 line 已被刷掉。
- 拿完整地址直接和 tag 比。

## Schedule

Schedule 相对简单，但不会只考最基础 FIFO/SJF/STCF/RR，还会考多级队列。

重点算法：

- FIFO / FCFS：先来先服务。
- SJF：Shortest Job First，非抢占。
- STCF：Shortest Time-to-Completion First，抢占式。
- Round Robin：按 time slice 轮转。
- MLFQ / MLQ：多级反馈/多级调度队列，具体规则会在题目里定义。

MLFQ 常见机制：

- 高优先级抢占低优先级。
- 同级队列内通常 RR。
- 高优先级时间片用完后降级。
- 定期 priority boost，防止 starvation。

可能题型：

- 给 job arrival trace，写出调度顺序。
- 算每个 job 的 turnaround time、response time 等 metric。
- 问某时刻 CPU 正在运行哪个 job。
- 定性解释 convoy effect。
- 定性解释 starvation 以及 priority boost 的作用。

老师明确建议参考 EXE7 里的两个题：普通 scheduling policy 和 MLFQ scheduling policy。

## Signal / System-Level I/O

这是复习课里最重要的修正之一：

- Signal 部分已经确定期末完全不考。
- System-level I/O 大概率也不会考。

老师仍然带了一点 I/O 背景：

- descriptor table。
- open file table。
- v-node / inode table。
- `dup2(fd4, fd1)` 会让 fd1 指向 fd4 的 open file table entry。

但这部分在期末优先级应显著低于 GPU/Cache/Schedule/VM/CON&NET 样题。

## Agent

Agent 题老师也不确定具体怎么出，大概率是概念/定性题。

重点：

- 根据 PPT 复习。
- 熟悉如何设计一个 Agent。
- 熟悉 Agent loop。
- 与 agent lab 关系密切。

可能题型：

- 填空或简答 Agent 的组成。
- 简述 Agent loop 如何设计。
- 简述 memory/tool/multi-agent 等系统设计问题。

## Virtual Memory

VM 是重点，内容多，题型很像 QUIZ。

重点：

- Page：普通页常见大小 4KB。
- Virtual memory 把连续虚拟地址空间映射到灵活的物理页。
- 有些 virtual pages 已 cached in physical memory，有些 swapped out，有些 on-demand 尚未加载。
- Page fault 可能考三类情况：
  - 物理内存放不下，页面被 swap out，再访问时 swap in。
  - On-demand paging，例如 `mmap` 后第一次访问才加载页面。
  - Segmentation fault，访问非法地址段或权限错误。
- VA 切成 VPN / VPO，PA 切成 PPN / PPO；VPO 和 PPO 相同。
- MMU 负责地址翻译。
- TLB 是地址翻译的 cache。
- Multi-level page table 节省页表内存。
- VIPT：地址翻译和 cache lookup 并行，关键条件是 page offset 要覆盖 cache index + cache offset，即 `CI + CO <= VPO`。
- `mmap` 可以减少 kernel 到 user buffer 的一次 copy，也体现 demand paging。
- Copy-on-Write：fork 后父子共享物理页，写某页时才复制该页。
- Page replacement：LRU 思想，但精确 LRU 开销大，所以用 clock algorithm 近似。
- Buddy system：内存分配速度与 fragmentation 的折中，按 2 的幂 split/merge。

可能题型：

- 基础 bit 位数计算。
- 多级页表占用内存计算。
- VIPT 机制判断。
- `mmap` / on-demand paging 机制解释。
- `fork` / COW 机制解释。
- Clock algorithm 按访问序列一步步写物理页状态和 reference bit/tag。
- Buddy system / allocator 小题。

老师说主体题型仍可参考 QUIZ，可能补考 QUIZ 没覆盖到的 allocator 部分。

## Concurrency & Network

这是较新的部分，不做完整复习，但给了样题方向。老师明确说 prethreaded server 期末肯定会考。

重点：

- Socket 接口：`connect`、`bind`、`listen`、`accept` 等，基本是概念解释。
- SNAT：公网/内网 IP 地址转换，server 看到公网 IP 而不是私网 IP。
- 用 semaphore / lock 保护 critical section，例如 bounded FIFO buffer 的入队/出队。
- Prethreaded server：维护 thread pool，用 bounded buffer 在 master 和 workers 之间传递 connected descriptors。

可能题型：

- 解释 socket API。
- 填 SNAT 前后 source/destination IP:port。
- 解释为什么 server 不直接看到 private IP。
- 写 bounded buffer 的 P/V 顺序。
- 说明为什么 P/V 顺序不对会 deadlock。
- 比较 prethreaded server、iterative server、thread-per-connection server 的优劣。
- 讨论 thread pool 大小和 CPU cores / I/O-bound workload 的性能权衡。

## 对原复习笔记的影响

需要调整：

- `06_CON_NET.md`：原来把 signal safety、sigsuspend、I/O tracing 放得太重。复习课明确 signal 不考，system-level I/O 大概率不考，所以这些应降为背景/低优先级；核心改为 socket、SNAT、bounded buffer、prethreaded server。
- `04_VM.md`：原来已覆盖 clock 和 buddy，但需要把 clock algorithm、buddy system 的考试可能性标得更明确。

基本不用改：

- `01_GPU.md`：和复习课 GPU 样题高度一致。
- `02_CACHE.md`：方向一致，尤其 valid bit、地址三段、miss rate。
- `03_SCHEDULE.md`：方向一致，重点就是 trace + metric + convoy/starvation。
- `05_AGENT.md`：方向一致，概念题/Agent loop 即可。
