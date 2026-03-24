# Chapter 3：多层安全纵深：从泄漏检测到 Prompt 注入防御 🔒🛡️

WASM 沙箱解决的是“你能碰到什么”。但一个长期联网、会处理外部文本、还会接触凭据的 agent，另一个同样危险的问题是：**即使它待在箱子里，它会不会把不该说的说出去、把不该信的信进去？**

IronClaw 在这一层做得非常重。它不是只有一个 sanitizer，也不是只有一个 allowlist，而是把安全问题拆成了几种不同威胁，然后给每种威胁都上了一层专门机制。

这章的核心结论可以先说在前面：

> IronClaw 的安全层不是单点拦截器，而是一个有顺序、有分工、有严重等级的流水线。

## 3.1 SafetyLayer：统一协调，而不是一堆零散 util

在 `src/safety/mod.rs` 里，`SafetyLayer` 是一个很清楚的组合器：

- `sanitizer`
- `validator`
- `policy`
- `leak_detector`
- `config`

这个结构本身就说明一件事：IronClaw 不把安全问题理解成“正则扫一下就完了”，而是分成至少四类：

1. **格式/输入合法性问题** → `validator`
2. **提示注入问题** → `sanitizer`
3. **策略违例问题** → `policy`
4. **秘密泄漏问题** → `leak_detector`

这四类问题来源不同、处理方式不同、严重程度也不同，所以被放在不同模块里是合理的。

## 3.2 它防的不是一种 secret，而是一整族 secret

在 `src/safety/leak_detector.rs` 里，IronClaw 的泄漏检测不是只盯着某一个厂商 API key，而是准备了一整族模式。

从源码看，默认 pattern 至少覆盖了这些家族：

- OpenAI keys
- Anthropic keys
- AWS credentials
- GitHub tokens
- Stripe tokens
- Google API tokens
- Slack tokens
- Twilio
- SendGrid
- NEAR AI
- PEM / SSH private keys
- Bearer token / Auth header 形式
- 高熵 hex 串

它还把每个模式都标了两层属性：

- `LeakSeverity`：Low → Medium → High → Critical
- `LeakAction`：Warn / Redact / Block

这两个枚举很关键，因为它们让系统不必把所有匹配都当成同一种事故。

也就是说，IronClaw 不只是“发现了就报警”，而是能根据模式家族和严重程度决定：

- 只是告警
- 打码后放行
- 直接阻断整个输出

这是一个成熟系统和 demo sanitizer 的明显区别。

## 3.3 Aho-Corasick + Regex：先快扫，再精扫

泄漏检测器的另一个关键点，是它不是只靠一堆重正则硬跑。

源码里可以看到它使用了两层检测方法：

- **Aho-Corasick**：适合做大量前缀/关键词的高效匹配
- **Regex**：处理更复杂的模式，例如某些 token 结构、URL 形式、变体格式

这个组合很实用：

- 前缀型 secret，例如 `sk-`、`ghp_`、`AKIA`，用 Aho-Corasick 扫非常快。
- 复杂格式，例如 JWT、数据库 URL、某些私钥块，用 regex 更合适。

这意味着 IronClaw 的泄漏扫描不是“为了有这个功能”，而是真按性能和表达能力分层设计过。

## 3.4 请求发出去之前先扫一次

很多系统只会在最终输出给用户或模型之前做清洗，但 IronClaw 更进一步：**在 HTTP 请求发出去之前就先扫。**

`scan_http_request()` 会检查：

- URL
- headers
- body

而且它会被用于至少两条路径：

- WASM channel / tool 的 HTTP 请求
- 内置 HTTP 工具

这一步的安全意义非常大，因为它防的是“主动外泄”。例如：

- 把 token 塞进 query param
- 把 secret 塞进 Authorization header
- 把敏感内容塞进 body
- 用 `https://user:pass@host/` 这种 URL userinfo 形式偷带凭据

也就是说，IronClaw 不只是担心“外部返回了脏内容”，它还担心“系统自己把秘密送出去”。

## 3.5 手动凭据检测：不信模型会乖乖走正规注入链路

在 `src/safety/credential_detect.rs` 中，IronClaw 还补了一层很有意思的防线：`params_contain_manual_credentials()`。

这个函数的目标不是检测真实 secret 值，而是检测“模型是不是试图手动构造凭据”。

它会检查：

- header 名里是否出现 `authorization`、`x-api-key`、`token`、`secret` 等
- header 值是否以 `Bearer `、`Basic `、`Token ` 等开头
- query 参数里是否出现 `api_key`、`access_token`、`client_secret`、`signature` 等
- URL 是否含 userinfo

为什么这一层重要？因为 IronClaw 同时又有“host boundary credential injection”。正常路径应该是：

- 扩展声明自己能访问哪个 host
- 宿主根据 host 自动注入对应 secret

如果模型绕开这套机制，手动在参数里拼认证信息，说明它要么越权，要么出错。于是系统有机会要求审批或者直接拦截。

这一步非常体现 IronClaw 的安全哲学：**不只保护 secret 值本身，还要保护 secret 的使用路径。**

## 3.6 Prompt Injection Defense：把提示注入当成输入污染问题

很多项目提到 prompt injection，会写成很抽象的一段安全说明。IronClaw 在 `src/safety/sanitizer.rs` 里做得更具体：它把注入看成文本污染模式。

它会检测类似这些内容：

- `ignore previous`
- `forget everything`
- `you are now`
- `system:` / `assistant:` / `user:`
- `[INST]` / `[/INST]`
- 某些代码块式系统指令
- `eval(...)`
- `exec(...)`
- 超长 base64 payload
- null byte

这说明它不是把 prompt injection 只理解成“恶意指令”，而是看成一组高风险文本形态。

更关键的是，如果碰到高严重度模式，它不是小修小补，而可能直接把整段内容 escape 掉，避免模型把它当成结构化指令继续消费。

这是一种很典型的“宁可牺牲一点可读性，也别把执行语义让出去”的防御取向。

## 3.7 Policy：不是所有问题都归 prompt injection

有一类风险，既不属于 secret leak，也不完全属于 prompt injection，例如：

- 系统文件访问痕迹
- shell injection 语义
- 可疑 SQL 语句
- 编码后的 exploit 片段
- 过量 URL
- 过度混淆的大块字符串

IronClaw 把这些放到 `src/safety/policy.rs` 里，用独立规则集处理。

默认规则里可以看到至少这些类别：

- `system_file_access`
- `crypto_private_key`
- `sql_pattern`
- `shell_injection`
- `excessive_urls`
- `encoded_exploit`
- `obfuscated_string`

这些规则对应的动作也不同：

- `Warn`
- `Review`
- `Sanitize`
- `Block`

这层设计很重要，因为它避免把所有文本风险都粗暴塞进一个 sanitizer 里。Policy 更像是一层“业务安全规则库”，用来表达系统明确不愿接受的内容类型。

## 3.8 工具输出净化顺序：它不是乱序叠加，而是明确流水线

`SafetyLayer::sanitize_tool_output()` 最值得看的地方是顺序。顺序决定了安全语义。

从源码逻辑看，流程大致是：

1. **长度检查**：太长先截断
2. **泄漏扫描**：发现 secret 先处理
3. **策略检查**：看有没有 policy block 或 force sanitize
4. **注入净化**：根据配置或策略结果运行 sanitizer
5. **包装给 LLM**：最终写成 `<tool_output ...>` 这种结构

这个顺序很合理，因为：

- 如果内容太长，先截断可以减小后续处理成本。
- secret 泄漏优先级最高，应该尽早处理。
- policy 决定后面是否需要强制 sanitize。
- 最后再包成稳定格式喂给模型。

这比“工具输出跑一个正则再塞给模型”成熟很多。IronClaw 真正在意的是：**输出在成为 prompt 上下文前，是否已经变成一种安全、稳定、可控的中间表示。**

## 3.9 XML 包装不是装饰，而是边界标记

IronClaw 会把工具输出包装成类似这样的结构：

```xml
<tool_output name="..." sanitized="true|false">
  ...
</tool_output>
```

这不是为了好看，而是为了给模型一个明确边界：

- 这是工具输出，不是系统指令
- 这是经过净化的内容
- 它带名字和状态标记

这种结构化包装和前面的 sanitizer/policy 一起，形成了一个核心目标：尽量减少模型把工具输出误认成新的控制指令。

这点在处理网页内容、第三方 API 返回、日志文本时尤其重要。

## 3.10 它真正做的是“秘密治理”，不是简单 redaction

把这一章往前压缩，你会发现 IronClaw 其实在做的不只是安全过滤，而是一整套“秘密治理”：

- Secret 不进 WASM
- Secret 的存在可以检查，但值不可读
- Secret 的注入有固定 host-boundary 路径
- Secret 的手动构造会被探测
- Secret 的出站请求会被扫描
- Secret 的响应回流也会被扫描
- Secret 的输出给模型前还会再清洗

这一整条链路连起来，就和很多“只在日志里打码一下”的系统不是一个量级了。

## 3.11 与 OpenClaw / ZeroClaw / OpenFang 的差异点

如果拿前面几个系统来对照，这一章最明显的差异有三个：

### 3.11.1 它更主动，而不是更被动

OpenFang 的 `zeroize` 很重要，ZeroClaw 的环境变量白名单也很重要，但这些更像“减少暴露面”的被动防御。IronClaw 多做的一步是主动扫描：

- 扫请求
- 扫响应
- 扫工具输出
- 扫手动拼接的认证参数

这会把很多“按理不该发生，但一旦发生就很糟”的情况提前拦住。

### 3.11.2 它更强调路径正确，而不只强调结果正确

很多系统只关心“最后有没有泄漏”。IronClaw 连 secret 的使用路径都管：

- 该由 host inject 的，不允许模型自己塞
- 该由 allowlist 保护的，不能直接外发
- 该由 tool output 包装的，不能裸文本直喂 LLM

所以它不是只在纠错，而是在强制系统走对的安全路径。

### 3.11.3 它把安全做成运行时基础设施

最重要的一点是，IronClaw 的安全层不是某个工具自己决定是否调用，而是被 agent loop、dispatcher、WASM runtime、HTTP 工具共同复用。

也就是说，这不是 feature，而是 runtime 基础设施。

## 3.12 代码定位

这一章最值得跟读的文件：

- `src/safety/mod.rs`
- `src/safety/leak_detector.rs`
- `src/safety/credential_detect.rs`
- `src/safety/sanitizer.rs`
- `src/safety/policy.rs`
- `src/tools/builtin/http.rs`
- `src/channels/wasm/wrapper.rs`
- `src/agent/dispatcher.rs`
- `src/agent/worker.rs`

下一章开始看 IronClaw 的另一处重头戏：它怎么把模型调用这一侧也做成一个有路由、断路器、重试和故障转移的韧性系统，而不是单个 provider 的薄封装。
