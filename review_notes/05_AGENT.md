# 05 Agent 复习提纲

## 考试定位

AGENT 是老师新加的专题，用户说明暂时没有真题，所以这里按填空/简答准备。重点参考 `courseware/2-16-agent_*.pdf` 和 `courseware/review.pdf`。

复习目标：能用系统课语言解释 AI Agent，而不是只背“LLM 很强”。关键词要能中英对照。

## 一句话理解

AI Agent 可以理解为：

```text
Agent = LLM + Harness
```

LLM（Large Language Model）负责推理、生成计划、决定行动；Harness 是外部执行框架，负责提供上下文、工具调用、权限控制、记忆管理、观察结果回填等系统工程能力。

课件中的 Agent Loop：

```text
goal -> observation -> LLM -> action -> observation -> LLM -> action ...
```

也就是不断观察、思考、行动、再观察。

## Agent Loop 四步闭环

1. 感知（Perception）

接收用户输入、系统提示词、环境状态，从 memory 中加载上下文。它是 Agent 的“感官”。

2. 认知（Cognition）

LLM 推理当前状态，制定计划，判断下一步该做什么。它是 Agent 的“大脑”。

3. 行动（Action）

调用 API、执行代码、查询数据库、调用工具。常见机制是 function calling / tool calling。

4. 观察（Observation）

接收工具结果，把结果追加回上下文，判断任务是否完成；未完成则回到 cognition 继续循环。

填空题记忆：

```text
Perception -> Cognition -> Action -> Observation
```

## Agent Memory 与 CSAPP

智能体记忆（agent memory）可以类比计算机系统中的存储层次。

三个朴素方案：

1. 分层存（hierarchy）

不是所有信息都放在上下文窗口里。快但小的层保存当前关键上下文，慢但大的层保存长期记忆或文件。

2. 按需取（on-demand）

需要时再加载相关信息，而不是一次性读入所有历史。

3. 扔旧的（replacement）

上下文放不下时，要选择淘汰旧的、不常用的或低价值信息。对应系统里的 replacement policy。

关键词：

- hierarchy：分层。
- on-demand loading：按需加载。
- replacement：替换。
- compression：压缩。
- long-term memory：长期记忆。
- working memory / context window：工作记忆 / 上下文窗口。

可答句式：Agent memory 需要像 OS 管理内存一样，在容量、速度和相关性之间折中；不能把所有历史都塞进 prompt，因此需要分层、检索、压缩和替换。

## Agent Tools 与 CSAPP

工具调用不是“模型自己会做所有事”，而是 LLM 决定调用外部工具，由系统执行。

和 CSAPP 的关系：

- MCP / CLI 可以类比 calling convention：工具调用需要约定输入、输出、错误格式。
- 启动工具对应 process creation / execution。
- 调用远程工具对应 network / RPC。
- 工具权限对应 protection / privilege / sandbox。
- 工具失败重试对应 robustness。
- speculative execution 可以类比多路径尝试或预测性执行，但要注意安全和回滚。

关键词：

- Tool calling / Function calling：工具调用 / 函数调用。
- MCP: Model Context Protocol，模型上下文协议。
- CLI: Command-Line Interface，命令行接口。
- API: Application Programming Interface，应用程序接口。
- Sandbox：沙箱。
- Permission control：权限控制。
- Robustness：鲁棒性。

简答题常用句式：LLM 只产生 action intent，真正的 action 由 harness 在权限和协议约束下执行；执行结果再作为 observation 返回给 LLM。

## Multi-Agent

多智能体协作（multi-agent collaboration）的优势：

- 专业分工：不同 Agent 负责写码、审查、测试、检索等。
- 降低复杂度：任务拆解后，每个上下文更清晰。
- 并行高效：多个子任务可同时推进。
- 相互校验：互审互评，减少 hallucination。
- 灵活扩展：按需增加或减少角色。

适用场景：

- 软件研发：coding / review / testing / debugging。
- 深度研究：parallel search + synthesis。
- 多视角决策：debate / voting / evaluation。
- 模拟与仿真：role-playing / social simulation。

和系统课的关系：

- 多进程调度（process scheduling）。
- 并发与同步（concurrency and synchronization）。
- 进程间通信（inter-process communication, IPC）。

## Agent 的系统工程难点

1. Context 管理

上下文窗口有限，需要选择放什么，不放什么。

2. Tool 安全

工具可能读写文件、联网、执行代码，必须有权限控制和隔离。

3. Observation 可信度

工具结果可能失败、截断或过期，Agent 要能检查错误。

4. Planning 与 replanning

计划可能在执行中被新观察推翻，因此需要循环修正。

5. Concurrency

多 Agent 或多工具并行时，需要同步、去重、合并结果，避免竞态。

## 高频英文术语

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
- MCP: Model Context Protocol。
- Sandbox：沙箱。
- Multi-agent collaboration：多智能体协作。
- Hallucination：幻觉。
- IPC: Inter-Process Communication，进程间通信。

## 可能填空/简答

Q: Agent = ?

A: `LLM + Harness`。

Q: Agent Loop 四步？

A: Perception, Cognition, Action, Observation。

Q: 为什么 Agent memory 需要分层？

A: 因为上下文窗口快但小，长期信息多但慢，需要像 memory hierarchy 一样按重要性和使用频率分层管理。

Q: 工具调用为什么需要系统工程？

A: 因为需要协议、进程执行、网络调用、权限、安全和错误处理；LLM 只是决策，harness 才负责可靠执行。

Q: 多智能体有什么好处？

A: 分工、并行、降低单个上下文复杂度、相互校验、易扩展。

## 考前速记

- `Agent = LLM + Harness`。
- Loop：感知、认知、行动、观察。
- Memory：分层、按需、替换、压缩。
- Tools：协议、进程、网络、权限、鲁棒性。
- Multi-Agent：分工、并行、校验、同步。
