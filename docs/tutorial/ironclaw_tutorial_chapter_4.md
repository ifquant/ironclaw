# Chapter 4：四层 LLM 韧性栈：从智能路由到断路器 🧠🔄

很多 agent 项目在 LLM 这一层的默认假设其实很脆弱：

- 提供者大多数时候可用
- 某次调用失败就重试一下
- 换模型只是配置项，不是运行时决策
- 成本控制靠人工自觉

一旦系统只是偶尔跑几次，这些假设还能勉强撑住；但如果它要长期在线、持续接消息、跑后台任务、处理工具调用，那么 LLM 这一层本身就会变成基础设施风险源。

IronClaw 在这里的做法，不是给 provider 套一个薄薄的 adapter，而是把 LLM 调用包进一套明确分层的韧性栈。它的目标很朴素：**在不把成本炸掉的前提下，让调用尽量成功；如果不能成功，也要尽量快速、清楚、可恢复地失败。**

这一章最重要的一句话是：

> IronClaw 不是把“路由、重试、故障转移、熔断、成本限制”当成五个零散 feature，而是把它们按执行顺序叠成了一条调用流水线。

## 4.1 先看全貌：五层不是并列的，而是套娃的

从源码结构看，IronClaw 的 LLM 韧性层大致可以理解成这样一个装饰器链：

```text
Cost Guard
  → Smart Routing
    → Circuit Breaker
      → Retry
        → Failover
          → Actual Provider
```

这条链的意义不是“组件很多”，而是每一层回答的是不同问题：

- **Cost Guard**：这一轮现在还能不能打？
- **Smart Routing**：该用便宜模型还是强模型？
- **Circuit Breaker**：这个 provider 当前是不是明显坏着？
- **Retry**：是不是短暂失败，值得在同一 provider 上再试？
- **Failover**：如果这家不行，下一家是谁？

把这几个问题分开处理，系统的行为会清楚很多。你不会把“一个 provider 彻底挂了”和“网络瞬时抖一下”混成同一类失败，也不会把“复杂安全审计任务”和“问今天星期几”送到同一个昂贵模型上。

## 4.2 Smart Routing：它不是换模型，而是先判断任务复杂度

在 `src/llm/smart_routing.rs` 里，IronClaw 的第一层不是 failover，而是 `Smart Routing`。这一步的思路很直接：

> 在决定“怎么救失败”之前，先决定“这次本来该打到哪一档模型”。

### 4.2.1 四档 Tier

源码里把任务分成四档：

- `Flash`
- `Standard`
- `Pro`
- `Frontier`

对应分数区间是：

- `0–15` → `Flash`
- `16–40` → `Standard`
- `41–65` → `Pro`
- `66+` → `Frontier`

这个分层很好理解：不是所有请求都值得上最贵的模型。问候、简单查时、轻量整理和复杂代码审计，本来就该落到不同成本区间。

### 4.2.2 13 维复杂度打分

更值得注意的是，它不是按 prompt 长度做一个粗糙判断，而是用 `score_complexity()` 做 13 维评分。源码里可以看到至少这些维度：

- reasoning words
- token estimate
- code indicators
- multi-step
- domain specific terms
- ambiguity
- creativity
- precision
- context dependency
- tool likelihood
- safety sensitivity
- question complexity
- sentence complexity

这些维度的权重也不是平均分配的。比如 reasoning words、token estimate、code indicators、multi-step 的权重更高，说明 IronClaw 在判断“这是不是一个值得上更强模型的问题”时，更看重推理需求、上下文规模和代码/多步骤特征。

这和“按模型名硬编码默认路由”不是一个层次。它已经开始把 LLM 调度当成运行时决策问题，而不是部署时配置问题。

### 4.2.3 Pattern Overrides：有些问题不用慢慢算

源码里还有一层很实用的捷径：pattern overrides。

例如：

- `hi` / `hello` / `thanks` 这种，直接归到 `Flash`
- `what time/date/day` 这类，直接归到 `Flash`
- `security audit/review/scan` 这类，直接拉到 `Frontier`
- `deploy production/mainnet` 这类，至少拉到 `Pro`

这一步非常工程化。因为有些请求的复杂度其实一眼就能判断，没必要每次都完整走评分器。它相当于在复杂度模型前面加了一个快速分流层。

## 4.3 Circuit Breaker：不是失败了再想，而是先承认“它现在坏着”

很多系统失败处理的最大问题在于，它会不断去碰一个已经明显坏掉的 provider。这样做不仅浪费时间，还浪费 token 预算和用户耐心。

IronClaw 在 `src/llm/circuit_breaker.rs` 里把这个问题单独抽成 `CircuitBreakerProvider`。它的状态机很经典：

- `Closed`
- `Open`
- `HalfOpen`

### 4.3.1 三态状态机

它的状态迁移逻辑可以概括成：

- 正常状态下是 `Closed`
- 连续 transient failure 达到阈值后 → `Open`
- 等到 `recovery_timeout` 到期后 → `HalfOpen`
- `HalfOpen` 下允许少量 probe call
- probe 连续成功到一定次数 → 回到 `Closed`
- probe 再失败 → 重新 `Open`

默认配置里：

- failure threshold = `5`
- recovery timeout = `30s`
- half-open successes needed = `2`

这组数字体现的意思很明确：

- 不会因为一两次短暂波动就熔断
- 但也不会无限忍耐坏 provider
- 恢复时不是完全相信，而是先小流量试探

### 4.3.2 它只对 transient error 敏感

更关键的是，它不会把所有错误都算进熔断阈值。

在 `is_transient(err)` 逻辑里，类似这些才会计入：

- `RequestFailed`
- `RateLimited`
- `InvalidResponse`
- 某些 `Http` / `Io` 错误
- `SessionExpired` / `SessionRenewalFailed` 一类会话类瞬时失败

而像这些则不应导致 breaker trip：

- `AuthFailed`
- `ContextLengthExceeded`
- `ModelNotAvailable`
- `Json` 结构类错误

这很重要，因为它区分了“provider 当前不稳定”和“请求本身就不该这么打”。如果把两者混在一起，断路器就会被误用成错误垃圾桶。

## 4.4 Retry：它重试的是短暂失败，不是所有失败

在 `src/llm/retry.rs` 里，`RetryProvider` 负责另一类问题：**如果这次失败只是瞬时抖动，要不要原地再试一下？**

这层和 Circuit Breaker 很像，但其实责任不同：

- Circuit Breaker 解决“provider 当前整体是否不健康”
- Retry 解决“这次单次请求是否值得立即再试”

### 4.4.1 指数退避 + 抖动

源码里的 `retry_backoff_delay(attempt)` 用的是很标准但非常必要的一套策略：

- attempt 0: `1s`
- attempt 1: `2s`
- attempt 2: `4s`

同时带 `±25%` jitter，下限不低于 `100ms`。

为什么 jitter 很重要？因为没有 jitter，多个并发请求在 provider 波动时会集体同频重试，形成惊群效应，结果把本来有机会恢复的 provider 继续拍死。

### 4.4.2 RateLimited 有专门分支

还有一个很成熟的细节：如果错误里明确带 `retry_after`，IronClaw 会优先尊重它，而不是强行套自己的 backoff。也就是说，provider 如果明确说“请 8 秒后再来”，系统会听，而不是死板地按本地公式算。

### 4.4.3 最大重试不是无限的

默认 `max_retries = 3`，也就是总共 4 次尝试（首次 + 3 次 retry）。

这表明它不打算把 Retry 变成“掩盖一切失败的黑箱”。重试是短暂修复手段，不是无限延期机制。

## 4.5 Failover：换 provider 不是 if-else，而是一个有冷却的调度器

在 `src/llm/failover.rs` 里，IronClaw 的 `FailoverProvider` 处理的是另一个层级的问题：**如果这家 provider 确实不行，下一家是谁，何时该跳过它，什么时候再重新尝试它？**

### 4.5.1 每个 provider 都有自己的 cooldown 状态

源码里 `ProviderCooldown` 持有两个原子状态：

- `failure_count: AtomicU32`
- `cooldown_activated_nanos: AtomicU64`

默认 cooldown 配置大致是：

- failure threshold = `3`
- cooldown duration = `300s`

也就是说，一个 provider 连续出现足够多可重试失败后，会进入冷却窗口。在这个窗口内，调度器会尽量跳过它。

### 4.5.2 它还是保证“永远至少试一个”

最成熟的细节之一是：就算所有 provider 都在 cooldown，也不会全部跳过。系统会选“最早进入 cooldown 的那个”再试一次。

这说明 IronClaw 的 failover 不是僵硬黑名单，而是一个始终保持最小活性的调度系统。因为完全拒绝尝试会让系统永远无法自恢复。

### 4.5.3 Lock-Free 冷却管理

这里还有一个工程偏好很明显的点：这个 cooldown 机制是 lock-free 的，依赖 `AtomicU32` / `AtomicU64`。

这在高并发或频繁调用场景下很实用，因为 provider 健康状态是热点路径。如果每次都上粗锁，代价会比想象中高。

## 4.6 Cost Guard：把“还能不能打”放到最外层

很多系统会先调用模型，最后再统计花了多少钱。IronClaw 反过来，在 `src/agent/cost_guard.rs` 把 `CostGuard` 放在外层，先判断预算是否允许，再允许真正发起调用。

这一步很关键，因为长期运行 agent 的成本问题不是报表问题，而是控制面问题。

### 4.6.1 两类限制

`CostGuard` 处理两种限制：

- 每日预算上限
- 每小时动作数量上限

对应的超限类型是：

- `DailyBudget { spent_cents, limit_cents }`
- `HourlyRate { actions, limit }`

这说明它把“钱”和“调用频率”视为两种不同资源，而不是混成一个抽象额度。

### 4.6.2 Daily Budget 是按日重置的真实累计

源码里 `DailyCost` 会记录：

- `total`
- `reset_date`

并在 UTC 日期变化时重置。它还会在接近阈值时给出 80% warning，并在超限后缓存一个 `budget_exceeded` 快速标志，避免每次都重复做昂贵检查。

### 4.6.3 Hourly Rate 用滑动窗口

小时动作限制则是用 `VecDeque<Instant>` 维护最近 3600 秒窗口。每次检查时先把过期事件弹掉，再看当前窗口内还有多少调用。

这个实现方式很朴素，但非常适合 agent 这种连续动作系统：你关心的是最近一小时的真实密度，而不是某个整点区间的粗糙计数。

### 4.6.4 它还会做 per-model 统计

`record_llm_call()` 不只是加总成本，还会按模型记录：

- input tokens
- output tokens
- accumulated cost

这说明 Cost Guard 不只是刹车器，也在顺手构建运营观察面。长期看，这些数据又会反过来帮助调优 routing 策略。

## 4.7 这五层是怎样真正叠起来的

如果把这一章压缩成一个请求流，可以这样理解：

### 4.7.1 第一步：预算检查

Agent 在真正调 LLM 前，先调用 `check_allowed()`：

- 今天预算还有没有超
- 最近一小时动作数有没有超

如果这里过不去，后面根本不发请求。

### 4.7.2 第二步：复杂度分档

通过预算检查后，Smart Routing 根据 prompt 复杂度和 override 规则，把请求分配到 `Flash` / `Standard` / `Pro` / `Frontier`。

这一步决定“默认该用哪档 provider / model”。

### 4.7.3 第三步：看 provider 是否已知不健康

Circuit Breaker 检查目标 provider 当前是否处于 `Open` 状态。如果是，就不浪费这次请求，而是尽快进入下一步。

### 4.7.4 第四步：对瞬时失败做本地修复

Retry 层会在同一 provider 上做有限次指数退避重试，只处理那些真正有希望恢复的 transient error。

### 4.7.5 第五步：切 provider

如果本地重试依然不行，Failover 才真正把请求切到下一个 provider，并更新对应 cooldown 状态。

### 4.7.6 第六步：成功后记录成本

调用完成后，再回到 Cost Guard 记录 token 使用和费用统计。

这条链说明 IronClaw 很清楚：

- 预算问题最外层先挡
- 路由问题在最前面决定
- 健康问题早判断
- 短时失败先局部修
- 全局失败再切 provider
- 最后再记账

这比“失败了就换一个 provider”细很多，也稳很多。

## 4.8 它和 ZeroClaw 那类“强容错解析链”不是同一种创新

前面比较里提到过 ZeroClaw 更强调解析链容错，例如模型返回结构乱掉时怎样修补 JSON、怎样做多级 fallback。那是一种很实用的鲁棒性方向。

IronClaw 这一章的创新点不同。它更关注的是：**在模型调用还没进入业务层之前，怎样把调度、成本、故障和恢复做成基础设施。**

换句话说：

- ZeroClaw 更擅长处理“模型吐出来的东西不规范”
- IronClaw 更擅长处理“模型调用这台机器本身不稳定、昂贵、可退化”

两者不是互斥关系，但侧重点完全不同。

## 4.9 这一章最值得借鉴到 OpenFang 的地方

如果把 IronClaw 这一章提炼成对 OpenFang 最有用的借鉴，我会优先挑这几条：

1. **把复杂度路由显式做成一层，而不是只在配置里选默认模型。**
2. **把 Circuit Breaker 从 provider adapter 里单独抽出来。**
3. **把 Retry 和 Failover 分层，而不是混成一套兜底逻辑。**
4. **把 Cost Guard 放到 agent 调用入口最外层，而不是事后统计。**

这些机制并不依赖 IronClaw 的全部架构，即使移植到别的 runtime 里也很值。

## 4.10 代码定位

这一章最值得配合源码看的文件：

- `src/llm/smart_routing.rs`
- `src/llm/circuit_breaker.rs`
- `src/llm/retry.rs`
- `src/llm/failover.rs`
- `src/agent/cost_guard.rs`
- `docs/smart-routing-spec.md`

下一章继续看另一个更像“长期驻留系统”的模块群：Self-Repair、Heartbeat、Routine、Scheduler、Workspace Hygiene。也就是 IronClaw 为什么不仅能跑，还敢长期跑。
