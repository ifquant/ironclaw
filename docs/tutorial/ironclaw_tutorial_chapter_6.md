# Chapter 6：技能系统与信任天花板：Prompt 扩展如何进入权限模型 🛠️🔒

到了这一章，IronClaw 又会显出一个和很多 agent 系统很不一样的判断：**prompt 扩展不是纯认知资产，它也是权限资产。**

很多系统里的 skills、prompts、agents instructions，本质上只是“给模型多塞一点文字”。它们影响回答风格、任务偏好、工具使用倾向，但很少直接影响能力边界。IronClaw 不这么看。

在它这里，skill 不只是“多一段提示词”，而是一个带来源、带要求、带信任级别、还能决定工具上限的运行时对象。

这一章最重要的一句话是：

> IronClaw 把 skill 系统从“提示词拼接器”升级成了“带信任约束的能力调制器”。

## 6.1 Skill 在 IronClaw 里不是文件，而是加载后的受控对象

先从文件格式看，IronClaw 的 skill 使用 `SKILL.md`，结构是：

- YAML frontmatter
- Markdown body

这和很多 prompt pack 看起来很像，但源码里的加载模型完全不是“读文本然后塞给 LLM”这么简单。

`SkillManifest` 至少会带这些信息：

- `name`
- `version`
- `description`
- `activation`
- `metadata`

其中 `activation` 又包含：

- `keywords`
- `patterns`
- `tags`
- `max_context_tokens`

加载后，系统里真正工作的对象是 `LoadedSkill`。它不仅有 prompt 内容，还有：

- `trust`
- `source`
- `content_hash`
- `compiled_patterns`
- 预先 lowercased 的 keywords / tags

这说明 IronClaw 根本没把 skill 当成一段随手拼装的文本，而是当成一个会参与选择、预算、权限和缓存的运行时单元。

## 6.2 Skill Source 决定 Trust，而不是作者自己自称可信

很多系统最大的风险在于：只要一个 prompt 文件被放进来，它就默认和用户自己写的指令享有一样高的权重与权限。IronClaw 在这里非常谨慎。

从加载来源看，skill 至少会区分这些 source：

- `Workspace(PathBuf)`
- `User(PathBuf)`
- `Bundled(PathBuf)`
- 注册表安装路径

而真正关键的是 `SkillTrust`。它不是一个抽象概念，而是明确枚举：

- `Installed = 0`
- `Trusted = 1`

这个顺序本身就带有语义：数值越低，权限越低。也就是说，“来源外部、通过注册表装进来的 skill”和“用户本地明确放置的 skill”在系统里不是同一种东西。

这一步非常重要，因为它打破了一个在 agent 系统里很危险的默认假设：**所有 prompt 扩展都同样可信。**

## 6.3 最关键的设计：取最小信任，而不是分别看每个 skill

这一章里最有辨识度、也最值得记住的机制，是所谓的“信任天花板”。

IronClaw 并不是给每个 skill 单独分配一套可用工具，然后把结果拼在一起。它做的是更保守的一步：

> 当前激活 skill 集合里的**最低信任等级**，决定整轮工具可见性的上限。

也就是说：

- 没有 active skills → 默认 `Trusted` 视角
- 全部是 trusted skills → 仍然保留完整工具集
- 只要混进任何一个 `Installed` skill → 整轮降到 Installed ceiling

这是一个非常不常见、但工程上很稳的选择。

因为一旦把不同信任级别 skill 混在同一轮推理里，LLM 并不会替你严格分辨“这段 prompt 只能读、那段 prompt 可以写”。在这种情况下，最安全的策略就是**按最低信任收敛**。

## 6.4 工具衰减不是事后拦截，而是 LLM 看见之前就裁掉

前面说“信任天花板”决定工具上限，但更关键的是它在哪里生效。

IronClaw 在 `attenuation.rs` 里做的不是：

- 先把所有工具都给 LLM 看
- 等它调用时再拒绝危险工具

而是：

- 先算出当前 active skills 的最小 trust
- 直接把工具定义列表裁掉
- 只把保留下来的工具发给 LLM

这一点极其重要，因为它意味着：

- LLM 根本不知道那些被裁掉的工具存在
- 不存在“被 prompt 操纵后还试图调用高权限工具”的机会
- 权限控制在认知面之前就完成了

这比运行时拒绝调用更强。因为后者仍然允许模型围绕“那个其实存在的危险工具”进行计划和试探，而前者直接从认知空间里拿掉它。

## 6.5 Read-Only 工具集是显式白名单，而不是模糊分类

一旦最低 trust 降到 `Installed`，IronClaw 不会靠“感觉上这些工具应该比较安全”来放行，而是使用一组硬编码只读工具白名单。

源码里这组只读工具大致包括：

- `memory_search`
- `memory_read`
- `memory_tree`
- `time`
- `echo`
- `json`
- `skill_list`
- `skill_search`

这个设计看起来很保守，但正因为保守，它才有很强的安全语义：**新工具默认不在只读集合内，因此默认被拦。**

这和那种“工具自己声明自己安全”完全不同。IronClaw 的逻辑是：默认不信，除非被宿主明确列入低风险白名单。

## 6.6 选择技能之前，不需要 LLM 参与

另一个很成熟的点，在 `src/skills/selector.rs`：技能选择是确定性的，不需要 LLM。

这是一个很重要的架构判断。因为如果 skill 选择本身还要靠模型，就会立刻出现几个问题：

- skill 内容可能反过来影响自己是否被选中
- selection 过程要额外消耗 token
- 选择机制本身变得不可预测、不稳定、难调试

IronClaw 明确避免了这些问题。它的 prefilter 是纯确定性的。

### 6.6.1 打分规则是明确的，而不是黑箱“相关性”

从源码看，至少有这样一套评分逻辑：

- keyword exact match：10 分
- keyword substring match：5 分
- tag match：3 分
- regex pattern match：20 分

而且每一类还有上限，防止有人通过堆很多关键词来刷分。

这套设计很实用，因为它平衡了两件事：

- 足够简单，开发者能预测为什么某个 skill 被激活
- 足够灵活，不至于只支持精确匹配

### 6.6.2 预算控制在选择阶段就开始了

skill 的 `max_context_tokens` 也不是摆设。选择器在排序后，还会考虑 token budget，把最终能装进上下文的 skill 数量限制住。

这说明 IronClaw 从一开始就承认：skill 不是越多越好。技能系统不仅要解决“选谁”，还要解决“塞多少”。

否则很容易出现另一个常见问题：skill 很聪明，但 prompt 被塞爆了。

## 6.7 Gating：skill 不是装上就能用，还得满足本地依赖

在 `src/skills/gating.rs`，IronClaw 又补了一层很有工程味的约束：skill 可以声明自己的运行前提。

例如 metadata 里可以要求：

- 某些 binary 必须存在
- 某些环境变量必须存在
- 某些配置文件必须存在

这一步的意义非常大，因为它把“skill 能否激活”从纯文本相关性问题，扩展成了“本机现在有没有这个执行前提”的问题。

更成熟的是，gating 失败时不是整个系统报错，而是**跳过该 skill**。

这很对。因为缺一个依赖并不该让整个 agent 崩掉，它只意味着这个 skill 在当前环境里暂时不可用。

## 6.8 Discovery 顺序也体现了信任模型

SkillRegistry 的发现顺序同样值得注意。大致上会按这样的优先级扫描：

1. workspace 目录
2. 用户目录
3. installed 目录

这里体现了很清楚的本地优先哲学：

- 距用户越近的 skill，越值得信任
- 外部安装来的 skill，即便存在，也处在更低 trust 层级

再加上同名冲突时前面的来源优先，这就让用户本地 skill 有能力覆盖或优先于外部技能，而不会被远端安装包悄悄抢占行为。

## 6.9 Skill 真正新颖的地方，不是“会选 skill”，而是“会降能力”

如果只看表面，很多系统都有类似 skill/plugin/prompt pack 功能。但 IronClaw 的新意不在有无，而在它回答了一个多数系统没有认真回答的问题：

> 当一段 prompt 扩展不是完全可信时，它到底能影响系统到什么程度？

IronClaw 给出的答案是：

- 它可以影响 LLM 上下文
- 但它不能因此获得更高工具权限
- 相反，它的存在还会拉低整轮可见工具上限

这就是“skill 进入权限模型”的真正含义。

它不只是让 prompt 和 permission 发生关系，而是直接用 prompt 来源去收敛 capability ceiling。

## 6.10 这对 OpenClaw / OpenFang 的启发

这一章最值得拿去对照前面几个系统的，不是 skill 文件格式，而是这几条设计判断：

1. **skill 选择要确定性，最好不依赖 LLM。**
2. **skill source 应该决定 trust，不要默认一律可信。**
3. **工具权限应在 LLM 看见工具之前先裁剪。**
4. **混合信任 skill 时，权限按最低信任收敛。**
5. **只读工具集应该是宿主显式白名单，而不是自报安全。**

这些判断如果放到 OpenFang 里，会直接影响插件与 skill 的组合安全边界；放到 OpenClaw 里，则会把原本偏认知层的技能系统推进到真正的能力治理层。

## 6.11 为什么这一章和安全章节不同样重要

Chapter 3 讲的是外部内容怎么不污染系统；这一章讲的则是：**系统内部引入的“帮助性提示资产”本身，也可能改变风险轮廓。**

很多人只盯着外部攻击，却忽略了另一种更隐蔽的风险：

- 安装了一个看起来很聪明的社区 skill
- 它确实提高了问题解法质量
- 但与此同时，它开始影响模型如何规划高权限工具调用

IronClaw 在这里的处理非常成熟：不是禁止 skill，而是让 skill 的能力影响可被显式限制。

## 6.12 代码定位

这一章最值得边读边看的文件：

- `src/skills/mod.rs`
- `src/skills/catalog.rs`
- `src/skills/selector.rs`
- `src/skills/gating.rs`
- `src/skills/attenuation.rs`

下一章进入最后一个最有新意的主题：IronClaw 如何把“构建新工具”本身做成运行时能力，以及这套 builder / registry / extension lifecycle 为什么值得单独拆开讲。
