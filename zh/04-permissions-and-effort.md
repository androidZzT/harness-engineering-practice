# Harness 怎么拿捏 agent：权限与 effort

上一篇《Harness 怎么扩展：skill、配置目录与 hook》说的是"往 harness 里加东西"——把可复用工作流打包成 skill，把约束写进配置目录，把监控逻辑挂到 hook。这篇转一个角度：已经给 agent 装好工具之后，怎么管住它？

两个具体问题：一，agent 改文件、执行命令、发网络请求，怎么决定哪些操作要问我、哪些自动放行；二，同一个模型，怎么按场景调节"想多深"，既不浪费 token，又不让复杂任务打折。

CC 和 Codex 对这两个问题都有解法。和前几篇一样，差异不在谁有谁没有，而在**面对同一个控制问题，两家工程上的解法走了不同的路**。

---

## 一、权限


### 解决什么问题

Agent 在运行中会改文件、执行命令、发网络请求。完全放开有风险，每次都问又拖垮效率。核心矛盾是：**怎样在"少打断"和"可控"之间取一个能调的平衡点**，并且让这个平衡点能按"个人 / 项目 / 企业"分层固定下来。

### Codex 怎么做

Codex 把授权拆成两个正交维度，组合成一张授权矩阵。

**维度一：沙箱策略**，定义 agent 的操作边界。枚举类型 `SandboxPolicy`（`codex-rs/protocol/src/protocol.rs:878`），4 个变体：

- `DangerFullAccess`（序列化 `"danger-full-access"`）—— 无限制
- `ReadOnly { network_access: bool }` —— 只读，网络默认关
- `ExternalSandbox { network_access }` —— 已在外部沙箱，放开磁盘但遵守传入的网络设置
- `WorkspaceWrite { writable_roots, network_access, exclude_tmpdir_env_var, ... }` —— 在 ReadOnly 基础上额外允许写当前工作区 + 列出的 `writable_roots`

**维度二：审批策略**，决定何时停下来问用户。枚举类型 `AskForApproval`（`codex-rs/protocol/src/protocol.rs:784`），5 个变体：

- `UnlessTrusted`（`"untrusted"`）—— 只有"已知安全且只读"的命令自动批，其余都问
- `OnFailure`（已标记 DEPRECATED）—— 沙箱内全自动批，失败才升级给用户
- `OnRequest`（`#[default]`，默认档）—— 由模型决定何时该问
- `Granular(GranularApprovalConfig)` —— 按类别细粒度开关，某类设 `true` 放行，`false` 直接自动拒（不弹给用户）
- `Never` —— 从不问，失败直接返回给模型，不升级

`Granular` 变体内的 `GranularApprovalConfig`（同文件 `protocol.rs`，紧随 `AskForApproval` 定义）有 5 个类别字段：`sandbox_approval`（shell 命令审批，含内联提权）、`rules`（execpolicy `prompt` 规则触发的提示）、`skill_approval`（skill 脚本执行）、`request_permissions`（`request_permissions` 工具触发）、`mcp_elicitations`（MCP elicitation 提示）。

两个维度的分工：沙箱策略管"边界"（文件读写范围 + 网络开关），审批策略管"打断节奏"（何时停下来问）。两者独立，可以任意组合。

### CC 怎么做

CC 把授权做成三层叠加：模式（baseline）+ 规则（allow/ask/deny）+ 分类器（auto mode 实时审）。

**6 种权限模式**（permission-modes 页），Shift+Tab 在 `default→acceptEdits→plan` 之间循环：

- `default` —— 只有只读操作不问
- `acceptEdits` —— 读 + 文件编辑 + 工作目录内的 `mkdir`/`touch`/`mv`/`cp`/`rm`/`rmdir`/`sed` 不问
- `plan` —— 只读，研究并给方案，不改源码
- `auto` —— 几乎都不问，但有独立分类器模型在动作执行前实时审；标注 research preview；需 Claude Code v2.1.83+，模型需 Opus 4.6+ 或 Sonnet 4.6
- `dontAsk` —— 只跑预批工具，其余一律自动拒；适合锁定的 CI 或脚本
- `bypassPermissions` —— 跳过所有检查全部不问，等价于 `--dangerously-skip-permissions`，只用于隔离容器/VM；`rm -rf /` 和 `rm -rf ~` 仍作熔断提示

除 `bypassPermissions` 外，对 protected paths（`.git`/`.claude`/`.vscode`/`.idea` 等）的写入永不自动批。

**规则三类**（permissions 页）：`allow`（不问直接用）、`ask`（每次确认）、`deny`（禁止）。评估序是 deny → ask → allow，首条匹配胜出，deny 始终优先。

细节值得记：bare 工具名 deny（如 `Bash`）会把工具从上下文整个移除；scoped deny（如 `Bash(rm *)`）保留工具，只拦匹配的调用。

**settings 5 级 precedence**：Managed（企业 IT 推送，最高，不可被任何层覆盖）> 命令行参数 > Local（`.claude/settings.local.json`）> Project（`.claude/settings.json`）> User（`~/.claude/settings.json`）。任一层 deny，其它层都不能 allow。

**auto mode 分类器**（permission-modes 页）有几个细节：

独立分类器模型在动作执行前审，拦截三类场景：超出请求范围的升级、指向不认识的基础设施、被读到的敌意内容驱动的动作。决策序是先走 allow/deny 规则，只读及工作目录内编辑直接放，其余交分类器，拦截时把理由回传给模型。

进入 auto mode 时会丢弃宽泛 allow 规则（`Bash(*)`、通配解释器、包管理 run、`Agent` 规则），离开时恢复。分类器只看 user 消息、工具调用、CLAUDE.md，工具结果被剥离（防注入）。

回退阈值：连续 3 次或累计 20 次被拦，auto mode 暂停，转回逐条提问（阈值不可配）。

会话里口头说的边界（"先别 push"）会被当作 block 信号，但不持久化，compaction 删了那条消息就可能失效，硬保证要写 deny 规则。

### 对比

| 维度 | Codex | CC |
|------|-------|----|
| 核心模型 | 两个正交维度：沙箱策略 × 审批策略 | 三层叠加：模式 + 规则 + 分类器 |
| 档位粒度 | 4 沙箱 × 5 审批，组合矩阵 | 6 种模式（含 dontAsk 全预批） |
| 最细粒度收口 | `Granular`：5 个类别字段独立开关 | scoped deny：`Bash(rm *)` 级别 |
| 语义审查 | 无（确定性规则组合） | auto mode 分类器，语义把关 |
| 企业策略锁定 | 沙箱策略 + 审批策略约束组合 | Managed settings，5 级优先级最高 |

Codex 偏"策略把关"：两个确定性维度正交组合，开源可证，行为可预测。CC 偏"模型把关 + 策略兜底"：auto mode 用独立分类器做语义判断（判断"是否超出请求 / 是否外部基础设施 / 是否被注入驱动"），deny 规则层兜底。

### 实操建议

**Codex**：日常用 `WorkspaceWrite` + `OnRequest`（默认组合）；无人值守批跑用 `Never`，但务必配合收窄的沙箱（关网络、限 `writable_roots`）；`Granular` 适合"放行 shell 但卡住 MCP / skill 脚本"这类精细场景。

**CC**：复盘式编码用 `acceptEdits`，配合编辑器或 `git diff` 事后看；长任务减少打断用 `auto`，但它是 research preview，敏感操作仍要人看；CI 用 `dontAsk` + 预批 allow 列表；`bypassPermissions` 只在隔离容器。

**通用**：企业安全边界用 Managed 或约束层锁死，不要试图用项目层覆盖；硬边界写 deny 规则，不要只靠对话里口头说（compaction 可能把那条消息删掉）。

---

## 二、effort


### 解决什么问题

同一个模型，可以"少想快答"，也可以"多想深答"。简单任务想太多浪费 token，还可能 overthinking；复杂任务想太少又不够用。所以需要一个能调"想多深"的档位，并能按模型、按场景设默认值。

### Codex 怎么做

**`ReasoningEffort` 枚举 6 档**（`codex-rs/protocol/src/openai_models.rs:45`）：

`None` / `Minimal` / `Low` / `Medium`（`#[default]`）/ `High` / `XHigh`，序列化为 `"none"/"minimal"/"low"/"medium"/"high"/"xhigh"`，默认 `Medium`。

**客户端只做一件事：透传**。`build_reasoning`（`codex-rs/core/src/client.rs:715`）把 effort 组装进请求，关键行是 `effort: effort.or(model_info.default_reasoning_level)`（`:722`），最终变成 OpenAI Responses API 请求体里的 `{"reasoning":{"effort":"high"}}`。

Codex 客户端不改 system prompt、不改工具列表、不调上下文窗口、不做任何本地推理增强。把字符串发给服务端，怎么把 `"high"` 变成实际算力是云端模型的事。

**三层 fallback**（`effective_reasoning_effort`，`codex-rs/core/src/session/turn_context.rs:125`）：当前 turn 用户设的值 → 模型默认档（`default_reasoning_level`）→ 模型不支持 reasoning 则整个 `reasoning` 字段不发（注意：是不发字段，不是发 `"none"`）。

**切模型时的档位平移**：`effort_rank`（`openai_models.rs:557-562`，None=0 / Minimal=1 / Low=2 / Medium=3 / High=4 / XHigh=5）配合 `nearest_effort`（`:566`），按 rank 差绝对值最小，映射到目标模型支持的最近档。

**Plan 模式独立 effort**：`plan_mode_reasoning_effort`（`codex-rs/core/src/config/mod.rs:873`）和执行阶段是独立字段，Plan 阶段可单独配更高档。

### CC 怎么做

**档位按模型而定**（model-config 页）：

- Opus 4.8 / Opus 4.7：`low` / `medium` / `high` / `xhigh` / `max`
- Opus 4.6 / Sonnet 4.6：`low` / `medium` / `high` / `max`
- 默认档：Opus 4.8、Opus 4.6、Sonnet 4.6 均为 `high`；Opus 4.7 为 `xhigh`
- 设了当前模型不支持的档，回退到不超过该档的最高支持档（例：在 Opus 4.6 上设 `xhigh` 实际跑 `high`）

**持久性差异**：`low`/`medium`/`high`/`xhigh` 跨会话持久；`max` 仅当前会话生效（除非走 `CLAUDE_CODE_EFFORT_LEVEL` 环境变量）。`max` 是最深推理、token 不封顶的档，官方文档原话是 "prone to overthinking, test before adopting broadly"。

**还有两样东西，不是模型档位，是 CC 自己加的**：

`ultracode` 值得单独多说，因为它常被当成"Opus 4.8 多出来的一个推理档"，其实不是。它是 CC 的会话级设置：启用后对模型发 `xhigh`，同时让 Claude 为每个实质任务自动编排一套 **dynamic workflow**——一段由 Claude 现写、在后台运行的 JavaScript 编排脚本，把工作拆给几十到上百个 subagent 并行跑（动态工作流于 v2.1.154 引入）。

它和普通 subagent 编排的关键区别：脚本自己持有循环、分支和中间结果，Claude 的上下文里只留最终答案。所以它适合"一个对话协调不过来"的大任务——全仓 bug 扫描、几百文件迁移、多源交叉验证的调研；脚本还能固化质量套路，比如让多个 agent 互相对抗式复核彼此的结论再汇报。

触发有两条路：prompt 里直接要（用自己的话，或写关键词 `ultracode`），或 `/effort ultracode` 让 Claude 在整个会话里对每个实质任务都自动规划一套。它只在支持 `xhigh` 的模型上出现在 `/effort` 菜单里（即 Opus 4.8 / 4.7），其余模型不提供。仅当前会话生效，不属于 `effortLevel` 设置、`--effort` flag 或 `CLAUDE_CODE_EFFORT_LEVEL`。

`ultrathink` 关键词：在 prompt 里任意位置写 `ultrathink`，请求当轮更深推理。不改会话 effort，发给 API 的 effort 值不变，只是在上下文里加了一条 in-context 指令。"think" / "think hard" 等不被识别为关键词。

**设置入口**：`/effort`（无参开滑块 / `/effort <档名>` 直接设 / `/effort auto` 回模型默认）；`/model` 里左右键调滑块；`--effort` flag；`CLAUDE_CODE_EFFORT_LEVEL` 环境变量；`effortLevel` 设置（仅接受 low~xhigh，`max` 和 `ultracode` 不接受）；skill / subagent frontmatter 的 `effort` 字段。优先级：环境变量 > 配置档 > 模型默认。

### 对比

| 维度 | Codex | CC |
|------|-------|----|
| 档位数量 | 6 档（None/Minimal/Low/Medium/High/XHigh） | 按模型，最多 5 档（low/medium/high/xhigh/max） |
| 默认档 | `Medium` | 多数模型 `high`（Opus 4.7 为 `xhigh`） |
| 客户端做不做加工 | 不加工（纯透传，只改 API 一个字段） | 有额外封装（ultracode 触发后台工作流编排） |
| 单轮临时加深 | 无专用关键词 | `ultrathink` 关键词（不改会话档） |
| max 持久性 | N/A | 仅当前会话（非环境变量设置时） |
| 切模型档位映射 | `nearest_effort`：绝对值最近档（可能升档） | 取不超过目标档的最高支持档（只降不升） |
| Plan 阶段独立 effort | 有（`plan_mode_reasoning_effort` 独立字段） | 无专用字段 |

两家共享 low/medium/high/xhigh 的中段命名，都有模型默认档和回退机制。差异在于：Codex 把 effort 当成一个只往 API 传、本地不加工的字段，源码可证；CC 在同样的档位之外又加了几样自家的——向上有 `max`（不封顶）、`ultracode`（effort 顺带后台工作流编排）、`ultrathink`（单轮临时加深，不动会话档）。

回退算法也不同：Codex `nearest_effort` 按 rank 差绝对值最小映射，可能比目标档更高；CC 取"不超过目标档的最高支持档"，只降不升。

### 实操建议

**Codex**：默认 `Medium` 适合多数场景；需要深推理调 `High` / `XHigh`；给 Plan 模式单独配更高档（规划值得多想，执行阶段不必同等档位）；记住调档只改一个传给服务端的字符串，本地不会"更努力"。

**CC**：日常 `high`（Opus 4.8 默认）够用；难题临时加深直接在 prompt 写 `ultrathink`，不用动会话设置；要 Claude 自动拆任务并行编排，用 `ultracode`；`max` 谨慎使用，可能 overthinking，先小范围试验。

**通用**：effort 是按模型校准的，同一个档名在不同模型不代表同一份算力，切模型后确认实际生效档。

---

## 小结

两节下来，两家控制面的方向可以各用一句话概括：

**Codex**：权限是"策略矩阵"（两个确定性维度正交组合，开源可证），effort 是"只传不改的档位"（拨到哪档就原样发哪档，本地不加工）。可预测性优先，行为可检查。

**CC**：权限是"模型把关 + 策略兜底"（auto mode 分类器语义审，deny 规则硬兜底），effort 是"档位 + 几样加料"（max 不封顶、ultracode 连带后台编排、ultrathink 临时加深）。灵活度优先，把控制面做厚。

两套解法没有对错，适合不同场景。需要行为强可预测、安全边界机器可验的，Codex 的组合矩阵更直接；需要减少人工干预、靠语义判断处理复杂边界的，CC 的分类器路径更省心。

---

## Harness Engineering 系列

围绕 coding agent 的「平台层（harness）与业务工程」，逐篇拆解：

1. [Harness 到底指什么](./01-what-is-harness.md) —— 平台层与业务工程的边界
2. [复杂任务的 Spec 怎么写](./02-how-to-write-specs.md) —— 多 Agent、编排者入口、rules / docs / skills 组织
3. [Harness 怎么扩展](./03-extending-the-harness.md) —— skill、配置目录与 hook：CC 与 Codex 的两套机制
4. [Harness 怎么拿捏 agent](./04-permissions-and-effort.md) —— 权限与 effort：CC 与 Codex 的控制面
5. Harness 怎么扛住长任务 —— compact、memory、goal（写作中）

> 其他语言：[English](../en/04-permissions-and-effort.md) · [한국어](../ko/04-permissions-and-effort.md) · [日本語](../ja/04-permissions-and-effort.md) · [中文](../zh/04-permissions-and-effort.md)

