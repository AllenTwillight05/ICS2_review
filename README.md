# ICS2 Review Notes

这是 ICS2 期末复习用的开卷速查笔记，按复习课提到的六道大题整理。

## 内容

- `review_notes/01_GPU.md`：CUDA/SIMT、线程索引、warp divergence、带宽/计算受限
- `review_notes/02_CACHE.md`：cache 地址拆分、hit/miss、LRU、write-back、miss rate
- `review_notes/03_SCHEDULE.md`：FIFO/SJF/STCF/RR、MLFQ、turnaround/response
- `review_notes/04_VM.md`：页表、TLB、VIPT、mmap、COW、clock、buddy
- `review_notes/05_AGENT.md`：Agent loop、memory、tools、multi-agent
- `review_notes/06_CON_NET.md`：socket、SNAT、bounded buffer、prethreaded server

## 复习建议

优先看每篇开头的“一分钟速查”和“题型对应来源”，再按题型模板刷对应 EXE/QUIZ。

复习课口径下，signal 不作为期末重点，system-level I/O 也大概率不考；CON&NET 重点放在 `concurrent&network` 样题。

## 说明

本仓库只追踪整理后的复习笔记，不包含原始课件、习题 PDF 或参考书。
