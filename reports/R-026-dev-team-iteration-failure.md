# R-026: AI Agent 开发团队项目迭代成功率研究

> 研究日期：2026-04-01
> 主题：以 Claude Code 等为鉴，分析 dev team 迭代失败根因并提出改进方案

---

## 一、Claude Code Plan Mode 与代码扫描机制分析

### 1.1 Plan Mode 实现机制

Claude Code 的 Plan Mode 本质上并不复杂——它是一套**prompt 驱动的状态机**，而非架构层面的特殊功能。根据 Armin Ronacher（Flask 作者）的逆向分析 [1]：

**三大核心要素：**

1. **系统提醒约束**：每次用户消息后自动注入 "Plan mode is active. You MUST NOT make any edits" 提示，且声明该指令优先级最高（"This supersedes any other instructions"）

2. **四阶段结构化工作流**（通过 prompt 注入）：
   - **Phase 1: Initial Understanding** — 阅读代码、向用户提问
   - **Phase 2: Design** — 设计实现方案，使用 Explore Agent 和 Plan Agent 并行收集信息
   - **Phase 3: Review** — 对齐计划与用户意图
   - **Phase 4: Final Plan** — 将最终计划写入 plan 文件（agent 唯一可写的文件）

3. **ExitPlanMode 工具**：计划完成后 agent 调用此工具请求用户审批。该工具不接收计划内容作为参数，而是自动从 plan 文件读取，确保审批与文件一致。

**关键洞察**：Plan Mode 并没有真正禁用写工具——只是通过 prompt 强化"只读"行为。Agent 可以编辑 plan 文件（使用 EditFile 工具），但不能修改项目代码。整个计划以 markdown 文件形式存在于 `.claude/plans/` 目录。

### 1.2 代码扫描与上下文管理

Claude Code 的架构 [2] 基于五个设计支柱：

| 支柱 | 机制 | 对我们的启示 |
|------|------|-------------|
| **Model-Driven Autonomy** | TAOR 循环（Think-Act-Observe-Repeat），模型决定下一步而非硬编码脚本 | dev team 的流程不应是固定的 spawn 链，而应让 dev-lead 自主决定下一步 |
| **Context as a Resource** | 自动 compaction（压缩）、语义搜索保护上下文窗口 | 我们需要上下文管理策略，而非无限堆叠历史 |
| **Layered Memory** | 6 层记忆在会话启动时加载（组织策略、个人偏好、MEMORY.md 等） | dev-coder/dev-qa spawn 时应自动注入项目关键文件 |
| **Declarative Extensibility** | 通过 .md/.json 声明 skills、agents、hooks | AGENTS.md 即是此模式，但需要更精细的结构 |
| **Composable Permissions** | 工具级别的 allow/deny/ask | dev-lead/dev-qa 的工具权限需要精确配置 |

**上下文管理细节：**
- **Auto-Compaction**：当对话接近上下文限制时，自动将历史 turn 压缩为摘要
- **Tool Search Tool**：优化工具在上下文中的表示方式，减少 token 消耗 [3]
- **6 层记忆加载**：会话开始时加载 CLAUDE.md、MEMORY.md、组织策略等

### 1.3 错误恢复与迭代机制

Claude Code 的关键设计：
- **不是多 agent 协作**，而是单 agent + 工具循环。这避免了 agent 间信息丢失问题
- **通过 Bash 实现万能适配**：git、npm、docker 等都通过 Bash 调用，意味着 git stash/checkout 天然可用
- **无显式回退机制**：依赖 agent 自主决定何时回退，通过 git 等工具实现

---

## 二、业界最佳实践对比

### 2.1 Cursor Agent 模式

- **多文件修改**：Agent 模式可运行数小时、完成多文件重构、迭代直到测试通过 [11]
- **关键实践**：当 agent 表现下降时，**启动新对话**而非继续修复。使用 `@Past Chats` 引用历史，将计划保存到 `.cursor/plans/` 作为团队文档和未来 agent 的上下文 [11]
- **Plan Mode**：Shift+Tab 切换，Agent 先搜索代码库→提出澄清问题→创建带文件路径和代码引用的计划→等待用户批准→再编码 [11]
- **上下文管理**：通过语义搜索和 grep 工具按需查找上下文，不需要手动标注所有文件 [11]
- **错误恢复**：出错时回退到 plan，revert 变更，refine 计划后重新执行——而非修复进行中的 agent [11]

### 2.2 Windsurf Cascade

- **双 Agent 架构**：后台专用 planning agent 持续优化长期计划，前台模型专注短期执行 [12]
- **Todo 列表**：在对话中创建 Todo 列表跟踪复杂任务进度，执行中可自动更新计划 [12]
- **工具调用限制**：每次 prompt 最多 20 次工具调用，达到限制后需按 continue 恢复 [12]
- **错误恢复**：丢弃变更→修改 plan markdown→点击 Implement 在新对话中重新执行 [12]

### 2.3 Devin

- **Playbook 方法**：Cognition 总结的最佳实践 [4]：
  - "告诉 agent 怎么做，而不只是做什么"（Say how, not just what）
  - "告诉 agent 从哪里开始"（Tell where to start）
  - "防御性提示"（Defensive prompting）
  - "给 agent 访问 CI、测试、类型检查器、linter 的权限"——**强反馈循环是 agent 自我纠错的关键**
  - "从 untyped Python 迁移到 typed Python"——更强的类型约束帮助 agent 减少错误
- **Autofix 闭环**：代码审查或 CI/lint 发现问题后自动修复 PR，反复迭代直到所有检查通过。Cognition 团队一周合并 659 个 Devin PR [13]
- **Devin 2.0**：比 1.0 完成 83% 更多任务，关键改进是"better error recovery and smarter resource allocation" [5]
- **关键局限**：当代码库超出 agent 上下文窗口时，会丢失架构决策和类型定义的追踪 [14]

### 2.4 OpenHands / SWE-agent

- **SWE-Bench 成绩**：OpenHands CodeAct 2.1 在 SWE-Bench 上达到 53% 解决率 [6]
- **架构**：单 agent + 工具循环（类似 Claude Code），而非多 agent 协作
- **SWE-EVO 惊人发现**：agent 在"连续修复"场景（SWE-EVO，21%）远不如单次修复（SWE-Bench Verified，65%）[7]——**这直接印证了我们的多轮迭代失败问题**
- **SWE-Compressor**：基于结构化上下文工作区（任务语义+长期记忆+短期交互），将上下文管理作为可调用工具集成到 agent 决策过程中，达到 57.6% [15]

### 2.5 Block (Goose) 的规划方案

- **三种规划策略** [16]：
  1. `/plan` 模式：交互式对话生成计划，**可用不同模型分别做规划和执行**
  2. 指令文件模式：用户写 markdown 计划，`goose run -i plan.md` 执行
  3. 对话式上下文构建
- **关键洞察**：规划和执行可以用不同模型——我们 dev-lead 可以用强推理模型做规划，dev-coder 用快速模型做执行

### 2.6 Martin Fowler 的 Context Engineering 总结

Martin Fowler 的分析 [3] 指出：
- **上下文不是越多越好**：过多上下文反而降低 agent 效果
- **LLM 自主决定何时加载上下文** 是无监督运行的前提，但存在不确定性
- **Claude Code 的 Skills 机制**：agent 按需加载额外指令和文档，而非一次性全部注入
- **逐步构建上下文**：不要一开始就塞入太多规则文件

---

## 三、多轮迭代退化的学术证据

### 3.1 "LLMs Get Lost In Multi-Turn Conversation"（微软研究院 & Salesforce, 2025）

这是**最直接相关的学术发现** [17]：

- LLM 在多轮对话中**平均性能下降 39%**，从单轮 90% 降至多轮 65%
- 即使仅 2 轮对话也会出现下降
- 退化可分解为：**轻微的能力下降 + 显著的不可靠性增加**
- "当 LLM 在多轮对话中走错方向时，它们会迷路且无法恢复"

**四个具体退化行为**（直接对应我们观察到的问题）：
1. 生成过于冗长的回复 → dev-coder 添加不必要代码
2. 过早提出最终方案 → 第 1 轮就用错误方案
3. 对未明确细节做错误假设 → 在不了解项目的情况下修改代码
4. **过度依赖之前（错误的）回答** → 7 轮修复都在同一个错误方向上

### 3.2 "SlopCodeBench"（威斯康星大学 & MIT, 2025）

- 编码 Agent 在长程迭代任务中**代码质量持续退化** [18]：
  - 冗余代码（verbalosity）在 **89.8%** 的轨迹中上升
  - 结构侵蚀（structural erosion）在 **80%** 的轨迹中上升
  - **没有任何 Agent 能在任何问题上端到端完成所有 checkpoint**，最高解决率仅 17.2%
- **关键发现**：初始代码质量可以通过 prompt 干预改善，但**无法阻止后续迭代的退化**

### 3.3 "Why Do Multi-Agent LLM Systems Fail?"（UC Berkeley, 2025）

多 Agent 系统的具体失败模式 [19]：
- **回合重复（step repetition）**：17.14%——agent 反复做相同的事
- **任务完成条件识别失败**：9.82%——不知道什么时候该停（即"假通过"）
- **上下文丢失**：3.33%

### 3.4 Agent 会话质量随时间退化

实测发现 [20]：
- **0-30 分钟**：精确编辑，正确的跨文件引用
- **30-60 分钟**：偶尔遗漏 import，仍可恢复
- **60-90 分钟**：单文件隧道视野，丢失架构上下文
- **90 分钟后**：重复尝试，与之前决策自相矛盾

### 3.5 Agent 成功率随步骤数指数级下降

- 5 步约 77%，10 步约 59%，20 步约 36% [21]
- 我们 4-agent 链至少 4 步，加上迭代循环轻松超过 20 步

### 3.6 多 Agent 退化的三个独立机制

1. **上下文压缩丢弃关键状态**
2. **推理一致性随 token 预算缩减而碎片化**
3. **Agent 间协调因缺乏共享真相而崩溃**

> "更长的上下文窗口不能修复以上任何一个问题" [20]

### 3.7 Ralph Loop 模式：一个验证过的解决方案

**核心思路**：每次迭代用**全新 LLM 实例** + 文件系统注入状态，**不依赖跨迭代的对话记忆** [20]

> "Spawn a fresh Claude instance per iteration; inject state from the filesystem; never rely on conversational memory beyond one iteration. The pattern works."

**这恰好是我们 dev team 的 spawn 模式——但缺少"文件系统注入状态"这一关键环节。**

---

## 四、我们 Dev Team 失败的根因分析

### 4.1 核心问题：多 Agent 上下文断裂 + 学术验证

我们的 4-agent 架构（dev-lead → dev-init → dev-coder → dev-qa）的失败现在有了**学术级解释**：

1. **每次 spawn 都是一次上下文重置**（上下文丢失占失败 3.33%）[19]
2. **QA 验证是不可靠的**（任务完成条件识别失败占 9.82%）[19]——这与 "dev-qa 报告通过但用户反馈问题依旧" 完全一致
3. **没有错误累积意识**（过度依赖之前错误的回答）[17]——7 轮修复都在同一错误方向
4. **回合重复占 17.14%** [19]——dev-coder 反复尝试相似的修复方案

### 4.2 对比业界：单 Agent 循环 vs 多 Agent 链

| 维度 | Claude Code / OpenHands | 我们的 dev team |
|------|------------------------|----------------|
| 架构 | 单 agent + 工具循环 | 4 agent 串行链 |
| 上下文 | 持续保持，自动压缩 | 每次 spawn 重建 |
| 验证 | 运行测试/linter 自验证 | dev-qa 独立验证（但不可靠） |
| 回退 | agent 自主 git checkout | 无回退机制 |
| 规划 | Plan Mode 四阶段 | dev-lead 直接分配任务 |
| 反馈循环 | 代码→测试→修复→测试 | 代码→QA→失败→重新分配 |
| 迭代策略 | 新对话+文件系统注入状态 | 同一 task 反复 spawn |

### 4.3 六大根因（更新版）

1. **上下文断裂**（最严重）：每次 agent spawn 都丢失前序 agent 的推理过程。学术证明这导致 39% 性能下降 [17]。Ralph Loop 模式证明"全新实例+文件系统状态注入"可以工作，**但我们缺少"状态注入"这一环**

2. **验证反馈循环缺失**：Devin 的 Autofix 闭环（CI→自动修复→CI→...直到通过）是一周合并 659 个 PR 的关键 [13]。我们的 dev-qa 是"另一个人看一眼"，不是自动化验证

3. **没有"失败记忆"**：7 轮修复同一问题，但多轮退化论文证明 LLM 会"过度依赖之前错误的回答" [17]。没有机制告诉 dev-coder"前 6 轮试了什么、为什么失败"

4. **角色分工错误**：dev-lead 既不理解代码细节（没有 browser），也不做规划。Cursor/Windsurf 都有专用 planning agent

5. **模型波动**：Block (Goose) 的方案证明规划和执行可以用不同模型 [16]，但我们没有根据任务类型选择模型

6. **God Agent 反模式**：每个 agent 配备 20+ 工具时路由准确率仅 70%，30% 错误率在多步工作流中复合放大 [22]。dev-qa 有 browser 但不真正用，就是工具配置不当的体现

---

## 五、具体改进方案

### 5.1 方案 A：引入 Plan Phase（借鉴 Claude Code Plan Mode + Cursor + Block Goose）

**改动文件**：AGENTS.md 中 dev-lead 的职责定义

**具体改动**：
```
dev-lead 的流程改为：
1. 收到任务后，先进入"Plan Mode"（只读）
2. 阅读相关代码文件（使用 read/grep 工具）
3. 输出结构化修改计划到 project-plan.md：
   - 问题分析（为什么当前代码有问题）
   - 影响范围（哪些文件需要改，附文件路径和关键代码行引用）
   - 修改步骤（具体的代码改动描述）
   - 风险点（可能影响其他功能的改动）
   - 验证步骤（怎么确认修复成功——必须是自动化验证命令）
4. 用户审批计划
5. spawn dev-coder 时，将完整计划作为 task 的一部分传入

借鉴 Block Goose：dev-lead（规划）用强推理模型，dev-coder（执行）用快速模型
借鉴 Cursor：出错时不修复进行中的 agent，而是 revert→refine plan→重新执行
```

### 5.2 方案 B：迭代上下文注入（解决"失败记忆"问题）—— 实现 Ralph Loop 模式

这是**基于学术验证的最重要改进**。

**新增机制**：在项目目录维护 `iteration-log.md`

```markdown
# 迭代日志

## Issue #7: 字体大小问题

### 尝试 1 (2026-03-28)
- 方案：修改 CSS font-size
- 结果：失败 — 改了错误的 CSS 文件
- 学习：字体大小由 JS 动态设置，不是纯 CSS

### 尝试 2 (2026-03-28)
- 方案：修改 JS 中的字体计算逻辑
- 结果：失败 — 修改了错误的组件
- 学习：字体大小在 renderer.js 第 42 行的 calculateFontSize 函数

### 尝试 3 (2026-03-29)
- 方案：修改 renderer.js calculateFontSize
- 结果：QA 通过，用户反馈仍有问题
- 学习：QA 的 browser 验证不充分，需要实际截图对比
```

**每次 spawn dev-coder 时，自动注入此日志的最新 N 条记录。**

**这就是 Ralph Loop 模式的核心**：全新实例 + 文件系统注入状态 = 不依赖对话记忆但保持迭代连续性。

### 5.3 方案 C：自动化验证替代 dev-qa（借鉴 Devin Autofix 闭环）

**核心思路**：与其让 dev-qa "看一眼"，不如让 dev-coder 自己运行验证，形成闭环。

**具体实现**：
- dev-coder 的 AGENTS.md 中增加："修改完成后，必须运行以下验证命令..."
- 对于 CSS/样式问题："修改后必须截图（browser 工具），对比修改前后截图"
- 对于功能问题："修改后必须运行相关测试，确保测试通过"
- **借鉴 Devin Autofix**：验证失败后自动修复，最多 3 轮，形成"代码→验证→修复→验证"闭环
- dev-qa 角色改为**审计者**：不是验证功能是否正常，而是检查 dev-coder 的验证过程是否充分

### 5.4 方案 D：Git 回退机制

**新增机制**：每次修改前自动 git stash/save checkpoint

```
dev-coder 的流程增加：
1. 收到任务后，先 git stash save "before-fix-issue-7"
2. 执行修改
3. 运行验证
4. 如果验证失败 3 次，自动 git stash pop 回退到修改前
5. 将回退信息写入 iteration-log.md
```

**借鉴 Cursor 的建议**：出错时 revert 变更，refine 计划后重新执行——而非修复进行中的 agent。

### 5.5 方案 E：项目上下文自动注入（借鉴 Claude Code 的 Layered Memory）

**新增配置**：在项目目录创建 `.context/` 目录

```
.context/
  PRODUCT.md          # 产品描述
  architecture.md     # 架构说明
  known-bugs.md       # 已知 bug 列表
  recent-changes.md   # 最近修改摘要（自动生成）
```

**每次 spawn 任何 dev agent 时，自动注入：**
1. PRODUCT.md（静态，每次都传）
2. known-bugs.md（动态，记录当前未解决问题）
3. 最近 3 次修改的文件路径和改动摘要

### 5.6 方案 F：会话长度控制（基于学术证据）

基于"Agent 会话质量在 90 分钟后急剧下降"的发现 [20]：

- dev-coder 单次 spawn 不超过 30 分钟
- 如果 30 分钟内未解决，写入 iteration-log.md，重新 spawn（即 Ralph Loop）
- 避免在单个 agent 会话中反复修复

### 5.7 优先级排序

| 优先级 | 方案 | 预期效果 | 学术/业界依据 | 实施难度 |
|--------|------|----------|--------------|----------|
| **P0** | B: 迭代上下文注入 | 直接解决"7轮修不好同一问题" | Ralph Loop [20]；多轮退化 39% [17] | 低 |
| **P0** | C: 自动化验证闭环 | 解决"QA假通过" | Devin Autofix [13]；完成条件识别失败 9.82% [19] | 中 |
| **P1** | A: Plan Phase | 提高首次修复成功率 | Claude Code [1]；Cursor [11]；Block Goose [16] | 中 |
| **P1** | E: 项目上下文注入 | 减少"不理解项目"导致的错误 | Claude Code 6层记忆 [2]；SWE-Compressor [15] | 低 |
| **P1** | F: 会话长度控制 | 防止长会话退化 | 90分钟退化曲线 [20] | 低 |
| **P2** | D: Git 回退 | 防止错误累积 | Cursor revert→refine [11] | 低 |

---

## 六、关于角色分工的重新思考

### 当前问题
- **dev-lead**：应该做规划，但实际只做任务分配
- **dev-init**：角色模糊，与 dev-lead 重叠
- **dev-coder**：执行代码修改，但缺少项目上下文
- **dev-qa**：有 browser 但不用，验证不可靠

### 建议调整

**方案 1：简化为 3 角色**（推荐）
- **dev-architect**（原 dev-lead + dev-init 合并）：Plan Mode 执行者，负责理解需求、阅读代码、输出修改计划。**使用强推理模型**。借鉴 Windsurf 的"后台 planning agent"模式 [12]。
- **dev-coder**：按计划执行 + 自验证闭环。**使用快速模型**。借鉴 Devin 的 Autofix 循环 [13]。
- **dev-reviewer**（原 dev-qa）：审查验证过程而非功能本身

**方案 2：保留 4 角色但重新定义**
- **dev-lead**：只做规划（Plan Mode），不直接修改代码
- **dev-init**：项目理解者——负责维护 `.context/` 目录和 `iteration-log.md`（即 Ralph Loop 的"文件系统状态管理"）
- **dev-coder**：执行修改 + 自动化验证闭环
- **dev-qa**：二次验证 + 回归测试

推荐**方案 1**。理由：
1. 业界趋势（Claude Code、OpenHands）都是单 agent + 工具
2. 多 agent 的信息传递损耗是学术验证的问题 [17][19]
3. 角色越多，spawn 链越长，成功率指数下降（20 步仅 36%）[21]
4. Windsurf 的双 agent（planning + execution）已经是业界最大胆的尝试 [12]

---

## 七、总结

我们的 dev team 迭代失败有**坚实的学术支撑**：微软研究院证明 LLM 多轮对话性能平均下降 39% [17]，MIT 证明编码 agent 代码质量在 89.8% 的长程轨迹中退化 [18]，UC Berkeley 证明多 agent 系统的回合重复占 17.14% [19]。

**根本原因**是：我们的多 agent 串行架构恰好放大了所有已知的退化机制——上下文断裂、验证反馈缺失、错误方向固化、会话超时退化。而业界成功的 coding agent（Claude Code、OpenHands、Devin）都采用单 agent 持续上下文 + 自动化验证闭环的模式。

**最有效的改进路径**：
1. **P0: 迭代上下文注入**（实现 Ralph Loop——全新实例 + 文件系统状态注入）
2. **P0: 自动化验证闭环**（借鉴 Devin Autofix，消除"假通过"）
3. **P1: Plan Phase**（借鉴 Claude Code/Cursor，提高首次修复成功率）
4. **P1: 项目上下文自动注入**（借鉴 Claude Code 6 层记忆）
5. **P1: 会话长度控制**（基于 90 分钟退化曲线，强制重新 spawn）

长期建议将 4 agent 简化为 3 角色（architect + coder + reviewer），减少 spawn 链长度和信息传递损耗。

---

## 参考资料

### 业界分析
1. [What Actually Is Claude Code's Plan Mode?](https://lucumr.pocoo.org/2025/12/17/what-is-plan-mode/) — Armin Ronacher 对 Plan Mode 的逆向分析
2. [Claude Code Architecture (Reverse Engineered)](https://vrungta.substack.com/p/claude-code-architecture-reverse) — 5 大设计支柱的深度分析
3. [Context Engineering for Coding Agents](https://martinfowler.com/articles/exploring-gen-ai/context-engineering-coding-agents.html) — Martin Fowler 对上下文工程的总结
4. [Coding Agents 101](https://devin.ai/agents101) — Devin/Cognition 的 agent 最佳实践
5. [Devin AI Complete Guide](https://www.digitalapplied.com/blog/devin-ai-autonomous-coding-complete-guide) — Devin 2.0 改进：83% 更多任务完成
6. [OpenHands CodeAct 2.1](https://localaimaster.com/blog/openhands-vs-swe-agent) — SWE-Bench 53% 解决率
7. [SWE-EVO: Benchmarking Coding Agents](https://arxiv.org/pdf/2512.18470) — 连续修复场景成功率骤降（65%→21%）
8. [Multi-model coding agents hitting 76% on SWE-bench](https://www.reddit.com/r/LocalLLaMA/comments/1onfjk6/) — 多模型协作提升成功率
9. [Implementing Claude Code Plan Mode in Your Own AI Agent](https://yag.xyz/en/post/ai-agent-plan-mode-example/) — Plan Mode 的 SDK 实现参考
10. [Building Effective AI Coding Agents for the Terminal](https://arxiv.org/html/2603.05344v3) — 终端 coding agent 的安全控制和上下文管理
11. [Cursor Agent Best Practices](https://cursor.com/blog/agent-best-practices) — Cursor 官方 agent 最佳实践
12. [Windsurf Cascade Docs](https://docs.windsurf.com/windsurf/cascade/cascade) — 双 agent 架构和 Cascade 模式
13. [How Cognition Uses Devin to Build Devin](https://cognition.ai/blog/how-cognition-uses-devin-to-build-devin) — Autofix 闭环和 659 PR/周
14. [Devin AI Engineers Production Realities](https://www.sitepoint.com/devin-ai-engineers-production-realities/) — 上下文窗口限制和架构决策丢失
15. [SWE-Compressor](https://arxiv.org/html/2512.22087v1) — 结构化上下文工作区，57.6% SWE-Bench
16. [Block Goose: Does Your AI Agent Need a Plan?](https://block.github.io/goose/blog/2025/12/19/does-your-ai-agent-need-a-plan/) — 不同模型分别做规划和执行
17. [LLMs Get Lost In Multi-Turn Conversation](https://arxiv.org/html/2505.06120v1) — 微软研究院 & Salesforce，多轮性能下降 39%
18. [SlopCodeBench](https://arxiv.org/html/2603.24755v1) — 威斯康星大学 & MIT，89.8% 轨迹代码冗余上升
19. [Why Do Multi-Agent LLM Systems Fail?](https://arxiv.org/html/2503.13657v3) — UC Berkeley，回合重复 17.14%
20. [The Forgetting Agent: Why Multi-Turn Conversations Collapse](https://blakecrosley.com/blog/agent-memory-degradation) — 90 分钟退化曲线和 Ralph Loop 模式
21. [Reddit: Why 90% of AI Agents Still Fail](https://www.reddit.com/r/AI_Agents/comments/1ovk0lx/) — 步骤数与成功率指数关系
22. [Why 90% of AI Agent Projects Fail](https://dev.to/nebulagg/why-90-of-ai-agent-projects-fail-and-the-patterns-that-fix-it-1dma) — God Agent 反模式
