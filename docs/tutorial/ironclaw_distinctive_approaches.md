# IronClaw 独特方案分析

> 对照 OpenClaw / ZeroClaw / OpenFang 已有讨论，梳理 IronClaw 中尚未被覆盖或落地深度显著更高的方案。

---

## 一览表

| # | IronClaw 方案 | 已有框架最接近概念 | 差距判定 |
|---|---|---|---|
| 1 | 自修复系统 (Self-Repair) | 无 | **全新** |
| 2 | 动态工具构建 (Dynamic Tool Building) | 无 | **全新** |
| 3 | WASM 频道隔离 (WASM Channels) | OpenFang WASM 工具沙箱 | **新维度** — 工具有，频道无 |
| 4 | 混合搜索 + RRF 融合 | OpenClaw SQLite-vec / OpenFang Qdrant | **显著更深** |
| 5 | 多层凭据泄漏检测 | ZeroClaw env_clear / OpenFang zeroize | **显著更深** |
| 6 | 信任天花板技能过滤 (Trust Ceiling) | OpenClaw skill 筛选 | **新机制** |
| 7 | 13 维复杂度智能路由 | ZeroClaw 提供者故障转移 | **显著更深** |
| 8 | 四层 LLM 提供者韧性栈 | ZeroClaw 六级解析链 | **新组合** |
| 9 | Per-Job 密码学令牌 Docker 沙箱 | OpenFang Docker 沙箱 | **显著更深** |
| 10 | 工作区卫生系统 (Workspace Hygiene) | 无 | **全新** |

---

## 1. 自修复系统 (Self-Repair)

**已有框架**: 无。OpenClaw / ZeroClaw / OpenFang 均无自动检测卡死作业并尝试恢复的机制。

**IronClaw 方案** (`src/agent/self_repair.rs`, ~529 行):

```
检测阶段
├─ 卡死作业: JobState::Stuck (超时未响应)
└─ 损坏工具: 连续失败 ≥ 5 次 → 标记 BrokenTool

修复阶段
├─ 卡死作业 → 重置为 InProgress，重新排队
└─ 损坏工具 → 调用 SoftwareBuilder 重新编译 WASM
    └─ LLM 分析失败日志 → 生成修复 → 重编译 → 验证

结果枚举 RepairResult
├─ Success       已修复
├─ Retry         需要更多尝试 (≤ max_repair_attempts, 默认 3)
├─ Failed        无法自动修复
└─ ManualRequired 需要人工介入
```

**启发**:
- 对 OpenFang 插件层: 可在 `PluginHostRegistry` 增加 `Degraded → Reloading` 自修复路径
- Node 工具崩溃后自动重试 worker session，而非直接标记 `Faulted`

---

## 2. 动态工具构建 (Dynamic Tool Building)

**已有框架**: 无。三大框架均无 LLM 驱动的运行时工具生成能力。

**IronClaw 方案** (`src/tools/builder/`, 8 步流水线):

```
用户描述需求 (自然语言)
  │
  ▼
1. Analyze    → LLM 分析需求，确定工具类型 (WASM/CLI/Script)
2. Scaffold   → 使用预置模板生成骨架 (含 WIT trait 文档)
3. Implement  → LLM 生成实现代码
4. Build      → cargo component build (WASM) / cargo build (CLI)
5. Test       → 运行测试套件
6. Validate   → 检查 capabilities，安全审查
7. Package    → 打包 .wasm artifact
8. Register   → 自动注册到 ToolRegistry

失败恢复: LLM 自动分析编译/测试错误 → 修复 → 重试 (最多 N 轮)
```

**WASM 特殊上下文注入**: Builder 自动注入 `Tool` trait 文档、宿主函数签名、capabilities 约束、验证规则，确保 LLM 生成的代码符合沙箱约定。

**启发**:
- OpenFang 可在 `openfang-skills` 增加动态插件构建能力
- 在 MCP 协议层增加 `tool/build` RPC，让 agent 请求创建新工具

---

## 3. WASM 频道隔离 (WASM Channels)

**已有框架**: OpenFang 已有 WASM 工具沙箱，但频道(通信渠道)仍为硬编码或原生集成。

**IronClaw 方案** (`wit/channel.wit` + `channels-src/`):

```
Channel WIT 接口
├─ on-start(config)           → 初始化 (注册 HTTP/WebSocket 端点、轮询配置)
├─ on-http-request(incoming)  → 处理 webhook
├─ on-poll()                  → 周期轮询
├─ on-respond(agent_response) → 将 agent 回复推送至频道
└─ on-status(status_update)   → 状态更新 (thinking/done/tool-started/auth-required)

隔离模型
├─ 每次回调创建全新 WASM 实例 (无共享状态，强制无状态)
├─ 写操作自动前缀: channels/<channel-name>/
├─ emit-message() 速率限制: 100 条/次执行
└─ pairing 系统: DM 白名单，unknown sender 需要授权

已实现频道 (均为 WASM 组件):
  whatsapp, slack, telegram, discord, gmail
```

**与工具沙箱的关键差异**: 频道是**长生命周期事件循环**，但 IronClaw 将其拆解为**逐回调隔离实例**，以获得工具级别的安全保证。

**启发**:
- OpenFang 可将 channel adapter 也纳入 WASM 沙箱
- 每个 webhook 回调一个实例 → 优雅处理频道 crash，不影响宿主

---

## 4. 混合搜索 + RRF 融合

**已有框架**:
- OpenClaw: SQLite + vec 扩展 (向量搜索)
- ZeroClaw: 全文搜索为主
- OpenFang: Qdrant 向量搜索

三者均为**单通道**检索。

**IronClaw 方案** (`src/workspace/search.rs`):

```
查询
  │
  ├── 全文搜索 (PostgreSQL ts_rank_cd)
  │     └── 返回 [(doc_id, fts_rank), ...]
  │
  └── 向量搜索 (pgvector cosine similarity)
        └── 返回 [(doc_id, vector_rank), ...]
  │
  ▼
Reciprocal Rank Fusion (RRF)
  score(d) = Σ 1/(k + rank_i)   k=60 (默认)
  │
  ▼
合并排序 → 返回 top-N
  每条结果含 fts_rank + vector_rank → 双通道命中为强信号
```

**优势**: RRF 对不同分数尺度鲁棒 (向量余弦相似度 0-1，BM25 分数范围不同)，比简单加权平均效果更好。

**启发**:
- OpenFang 可在 RAG 层组合 Qdrant 向量搜索 + PostgreSQL 全文搜索 + RRF
- 向 `openfang-kernel` 的知识检索管线增加融合策略

---

## 5. 多层凭据泄漏检测

**已有框架**:
- ZeroClaw: `env_clear` 白名单环境变量
- OpenFang: `zeroize` 内存清除敏感数据

均为**被动防御** — 减少暴露面，但不主动扫描输出。

**IronClaw 方案** (`src/safety/leak_detector.rs` + `src/safety/credential_detect.rs`):

```
四层主动检测

Layer 1: Aho-Corasick 快速前缀匹配
  sk-, pk-, gh_pat, ghp_, AKIA, -----BEGIN PRIVATE KEY-----

Layer 2: 正则模式匹配
  JWT (eyJ...), 数据库 URL (postgres://user:pass@), OAuth (ya29.)

Layer 3: HTTP 专项扫描 (请求发出前)
  ├─ URL query params (api_key=, access_token=)
  ├─ Header values (Authorization: Bearer)
  ├─ Body 中的 JSON secret 字段
  └─ URL userinfo (https://user:pass@host/)

Layer 4: 响应扫描 (返回 WASM 前)
  ├─ 扫描 API 响应 body
  └─ 检查是否有 secret 回显

处理策略 (LeakAction)
├─ Block    → 严重泄漏，拒绝整个操作
├─ Redact   → 替换为 [REDACTED]
└─ Warn     → 日志告警，放行

严重等级 (LeakSeverity)
  Low → Medium → High → Critical
```

**加分**: `params_contain_manual_credentials()` 在 LLM 尝试手动注入凭据时要求审批，而非盲目执行。

**启发**:
- OpenFang 插件桥 RPC 层可增加出站扫描: 插件发送的每个 HTTP 请求和工具输出均过 LeakDetector
- 在 `openfang-extensions` import 阶段增加插件代码静态扫描

---

## 6. 信任天花板技能过滤 (Trust Ceiling)

**已有框架**: OpenClaw 技能系统有 scoring 和 prefilter，但无信任天花板机制。

**IronClaw 方案** (`src/skills/`):

```
SKILL.md 格式: YAML frontmatter + Markdown prompt body

信任层级
├─ Trusted   → 用户直接放置的技能 → 完整工具访问
└─ Installed → 注册表安装的技能   → 仅只读工具 (8 个)
    memory_search, time, echo, json, ...

天花板规则 (关键创新):
  只要存在任何一个 Installed 级别技能
  → 所有技能 (包括 Trusted) 降级到只读工具集

评分: 确定性预过滤 (keywords, tags, regex)，无 LLM 调用
Token 预算: 防止技能 prompt 膨胀
```

**设计哲学**: 最低信任约束。混合使用不同信任级别的技能时，系统取最严格限制，而非最宽松。

**启发**:
- OpenFang 插件权限模型可借鉴: 当任何一个未信任插件激活时，收紧所有插件的权限上限
- `PluginLoadState` 增加 `TrustLevel` 维度

---

## 7. 13 维复杂度智能路由

**已有框架**: ZeroClaw 有提供者故障转移和模型别名，但路由决策基于简单规则 (rate-limit 后切换)。

**IronClaw 方案** (`src/llm/smart_routing.rs`, ~700 行):

```
复杂度评分器 (13 维)

输入特征:
  1.  消息长度
  2.  工具数量
  3.  活跃工具类别
  4.  对话轮次
  5.  系统 prompt 复杂度
  6.  用户意图类别 (code/analysis/creative/simple)
  7.  多模态附件数量
  8.  上下文窗口使用率
  9.  历史错误率
  10. 任务紧急度
  11. 成本预算剩余
  12. 响应质量要求
  13. 并行子任务数

评分结果 → 路由矩阵
├─ score < threshold_low  → cheap model (e.g. Haiku)
├─ score < threshold_high → primary model (e.g. Sonnet)
└─ score ≥ threshold_high → premium model (e.g. Opus)

运行时自适应: 历史调用结果反馈评分权重
```

**启发**:
- OpenFang 可在代理网关层增加请求复杂度评估，动态选择模型
- 对 `openfang-kernel` 的 LLM 调度器，比简单 round-robin 更经济

---

## 8. 四层 LLM 提供者韧性栈

**已有框架**: ZeroClaw 六级解析链 (JSON 容错)；OpenFang 有基础故障转移。

**IronClaw 将四个独立系统组合为一体**:

```
请求
  │
  ▼
Layer 1: Smart Routing (§7)
  → 选择模型 + 提供者
  │
  ▼
Layer 2: Circuit Breaker (src/llm/circuit_breaker.rs, ~500 行)
  状态机: Closed → Open → HalfOpen
  ├─ 连续失败 ≥ threshold → Open (直接跳过该提供者)
  ├─ cooldown 到期 → HalfOpen (允许一次探测)
  └─ 探测成功 → Closed (恢复正常)
  │
  ▼
Layer 3: Retry with Jitter (src/llm/retry.rs, ~300 行)
  指数退避: 1s → 2s → 4s
  ±25% 随机抖动 (防止惊群)
  │
  ▼
Layer 4: Failover (src/llm/failover.rs, ~900 行)
  多提供者顺序尝试
  Per-provider 无锁冷却: cooldown 300s, threshold 3 次失败
  Lock-free (AtomicU64) 实现
```

**关键差异**: 不是 4 个独立机制，而是协同工作的流水线。Circuit Breaker 跳过已知不可用提供者 → Retry 处理暂时性错误 → Failover 切换到下一个提供者。

**启发**:
- OpenFang 的 LLM 层可直接采用这一四层架构
- 特别是 Circuit Breaker 状态机，避免对已知宕机的提供者浪费请求

---

## 9. Per-Job 密码学令牌 Docker 沙箱

**已有框架**: OpenFang 有 Docker 沙箱，但认证机制细节未见设计。

**IronClaw 方案** (`src/orchestrator/auth.rs` + `src/worker/api.rs`):

```
令牌生命周期

1. 作业创建 → 生成 32 字节随机令牌 (64 hex chars)
2. 存储   → 内存级 TokenStore (不持久化，重启即失效)
3. 注入   → IRONCLAW_WORKER_TOKEN 环境变量传入容器
4. 验证   → constant-time comparison (subtle crate，防时序攻击)
5. 撤销   → 作业清理时立即从 TokenStore 移除

内部 API (:50051)
├─ LLM 代理: 容器不直接访问 LLM API，通过宿主代理
├─ 凭据注入: 容器请求凭据，宿主验证令牌后注入
└─ 状态上报: 进度、结果、错误
```

**安全属性**:
- 容器无法伪造令牌 (CSPRNG + 256 bit entropy)
- 容器无法嗅探其他作业令牌 (网络隔离)
- 即使容器被攻陷，令牌仅授权单一作业
- 时序攻击防护 (constant-time comparison)

**启发**:
- OpenFang worker session 可采用 per-session 令牌绑定
- Bridge RPC 认证从简单 process ID 升级为密码学令牌

---

## 10. 工作区卫生系统 (Workspace Hygiene)

**已有框架**: 无。三大框架均无自动清理机制。

**IronClaw 方案** (`src/workspace/hygiene.rs`, ~400 行):

```
自动清理
├─ 每日日志 > 30 天自动删除
├─ 临时文件定期回收
└─ 状态文件压缩

安全保障
├─ AtomicBool 全局锁 → 防止并发清理
├─ 原子状态保存: 写临时文件 → rename (防止写入中断导致损坏)
└─ 节奏控制: 12 小时间隔 (默认)，平衡清洁度与性能

触发方式
├─ Heartbeat 结束后异步触发 (fire-and-forget)
└─ 手动触发
```

**启发**:
- OpenFang 长期运行时需要类似机制，防止日志和临时文件膨胀
- 可在 `openfang-kernel` 增加 `HousekeepingService`

---

## 交叉总结: 对三大框架的潜在借鉴

### 对 OpenFang 最直接的借鉴

| IronClaw 方案 | OpenFang 落地建议 | 优先级 |
|---|---|---|
| 自修复 | `PluginHostRegistry` 增加 `Degraded → Reloading` 路径 | P1 |
| 凭据泄漏主动扫描 | Bridge RPC 出站流量过 LeakDetector | P1 |
| Per-Job 密码学令牌 | Worker session 令牌绑定 + constant-time 验证 | P1 |
| WASM 频道隔离 | Channel adapter 纳入 WASM 沙箱 | P2 |
| 四层韧性栈 | LLM 层增加 Circuit Breaker + Retry Jitter | P2 |
| 工作区卫生 | 增加 HousekeepingService | P3 |

### 对 ZeroClaw 的借鉴

| IronClaw 方案 | ZeroClaw 落地建议 |
|---|---|
| RRF 混合搜索 | 组合现有全文搜索 + 向量搜索 |
| 信任天花板 | 插件权限模型增加最低信任约束 |
| 13 维智能路由 | 升级简单故障转移为复杂度感知路由 |

### 对 OpenClaw 的借鉴

| IronClaw 方案 | OpenClaw 落地建议 |
|---|---|
| 动态工具构建 | 技能系统增加运行时工具生成 |
| 多层泄漏检测 | 扩展现有安全层，增加输出扫描 |
| 工作区卫生 | 长期运行实例的自动清理 |

---

## 附录: IronClaw 关键文件索引

| 子系统 | 文件路径 | 行数 |
|---|---|---|
| Self-Repair | `src/agent/self_repair.rs` | ~529 |
| Heartbeat | `src/agent/heartbeat.rs` | ~448 |
| Cost Guard | `src/agent/cost_guard.rs` | ~403 |
| Routine Engine | `src/agent/routine_engine.rs` | ~900+ |
| Scheduler | `src/agent/scheduler.rs` | ~400+ |
| Workspace Hygiene | `src/workspace/hygiene.rs` | ~400+ |
| Smart Routing | `src/llm/smart_routing.rs` | ~700 |
| Circuit Breaker | `src/llm/circuit_breaker.rs` | ~500 |
| Retry | `src/llm/retry.rs` | ~300 |
| Failover | `src/llm/failover.rs` | ~900+ |
| Leak Detector | `src/safety/leak_detector.rs` | — |
| Credential Detect | `src/safety/credential_detect.rs` | — |
| Tool Builder | `src/tools/builder/` | — |
| WASM Tool Runtime | `src/tools/wasm/` | — |
| Hybrid Search | `src/workspace/search.rs` | — |
| WIT (Tool) | `wit/tool.wit` | — |
| WIT (Channel) | `wit/channel.wit` | — |
| Skills | `src/skills/` | — |
