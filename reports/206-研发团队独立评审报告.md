# R-015b Meta-Review：独立评审报告

> 评审时间：2026-03-30 | 评审者：Research Lead (独立 subagent) | 评审对象：R-015b-dev-team-quality-first-v2.md

---

## 逐维度评审

### 1. 对 Anthroid 文章的理解深度 — 9/10

**优点：**
- 直接阅读了 agent.py、progress.py、security.py、client.py 源码，不是二手转述。关键代码段（main loop、count_passing_tests、allowlist 15 命令）都有引用。
- 准确区分了"机制"（fresh client per session、3s delay、feature_list.json existence check）和"精神"（Clean State = 可合并到 main 的代码）。
- 最有价值的发现：确认 Anthropic 是 TWO-agent 而非 three-agent，blog 中提到的 "testing agent, QA agent" 是展望而非实现。这直接影响了设计决策。

**不足：**
- 未引用 coding_prompt.md 的具体段落（虽然提到了 "MANDATORY: test the feature end-to-end"）。编码 prompt 中关于 "END SESSION CLEANLY" 的具体指令可以更完整地引用。
- 对 Anthropic 为什么选择 self-verify（而非独立 QA）的原因分析不足——是技术限制还是刻意设计？这影响我们是否应该偏离。

### 2. 质量保障体系设计 — 8/10

**优点：**
- 三层 Gate（Feature/Regression/Clean State）结构清晰，每个 gate 有触发条件、执行者、通过标准、失败动作。这是 R-015 叙述式铁律的重大改进。
- Regression 分级策略（核心 1-2 / 3 功能全量 / 按需全量）解决了 R-015 的"regression 太慢"问题，且与 Anthropic coding_prompt 原文一致。

**不足：**
- Gate 2 的 "关键 steps 的子集" 定义模糊——由谁决定哪些是"关键 steps"？应要求 dev-init 在 feature_list 中标注 `critical_steps`。
- Gate 3 的 "spawn dev-coder 修复" 与 dev-lead 无 exec 权限矛盾——dev-lead 只能 spawn agent，不能直接执行 git 操作来检查 clean state。但 dev-lead 有 `read` 权限可以读 git status 吗？需要确认。
- 缺少超时机制：如果 QA session 挂起或无限循环怎么办？

### 3. Agent 角色设计 — 7/10

**优点：**
- 测试者/实现者分离彻底：dev-coder 不能改 passes，dev-qa 不能写代码，dev-lead 不写代码。三权分立清晰。
- 权限矩阵中 dev-qa 是唯一能改 passes 的角色，消除了 R-015 中 ACP prompt 允许 coder 改 passes 的矛盾。

**逻辑矛盾：**
1. **dev-lead 运行 init.sh**：权限矩阵显示 dev-lead ❌ 运行 init.sh，但 Gate 3 要求 dev-lead 检查 "init.sh 冒烟测试通过"。如果 dev-lead 无 exec 权限（openclaw.json 确认 deny exec），它无法运行 init.sh。它需要 spawn dev-qa 来跑冒烟测试，但这与 Gate 3 "由你自己执行" 矛盾。
2. **dev-qa git commit**：权限矩阵显示 dev-qa ✅ git commit (test)，但 openclaw.json 中 dev-qa 有 `write` 权限（可以写文件）和 `exec` 权限（可以执行 git）。这没问题，但 AGENTS.md 中未明确写 dev-qa 可以 git commit，容易遗漏。
3. **dev-coder 的 process 权限**：openclaw.json 给 dev-coder `process` 工具，但 AGENTS.md 中未说明 process 用于什么场景。如果是为了启动 dev server，应该在 AGENTS.md 中明确。
4. **dev-lead 的 subagents 权限**：openclaw.json 中 dev-lead 的 tools 包含 `sessions_spawn` 和 `subagents`，但权限矩阵中 "spawn agent" 操作未列出 dev-lead ✅——这是隐含的，但应该显式标注。

### 4. AGENTS.md 质量 — 8/10

**优点：**
- 每个 agent 的 AGENTS.md 都有清晰的"绝对不做什么"列表，强措辞充分（"IT IS CATASTROPHIC"）。
- Session 启动流程（pwd → progress.md → git log → feature_list.json → init.sh）完整且有序。
- dev-qa 的 Gate 1/2 流程步骤清晰。

**不足：**
- dev-lead AGENTS.md 中 "不直接执行任何命令" 但 Gate 3 需要检查 git status clean——这需要 exec 或 read+特定路径。当前 dev-lead 有 `read` 但没有 `exec`，无法运行 `git status`。
- dev-init 的 AGENTS.md 缺少 "init.sh 必须包含冒烟测试" 的明确要求（虽然模板中有，但 AGENTS.md 指令中未强制）。
- 缺少统一的错误处理格式：当 agent 遇到无法处理的问题时，应该输出什么格式的错误报告？

### 5. 模板和配置 — 8/10

**优点：**
- feature_list.json 模板 BDD 风格，steps 用业务语言、可验证、有 Given/When/Then 指南。
- init.sh 包含完整的冒烟测试（4 步：环境检查、lint、dev server、smoke test）。
- progress.md 模板结构化，含 Gate 结果、Milestone、统计。
- openclaw.json 配置完整，4 个 agent 的 tools allow/deny 清晰。

**不足：**
- feature_list.json 的 `qa_status` 字段有 4 种值（pending/needs-fix/regression-fail/verified），但 JSON schema 中没有定义为 enum，agent 可能使用不一致的值。
- init.sh 的 dev server 启动逻辑假设端口 3000，但不同项目可能不同。应该支持从 config 或 feature_list 中读取端口。
- ACP task prompt 模板中 `$featureId` 和 `$description` 是模板变量，但没有说明 dev-lead 如何填充这些变量。

### 6. 可行性 — 6/10

**关键风险：**

1. **browser 工具能力**：R-015b 自身承认未解决。dev-qa 的 Gate 1 核心依赖 browser e2e 测试（"像人类用户一样操作"），但 OpenClaw 的 browser 工具是否支持完整的点击、输入、导航、验证？如果 browser 不可靠，整个 QA 流程崩溃。这是**致命风险**。

2. **Command allowlist 无法硬性执行**：方案正确指出了 OpenClaw 与 Anthropic SDK 的架构差异——OpenClaw 的工具权限是 tool-level（允许/禁止 exec），不是 command-level。AGENTS.md 中的安全约束是"agent 自行遵守"的软约束。一个偏离指令的 agent 可以执行任何命令。这是**重大风险**。

3. **QA agent 模型能力**：建议用 glm-4.7 做 QA，但 QA 需要理解业务逻辑、识别 UI bug、执行 browser e2e。glm-4.7 的 instruction-following 和 browser 操作能力未经验证。

4. **dev-lead 无 exec 权限**：dev-lead 不能执行任何命令，只能 spawn subagent。这意味着 Gate 3 的 "git status clean" 检查无法直接执行——需要 spawn dev-qa 来检查，但这增加了复杂性和延迟。

5. **并发安全**：多个 agent（coder + QA）共享 feature_list.json，虽然 dev-lead 串行 spawn，但如果 ACP harness session 和 dev-qa 重叠（coder 还在运行时 QA 被触发），可能产生竞态条件。

**非致命但需注意：**
- sessions_spawn 的 agentId 对应 openclaw.json 中的 agents.list[].id，这个映射关系在方案中是正确的。
- ACP task prompt 的模板变量替换需要 dev-lead 的 AGENTS.md 中有明确的模板逻辑。

### 7. 部署路径 — 8/10

**优点：**
- 4 Phase 部署（环境准备 → 核心循环 → Regression → ACP 集成）清晰合理。
- 每个阶段有具体测试用例编号和验收标准。
- Phase 1 用 5 功能项目做 end-to-end 验证，规模适中。

**不足：**
- Phase 0 的 "确认 browser 工具可用" 只说 "在主 agent 中发送截个屏"，这不是系统性验证。应该有具体的 browser 命令测试清单（navigate、click、type、screenshot、verify text）。
- 缺少回滚方案：如果某个 Phase 失败，如何回退？
- 未考虑成本：每个功能至少 2 个 session（1 coder + 1 QA），20 功能项目就是 40+ session。模型调用成本未评估。

### 8. 创新性 — 8/10

**相比 Anthropic 的实质性改进：**
1. **独立 QA Agent**：Anthropic 只有 self-verify，R-015b 实现了 Anthropic blog 中想做但没做的独立验证。有行业佐证（OpenAI precision-first、Red/Green Team 分离）。
2. **三层 Gate**：Anthropic coding_prompt 中散布的质量要求被结构化为三个独立 gate。
3. **Regression 分级**：比 Anthropic 的 "1-2 个核心" 更系统化。

**不足：**
- Mutation Testing 和 Agent-Separated TDD 提到了但未集成到方案中，只是"未来可做"。
- 没有探索 Anthropic 之外的创新方向（如基于 LLM 的 property-based testing、visual regression testing）。

---

## 逻辑矛盾汇总

| # | 矛盾 | 严重程度 | 修复建议 |
|---|------|---------|---------|
| 1 | dev-lead 无 exec 权限但 Gate 3 要求运行 init.sh 冒烟测试 | **高** | 改为 dev-lead spawn dev-qa 执行 Gate 3 的 init.sh 部分，或给 dev-lead 添加有限 exec |
| 2 | dev-coder 有 process 工具但 AGENTS.md 未说明用途 | 低 | 在 dev-coder AGENTS.md 中明确 process 用于启动/管理 dev server |
| 3 | feature_list.json 的 qa_status 无 enum 定义 | 低 | 在 schema 中定义 enum 或在 AGENTS.md 中列出所有合法值 |
| 4 | 权限矩阵未列出 "spawn agent" 操作 | 低 | 添加此行，明确 dev-lead ✅、其他 ❌ |

---

## Anthroid 文章中提到但 R-015b 未体现的

| # | 漏洞 | 来源 | 重要程度 |
|---|------|------|---------|
| 1 | **max_turns=1000 的会话长度限制** | client.py | 中 — 应在 openclaw.json 或 ACP 配置中设置类似限制 |
| 2 | **3 秒 session 间延迟** | agent.py AUTO_CONTINUE_DELAY_SECONDS | 低 — 人类参与时可能不需要 |
| 3 | **每次 session 创建 fresh client** | agent.py | 低 — OpenClaw subagent 每次也是新 context |
| 4 | **Initializer 要求 200+ features、25+ 有 10 steps** | initializer_prompt.md | 中 — 方案只说"参考"但未给出具体项目的 feature 数量指导 |
| 5 | **8 个 Puppeteer 工具的具体能力** | client.py | 高 — 方案说 "browser 工具" 但未分析 OpenClaw browser 与 Puppeteer 的能力差异 |
| 6 | **Progress 作为 passes=true 的百分比** | progress.py | 低 — 方案的 progress.md 更丰富，但可增加百分比统计 |

---

## 与 R-014 对比

| 维度 | R-014 | R-015b | 判定 |
|------|-------|--------|------|
| Anthropic 理解深度 | 间接引用，无源码 | 完整源码分析 | **R-015b 明显更好** |
| 质量体系 | 无结构化门禁 | 三层 Gate + 分级 regression | **R-015b 明显更好** |
| 测试者/实现者分离 | 无（coder 自测） | 独立 QA agent | **R-015b 明显更好** |
| 安全约束 | 基础红线 | security.py allowlist | **R-015b 更好** |
| 简洁性 | 4 agent（dev-harness 是 ACP session） | 4 agent（dev-qa 替代 dev-harness） | **R-014 更简洁**（dev-harness 不是注册 agent，减少管理负担） |
| 可行性 | 不依赖 browser | 严重依赖 browser | **R-014 风险更低** |
| AGENTS.md 完整度 | 有完整 AGENTS.md | 有完整 AGENTS.md + 行业佐证 | **R-015b 更好** |
| 部署路径 | 类似 | 更详细（具体测试用例编号） | **R-015b 稍好** |

**总结**：R-015b 在几乎所有维度优于 R-014，但引入了对 browser 工具的强依赖，这是一个 R-014 中不存在的风险。

---

## 总体评分

| 维度 | 权重 | 得分 | 加权 |
|------|------|------|------|
| 1. Anthroid 理解深度 | 10% | 9 | 0.90 |
| 2. 质量保障体系 | 15% | 8 | 1.20 |
| 3. Agent 角色设计 | 15% | 7 | 1.05 |
| 4. AGENTS.md 质量 | 15% | 8 | 1.20 |
| 5. 模板和配置 | 10% | 8 | 0.80 |
| 6. 可行性 | 20% | 6 | 1.20 |
| 7. 部署路径 | 10% | 8 | 0.80 |
| 8. 创新性 | 5% | 8 | 0.40 |
| **总计** | **100%** | | **7.55** |

---

## 改进建议（按优先级）

1. **【P0-致命】验证 browser 工具能力**：在 Phase 0 中系统测试 browser 工具的 navigate/click/type/screenshot/verify 能力。如果不可靠，需要降级方案（如改为 curl + DOM 解析，或人工 QA checkpoint）。

2. **【P0-致命】解决 dev-lead Gate 3 执行矛盾**：方案 A：给 dev-lead 添加 exec 权限（仅 git 和 init.sh）；方案 B：Gate 3 的 init.sh 部分由 dev-qa 执行，dev-lead 只检查 feature_list 状态和 progress.md。

3. **【P1-重要】定义 qa_status enum**：在 AGENTS.md 和 feature_list.json schema 中明确 qa_status 的合法值（pending / needs-fix / regression-fail / verified），防止 agent 使用不一致的值。

4. **【P1-重要】增加 QA session 超时机制**：如果 dev-qa session 超过 N 分钟未返回，dev-lead 应有超时处理逻辑。

5. **【P1-重要】补充 dev-coder process 工具说明**：明确 process 工具用于启动/停止 dev server。

6. **【P2-建议】增加 feature_list.json 中的 critical_steps 标注**：让 dev-init 标注每个 feature 的关键 steps，供 Gate 2 regression 使用。

7. **【P2-建议】评估成本**：估算不同规模项目的 session 数和模型调用成本。

8. **【P2-建议】init.sh 端口配置化**：支持从 config 中读取 dev server 端口。

---

## 结论

**需要修改后进入创建阶段。**

R-015b 是一份高质量的设计文档（7.55/10），在 Anthropic 源码分析深度、质量体系结构化、测试者/实现者分离等方面明显优于 R-014。但存在两个需要创建前解决的致命问题：

1. **browser 工具能力未验证**——这是 QA agent 的核心依赖，如果不可用，整个独立 QA 设计需要降级。
2. **dev-lead Gate 3 执行矛盾**——dev-lead 无 exec 权限但需要运行 init.sh，这是架构层面的逻辑矛盾。

建议：先用 1-2 天执行 Phase 0 的 browser 工具验证。如果 browser 可用，修复 Gate 3 矛盾后即可进入创建阶段。如果 browser 不可用，需要设计降级方案（如改为基于 exec + curl 的非 browser QA，或引入人工 QA checkpoint），这可能需要更大范围的修改。
