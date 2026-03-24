# Chapter 7：动态工具构建与扩展生态：把“长新能力”做成运行时能力 🛠️🧩

到了最后一章，IronClaw 最激进、也最能拉开差距的一点终于要展开了：**它不只是运行工具，它还试图在运行时长出新工具。**

前面几章我们已经看过它如何控制输入、隔离执行、保护密钥、限制模型、修复卡死、管理 skill。现在还剩下最后一个大问题：

> 当现有工具不够用时，这个系统是停下来等开发者发版，还是能自己把缺的能力补出来？

多数 agent 系统在这里会停下，因为“构建新工具”通常被视为开发期流程，不属于运行时。IronClaw 则把它推进了一步：它把工具构建、注册、安装、认证、激活这一整条链，视为 runtime 的一部分。

这就是为什么我会把这一章视为 IronClaw 最有未来感的一章。因为它讨论的已经不是“如何调用现有世界”，而是：**系统如何在受控前提下扩张自己的能力边界。**

这一章最重要的一句话是：

> IronClaw 把“扩展生态”从包管理问题推进成了“运行时能力生产线”。

## 7.1 动态构建器的目标，不是脚手架，而是能力生产

很多项目也有 generator、template、CLI scaffold，但那些工具的默认使用者是人。IronClaw 里的 builder 不一样，它服务的对象首先是 agent runtime 本身。

从源码结构看，builder 接收的不是“项目名 + 语言模板”这种传统参数，而是一个 `BuildRequirement`。里面至少包含：

- `name`
- `description`
- `software_type`
- `language`
- `input_spec`
- `output_spec`
- `dependencies`
- `capabilities`

这说明它不是在做普通脚手架，而是在做一种“从需求描述到可执行能力体”的转换器。

特别是 `software_type` 这个枚举，很关键。它允许系统区分要生成的是：

- `WasmTool`
- `CliBinary`
- `Library`
- `Script`
- `WebService`

也就是说，IronClaw 一开始就在把“新能力的物理形态”纳入设计，而不是假设一切都是同一类插件。

## 7.2 BuildPhase：构建过程被显式建模成状态机

另一个很成熟的点，是构建器没有把整个流程揉成一个黑箱“build()”。它用 `BuildPhase` 显式表示生命周期。

至少能看到这些阶段：

- `Analyzing`
- `Scaffolding`
- `Implementing`
- `Building`
- `Testing`
- `Fixing`
- `Validating`
- `Registering`
- `Packaging`
- `Complete`
- `Failed`

为什么这很重要？因为一旦你把“构建新工具”交给 runtime，你就必须能回答三个问题：

1. 它现在进行到哪一步了？
2. 是在哪一步失败的？
3. 失败后应该修哪里？

如果没有阶段状态，builder 只是一个巨大黑盒。一旦出错，你既没法向用户解释，也没法让系统自己进入“修复-重试”回路。

IronClaw 在这里的判断非常清楚：**动态构建如果要可信，首先必须可观察。**

## 7.3 它不是一次性生成，而是可以迭代修复

builder 最像 IronClaw 其它系统的地方，在于它不把失败视为终点，而是把失败也纳入正常路径。

从配置和结果结构上能看到几个关键字段：

- `max_iterations`
- `timeout`
- `cleanup_on_failure`
- `run_tests`
- `validate_wasm`
- `auto_register`

再加上 `BuildResult` 里会记录：

- `logs`
- `iterations`
- `validation_warnings`
- `tests_passed`
- `tests_failed`
- `registered`

这意味着 builder 的真实目标不是“第一次就完美生成”，而是：

- 先生成
- 编译
- 测试
- 如果失败，再根据失败信息修
- 直到达到收敛，或者达到最大迭代次数

从系统设计角度看，这一点非常关键。因为“让模型一次写对”并不现实；但“给模型一个结构化的改错回路”则是现实的。

## 7.4 WASM Tool 是 builder 的核心落点

虽然 builder 支持多种 `SoftwareType`，但从整个 IronClaw 架构看，最关键的落点仍然是 `WasmTool`。

原因很简单：

- Chapter 2 已经建立了完整的 WASM 沙箱
- WASM tool 天然最适合作为受控扩展单元
- 构建出来以后，可以直接进入现有 runtime 的能力体系

也正因如此，WASM 构建路径带了明显的特殊处理：

- 会注入 Tool trait / host function 相关上下文
- 编译目标是 `wasm32-wasip2`
- 构建后要经过 `WasmValidator`
- 验证通过后再进入注册和产物保存

这说明 IronClaw 的 builder 不是“写点代码就算扩展”，而是明确围绕 WASM 能力单元构建的。

## 7.5 WasmValidator：能力不是你说有就有，接口也不是你说对就对

如果一个系统允许 runtime 自己生成工具，那么最危险的地方就在于：生成物可能只是“看起来像工具”，但并不真正符合宿主契约。

这就是 `WasmValidator` 的价值。

从验证器结构看，它至少会检查：

- 大小是否超限
- 必要导出函数是否存在
- 函数签名是否符合要求
- import module 是否都在允许列表里
- 模块整体是否合法

典型要求大致包括：

- 默认最大 10MB
- 必须带 `run` export
- 允许 import 的模块只限少数宿主约定项，例如 `env` / `wasi` 相关

这里最重要的不是这些规则本身，而是它表达的系统原则：**builder 生成的产物并不会因为“是自己生成的”就自动被信任。**

它仍然要过宿主验证。

这点非常关键，因为如果没有这一层，动态生成能力很容易变成“把风险自动化”。IronClaw 在这里保留了运行时治理边界。

## 7.6 Registry：扩展不只是文件夹，还是一个受管理的目录体系

构建出新工具之后，还需要一个地方让系统管理它们。这里就进入了 registry 与 extension manager 的世界。

IronClaw 的 registry 不是一个“下载列表”那么简单，它至少有：

- `registry/tools/`
- `registry/channels/`
- `_bundles.json`

这意味着扩展生态在运行时里是按类别和分组组织的。

特别是 `_bundles.json` 很值得注意，它允许把多个扩展按场景打包成 bundle，例如：

- Google Suite
- Messaging
- Recommended Set

这看起来像 UI 便利功能，但其实它意味着 IronClaw 对扩展生态的理解不是“一个个散装插件”，而是“按工作流组合的能力包”。

这会直接影响后面的安装体验、授权体验和推荐逻辑。

## 7.7 Curated Registry：先防止名字碰瓷，再谈开放发现

另一个很成熟的点，是 IronClaw 并不完全信任在线发现结果。

它有 curated registry，也有 online discovery，但两者不是平权的。对于内建或已知的 registry entries，同名同 kind 的 discovered extension 不能随意覆盖它们。

这个决定非常重要，因为只要系统允许在线发现扩展，就一定会遇到一个问题：

- 一个远端结果伪装成“官方 telegram”
- 名字一样，功能描述也很像
- 用户或模型很容易误装

IronClaw 的处理方式很简单也很正确：**先保证 curated entries 的名字空间稳定，再允许发现新东西。**

这相当于给扩展生态先加了一层供应链基本秩序。

## 7.8 ExtensionKind：它把不同扩展类型放到同一套生命周期里

从 `ExtensionKind` 看，IronClaw 至少统一处理三类扩展：

- `McpServer`
- `WasmTool`
- `WasmChannel`

这很关键，因为它说明 extension manager 的目标不是只服务某一种插件，而是做统一扩展控制平面。

而且三者虽然形态不同，却都要经过类似的问题：

- 从哪里拿到源
- 如何安装
- 是否需要认证
- 何时激活
- 是否要持久化激活状态

把这些问题统一到同一个 manager 里，会让系统扩展面保持一致，而不是每种扩展都自带一套生命周期脚本。

## 7.9 ExtensionSource：下载、构建、发现，全都算正规来源

IronClaw 在来源建模上也很清楚。扩展来源不是单一 download URL，而是被建模成 `ExtensionSource`：

- `McpUrl`
- `WasmDownload`
- `WasmBuildable`
- `Discovered`

这说明它已经明确接受几个现实：

- 有些扩展适合直接下载产物
- 有些扩展更适合本地构建
- 有些扩展来自在线发现
- MCP 服务型扩展本来就不是同一物理形态

这种来源多态，是运行时级扩展系统必须具备的能力。否则一旦生态变复杂，系统就会被迫在“只支持下载”或“只支持源码构建”之间选一个低上限路线。

## 7.10 认证不是扩展的私事，而是生命周期的一部分

扩展真正难做的通常不是加载代码，而是认证。

IronClaw 在这方面同样没有偷懒。你可以从这些类型看出它已经把 auth 问题拉进扩展生命周期：

- `AuthHint`
- `ToolAuthState`
- `AuthStatus`

其中 `AuthHint` 至少包括：

- `Dcr`
- `OAuthPreConfigured`
- `CapabilitiesAuth`
- `None`

这意味着扩展在进入系统之前，宿主已经知道“它大概需要哪种认证路径”。而 `AuthStatus` 又把运行态细分成：

- 已认证
- 不需要认证
- 等待授权
- 等待 token
- 需要先做 setup

这说明 IronClaw 对扩展认证的处理不是“失败时报错”，而是把它当成一个完整状态机来看待。

这对于用户体验和系统自动化都非常重要。因为对长期运行 runtime 来说，“需要授权但尚未完成”本身就是一种重要状态，不该被简化成失败。

## 7.11 ExtensionManager：真正的控制面在这里

如果说 registry 是目录，builder 是生产线，那么 `ExtensionManager` 就是控制塔。

它内部管理的东西非常多，包括：

- registry
- online discovery
- MCP session manager
- MCP clients
- WASM tool runtime
- WASM tools dir / channels dir
- channel runtime
- secrets store
- tool registry
- active channel names
- activation errors
- SSE sender
- pending OAuth flows

这说明扩展管理在 IronClaw 里不是一个小角落，而是横跨：

- 获取
- 安装
- 认证
- 激活
- UI 反馈
- 热加载
- 重启恢复

的中心枢纽。

这很重要，因为一旦扩展生态被认真做起来，它就不再是“往某个目录里扔文件”的问题，而是一个运行时控制问题。

## 7.12 生命周期：Search → Install → Auth → Activate

把 extension manager 的行为压缩成一条主线，大致就是：

1. `search()`：在 registry 和 discovery 源里查找扩展
2. `install()`：下载、复制或准备扩展
3. `start_*_auth()` / `authenticate()`：如果需要，先完成认证
4. `activate()`：真正让扩展进入运行时

这条主线本身不惊艳，但有两个细节特别重要。

### 7.12.1 激活状态是可持久化的

对于 channel 一类扩展，IronClaw 会把 active channel names 存起来，下次重启后自动重新加载。

这意味着“当前启用了哪些扩展”不是一次性会话状态，而是运行时配置的一部分。

### 7.12.2 WASM Channel 支持热激活

更有意思的是，WASM channel runtime 可以在系统运行后再挂进来，这相当于支持一种真正的 hot activation。

这点非常值，因为它意味着：

- 不是每加一个 channel 都要重启系统
- 扩展生态的变化可以更接近“在线演进”
- 动态生成或新安装的扩展更容易进入现有运行时

从产品角度说，这是扩展生态能不能顺畅增长的关键一环。

## 7.13 动态 builder 和 extension manager 组合起来，才是真正的新意

如果只看 builder，你会觉得这是“AI 帮你写工具”；如果只看 extension manager，你会觉得这是“运行时安装扩展”。

但把两者放一起，IronClaw 做的事情就完全不一样了：

- builder 负责从需求生成能力产物
- validator 负责确认产物符合宿主契约
- registry / manager 负责把产物纳入生态
- auth / activation 负责让产物真正可用
- hot activation / persisted state 负责让它进入长期运行系统

这条链完整后，系统就具备了一种很少见的能力：**把“需要一个新工具”直接连接到“系统获得一个新能力”。**

这并不意味着它已经完全自动、自主、无风险；但它已经把这件事认真建模成一条正规路径，而不是零散脚本和人工操作。

## 7.14 为什么这一章是 IronClaw 最“未来式”的部分

前面的 WASM 沙箱、安全纵深、韧性栈、自修复系统，都可以理解成“为了把现有能力跑稳”。这一章不一样，它讨论的是“如何扩张能力边界”。

而一旦一个系统开始能在运行时生产和接入新能力，它的性质就变了：

- 它不再只是静态工具集合
- 它开始具备受控增长能力
- 它的核心挑战也从“有没有功能”变成“增长是否被治理”

IronClaw 在这里最可取的地方，不是它敢做 dynamic builder，而是它没有为了动态性牺牲治理：

- 有 phase
- 有 validation
- 有 trust boundary
- 有 registry order
- 有 auth state
- 有 hot activation but still under manager control

这使得它的“未来感”不是空想，而是建立在 runtime discipline 上。

## 7.15 对 OpenFang / OpenClaw 最有价值的启发

如果要把这一章压缩成对另外两个系统最有价值的借鉴，我会挑这几条：

1. **把构建流程显式建模成阶段状态，而不是黑箱生成器。**
2. **构建产物进入系统前必须经过宿主验证，而不是“自己生成的所以默认可信”。**
3. **扩展管理应统一处理 search/install/auth/activate，而不是每类扩展各自为政。**
4. **在线发现必须让位于 curated registry 的名字空间稳定性。**
5. **WASM channel 的热激活值得单独借鉴。**

这些都是“能让扩展生态长期健康”的基础设施，而不只是让 demo 跑起来的技巧。

## 7.16 代码定位

这一章最值得跟着看的文件：

- `src/tools/builder/`
- `src/extensions/manager.rs`
- `registry/tools/`
- `registry/channels/`
- `registry/_bundles.json`
- `src/tools/wasm/` 相关验证与运行时代码

到这里，IronClaw 教程的技术主线已经完整了：

- 它如何看待自己
- 一条消息如何流过系统
- 工具和频道如何被沙箱化
- 安全层如何做纵深防御
- LLM 调度如何变成韧性基础设施
- 长期运行如何靠主动系统维持
- skill 如何被纳入权限模型
- 新工具如何被构建、验证、注册、激活

这八章合起来，解释了 IronClaw 的主体机制；接下来的 [ironclaw_tutorial_chapter_8.md](ironclaw_tutorial_chapter_8.md) 会把这些机制重新投回 OpenClaw / ZeroClaw / OpenFang 的比较坐标里，给出真正可迁移的判断。
