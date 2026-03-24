# Chapter 5：自修复与主动系统：Heartbeat、Routine、Self-Repair 🔄🛠️

到了这一章，IronClaw 和多数“会聊天、会调工具”的 agent 项目之间的差距会变得非常明显。

前几章里我们已经看过它如何把输入、工具、沙箱、安全、LLM 调度都做成运行时基础设施。但这些能力本身还不足以让系统长期工作。因为一个真正长期运行的 agent，迟早会遇到这些问题：

- job 卡死了怎么办？
- 某个工具连续坏了怎么办？
- 没有用户消息时，系统还要不要主动检查环境？
- 周期任务、事件触发、Webhook 触发，这些后台动作如何统一调度？
- 日志、状态、工作区垃圾越积越多时怎么办？

很多系统会把这些事情留给外部 cron、人工运维或“以后再做”。IronClaw 则把它们直接写进核心运行时里。这就是为什么我会说，它不像一个只在消息到来时才醒来的聊天 agent，更像一个**半自治、长期驻留的个人 AI runtime**。

这一章最关键的一句话是：

> IronClaw 默认系统会老化、会卡住、会堆垃圾、会遇到无人值守的时段，所以它在架构上预留了主动维护和自我恢复的回路。

## 5.1 Self-Repair：系统先承认“作业会卡死”

在很多 agent 设计里，job 一旦进入异常状态，往往就只剩两个选项：

- 失败
- 等人来处理

IronClaw 在 `src/agent/self_repair.rs` 里走了另一条路。它首先把“卡死”明确建模成一种状态，而不是把它混进普通失败里。

从上下文状态可以看到，`JobState` 里单独有 `Stuck`。这说明 IronClaw 认为：

- `Failed` 表示这次执行已经结束，但结果失败
- `Stuck` 表示系统怀疑这次执行并没有正常收束，而是挂在了中间

这是一个非常重要的语义区别。因为如果你分不清“失败”和“卡住”，你就很难做自动恢复。

### 5.1.1 StuckJob 和 BrokenTool 是两类不同问题

`self_repair.rs` 里并不只盯 job，它还区分了两类对象：

- `StuckJob`
- `BrokenTool`

`StuckJob` 关注的是单个作业的运行状态，例如：

- job_id
- last_activity
- stuck_duration
- repair_attempts

而 `BrokenTool` 关注的则是某个工具是否进入了持续性损坏，例如：

- failure_count
- last_error
- first_failure / last_failure
- last_build_result
- repair_attempts

这个区分非常成熟。因为有时问题在“这次执行上下文跑坏了”，有时问题在“底层工具本身坏了”。这两类故障的修复策略完全不同。

## 5.2 Stuck Job Recovery：不是重启整个系统，而是局部回退

Self-Repair 的 job 修复流并不复杂，但它的价值就在于足够明确。

源码里的大致逻辑是：

1. 调 `ContextManager::find_stuck_jobs()` 找所有疑似卡死作业
2. 再把这些上下文取出来，过滤出当前确实是 `JobState::Stuck` 的对象
3. 检查 `repair_attempts` 是否已经超过 `max_repair_attempts`
4. 如果没超过，就调用 `attempt_recovery()`
5. 把 job 从 `Stuck` 拉回 `InProgress`

这里对应的修复结果用 `RepairResult` 表达，至少包括：

- `Success`
- `Retry`
- `Failed`
- `ManualRequired`

这套设计最值得注意的点，不是它“会自动恢复”，而是它不会装作自己永远能恢复。超过最大修复次数后，它会明确进入 `ManualRequired`，把问题交还给人类。

这种克制很重要。自动修复系统最危险的地方，恰恰是它不知道什么时候该停。

## 5.3 Broken Tool Recovery：把“修工具”也看成运行时职责

更激进的一步，是 IronClaw 不只恢复作业，还试图恢复工具。

在 `BrokenTool` 这条线上，系统会跟踪某个工具的连续失败次数以及最近构建结果。如果失败持续到某个阈值，它不只是标红，而是准备走“重新构建”的路径。

这里就和前面 Chapter 7 的动态工具构建器接上了：IronClaw 不把“修工具”单纯当作开发阶段行为，而是允许运行时把它当成一类恢复手段。

这件事在别的框架里很少见。多数系统对工具的默认态度是：

- 坏了就停用
- 下次发版再修

IronClaw 多走了一步：**工具坏了时，系统至少尝试把它拉回来。**

这未必适合所有生产环境，但作为个人 agent runtime，它非常符合“尽量减少人为干预”的目标。

## 5.4 Heartbeat：没有用户消息时，系统也可以醒着

到了 `src/agent/heartbeat.rs`，IronClaw 的“长期驻留”味道就更明显了。

很多 agent 只有在收到用户输入时才有行为。IronClaw 则允许系统在没有任何外部消息时，基于 `HEARTBEAT.md` 自己做周期检查。

这意味着它不再只是消息驱动程序，而是带有一定主动性的维护体。

### 5.4.1 HEARTBEAT.md 是本地化的主动检查协议

Heartbeat 的核心输入不是硬编码逻辑，而是工作区里的 `HEARTBEAT.md`。

这点很有意思，因为它意味着“系统应该定期检查什么”是用户可编辑、可审阅、可版本化的本地资产，而不是二进制里写死的一套巡检项。

大致流程是：

1. 从 workspace 读取 `HEARTBEAT.md`
2. 判断它是不是“有效为空”
3. 如果只是标题、空白、注释、空 checkbox，就直接跳过
4. 只有在里面确实有内容时，才发起 LLM 调用

这个“空内容就不调模型”的设计非常关键，因为它体现了 IronClaw 的另一个成熟点：**主动系统也必须受成本约束。**

### 5.4.2 它不是每次都通知，而是只有需要注意时才发声

Heartbeat 发给模型的协议也很克制：

- 读取 `HEARTBEAT.md`
- 严格按其清单执行
- 如果没什么需要注意的，必须返回精确的 `HEARTBEAT_OK`

之后系统按返回值决定：

- 如果是 `HEARTBEAT_OK` → 静默
- 如果不是 → 记为 `NeedsAttention`，向指定用户/频道发送通知

这让 Heartbeat 的产品语义非常清楚：它不是定时打卡器，而是**例外上报器**。

这比“每 30 分钟给你发一次我很好”合理得多。

### 5.4.3 Heartbeat 自己也会失败，而且失败会被计数

更细的一点是，heartbeat loop 本身也会跟踪连续失败次数。如果连续失败到一定阈值，会主动停掉这条 loop。

这再次体现出 IronClaw 的运行时哲学：

- 主动系统很有用
- 但主动系统自己也可能出故障
- 所以主动系统也必须有自限机制，不能无限制制造噪音或死循环

## 5.5 Routine Engine：把后台动作做成统一触发模型

如果说 Heartbeat 是“周期检查”，那么 `src/agent/routine.rs` 和 `src/agent/routine_engine.rs` 则把更一般的后台自动化抽象成了 `Routine`。

它不是简单 cron wrapper，而是一个统一的触发模型。

### 5.5.1 四种 Trigger

源码里 `Trigger` 这个枚举至少支持四种形式：

- `Cron { schedule }`
- `Event { channel, pattern }`
- `Webhook { path, secret }`
- `Manual`

这四种触发方式非常关键，因为它们把后台任务从“定时器”扩展成了“运行时事件系统”：

- `Cron` 负责标准周期性动作
- `Event` 负责对消息内容或通道事件做反应
- `Webhook` 负责接收外部系统触发
- `Manual` 负责 CLI 或工具侧的人为触发

也就是说，Routine Engine 不只是补一个 scheduler，而是在 agent runtime 内部提供了一个统一的自动化入口。

### 5.5.2 两类 Action：轻量动作和完整 Job

Routine 的执行目标也不是单一的。它至少分为两种：

- `Lightweight`
- `FullJob`

`Lightweight` 更像一次轻量的 LLM 检查：

- 有 prompt
- 有 context_paths
- 有 max_tokens

而 `FullJob` 则会走正式 job 分发：

- title
- description
- max_iterations

这个设计很合理，因为并不是所有后台任务都值得拉起完整 agentic loop。有些只是“帮我定期检查这个状态文件”，有些才是真正需要多轮工具执行的任务。

### 5.5.3 Guardrails：Routine 默认就带冷却和并发限制

Routine Engine 最体现工程成熟度的地方，是 guardrails 不是后补，而是 routine 定义的一部分。

源码里至少能看到这些限制项：

- `cooldown`
- `max_concurrent`
- `dedup_window`

默认 cooldown 大致是 300 秒，默认单 routine 并发数是 1。

这说明 IronClaw 很清楚，后台自动化最危险的地方不是“不会触发”，而是“触发太多”。没有这些 guardrails，event-based routine 特别容易把系统拖进风暴。

## 5.6 Event Routine：消息来了，但不一定进入主聊天路径

Routine Engine 的另一个高级点，在于它允许某些消息不仅进入 agent loop，还触发 event routine。

`routine_engine.rs` 里会维护 event routine 缓存，把 regex 编译后放进内存，然后对到来的消息做快速匹配。匹配后还要继续做：

- channel filter
- regex match
- cooldown check
- running count check
- global capacity check

通过这些检查后，routine 才会异步触发。

这说明 IronClaw 的事件系统不是“看到关键字就立刻跑”，而是一个有容量意识、有并发意识的后台触发器。

## 5.7 Scheduler：并行作业不是顺便 spawn 一下

到了 `src/agent/scheduler.rs`，又能看到一种很典型的 IronClaw 风格：把本来可以“简单做”的事，做成一个对长期运行更稳的结构。

Scheduler 不只是一个 `tokio::spawn` 集合，它至少负责：

- 创建 job
- 持久化 job 到数据库
- 检查并发容量上限
- 为每个 job 建立 worker 通道
- 跟踪活跃作业集合
- 在作业结束后自动清理，防止 capacity leak

### 5.7.1 它先落库，再调度

一个很关键的细节是，scheduler 会先把 job 持久化，再真正调度。这背后的考虑很现实：如果系统还要做事件记录、UI 可见性、外键关联、重载后恢复，那 job 不能只是一个内存中的 future。

### 5.7.2 它用清理任务保证“容量真的会释放”

另一个很值的细节是，每个作业旁边还有一个 cleanup 任务，会轮询 `handle.is_finished()`，一旦作业结束就把它从活跃 job 集合里移除。

这解决的是一种很典型的长期运行 bug：逻辑上 job 已结束，但调度器内部还以为它占着槽位，最终导致系统越跑越“满”。

IronClaw 在这里的思路不是“希望不会漏”，而是明确做一个清理回路保证不会漏。

## 5.8 Workspace Hygiene：长期运行系统一定会长垃圾

`src/workspace/hygiene.rs` 这一块，几乎可以说是 IronClaw 最不像普通 agent 项目的地方之一。

因为大多数 agent 项目默认不会认真考虑：

- 每天产生的工作区文档谁来清
- 状态文件如何防损坏
- 多个后台任务同时做清理怎么办
- 清理任务自己如何避免重复执行

IronClaw 把这些问题都正面处理了。

### 5.8.1 它不是“有空就清”，而是有节奏地清

HygieneConfig 至少包含：

- `retention_days`
- `cadence_hours`
- `enabled`
- `state_dir`

默认 retention 大致是 30 天，cadence 大致是 12 小时。

这说明它不是每次 heartbeat 都暴力扫全盘，而是按节奏窗口决定是否需要做一次卫生清理。

### 5.8.2 它先抢占运行窗口，再做清理

更成熟的点在于，hygiene pass 不会盲目并发运行。它用了一个全局 `AtomicBool` 作为运行锁，先保证当前没有别的清理任务在跑。

这能直接避免两个后台触发源同时启动清理，互相踩状态。

### 5.8.3 状态先写，再清理，而且是原子写

还有一个特别好的细节：它会先把“本次已经开始跑”的状态写入状态文件，而且是临时文件 + rename 的原子写法。

这个顺序的意义很大：

- 即使中途崩了，也不会让多个并发任务都以为自己该跑
- 不会写出半截状态文件
- Windows 这类平台也更稳

这是典型的长期运行系统工程思维，而不是 demo 级脚本思维。

## 5.9 Heartbeat、Routine、Self-Repair、Hygiene 之间怎么协同

这一章最容易被低估的一点，是这些模块并不是各干各的。

它们实际构成了一个闭环：

- **Heartbeat** 负责定期巡检和提醒
- **Routine Engine** 负责按事件/时间/外部回调触发自动化动作
- **Scheduler** 负责把这些动作变成可管理 job
- **Self-Repair** 负责在 job 卡死时试着拉回来
- **Hygiene** 负责清理由长期运行带来的残留和老化

这组系统放在一起，才真正构成 IronClaw 的“长期驻留特征”。

如果只拿掉其中一个，你还能做 agent；但把这几个凑齐以后，系统开始接近一种能够自我维持基本运行秩序的形态。

## 5.10 这为什么是 OpenClaw / ZeroClaw / OpenFang 里还没真正展开的点

前面对比里已经提到过，其他框架也许有某些相邻概念：

- OpenClaw 有很多交互期的安全与执行治理
- ZeroClaw 有很强的鲁棒性和容错链路
- OpenFang 正在形成宿主内核与插件兼容层

但像 IronClaw 这样，把下面这些合并成一个明确的 runtime 子系统群，仍然是很少见的：

- job stuck detection
- repair attempts and manual escalation
- user-editable heartbeat protocol
- event/webhook/cron/manual 四类 routine trigger
- capacity-aware scheduler cleanup
- workspace hygiene with atomic cadence state

所以我会把这一章视为 IronClaw 最新颖的部分之一。因为它真正回答了一个很多 agent 项目都还没认真面对的问题：

> 当没有人在盯着它时，这个系统还能不能像个系统一样维持自己？

## 5.11 对 OpenFang 最值得借鉴的点

如果把这一章压缩成 OpenFang 最值得学的几条，我会优先挑这些：

1. **把卡死从失败中分离出来，单独建模。**
2. **把自修复做成状态迁移，而不是 ad-hoc 重试。**
3. **让例程系统统一承接 cron、event、webhook、manual 四种触发。**
4. **给 scheduler 增加显式 cleanup 回路，避免长期运行后的容量假占用。**
5. **加入 workspace hygiene，防止运行时数据长期膨胀。**

这些都不是“可有可无的小 feature”，而是系统从“能跑”走向“能长期跑”的分水岭。

## 5.12 代码定位

这一章最值得跟着看的文件：

- `src/agent/self_repair.rs`
- `src/agent/heartbeat.rs`
- `src/agent/routine.rs`
- `src/agent/routine_engine.rs`
- `src/agent/scheduler.rs`
- `src/agent/job_monitor.rs`
- `src/workspace/hygiene.rs`

下一章回到权限与提示边界的另一面：Skills。也就是 IronClaw 怎样用一个确定性的 skill 选择器和“最低信任天花板”机制，把 prompt 扩展系统也纳入权限模型。
