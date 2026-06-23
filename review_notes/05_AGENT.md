# 05 Agent 开卷速查

## 一分钟速查

Agent 目前没有 EXE/QUIZ 真题，老师在复习课里也说不太确定最后怎么出，大概率是概念/定性题。复习重点不是死背 buzzwords，而是会用系统课语言解释 Agent 是怎么工作的。

来源优先级：

- `courseware/2-16-agent_*.pdf`：Agent 专题课件。
- `courseware/review.pdf`：Agent review slides。
- `reviewClass.txt / reviewClass_summary.md`：复习课口径，重点是 Agent 设计和 Agent loop。

题型对应来源：

| 题型 | 主要来源 |
|---|---|
| `Agent = LLM + Harness`、Agent loop 四步 | `courseware/review.pdf`；`2-16-agent_*.pdf` |
| Agent memory / tools / multi-agent 与 CSAPP 的联系 | `courseware/2-16-agent_*.pdf` |
| Agent 设计类简答题 | `reviewClass.txt` |

核心句：

> Agent 不是“一个会聊天的模型”，而是 LLM 在 harness 支撑下，围绕 goal 持续感知、推理、行动、观察的系统。

## 一句话定义

```text
Agent = LLM + Harness
```

- LLM：负责推理、计划、决定下一步做什么。
- Harness：负责上下文、工具调用、权限、安全、记忆管理、执行结果回填。

老师如果问“为什么不能只有 LLM”，标准思路就是：

> 因为真正完成任务还需要工具、状态、权限、执行和反馈闭环，这些都属于 harness 的系统工程部分。

## Agent Loop

课件中的四步闭环：

```text
Perception -> Cognition -> Action -> Observation
```

### 1. Perception

- 接收用户输入
- 读取系统提示词
- 从 memory 载入上下文

相当于 Agent 的“感官”。

### 2. Cognition

- LLM 分析当前状态
- 制定计划
- 决定下一步行动

相当于 Agent 的“大脑”。

### 3. Action

- 调 API
- 调工具
- 执行代码
- 查数据库

相当于 Agent 的“手”。

### 4. Observation

- 接收工具执行结果
- 把结果追加进上下文
- 判断任务是否完成

如果没完成，就回到 Cognition 继续循环。

## 题型模板

### 1. 填空：Agent = ?

来源：`courseware/review.pdf`。

答案：

```text
Agent = LLM + Harness
```

### 2. 填空：Agent Loop 四步

来源：`courseware/review.pdf`、`2-16-agent_*.pdf`。

答案：

```text
Perception, Cognition, Action, Observation
```

### 3. 简答：为什么 Agent memory 需要分层

来源：`courseware/2-16-agent_*.pdf`。

关键词：

- hierarchy
- on-demand loading
- replacement
- compression

答题思路：

> 上下文窗口快但小，长期信息多但慢，不能把所有历史都塞进 prompt。要像 memory hierarchy 一样分层管理，并按需加载、压缩、替换。

### 4. 简答：Agent tools 与系统课有什么关系

来源：`courseware/2-16-agent_*.pdf`。

对应关系：

- tool calling / function calling -> calling convention
- 启动工具 -> process creation / execution
- 远程工具 -> network / RPC
- 权限与隔离 -> protection / sandbox
- 错误恢复 -> robustness

答题句式：

> LLM 只决定 action intent，真正的 action 由 harness 在协议、权限和执行环境约束下完成；结果再回流为 observation。

### 5. 简答：Multi-Agent 的好处

来源：`courseware/2-16-agent_*.pdf`。

高频点：

- 分工
- 并行
- 降低单个上下文复杂度
- 互相校验
- 易扩展

一句话：

> 多智能体协作像多进程/多线程分工一样，可以把复杂任务拆开，让不同角色并行推进并互相检查。

## Agent memory 与 CSAPP

Agent memory 可类比系统中的存储层次：

1. 分层存（hierarchy）
2. 按需取（on-demand）
3. 扔旧的（replacement）
4. 必要时压缩（compression）

常见类比：

- context window -> working memory
- 长期记忆库 -> slower / larger storage
- 检索 -> demand paging / on-demand loading
- 替换旧上下文 -> replacement policy

## Multi-Agent 与系统课

课件里强调的联系：

- process scheduling
- concurrency and synchronization
- IPC

换句话说，Agent 系统不是“只靠模型本身”，而是要处理：

- 谁先做
- 谁和谁通信
- 并发结果怎么合并
- 工具失败时如何恢复

## 考场检查清单

1. 一上来先记住：`Agent = LLM + Harness`。
2. Agent Loop 四步必须能默写。
3. 简答题尽量用系统课语言，不要只说“模型很聪明”。
4. memory 题想到 hierarchy / on-demand / replacement。
5. tools 题想到 process / network / permission / robustness。
6. multi-agent 题想到分工、并行、校验、同步。

## 高频术语

- Agent：智能体。
- LLM: Large Language Model，大语言模型。
- Harness：执行框架 / 外部支撑系统。
- Agent loop：智能体循环。
- Perception：感知。
- Cognition：认知。
- Action：行动。
- Observation：观察。
- Tool calling / Function calling：工具调用 / 函数调用。
- Memory hierarchy：存储层次。
- On-demand loading：按需加载。
- Replacement policy：替换策略。
- Compression：压缩。
- Sandbox：沙箱。
- Multi-agent collaboration：多智能体协作。
- Hallucination：幻觉。
- IPC: Inter-Process Communication，进程间通信。
