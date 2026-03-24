# Chapter 1：一条消息的奇幻漂流：从 Channel 到工具执行再到回复 📡🧠🛠️

上一章讲的是全景。这一章不再俯视，而是盯住一条消息，看它怎样穿过 IronClaw 的输入面、决策面、行动面，再回到用户手里。

我们用一个具体场景来讲：

> 周五晚上，你在 WhatsApp 里给 IronClaw 发了一句话：
> “帮我检查一下当前工作区里和 webhook 相关的代码，然后总结一下哪里最脆弱。”

这句话从外部世界进入 IronClaw 后，不会直接扔给模型。它会先被收编成统一消息，再和系统 prompt、skills、tool definitions、cost guard、safety layer 一起进入 agentic loop。也就是说，**真正运行的不是“消息 → LLM”两步，而是“消息 → runtime 编排 → LLM → 工具 → 再推理 → 回复”。**

## 1.1 第一站：Channel 不是输入源，而是协议适配器

在 IronClaw 里，Channel 的职责非常明确：它不负责思考，它只负责把外部来源的事件转换成统一的 `IncomingMessage`。

这背后有一个关键判断：Telegram、Slack、CLI、HTTP webhook、Web UI，看起来是不同产品形态，但对 agent runtime 来说，它们都只是“消息输入协议”。

所以 `src/channels/mod.rs` 里最重要的不是某个实现，而是统一抽象：

- `Channel`
- `IncomingMessage`
- `OutgoingResponse`
- `StatusUpdate`
- `MessageStream`

这几个类型把外部世界切平了。

换句话说，IronClaw 的第一层架构判断是：**先消灭渠道差异，再谈智能。**

## 1.2 第二站：ChannelManager 把世界变成一个流

如果你只有一个 channel，事情很简单；难点在于 IronClaw 同时支持多个 channel，而且还允许后台系统自己往主循环里塞消息。

`src/channels/manager.rs` 做的就是这件事：

- 启动每个 channel。
- 拿到每个 channel 返回的 `MessageStream`。
- 用 `select_all` 把多个 stream 合成一个统一流。
- 额外再并进一个 injection channel，让后台任务也能投递消息。

这个 injection channel 很重要。它意味着进入 agent loop 的不只是“真人发来的话”，还可能包括：

- job monitor 的状态回推
- heartbeat 触发后的检查任务
- routine engine 触发的新任务
- 系统内部的恢复或提醒消息

所以从第二站开始，IronClaw 的世界观就已经和普通聊天机器人分开了：**agent 接收的是事件流，不只是聊天消息流。**

## 1.3 第三站：Agent loop 是总控，不是全部逻辑

很多教程喜欢把 agent 的核心理解成一个 while loop。但在 IronClaw 里，`src/agent/agent_loop.rs` 更像总调度台，而不是“所有事情都在这里发生”的大循环。

它主要负责：

- 启动 channels，拿到合并后的消息流。
- 启动 self-repair、heartbeat、routine 这些后台机制。
- 在 `tokio::select!` 中同时等待 Ctrl+C 和新消息。
- 收到消息后调用 `handle_message()`。
- 在退出时做 scheduler 和 channel 的清理。

这说明 IronClaw 把核心控制逻辑拆得很清楚：

- `agent_loop.rs` 管总线。
- `router.rs` 管显式命令。
- `dispatcher.rs` 管工具调用循环。
- `worker.rs` 管单个 job 的实际执行。
- `session` / `session_manager` 管线程态和审批态。

这和把所有逻辑塞进一个超大 ReAct loop 的写法差别很大。它更像一个异步事件系统里的控制器。

## 1.4 第四站：Router 只识别显式命令，不假装理解自然语言

`src/agent/router.rs` 很克制。它不试图用脆弱的规则去猜自然语言意图，而是只做一件确定的事：**识别显式命令前缀**。

例如：

- `/job` → 创建任务
- `/status` → 查状态
- `/cancel` → 取消任务
- `/list` → 列任务
- `/help` → 帮助或 job help

除此之外，自然语言消息不会被这里做复杂模式匹配，而是进入后面的 LLM 推理回路。

这个选择很成熟，因为经验上自然语言路由如果写太早、写太死，后面一定变成 bug 温床。IronClaw 的态度很明确：

- 确定性能用规则解决的，用规则。
- 需要语义理解的，交给模型。

这能让 router 长期保持小而稳定。

## 1.5 第五站：handle_message 先把消息变成“提交物”

在 `agent_loop.rs` 的 `handle_message()` 里，IronClaw 没有直接调用 LLM，而是先做了一层 submission parsing。

这一步的价值在于把输入分类成不同提交类型，例如：

- 用户输入
- 系统命令
- 审批回复
- 线程操作
- 可能的系统内部事件

这样做有两个好处：

1. 主循环不需要把所有分支揉成一团。
2. 同一个 runtime 可以接纳人类消息、系统命令、审批反馈等多种输入，而不用为每种输入单独造入口。

在一个支持 tool approval、thread interruption、background jobs 的系统里，这层抽象几乎是必需的。

## 1.6 第六站：真正的 agentic loop 在 dispatcher 里

如果说 `agent_loop.rs` 是交通警察，那么 `src/agent/dispatcher.rs` 才是最像“agent 大脑工作现场”的地方。

这里的核心循环是：

1. 组装 context。
2. 调用 LLM。
3. 如果返回文本，直接结束。
4. 如果返回 tool calls，先预检，再执行，再把结果塞回上下文。
5. 再次调用 LLM。
6. 重复，直到得到最终文本，或者达到工具调用上限。

但它不是裸循环，它在每一轮外面包了很多结构：

- 系统 prompt 注入
- skill context 注入
- tool definitions 动态刷新
- trust-based tool attenuation
- cost guard 检查
- group chat 模式
- approval gating
- hook 调用
- safety sanitize
- interrupt 检查

所以更准确地说，这不是一个“LLM 会不会调工具”的简单流程，而是一个**受预算、受权限、受中断、受安全层约束的推理循环**。

## 1.7 第七站：系统 prompt 不是常量，而是工作区资产

一个很容易被忽略的点是，dispatcher 在每轮推理前会先尝试从 workspace 读取系统 prompt。

这意味着系统 prompt 不是硬编码在二进制里的一段字符串，而是和本地工作区绑定的上下文资产。像这些文件都可能参与构造 agent 的“人格”和“操作边界”：

- `AGENTS.md`
- `SOUL.md`
- `HEARTBEAT.md`
- 其他 workspace identity files

这个设计很关键，因为它让 agent 的行为框架可以被用户在本地显式管理，而不必每次都在模型请求前拼接一堆临时 prompt。

## 1.8 第八站：Skills 不只是提示词，它们还会削权限

IronClaw 的 skills 不只是往 prompt 里加几段说明，它还会参与工具衰减。

在 dispatcher 里，active skills 被选中后，会发生两件事：

1. 它们的 prompt 内容被包进 skill context 注入到推理上下文。
2. 当前轮可见的工具定义会被 `attenuate_tools()` 收缩。

这一步很重要，因为它把“提示影响行为”推进到“提示影响能力边界”。

也就是说，skills 在 IronClaw 里不是纯认知层资产，它们还是权限层资产。这个设计在后面的信任天花板机制里会更明显。

## 1.9 第九站：工具调用前有三段闸门

当 LLM 给出 tool calls 后，dispatcher 不会立刻执行，而是先进入一个三阶段流程。

### 1.9.1 Preflight

预检阶段按原顺序扫描每个 tool call，主要检查：

- 这个工具是否存在。
- hook 会不会拒绝。
- 当前参数是否需要人工审批。
- 是否应被 auto-approve 策略放行。

如果某个工具需要审批，dispatcher 会先停下来，把 `PendingApproval` 记录进 session，而不是莽撞执行。

### 1.9.2 Parallel Execution

通过预检的工具，如果数量大于 1，就会并行执行。这里 IronClaw 用的是 `JoinSet`，而不是自己手搓线程池。

这使得多个工具调用可以并发跑，同时仍然保留每个 tool call 的结果槽位，后面好按原顺序回填上下文。

### 1.9.3 Post-flight

执行完后，再按原始顺序把结果写回：

- 原始输出先过 safety layer。
- 如果需要，会包成适合 LLM 读取的 XML 化 tool result。
- 某些结果会触发 auth mode。
- 每次工具结果都被放回 `context_messages`，供下一轮推理使用。

这三段结构看起来啰嗦，但工程上很值，因为它清楚地区分了：

- 哪些是“能不能执行”的问题。
- 哪些是“怎么并发执行”的问题。
- 哪些是“结果怎么喂回模型”的问题。

## 1.10 第十站：Worker 是 job 的执行肌肉

对交互式对话来说，dispatcher 已经很核心；但对后台 job 来说，`src/agent/worker.rs` 才是执行肌肉。

Worker 会持有一组打包好的依赖：

- ContextManager
- LLM provider
- SafetyLayer
- ToolRegistry
- Database
- HookRegistry
- timeout 配置
- planning 开关
- SSE 广播器

这说明 IronClaw 对 job 的理解不是“消息线程的附属物”，而是一个可以独立运行、持久化状态、产生日志、向 Web UI 推送事件的执行单元。

更具体地说，worker 负责：

- 读取 job context
- 执行 reasoning
- 记录事件到数据库
- 广播 SSE 到 web gateway
- 更新 job 状态
- 把工具结果和中间消息保存下来

这就是为什么 IronClaw 能把聊天、后台任务、web 界面监控统一起来。

## 1.11 第十一站：状态更新和回复不是一回事

很多系统只处理“最终回复”。IronClaw 把 `StatusUpdate` 单独建模，意味着它承认一个事实：

> 对用户而言，系统正在想、正在调用工具、正在等待授权、已经完成，这些状态本身就是产品体验的一部分。

所以 channel 不只要会 `respond()`，还要会处理状态消息。这对于：

- Web UI 的实时展示
- 长任务的中间反馈
- 需要审批的工具调用
- background job 的进度流

都非常关键。

也正因为有这个建模，IronClaw 才能把“thinking”“tool started”“auth required”这些状态往不同 channel 推回去，而不必都塞进最终文本回复里。

## 1.12 一条消息的完整时序

把前面几个站点压缩成一条链，大致是这样：

```text
WhatsApp / Slack / CLI / Web
  → Channel implementation
  → IncomingMessage
  → ChannelManager.select_all()
  → Agent.handle_message()
  → SubmissionParser
  → Router / thread ops / dispatcher
  → load workspace prompt + active skills
  → build tool definitions
  → cost guard check
  → LLM call
  → tool preflight
  → tool execution (possibly parallel)
  → safety sanitize + tool_result wrap
  → next LLM turn
  → final text response
  → Channel.respond()
```

如果是后台 job，旁路上还会多出：

```text
Scheduler
  → Worker
  → DB event persistence
  → SSE broadcast
  → job status updates
```

## 1.13 这一章真正想说明什么

这条消息旅程最值得记住的，不是它经过了多少模块，而是 IronClaw 对“消息”的定义已经远远超出聊天内容本身。

在这个系统里，一条消息会被逐步嵌入到一个更大的运行时约束里：

- 它要经过 channel 统一抽象。
- 它要接受 submission 分类。
- 它要进入带系统 prompt 和 skills 的上下文。
- 它要受预算、审批、安全、权限约束。
- 它可能触发后台 job、SSE 广播、auth mode。

所以 IronClaw 的消息处理，不是“把一句话喂给模型”，而是**把一句话接入一个长期运行的代理操作系统。**

## 1.14 代码定位

这一章最值得边看边 grep 的文件：

- `src/channels/mod.rs`
- `src/channels/manager.rs`
- `src/agent/agent_loop.rs`
- `src/agent/router.rs`
- `src/agent/dispatcher.rs`
- `src/agent/worker.rs`
- `src/agent/session.rs`

下一章开始进入 IronClaw 最有辨识度的重头戏：为什么它不仅给工具上 WASM 沙箱，连频道也要一起沙箱化。
