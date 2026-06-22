# 02 Cache 复习提纲

## 考试定位

Cache 是真题/练习题最稳定的计算题来源。重点参考 `courseware/2-6-locality_*.pdf`、`courseware/2-7-cache_*.pdf`、`courseware/review.pdf`、`EXE/EXE3.pdf`、`EXE/EXE4.pdf`、`QUIZ/ics2-quiz.pdf`。

高频题型：

- cache 总容量计算。
- 地址拆分：tag / set index / block offset。
- 给定 cache 表，判断 hit / miss 和返回 byte。
- LRU 与 write-back 下的状态更新。
- 程序访存 miss rate。
- 硬件参数或地址布局如何优化 miss rate。

## 核心知识点

局部性（locality）是 cache 有效的根本原因：

- 时间局部性（temporal locality）：最近访问过的数据很可能再次访问。
- 空间局部性（spatial locality）：访问某地址后，很可能访问附近地址。

Cache 是按 block / line 传输和存储的。一次 miss 加载整个 block，而不是只加载一个 byte。

基本参数：

- `S = 2^s`：sets 数。
- `E`：每个 set 的 lines 数，也叫 associativity。
- `B = 2^b`：每个 block 的 bytes 数。
- `m`：物理地址位数。
- `C = S * E * B`：cache data capacity，不含 tag / valid / dirty 等元数据。

地址拆分：

```text
| tag: t bits | set index: s bits | block offset: b bits |
```

其中：

```text
b = log2(B)
s = log2(S)
t = m - s - b
```

访问流程：

1. 用 set index 找到 set。
2. 在 set 内比较 valid bit 和 tag。
3. 命中（hit）时，用 block offset 取 byte。
4. 未命中（miss）时返回通常填 `-` 或 `--`，并把内存 block 装入 cache。

## 题型 1：容量与地址拆分

例：16-bit machine，2-way set associative，block size 8 bytes，2 sets。

```text
C = S * E * B = 2 * 2 * 8 = 32 bytes
b = log2(8) = 3
s = log2(2) = 1
t = 16 - 1 - 3 = 12
```

答：

```text
tag 12 bits, set-index 1 bit, block-offset 3 bits
```

例：64-bit machine，4 sets，8-byte block，2-way。

```text
b = 3
s = 2
t = 64 - 2 - 3 = 59
```

## 题型 2：给地址判断 hit / miss

最稳步骤：

1. 先算 `block offset`：地址低 `b` 位。
2. 再算 `set index`：offset 上面的 `s` 位。
3. 剩余高位是 `tag`。
4. 到对应 set 里找 valid=1 且 tag 相同的 line。
5. 如果 hit，返回 `Byte[offset]`。
6. 如果 miss，返回 `-`；若题目要求继续跟踪，要更新 cache 状态。

快速算法：

```text
block_number = address / B
set_index = block_number % S
tag = block_number / S
offset = address % B
```

注意：题目表格里 tag 常以十六进制给出，不要直接拿完整地址比 tag。

## 题型 3：LRU 与 write-back 状态更新

LRU（Least Recently Used）替换：

- hit：被访问 line 变成最近使用。
- miss 且 set 有 invalid line：优先填 invalid line。
- miss 且 set 满：替换最久未使用 line。

Write-back（写回）：

- 写 hit：修改 cache line，并把 dirty bit 置 1，不立刻写内存。
- 写 miss：通常题目默认把 block 装入 cache 后写入，dirty 置 1。
- dirty line 被 eviction 或 flush 时，才写回内存。

题目问“最终有多少 cache lines 写回内存”时：

```text
总写回数 = 过程中被 eviction 的 dirty lines + 最后 flush 时仍 dirty 的 lines
```

EXE4 解答中出现过：最后 dirty lines 有 4 条，加上过程中 1 次 dirty eviction，总计 5 次写回。

## 题型 4：程序 miss rate

答题步骤：

1. 明确只考虑 data cache，忽略 instruction cache。
2. 写出每次循环访问的地址序列。
3. 对每个地址算 set / tag / offset。
4. 跟踪每个 set 的 line 状态。
5. 统计 hits 和 misses。
6. `miss rate = misses / total accesses`。

例如 `short A[N], B[N], C[N]`，`short` 是 2 bytes：

```c
C[i] = A[i] * B[i] + C[i];
```

一次循环至少涉及：

- read `A[i]`
- read `B[i]`
- read `C[i]`
- write `C[i]`

如果题目认为读写 `C[i]` 都是 data access，就要分别算；如果答案表述给了具体访问序列，按题面为准。

常见陷阱：`A[0] = 0xa00, B[0] = 0xb00, C[0] = 0xc00` 这类地址低位可能完全一样，会映射到同一个 set，导致 conflict misses。

## 题型 5：为什么用中间位做 set index

标准回答：

如果用高位做 index，连续内存块的高位很长时间不变，顺序扫描数组时大量相邻 blocks 会映射到同一个 set，其他 sets 空着，cache 利用率很差。用中间位作为 set index，可以让相邻 blocks 分散到不同 sets，更好利用整个 cache。

关键词：

- high-order bits：高位。
- middle bits：中间位。
- sequential scan：顺序扫描。
- conflict miss：冲突不命中。

## 题型 6：硬件参数如何改善 miss rate

常见选项：

1. Double the number of sets while keeping cache size unchanged.
2. Double the block size while keeping cache size unchanged.
3. Increase cache size by doubling lines per set / associativity.

判断逻辑：

- 增大 block size：可能利用空间局部性，顺序访问数组时 miss rate 下降；但 block 变大而总容量不变时，line 总数减少，冲突可能更严重。
- 增大 associativity / lines per set：通常减少 conflict misses。
- 增大 sets：如果总容量不变，会减少 associativity 或 block 数，效果要看具体地址映射；不能机械认为一定更好。

对于连续访问 `short` 或 `int` 数组，block size 增大常常能一次带入更多相邻元素，因此有利于空间局部性。

对于多个数组首地址映射到同一 set 的情况，增加 associativity 或调整数组起始地址更直接。

## 题型 7：软件优化 cache hit rate

高频回答方向：

- 调整数组起始地址（padding / alignment），让多个数组不要映射到同一个 set。
- 改变循环顺序，让内层循环访问连续地址。
- blocking / tiling，提高局部性。
- 合并或重排访问，减少反复冲突。

QUIZ 中的典型答法：移动某个数组的起始地址，例如让 `C[0]` 从 `0xC00` 改到 `0xC08`，使 `A/B/C` 不全部落在同一个 set。

## 高频英文术语

- Cache hit / miss：缓存命中 / 不命中。
- Cache line / block：缓存行 / 块。
- Set associative cache：组相联缓存。
- Direct-mapped cache：直接映射缓存，`E = 1`。
- Fully associative cache：全相联缓存，`S = 1`。
- Valid bit：有效位。
- Dirty bit：脏位。
- Tag：标记。
- Set index：组索引。
- Block offset：块内偏移。
- LRU: Least Recently Used，最近最少使用。
- Write-back：写回。
- Write-through：写直达。
- Conflict miss：冲突不命中。
- Capacity miss：容量不命中。
- Cold miss / compulsory miss：冷不命中 / 强制不命中。

## 考前速记

- `C = S * E * B`。
- `b = log2(B)`, `s = log2(S)`, `t = m - s - b`。
- hit 必须同时满足 valid=1 且 tag 相同。
- miss 时返回 byte 通常写 `-`。
- dirty 只有 eviction 或 flush 才写回。
- 程序题先列访存序列，再算 set/tag，不要凭感觉。
