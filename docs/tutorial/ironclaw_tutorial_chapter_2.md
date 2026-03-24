# Chapter 2：WASM 沙箱：工具与频道的双重隔离 🔒🧩

如果说很多 agent 系统只是“支持插件”，那 IronClaw 的问题更尖锐一些：**当你允许第三方代码接入 agent 时，你到底信不信它？**

IronClaw 给出的答案非常明确：默认不信。正因为默认不信，它没有把扩展简单当成可加载模块，而是把它们放进了一个严格受控的 WASM 执行边界里。更有意思的是，它不只把工具放进去，连频道也一起放进去。

这就是 IronClaw 和多数同类系统拉开距离的地方。

> 这一章最值得记住的不是“它用了 WASM”，而是：它围绕 WASM 建立了一整套默认拒绝、能力声明、资源计量、路径封闭、凭据注入、泄漏扫描的完整执行制度。

## 2.1 为什么它连 Channel 也要沙箱化

很多系统只会给工具上沙箱，因为工具最像“会做危险动作的代码”。频道通常被默认当成 host-native adapter，例如 Telegram、Slack、Discord 这些集成往往直接跑在宿主进程里。

IronClaw 反过来想了一步：频道其实也很危险。

因为频道不只是接收消息，它还可能：

- 接收 webhook
- 处理外部附件和 payload
- 轮询远端 API
- 向外部系统发送 agent 回复
- 持有某种认证配置

也就是说，频道本质上也是一个带外部 I/O 的不可信边界。所以 IronClaw 在 `wit/channel.wit` 里单独定义了一套 `sandboxed-channel` world，把 channel 也变成 capability-based 的 WASM 组件。

这和 OpenFang 当前“有 WASM 工具沙箱，但 channel 仍主要是宿主集成”的路线不同。IronClaw 的判断更激进：**消息入口本身也该被隔离。**

## 2.2 WIT 合约：它先定义边界，再谈实现

IronClaw 的工具和频道都不是直接暴露宿主 API，而是先通过 WIT 声明边界。

### 2.2.1 工具 world：sandboxed-tool

`wit/tool.wit` 里的 `sandboxed-tool` world 把工具能看到的宿主能力限制在一小撮明确接口上：

- `log()`
- `now-millis()`
- `workspace-read()`
- `http-request()`
- `tool-invoke()`
- `secret-exists()`

工具侧导出的能力也很克制：

- `execute(request)`
- `schema()`
- `description()`

这里最关键的不是列表本身，而是背后的安全语义：

- 工具可以知道某个 secret 存不存在，但**不能直接读取 secret 值**。
- 工具可以发 HTTP，但必须经过 allowlist 和 host boundary。
- 工具可以读工作区，但要经过路径校验。
- 工具能调用别的工具，但也受 capability 和 rate limit 约束。

换句话说，WIT 在这里不是“跨语言接口文档”，而是**最小权限宪法**。

### 2.2.2 频道 world：sandboxed-channel

`wit/channel.wit` 则为频道定义了另一套接口：

- `on-start(config)`
- `on-http-request(incoming)`
- `on-poll()`
- `on-respond(agent_response)`
- `on-status(status_update)`

它还额外暴露了频道专属的 host 能力：

- `emit-message()`
- `workspace-write()`
- pairing 相关接口

这说明 IronClaw 对 channel 的建模不是“长期持有一个对象”，而是“让宿主驱动一个受控的回调接口集合”。

这为后面的“逐回调 fresh instance”做了铺垫。

## 2.3 Compile Once，Instantiate Fresh

IronClaw 的 WASM 运行时最值得看的点之一，是它没有走“每次都重新编译”，也没有走“一个实例一直跑到底”，而是采取了中间路线：

- **编译一次**，缓存 `PreparedModule`
- **每次执行都新建 Store**，实例用完即弃

在 `src/tools/wasm/runtime.rs` 里，`WasmToolRuntime` 会把预编译后的 `Component` 缓存在 `RwLock<HashMap<String, Arc<PreparedModule>>>` 里。这个 `PreparedModule` 包含：

- 工具名
- 预编译的 `wasmtime::component::Component`
- 资源限制配置

这意味着注册或加载时付出编译成本，运行时则尽量减少冷启动开销。

但真正关键的是第二步。在 `src/tools/wasm/wrapper.rs` 里，每次工具执行都会新建一份 `StoreData`，然后：

- `Store::new(&engine, store_data)`
- 设定 epoch deadline
- 实例化 component
- 调用导出函数
- 丢弃整个 store

这个模型的好处很直接：

1. **没有脏状态继承**。上一次执行留下的内存、句柄、运行痕迹不会带到下一次。
2. **故障边界清晰**。某次调用出错，不会把整个扩展进程污染掉。
3. **安全语义稳定**。每次执行都重新带着 capability 和限制进场。

频道 runtime 也采用同样思路。在 `src/channels/wasm/wrapper.rs` 中，每次 `on-http-request`、`on-poll`、`on-respond` 都会创建新的 `ChannelStoreData`。所以 IronClaw 的 channel 其实不是长驻实例，而是**长生命周期协议 + 短生命周期执行体**。

## 2.4 Default-Deny 能力系统

WASM 只是容器，真正决定它能干什么的是 capability 模型。

在 `src/tools/wasm/capabilities.rs` 中，工具能力是一个默认空的结构：

- `workspace_read`
- `http`
- `tool_invoke`
- `secrets`

全都以 `Option<...>` 出现，意思非常明确：**不声明，就没有。**

尤其是 HTTP capability，不只是“允许联网”这么粗糙，而是包含：

- `allowlist`
- `credentials`
- `rate_limit`
- `max_request_bytes`
- `max_response_bytes`
- `timeout`

也就是说，IronClaw 对网络能力的理解不是单一开关，而是一组组合约束。

频道能力在 `src/channels/wasm/capabilities.rs` 里又再加了一层：

- `allowed_paths`
- `allow_polling`
- `min_poll_interval_ms`
- `workspace_prefix`
- `emit_rate_limit`
- `max_message_size`
- `callback_timeout`

这里最有代表性的字段是 `workspace_prefix`，它会强制把频道写入限定在 `channels/{name}/` 命名空间下。这不是约定，而是能力对象本身的一部分。

## 2.5 网络不是“能发请求”就完了，而是三重约束

IronClaw 的 HTTP 能力特别值得看，因为它不是简单在 WASM 里塞一个 fetch。

### 2.5.1 Allowlist Validator

在 `src/tools/wasm/allowlist.rs` 里，`AllowlistValidator` 会检查：

- URL 是否合法
- scheme 是否为 HTTPS
- 是否含 `user:pass@host` 这种 userinfo
- host 是否允许
- path prefix 是否允许
- HTTP method 是否允许

并且它不是统一报一个“拒绝”，而是区分：

- `HostNotAllowed`
- `PathNotAllowed`
- `MethodNotAllowed`
- `InsecureScheme`
- `InvalidUrl`

这说明这个 allowlist 不是临时补丁，而是认真做成了一个对开发者友好的网络策略层。

### 2.5.2 Host-Boundary Credential Injection

在 `src/tools/wasm/credential_injector.rs` 里，IronClaw 定义了 `SharedCredentialRegistry` 和 `CredentialMapping`。凭据映射会描述：

- secret_name
- host_patterns
- location

其中 `location` 可以是 Bearer、Basic、Header、QueryParam。

重要的点不是“它支持多种注入方式”，而是：**凭据的解析和注入发生在宿主边界，不在 WASM 内部。**

这意味着：

- WASM 工具拿不到真实密钥值。
- WASM 最多只能通过占位符或 host 匹配触发注入。
- 返回日志和错误里还会经过 redaction，避免把密钥打回文本。

这是 IronClaw 安全模型里非常关键的一层。相比“把 API key 当环境变量传给插件”，这条路线干净得多。

### 2.5.3 Leak Scan Before and After

更进一步，它不是只做凭据注入，还会在请求前和响应后做泄漏扫描。

这意味着防线不是一条，而是三条：

1. allowlist 控制你能请求哪里
2. credential injection 控制密钥怎么进入请求
3. leak detection 控制密钥有没有被试图带出去或意外带回来

所以 IronClaw 的网络能力不是“给插件开网”，而是“给插件一条带海关、安检、边防和密钥托管的专线”。

## 2.6 资源计量：不只怕越权，也怕跑飞

很多人提起沙箱只想到权限，其实长期运行系统更怕另一类问题：跑飞。

IronClaw 在 `src/tools/wasm/limits.rs` 和 runtime 初始化里做了两层资源控制。

### 2.6.1 Memory Limit

`WasmResourceLimiter` 会在 `memory_growing()` 时拦截增长请求，默认内存上限是 10MB。超了就拒绝增长。

这意味着扩展不是想 malloc 多大就多大，而是天然在一个小箱子里活动。

### 2.6.2 Fuel + Epoch Interruption

更关键的是执行时间控制。IronClaw 开启了：

- `consume_fuel(true)`
- `epoch_interruption(true)`

Fuel 是指令计量，epoch interruption 是额外的超时保险。runtime 里还有一个每 500ms tick 一次的 epoch thread，会触发到期实例的 trap。

这是一种很成熟的双保险：

- fuel 负责控制“算了多少步”
- epoch 负责控制“墙上时间过了多久”

所以它不只是怕恶意代码“越权”，还怕它“卡死 CPU”或者“长时间不返回”。

## 2.7 Rate Limiting：工具和频道各有自己的节流逻辑

IronClaw 不把 rate limiting 只留给 HTTP API。它把 rate limit 看成运行时通用约束。

### 2.7.1 Tool 级限流

在 `src/tools/rate_limiter.rs` 里，`RateLimiter` 用 `(user, tool)` 作为键，记录分钟级和小时级窗口。

这意味着某个用户疯狂调用某个工具时，不会把整个系统拖进滥用状态。

### 2.7.2 Channel emit 限流

在 `src/channels/wasm/host.rs` 中，`emit_message()` 还带了频道级别的限流：

- 单次执行最多 100 条 emit
- 单条消息大小上限 64KB
- 达到上限后会开始丢弃，而不是让整个执行失败

这个选择很有意思。它说明 IronClaw 更关心“系统继续活着”，而不是“扩展必须拿到所有输出机会”。

## 2.8 Channel 命名空间隔离：不是建议，而是强制

如果频道能写工作区，一个显而易见的问题就是：它能不能逃出自己的目录？

IronClaw 的答案是不能，而且不是靠扩展自觉，而是靠宿主强制。

在 `src/channels/wasm/capabilities.rs` 里，`validate_workspace_path()` 会拒绝：

- 绝对路径
- `..`
- null byte

通过校验后，路径还会被自动前缀成：

```text
channels/{channel-name}/{path}
```

然后 `src/channels/wasm/host.rs` 再把它记成 pending write。

这说明 channel workspace write 的真正语义不是“自由写文件”，而是“写你自己的命名空间”。

这是个很重要的工程细节，因为它把“多 channel 并存”从文件系统角度也隔离开了。

## 2.9 这套沙箱为什么比“仅支持 WASM 工具”更进一步

只给工具上沙箱，解决的是“动作执行”的风险；把 channel 也放进沙箱，解决的是“外部入口”和“回复出口”的风险。

所以 IronClaw 的进步不在于“也用了 Wasmtime”，而在于它把沙箱思路扩展到两个方向：

- **动作侧**：WASM tools
- **通信侧**：WASM channels

这两个方向一结合，就把 agent 运行时里最容易与外部世界接触的边界都包了起来。

这也是为什么在前面的对比里，我把“WASM channels”判定为新维度，而不是普通 feature parity。

## 2.10 与 OpenClaw / OpenFang 的对照

如果做一个粗线条对比，可以这样记：

- OpenClaw 更像“厚壳 + 原生集成”，重点在安全执行和工程外壳。
- OpenFang 已经明显朝“WASM 工具宿主”方向走，但 channel 这一层的统一隔离还没有像 IronClaw 这么激进。
- IronClaw 则把扩展模型贯彻得更彻底：**工具和频道都视为不可信能力体，都必须带着最小权限进入 runtime。**

这不一定适合所有项目，但对“长期运行的个人 agent”而言，非常有说服力。因为个人 agent 通常会长期连着消息入口和外部服务，这两个边界都不能放松。

## 2.11 代码定位

这一章最值得配合源码一起看的文件：

- `wit/tool.wit`
- `wit/channel.wit`
- `src/tools/wasm/runtime.rs`
- `src/tools/wasm/wrapper.rs`
- `src/tools/wasm/capabilities.rs`
- `src/tools/wasm/allowlist.rs`
- `src/tools/wasm/credential_injector.rs`
- `src/tools/wasm/limits.rs`
- `src/channels/wasm/runtime.rs`
- `src/channels/wasm/wrapper.rs`
- `src/channels/wasm/capabilities.rs`
- `src/channels/wasm/host.rs`

下一章继续沿着这条边界往里走：WASM 沙箱只是外层容器，真正让它不泄密、不被注入、不乱出网的，是后面的多层安全纵深系统。
