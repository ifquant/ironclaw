# IronClaw 架构解剖：个人 AI Agent 的工程深潜

> 聚焦 IronClaw 相对于 OpenClaw / ZeroClaw / OpenFang 的新颖方案，从架构哲学到实现细节完整拆解。

---

## 阅读指南

| 标记 | 含义 |
|---|---|
| 🦀 | Rust 实现细节 |
| 🔒 | 安全 / 隔离相关 |
| 🧠 | 智能决策 / LLM 相关 |
| 🛠️ | 工具 / 扩展系统 |
| 🔄 | 韧性 / 自修复 |
| 📍 | 代码定位（grep 提示） |

**快速通道** (30 min): Ch.0 概览 → Ch.1 消息旅程 → Ch.5.1 自修复 → Ch.8 综合判断

**推荐顺序**: Ch.0 → Ch.1 → Ch.2 → Ch.3 → Ch.4 → Ch.5 → Ch.6 → Ch.7 → Ch.8

## 速读入口

- [ironclaw_tutorial_quick_overview.md](ironclaw_tutorial_quick_overview.md) — 10 分钟读完 IronClaw 的新意与可迁移结论

## 术语约定

- `运行时 (runtime)`：IronClaw 作为长期运行系统的整体执行环境
- `宿主 (host)`：为工具、频道、技能、扩展提供能力边界与治理规则的内核侧
- `扩展 (extension)`：统一指可被安装、认证、激活、管理的外部能力包
- `工具 (tool)`：被 LLM 或作业调用的动作单元，包含内置工具、WASM 工具、MCP 工具
- `频道 (channel)`：消息输入/输出适配层，包含原生频道与 WASM 频道
- `插件 (plugin)`：仅在对照 OpenClaw / OpenFang 插件语境时使用，不作为 IronClaw 主术语

## 章节入口

- [ironclaw_tutorial_quick_overview.md](ironclaw_tutorial_quick_overview.md) — 10 分钟总览
- [ironclaw_tutorial_chapter_0.md](ironclaw_tutorial_chapter_0.md) — 哲学与全景
- [ironclaw_tutorial_chapter_1.md](ironclaw_tutorial_chapter_1.md) — 消息旅程
- [ironclaw_tutorial_chapter_2.md](ironclaw_tutorial_chapter_2.md) — WASM 双重沙箱
- [ironclaw_tutorial_chapter_3.md](ironclaw_tutorial_chapter_3.md) — 安全纵深
- [ironclaw_tutorial_chapter_4.md](ironclaw_tutorial_chapter_4.md) — LLM 韧性栈
- [ironclaw_tutorial_chapter_5.md](ironclaw_tutorial_chapter_5.md) — 自修复与主动系统
- [ironclaw_tutorial_chapter_6.md](ironclaw_tutorial_chapter_6.md) — 技能系统与信任天花板
- [ironclaw_tutorial_chapter_7.md](ironclaw_tutorial_chapter_7.md) — 动态工具构建与扩展生态
- [ironclaw_tutorial_chapter_8.md](ironclaw_tutorial_chapter_8.md) — 综合对照与借鉴价值

---

## 结构总览

### Act 0: 全景 (Panorama)

**Chapter 0 — 哲学与全景：一个 Rust 单体的 AI Agent 为何这样设计**
- 0.1 设计哲学：个人助理 vs 多租户平台
- 0.2 架构一览：从 main.rs 到 30 个模块
- 0.3 与 OpenClaw / ZeroClaw / OpenFang 的定位差异
- 0.4 十大新颖方案速览（→ 详见 `ironclaw_distinctive_approaches.md`）

### Act 1: 旅程 (Journey)

**Chapter 1 — 一条消息的奇幻漂流：从 WhatsApp 到工具执行再到回复**
- 1.1 入口：Channel trait → WASM Channel → ChannelManager
- 1.2 路由：MessageIntent 分类与 Router
- 1.3 调度：Scheduler → Worker → Agent Loop
- 1.4 工具分发：Dispatcher → 内置/WASM/MCP 工具
- 1.5 回复：on-respond → Channel → 用户
- 1.6 全链路时序图

### Act 2: 深潜 (Deep Dives) — 新颖方案专题

**Chapter 2 — 🔒 WASM 沙箱：工具与频道的双重隔离**
- 2.1 WIT 接口设计：tool.wit 与 channel.wit 的最小权限合约
- 2.2 工具沙箱：compile-once / instantiate-fresh 模型
- 2.3 频道沙箱：逐回调隔离的无状态设计
- 2.4 能力系统：default-deny → capabilities.json 声明
- 2.5 资源计量：fuel metering、内存上限、epoch 中断
- 2.6 凭据注入：代理层注入，WASM 永不见密
- 2.7 与 OpenFang WASM 沙箱的对比

**Chapter 3 — 🔒 多层安全纵深：从泄漏检测到 Prompt 注入防御**
- 3.1 安全层全貌：SafetyLayer 的五道闸
- 3.2 凭据泄漏检测（4 层）：Aho-Corasick → Regex → HTTP 专项 → 响应扫描
- 3.3 LeakAction 决策树：Block / Redact / Warn
- 3.4 手动凭据检测：LLM 尝试注入凭据时的审批拦截
- 3.5 Prompt 注入防御：Sanitizer 的模式识别与转义
- 3.6 策略引擎：SQL 注入、Shell 注入、系统文件访问的规则库
- 3.7 工具输出净化管线：截断 → 泄漏扫描 → 策略检查 → 注入转义 → XML 包装
- 3.8 与三大框架安全机制的对比

**Chapter 4 — 🧠 四层 LLM 韧性栈：从智能路由到断路器**
- 4.1 全貌：四层如何协同（Smart Routing → Circuit Breaker → Retry → Failover）
- 4.2 13 维复杂度评分器：输入特征与路由矩阵
- 4.3 断路器状态机：Closed → Open → HalfOpen
- 4.4 指数退避 + 抖动：防止惊群的重试策略
- 4.5 无锁故障转移：AtomicU64 实现的 per-provider 冷却
- 4.6 成本守卫：日预算 + 小时速率 + 80% 告警
- 4.7 端到端请求流：一次 LLM 调用穿越四层的完整路径
- 4.8 与 ZeroClaw 六级解析链的对比

**Chapter 5 — 🔄 自修复与主动系统：Heartbeat、Routine、Self-Repair**
- 5.1 自修复系统：卡死检测 → 自动恢复 → RepairResult 状态机
- 5.2 损坏工具重建：SoftwareBuilder 驱动的 WASM 重编译
- 5.3 心跳系统：HEARTBEAT.md 清单 → 成本优化 → 条件通知
- 5.4 例程引擎：四种触发器（Cron / Event / Webhook / Manual）
- 5.5 防抖与限流：cooldown、max_concurrent、事件缓存
- 5.6 工作区卫生：定期清理 → 原子保存 → 并发锁
- 5.7 调度器：并行作业管理 → SSE 广播 → 容量泄漏防护
- 5.8 这些系统在三大框架中的空白

**Chapter 6 — 🛠️ 技能系统与信任天花板**
- 6.1 SKILL.md 格式：YAML frontmatter + Markdown prompt
- 6.2 确定性预过滤：keywords、tags、regex（零 LLM 开销）
- 6.3 信任天花板机制：一个 Installed 技能 → 全局降级
- 6.4 工具衰减：从完整工具集到只读子集的映射
- 6.5 Token 预算：防止技能 prompt 膨胀
- 6.6 与 OpenClaw 技能系统的对比

**Chapter 7 — 🛠️ 动态工具构建与扩展生态**
- 7.1 八步构建流水线：Analyze → Scaffold → ... → Register
- 7.2 WASM 上下文注入：WIT trait 文档、宿主函数、capabilities
- 7.3 迭代修复：编译失败 → LLM 分析 → 自动重试
- 7.4 注册表 (ClawHub)：channels/ + tools/ + _bundles.json
- 7.5 保护名称：40+ 内置工具不可覆盖
- 7.6 扩展管理器生命周期：search → install → activate
- 7.7 Per-Job 密码学令牌：Docker 沙箱的认证设计
- 7.8 混合搜索：pgvector + FTS + Reciprocal Rank Fusion

### Act 3: 综合 (Synthesis)

**Chapter 8 — 交叉分析：IronClaw 方案对三大框架的借鉴价值**
- 8.1 OpenFang 可直接采纳的方案（P1/P2/P3 优先级）
- 8.2 ZeroClaw 可增强的子系统
- 8.3 OpenClaw 可扩展的能力
- 8.4 共性趋势：AI Agent 工程的演进方向

---

## 参考文档

- [ironclaw_tutorial_quick_overview.md](ironclaw_tutorial_quick_overview.md) — 速读版导航
- [ironclaw_distinctive_approaches.md](ironclaw_distinctive_approaches.md) — 十大新颖方案详细对比
- [IronClaw README](../ironclaw/README.md) — 项目概述
- [IronClaw FEATURE_PARITY.md](../ironclaw/FEATURE_PARITY.md) — OpenClaw↔IronClaw 功能对照
- [IronClaw CLAUDE.md](../ironclaw/CLAUDE.md) — 开发指南与项目结构
