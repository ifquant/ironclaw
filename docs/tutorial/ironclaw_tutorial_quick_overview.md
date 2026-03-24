# IronClaw 10 分钟总览

> 给只想快速判断 IronClaw 新意、边界和借鉴价值的读者。

## 一句话结论

IronClaw 不是“另一个会调工具的 agent”，而是一套已经朝 **个人 AI runtime** 演化的系统：它把长期运行、不可信扩展、能力增长这三类问题都提前做成了一等结构。

## 先看它和另外三个项目的相对位置

- **OpenClaw**：强在厚壳，擅长把操作系统和真实环境里的脏问题挡在模型外面。
- **ZeroClaw**：强在工程稳态与多模型兼容，擅长解析容错、兼容层和 provider 适配。
- **OpenFang**：强在宿主内核化和插件兼容层，已经明显朝平台宿主演进。
- **IronClaw**：再往前一步，重点不是“这轮能不能调用工具”，而是“系统能不能长期跑、边界能不能守住、能力能不能有治理地长出来”。

## IronClaw 最值得看的 8 个点

### 1. WASM 双重沙箱

它不只把工具放进 WASM，还把频道也放进 WASM。

这意味着：

- 工具是受控能力单元
- 频道也是不可信边界
- 消息入口和动作出口都被纳入沙箱治理

如果你只记一个结构创新，先记这个。

对应章节：
- [ironclaw_tutorial_chapter_2.md](ironclaw_tutorial_chapter_2.md)

### 2. 主动安全纵深

它不是只在 import 或日志阶段做被动防御，而是主动扫描：

- 出站请求
- 入站响应
- 工具输出
- 手工拼接的认证参数

同时配合 host-boundary credential injection，让 secret 不直接暴露给 WASM 扩展。

对应章节：
- [ironclaw_tutorial_chapter_3.md](ironclaw_tutorial_chapter_3.md)

### 3. 四层 LLM 韧性栈

它把模型调用做成：

- Smart Routing
- Circuit Breaker
- Retry
- Failover
- Cost Guard

这不是 provider adapter，而是运行时基础设施。

对应章节：
- [ironclaw_tutorial_chapter_4.md](ironclaw_tutorial_chapter_4.md)

### 4. Self-Repair + Heartbeat + Routine + Hygiene

这是 IronClaw 最像“长期驻留系统”的一组模块。

它默认系统会：

- 卡死
- 老化
- 堆垃圾
- 在无人看管时仍然需要维持基本秩序

所以它内置了：

- stuck job 恢复
- heartbeat 巡检
- 例程引擎
- scheduler cleanup
- workspace hygiene

对应章节：
- [ironclaw_tutorial_chapter_5.md](ironclaw_tutorial_chapter_5.md)

### 5. Skill 信任天花板

IronClaw 的 skill 系统最不一样的地方不是“会选 skill”，而是：

- skill 带 trust level
- 选择是确定性的，不依赖 LLM
- 只要混入任何 `Installed` skill，整轮工具上限就按最低 trust 收敛

这会把 prompt 扩展系统直接接进权限模型。

对应章节：
- [ironclaw_tutorial_chapter_6.md](ironclaw_tutorial_chapter_6.md)

### 6. 动态工具构建器

它不只是装扩展，还尝试在运行时长出新工具：

- analyze
- scaffold
- implement
- build
- test
- fix
- validate
- register

关键点不是“AI 写代码”，而是这条链被做成了受治理的构建状态机。

对应章节：
- [ironclaw_tutorial_chapter_7.md](ironclaw_tutorial_chapter_7.md)

### 7. Curated registry + activation lifecycle

IronClaw 的扩展生态不是简单插件目录，而是带：

- curated registry
- online discovery
- auth lifecycle
- activate lifecycle
- persisted active state
- hot activation

这说明它在认真做扩展控制平面，而不是只做加载器。

对应章节：
- [ironclaw_tutorial_chapter_7.md](ironclaw_tutorial_chapter_7.md)

### 8. 它代表“agent runtime 第二阶段”

如果把整个判断压缩成一句话：

- 第一阶段 agent 系统关心“能不能跑一轮任务”
- 第二阶段 agent 系统开始关心“长期运行、边界治理、能力增长”

IronClaw 已经明显进入第二阶段问题域。

对应章节：
- [ironclaw_tutorial_chapter_8.md](ironclaw_tutorial_chapter_8.md)

## 对 OpenFang / ZeroClaw / OpenClaw 的直接借鉴

### 对 OpenFang

最值得优先吸收：

- 插件桥的主动泄漏扫描
- stuck / degraded / reloading 这类可恢复状态
- Cost Guard + Retry + Failover + Circuit Breaker
- 为 channel sandbox 预留架构口子

### 对 ZeroClaw

最值得优先吸收：

- smart routing
- cooldown-based failover
- heartbeat / routine / hygiene 这类长期运行机制
- trust-aware skill ceiling

### 对 OpenClaw

最值得优先吸收：

- 从交互期治理扩展到后台主动治理
- 从宿主执行安全推进到扩展边界治理
- host-boundary credential injection
- 更完整的扩展注册与激活模型

## 推荐阅读路线

### 10 分钟路线

1. [ironclaw_tutorial_chapter_0.md](ironclaw_tutorial_chapter_0.md)
2. [ironclaw_tutorial_chapter_2.md](ironclaw_tutorial_chapter_2.md)
3. [ironclaw_tutorial_chapter_5.md](ironclaw_tutorial_chapter_5.md)
4. [ironclaw_tutorial_chapter_8.md](ironclaw_tutorial_chapter_8.md)

### 30 分钟路线

1. [ironclaw_tutorial_chapter_0.md](ironclaw_tutorial_chapter_0.md)
2. [ironclaw_tutorial_chapter_1.md](ironclaw_tutorial_chapter_1.md)
3. [ironclaw_tutorial_chapter_2.md](ironclaw_tutorial_chapter_2.md)
4. [ironclaw_tutorial_chapter_3.md](ironclaw_tutorial_chapter_3.md)
5. [ironclaw_tutorial_chapter_5.md](ironclaw_tutorial_chapter_5.md)
6. [ironclaw_tutorial_chapter_8.md](ironclaw_tutorial_chapter_8.md)

### 全量路线

从 [ironclaw_tutorial_toc.md](ironclaw_tutorial_toc.md) 按顺序读完 0 到 8。

## 文档地图

- [ironclaw_tutorial_toc.md](ironclaw_tutorial_toc.md) — 完整目录与术语约定
- [ironclaw_distinctive_approaches.md](ironclaw_distinctive_approaches.md) — 十大新颖方案速查表
- [ironclaw_tutorial_chapter_8.md](ironclaw_tutorial_chapter_8.md) — 最终综合判断

## 最后判断

如果你只问一个问题：**IronClaw 最值得注意的到底是什么？**

答案不是某个单独 feature，而是它把下面三件事同时做了：

- 把不可信扩展纳入严格边界
- 把长期运行问题纳入运行时核心
- 把能力增长做成受治理的正式路径

这三件事合在一起，才让它和 OpenClaw、ZeroClaw、OpenFang 形成了真正的差异化。 
