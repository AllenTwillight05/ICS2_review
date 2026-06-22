# 03 Schedule 复习提纲

## 考试定位

Scheduling 题以手工画时间线和算指标为主。重点参考 `courseware/2-13-sched.pdf`、`courseware/review.pdf`、`EXE/EXE7.pdf` 中的 scheduling problem。

高频题型：

- FIFO / SJF / STCF / RR 的执行时间线。
- average turnaround time 和 average response time。
- 某一时刻 CPU 上运行哪个 job。
- convoy effect。
- MLFQ trace、starvation、priority boost。

## 核心指标

周转时间（turnaround time）：

```text
T_turnaround = T_completion - T_arrival
```

响应时间（response time）：

```text
T_response = T_first_run - T_arrival
```

答题时要为每个 job 记录：

- arrival time：到达时间。
- run time / burst time：需要 CPU 运行多久。
- first run time：第一次被调度上 CPU 的时间。
- completion time：完成时间。

平均值：

```text
average turnaround = sum(T_turnaround) / number of jobs
average response = sum(T_response) / number of jobs
```

## FIFO / FCFS

FIFO（First In, First Out）也叫 FCFS（First Come, First Served）。

规则：按 arrival order 运行，非抢占（non-preemptive），一个 job 开始后运行到完成。

优点：简单。  
缺点：容易产生 convoy effect。

Convoy effect：一个长 job 排在前面，后面许多短 job 被迫等待，平均 response / turnaround 被拉高。

做题方法：

1. 按 arrival time 排队。
2. CPU 空闲时取队首。
3. 运行到完成。
4. 算每个 job 的 completion / first_run。

## SJF

SJF（Shortest Job First）是非抢占调度。

规则：每次 CPU 空闲、需要选择下一个 job 时，在已到达的 jobs 中选择 run time 最短的。

注意：SJF 不会因为新来的短 job 打断当前 job。若一个长 job 已经开始，新来的短 job 只能等它完成。

特点：

- 在所有 job 同时到达、运行时间已知时，SJF 对 average turnaround 很好。
- 对 response time 不一定好，长 job 可能被短 job 挤到后面。

## STCF

STCF（Shortest Time-to-Completion First）是抢占版 SJF，也常叫 SRTF（Shortest Remaining Time First）。

规则：每当新 job 到达时，重新比较所有 ready jobs 的 remaining time，选择剩余时间最短的运行。

做题方法：

1. 在每个 arrival time 和 completion time 处重新决策。
2. 当前 job 若不再是 remaining time 最短，就被 preempt。
3. 画时间线时标出每段运行。

典型优势：如果新来的 jobs 都比当前剩余时间短，它们可以马上运行，response time 可能为 0。

典型缺点：长 job 可能一直被短 job 抢占，出现 starvation 风险。

## RR

RR（Round Robin）是时间片轮转。

规则：

- 每个 job 最多运行一个 time quantum。
- quantum 用完还没结束，就放到 ready queue 队尾。
- 如果 job 提前完成，不用跑满 quantum。

做题关键：

1. 明确 quantum，例如 2 ms。
2. 每个时间片结束时，把新到达的 job 加入队列。
3. 未完成的当前 job 通常放队尾。
4. 记录 first_run 和 completion。

RR 的 trade-off：

- quantum 小：response time 通常好，但 context switch overhead 高。
- quantum 大：overhead 小，但接近 FIFO，response time 变差。

Context switch cost 可能包括 privilege switch、save/restore context、TLB flush、pipeline state flush。若题面说忽略，就不要算。

## I/O 与调度

当 job 发起 I/O：

- 它会阻塞，不再占用 CPU。
- 调度器应让其他 ready job 使用 CPU。
- I/O 完成后，job 回到 ready queue。

核心思想：overlap CPU and I/O。好的 schedule 会让一个 job 做 I/O 时，另一个 job 用 CPU，提高资源利用率。

## MLFQ

MLFQ（Multi-Level Feedback Queue）试图在不知道 job 长短的情况下，同时兼顾 turnaround 和 response。

课件规则版本：

1. 如果 `Priority(A) > Priority(B)`，运行 A。
2. 如果优先级相同，用 RR。
3. 新 job 进入最高优先级队列。
4. 一个 job 在某一层用完 time allotment 后，降低优先级。
5. 每隔一段时间 S，把所有 jobs 提升到最高优先级，防止 starvation。

做 MLFQ trace：

1. 新 job 一律进最高队列。
2. 高优先级队列非空时，低优先级 job 不运行。
3. 同队列内按 RR。
4. job 用完该层 allotment 才降级。
5. 若有 priority boost，全部回最高队列。

常见问法：是否会 starvation？

标准答法：如果高优先级队列持续有短 job 到达，低优先级长 job 可能长期得不到 CPU。Rule 5 的 periodic priority boost 可以缓解或避免 starvation。

## EXE7 题型模板

若题目给：

```text
Policy | Avg turnaround | Avg response | CPU at t=6 | CPU at t=12
FIFO
SJF
STCF
RR
```

答题流程：

1. 分别画四条时间线。
2. 对每个 job 标 first_run 和 completion。
3. 计算每个 job 的 turnaround / response。
4. 填平均值。
5. 查时间线中 `t=6` 和 `t=12` 落在哪一段。

注意边界：若某段是 `t=4-6`，通常表示运行到 6 之前；`t=6` 时可能已经切到下一段。按题目或课上习惯处理，最好在草稿上用半开区间 `[start, end)`。

## 高频英文术语

- Scheduling policy：调度策略。
- Mechanism：机制，例如 context switch。
- Policy：策略，例如选择哪个 job。
- Turnaround time：周转时间。
- Response time：响应时间。
- FIFO / FCFS：先来先服务。
- SJF: Shortest Job First，短作业优先。
- STCF: Shortest Time-to-Completion First，最短剩余完成时间优先。
- RR: Round Robin，时间片轮转。
- Time quantum / time slice：时间片。
- Preemption：抢占。
- Convoy effect：护航效应 / 长任务堵住短任务。
- MLFQ: Multi-Level Feedback Queue，多级反馈队列。
- Starvation：饥饿。
- Priority boost：优先级提升。

## 考前速记

- `turnaround = completion - arrival`。
- `response = first_run - arrival`。
- FIFO/SJF 是非抢占；STCF/RR 是抢占。
- SJF 只在 CPU 空闲时选择；STCF 每次新 job 到达都可能抢占。
- RR 必须维护 ready queue。
- MLFQ：新 job 最高优先级，用完 allotment 降级，周期 boost 防饥饿。
