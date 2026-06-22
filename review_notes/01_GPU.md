# 01 GPU 复习提纲

## 考试定位

GPU 题大概率不是考复杂 CUDA 语法，而是考你能不能把 CUDA 的并行模型翻译成索引、线程数量、分支发散和带宽估算。重点参考 `courseware/2-3-gpu_*.pdf`、`courseware/review.pdf`、`EXE/EXE2.pdf`、`EXE/gpu_exe.pdf`。

## 核心知识点

GPU（Graphics Processing Unit）适合大量相同操作作用在不同数据上的任务。CPU 更强调低延迟和复杂控制流，GPU 更强调高吞吐（throughput）和高内存带宽（memory bandwidth）。

SM（Streaming Multiprocessor）是 NVIDIA GPU 的基本计算单元，可以粗略类比 CPU core，但一个 SM 内部有很多 CUDA cores、load/store units、special function units、register file、shared memory / L1 cache、warp scheduler。

SIMT（Single Instruction Multiple Threads）是 GPU 的执行模型：一个 warp 中的线程共享同一条指令流，但每个线程有自己的寄存器和数据。NVIDIA 课件中默认 warp size = 32 threads。

CUDA 线程层次（thread hierarchy）：

- Grid：一次 kernel launch 启动的全部 blocks。
- Block：一组 threads，同一个 block 内线程可以较自然地共享资源。
- Thread：实际执行 kernel 代码的最小逻辑单位。
- Kernel：在 GPU 上运行的函数，用 `__global__` 声明。

常用内建变量：

- `blockIdx.x/y/z`：当前 block 在 grid 中的编号。
- `threadIdx.x/y/z`：当前 thread 在 block 中的编号。
- `blockDim.x/y/z`：每个 block 的维度。
- `gridDim.x/y/z`：grid 的维度。

一维全局索引最常考：

```c
int idx = blockIdx.x * blockDim.x + threadIdx.x;
```

三维全局索引最常考：

```c
int x = blockIdx.x * blockDim.x + threadIdx.x;
int y = blockIdx.y * blockDim.y + threadIdx.y;
int z = blockIdx.z * blockDim.z + threadIdx.z;
int idx = z * (width * height) + y * width + x;
```

如果数据规模不一定整除 block size，grid 维度用向上取整：

```c
dim3 gridDim((width  + blockDim.x - 1) / blockDim.x,
             (height + blockDim.y - 1) / blockDim.y,
             (depth  + blockDim.z - 1) / blockDim.z);
```

必须在 kernel 中用 bounds check 防止越界：

```c
if (idx < n) {
    out[idx] = ...;
}
```

## 题型 1：线程数量与有效线程

题面常给：

```c
kernel<<<numBlocks, threadsPerBlock>>>(...);
```

解法：

1. 总线程数 = `numBlocks * threadsPerBlock`。
2. 有效线程数 = 满足 `idx < n` 的线程数，通常就是 `n`。
3. 无效线程数 = 总线程数 - 有效线程数。

例：`<<<16, 64>>>` 且 `n = 1000`。

- 总线程数 = `16 * 64 = 1024`。
- 有效写入 `out` 的线程 = `idx = 0..999`，共 1000。
- 无效线程 = 24。

问某个线程的 `idx`：

```text
idx = blockIdx.x * blockDim.x + threadIdx.x
```

例如 `blockIdx.x = 5, threadIdx.x = 17, blockDim.x = 64`：

```text
idx = 5 * 64 + 17 = 337
```

## 题型 2：最后一个 block 哪些线程无效

通用步骤：

1. 先找最后一个 block 的 `blockIdx.x`。
2. 写出该 block 的索引范围。
3. 与 `idx < n` 比较。

例：`<<<16,64>>>`, `n = 1000`。

最后一个 block 是 `blockIdx.x = 15`：

```text
idx = 15 * 64 + threadIdx.x = 960 + threadIdx.x
```

有效条件 `idx < 1000`，所以：

```text
960 + threadIdx.x < 1000
threadIdx.x < 40
```

有效：`threadIdx.x = 0..39`。  
无效：`threadIdx.x = 40..63`。

## 题型 3：SIMT 分支发散

分支发散（thread divergence / branch divergence）只在同一个 warp 内讨论。若同一个 warp 中一部分 lanes 走 if，另一部分 lanes 走 else，GPU 需要串行执行两个分支路径，每次只激活对应 lanes。

判断步骤：

1. 写出该 warp 覆盖的 `idx` 范围。
2. 先应用外层 `idx < n`，得到真正活跃 lanes。
3. 再看内层条件，例如 `idx % 2 == 0`。
4. 分别数每条路径 active lanes。

例：第一个 warp 覆盖 `idx = 0..31`，条件是：

```c
if (idx % 2 == 0)
    out[idx] = 2.0f * v;
else
    out[idx] = v + 1.0f;
```

偶数 16 个 lanes，奇数 16 个 lanes，因此发生 divergence。SIMT 会执行两个分支，每条分支 16 lanes active。

例：最后 block 第二个 warp 覆盖 `idx = 992..1023`，`n = 1000`：

- 外层 `idx < n` 后，只有 `992..999` 共 8 lanes active。
- 其中偶数 4 个，奇数 4 个。
- 所以内层 even 分支 4 active lanes，odd 分支 4 active lanes，其余 24 lanes 被 bounds check 屏蔽。

## 题型 4：CUDA 改写为 CPU 单线程函数

CUDA kernel 的本质通常是“每个线程处理一个元素”。改写成 CPU 代码时，把 `idx` 变成 for 循环变量。

CUDA：

```c
__global__ void f(float *x, float *y, float *out, int n) {
    int idx = blockDim.x * blockIdx.x + threadIdx.x;
    if (idx < n) {
        float v = x[idx] + y[idx];
        if (idx % 2 == 0)
            out[idx] = 2.0f * v;
        else
            out[idx] = v + 1.0f;
    }
}
```

CPU：

```c
void f_cpu(float *x, float *y, float *out, int n) {
    for (int idx = 0; idx < n; idx++) {
        float v = x[idx] + y[idx];
        if (idx % 2 == 0)
            out[idx] = 2.0f * v;
        else
            out[idx] = v + 1.0f;
    }
}
```

答题句式：CUDA 版本启动许多 GPU threads，每个 thread 负责一个 `idx`；CPU 版本用一个循环顺序处理所有 `idx`。

## 题型 5：带宽受限还是计算受限

操作强度（operational intensity）：

```text
OI = FLOPs / bytes
```

题中常给每个元素：

- 读 `x[idx]`：4 bytes。
- 读 `y[idx]`：4 bytes。
- 写 `out[idx]`：4 bytes。
- 共 12 bytes。
- 做 2 floating-point operations。

则：

```text
OI = 2 / 12 = 1/6 FLOPs/byte
```

判断：每读写很多字节只做很少计算，通常是 bandwidth-bound（带宽受限），不是 compute-bound（计算受限）。

## 题型 6：CUDA 程序流程填空

标准流程：

1. 在 CPU 端准备 host data。
2. `cudaMalloc` 分配 device memory。
3. `cudaMemcpy(..., cudaMemcpyHostToDevice)` 把数据从 host 拷到 device。
4. 设置 `blockDim` 和 `gridDim`。
5. `kernel<<<gridDim, blockDim>>>(...)` 启动 kernel。
6. `cudaDeviceSynchronize()` 等待 GPU 完成。
7. `cudaMemcpy(..., cudaMemcpyDeviceToHost)` 拷回结果。
8. `cudaFree` 释放 device memory。

## 高频英文术语

- GPU: Graphics Processing Unit，图形处理器。
- GPGPU: General Purpose GPU，通用 GPU 计算。
- SM: Streaming Multiprocessor，流式多处理器。
- SIMT: Single Instruction Multiple Threads，单指令多线程。
- Warp: 一组 SIMT 执行线程，课件默认 32 threads。
- Kernel: GPU 上执行的函数。
- Grid / Block / Thread: CUDA 线程层次。
- Device memory: GPU 内存。
- Host memory: CPU 侧内存。
- Thread divergence: 分支发散。
- Memory bandwidth: 内存带宽。
- Operational intensity: 操作强度，FLOPs per byte。

## 考前速记

- 一维索引：`idx = blockIdx.x * blockDim.x + threadIdx.x`。
- 三维线性化：`idx = z * width * height + y * width + x`。
- grid 向上取整：`(n + block - 1) / block`。
- divergence 只在同一 warp 内讨论。
- 先看 bounds check，再看内层分支。
- 读写多、计算少：bandwidth-bound。
