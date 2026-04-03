# R-024: AI Agent 团队中"研究团队做产品设计"的最佳实践

## 弥合设计与实现的差距

> 研究日期：2026-03-31 | 方法：多源并行调研（业界实践 + AI Agent 架构 + 可行性验证）

---

## 一、业界最佳实践总结

### 1.1 Google Design Doc 文化

**核心机制**：在编码前写设计文档，强制评审，关注 trade-offs。

**可行性保障要素**：
- **Goals / Non-goals**：明确边界，防止 scope creep，Non-goals 是"可以合理成为目标但明确排除的事项"
- **Alternatives Considered**：必须列出替代方案及其 trade-offs，证明所选方案最优
- **原型链接**：要求"链接到原型以展示设计的可实施性"（link to prototypes that show the implementability）
- **跨切面审查**：安全、隐私、可观测性必须有专项审查
- **规模控制**：大项目 10-20 页，增量改进 1-3 页 mini design doc

**关键启示**：Google 不要求研究团队有 exec 权限，而是要求**设计文档提供可实施性证据**（原型链接、替代方案分析）。可行性验证的责任在设计者，验证手段是评审而非执行。

### 1.2 Amazon Working Backwards (PRFAQ)

**核心机制**：从客户视角倒写新闻稿和 FAQ，在投入开发前验证产品价值。

**可行性保障要素**：
- 先写对外的新闻稿（press release），如果无法清晰描述客户价值则重新思考
- FAQ 部分必须回答技术可行性问题
- 6-pager 叙事文档要求定量分析支撑

**关键启示**：Amazon 的方法强制"先想清楚再动手"，但可行性验证发生在**评审会议**中（silently read + discuss），而非靠撰写者自己执行。

### 1.3 Stripe RFC 机制

**核心机制**：RFC（Request for Comments）流程，设计提案公开评审。

**可行性保障要素**：
- RFC 有标准模板和生命周期（draft → review → approved/rejected）
- 任何工程师都可以评审和提出质疑
- 强调在投入大量工程时间前获得共识

### 1.4 Stage-Gate 流程（工业界通用）

**核心机制**：每个阶段结束有 Gate Review，gatekeeper 用严格标准筛除弱势项目。

**关键原则**：
- 评审标准必须"清晰、严格、简单"，不留模糊空间
- 技术可行性必须在 Gate Review 中被明确验证
- 跨职能评审团队

---

## 二、AI Agent 多 Agent 协作的关键模式

### 2.1 Anthropic 的多 Agent 研究系统

**架构**：Lead Agent（Opus 4）规划 + 子 Agent（Sonnet 4）并行执行。

**关键设计**：
- 子 Agent 各自有独立上下文窗口和工具
- 核心价值是**上下文压缩**：子 Agent 并行处理后精炼关键信息给主 Agent
- 多 Agent 系统比单 Agent 性能高 90.2%

**与我们的对应**：我们的 research lead → sub search agents 架构与此高度一致。但 Anthropic 的研究系统**不涉及实现**——研究和实现是不同系统。

### 2.2 Cursor 的 Plan Mode

**架构**：Agent 有专门的 Plan Mode（Shift+Tab），先研究代码库、提出澄清问题、创建实施计划。

**关键设计**：
- Plan Mode 下 Agent **不写代码**，只做研究和规划
- 计划保存为 Markdown 文件在 `.cursor/plans/` 目录
- 计划包含文件路径和代码引用，等待用户批准后才执行
- 保存的计划支持团队共享和后续 Agent 上下文延续

**关键启示**：Cursor 将"研究/规划"和"实现"显式分离为两个模式，但它们**共享同一代码库上下文**，计划文件是衔接点。这正是我们需要学习的。

### 2.3 CrewAI Planner-Executor 模式

**架构**：Planner Crew 和 Executor Crew 是两个独立的多 Agent 系统。

**关键设计**：
- 规划和执行分离为独立 Crew
- 通过 Flows（事件驱动）衔接
- 每个 step 只有一次 LLM 调用，控制成本

### 2.4 LangGraph Plan-and-Execute 模式

**架构**：规划 Agent 先制定计划，执行 Agent 按计划逐步执行，可重新规划。

**关键设计**：
- 比 ReAct 等 Agent 设计更快、更便宜
- 强调控制（control）和持久性（durability）而非高级抽象
- Agent 开发三大挑战：高延迟管理、长任务重试、非确定性需检查点和审批

### 2.5 Andrew Ng 的多 Agent 协作观点

- 多 Agent 是四大 AI Agent 设计模式之一
- 核心价值：提供分解复杂任务的框架
- 即使底层调用同一 LLM，角色分解也能优化子任务
- 典型角色：软件工程师、产品经理、设计师、QA 工程师

---

## 三、设计文档可实施性验证最佳实践

### 3.1 验证时机

**业界共识**：技术可行性验证应在设计早期（低保真阶段）进行，而非等到设计完善后。

> "It's best to do it in the early stages of your design solution when all you have is the user journey or a low-fidelity wireframe. Avoid investing time in polishing an interface without having validated it previously." — DevChecklists

### 3.2 验证责任人

**Tech Lead 主导**：
> "In most teams, the tech lead knows more about feasibility constraints than the project manager. However, you might need input from more than one person before reaching the final solution." — DevChecklists

### 3.3 技术可行性验证清单

- 功能复杂度评估：涉及哪些产品组件？技术限制是什么？
- 组件验证：需要创建新组件还是复用已有组件？
- 性能影响分析
- 代码重构需求评估

### 3.4 PoC 最佳实践

- 明确目标与范围（验证特定技术，非构建完整产品）
- 现实的时间线和资源分配
- 跨职能协作
- 迭代改进

### 3.5 可行性研究的核心问题

1. 团队是否有工具/资源完成项目？
2. 投资回报是否足够高？

---

## 四、对四个初步方案的评估

### 方案 A：研究团队加 exec 权限

| 维度 | 评估 |
|------|------|
| 优点 | 研究团队可自行验证 API、依赖可用性 |
| 缺点 | 破坏职责分离原则；增加安全风险；与 Anthropic/Cursor 等业界实践相悖（它们都保持规划与执行的分离） |
| 业界参考 | Cursor Plan Mode 明确在规划模式下**不执行代码**；Google 靠评审而非执行来验证 |
| 结论 | ❌ **不推荐**。引入 exec 权限会让研究团队变为"迷你 dev team"，失去职责清晰的架构优势 |

### 方案 B：设计文档强制标注"已验证/未验证"

| 维度 | 评估 |
|------|------|
| 优点 | 低成本、低风险；显式标注让不确定性可见 |
| 缺点 | 仅标注不解决问题，需要配套的验证流程 |
| 业界参考 | Google 要求"链接到原型以展示可实施性"——比标注更强 |
| 结论 | ✅ **推荐作为基础措施**，但不够充分。应扩展为"验证状态 + 验证方式 + 风险等级"的三维标注 |

### 方案 C：双审机制（研究出设计 → dev-lead 做可实施性评审）

| 维度 | 评估 |
|------|------|
| 优点 | 符合业界 Gate Review / Stage-Gate 模式；保持职责分离；利用 dev-lead 的实现经验 |
| 缺点 | 增加 dev-lead 工作量；需要明确的评审标准和 SLA |
| 业界参考 | Google 的 cross-cutting review、Stage-Gate 的 gatekeeper 机制、Cursor 的 Plan Mode 审批流程 |
| 结论 | ✅ **强烈推荐作为核心机制**。这是最符合业界实践且对现有架构改动最小的方案 |

### 方案 D：合并为"产品团队"

| 维度 | 评估 |
|------|------|
| 优点 | 消除交接问题 |
| 缺点 | 丧失专业分工优势；研究深度可能下降；Agent 角色模糊；与 Andrew Ng 强调的"角色分解优化子任务"相悖 |
| 业界参考 | CrewAI 用 Planner Crew + Executor Crew 保持分离但协作 |
| 结论 | ❌ **不推荐**。业界趋势是更好的衔接机制，而非合并角色 |

---

## 五、推荐方案：方案 B+C 的增强版——"标注验证 + Gate Review + 共享上下文"

### 5.1 核心机制

```
Research Agent 写设计文档
    ↓ (文档含验证状态标注)
Dev-Lead Agent 做 Gate Review (可实施性评审)
    ↓ (通过 → 进入实施 / 不通过 → 标注问题退回)
Dev Team 实施
```

### 5.2 设计文档增强字段

每个 finding/设计决策必须包含：

```markdown
## 验证状态矩阵

| 设计决策 | 验证状态 | 验证方式 | 风险等级 | 备注 |
|---------|---------|---------|---------|------|
| 使用 XXX API | ⬜ 未验证 | 需 exec 验证 | 🔴 高 | 未确认 API 是否可用 |
| YYY 库实现 ZZ | ✅ 已验证 | 文档确认 | 🟢 低 | 官方文档确认支持 |
| WWW 方案 | ⚠️ 部分验证 | 搜索结果推断 | 🟡 中 | 基于类似案例推断 |
```

### 5.3 Gate Review 清单

Dev-Lead Agent 评审时检查：

1. **API/依赖验证**：所有外部依赖是否已确认可用？
2. **复杂度评估**：实现是否在合理范围内？
3. **组件复用**：是否最大化复用已有组件？
4. **性能影响**：是否有性能风险？
5. **未验证项风险**：未验证项是否已标注并有替代方案？
6. **实施顺序**：是否提供了合理的实施步骤？

### 5.4 共享上下文机制（学习 Cursor）

- 研究报告和设计方案写入**共享目录**（已有：`shared/results/`）
- 设计文档采用统一 Markdown 格式，包含文件路径引用
- Dev-Lead 可以直接读取研究报告作为上下文
- **设计文档就是两个团队之间的接口协议**

### 5.5 PoC 机制

对于高风险设计决策：
- Research Agent 在报告中标注"建议 PoC 验证"
- Dev-Lead 在 Gate Review 中判断是否需要 PoC
- PoC 由 Dev Team 执行（他们有 exec 权限），结果回填到设计文档

---

## 六、AGENTS.md 改动建议

### 研究团队 (research) 的 AGENTS.md 增加：

```markdown
## 设计文档输出规范

### 必须包含的验证标注
每个技术方案/设计决策必须标注：
- **验证状态**：✅ 已验证 | ⚠️ 部分验证 | ⬜ 未验证
- **验证方式**：exec 验证 / 文档确认 / 搜索推断 / 专家判断
- **风险等级**：🔴 高 | 🟡 中 | 🟢 低

### 必须包含的章节
1. Goals / Non-Goals（学习 Google）
2. Alternatives Considered（学习 Google）
3. 验证状态矩阵（上述表格）
4. 实施建议顺序
5. 建议 PoC 项（如有）

### 红线补充
- ❌ 不要写无法验证的 API 端点或依赖——如果未验证，必须标注为"⬜ 未验证"
- ❌ 不要假设某个库/API 可用——要么从文档确认，要么标注为未验证
```

### 新增 Dev-Lead Agent（或在现有 dev team 配置中增加）：

```markdown
## Gate Review 职责

### 触发条件
当 shared/results/ 中出现新的设计文档时，执行可实施性评审。

### 评审清单
1. 所有外部依赖/API 是否已验证或标注风险？
2. 实现复杂度是否合理？
3. 是否有组件可复用？
4. 是否需要 PoC 验证？
5. 实施顺序是否可行？

### 输出
在原设计文档末尾追加 Gate Review 结果：
```markdown
---
## Gate Review (by dev-lead, {date})

- **结果**：✅ 通过 / ⚠️ 有条件通过 / ❌ 需修订
- **评审详情**：...
- **需要 PoC 的项**：...
- **实施注意事项**：...
```
```

---

## 七、实施路径（最小改动）

### Phase 1（立即可做，0 代码改动）
1. ✅ 在研究团队的 AGENTS.md 中加入验证标注规范
2. ✅ 研究报告模板增加"验证状态矩阵"章节
3. ✅ 利用现有 dev team 的 agent 做 Gate Review（作为额外 spawn 任务）

### Phase 2（需要配置改动）
1. 设置 dev-lead agent（或复用现有 dev agent），配置 Gate Review 任务
2. 在共享目录中建立标准化的设计文档模板
3. 建立 PoC 请求 → Dev Team 执行 → 结果回填的工作流

### Phase 3（可选优化）
1. 自动检测 shared/results/ 中的新文档，触发 Gate Review
2. 验证状态的自动更新（PoC 结果回填后自动标记为已验证）

---

## 八、知识缺口

1. **Amazon PRFAQ 的具体技术可行性验证机制**未深入获取（搜索受限）
2. **Meta/字节跳动的内部技术评审流程**公开资料不足
3. **AI Agent 团队中 Planner-Executor 之间上下文传递的具体协议格式**业界尚无标准
4. **Windsurf 的 Agent 模式架构**公开信息有限
5. "已验证/未验证"标注在 AI Agent 系统中的实际效果尚无实证研究

---

## 九、来源列表

| # | 来源 | URL | 类型 |
|---|------|-----|------|
| 1 | Industrial Empathy - Google Design Docs | https://www.industrialempathy.com/posts/design-docs-at-google/ | 博客（前 Google 工程师） |
| 2 | Anthropic Engineering - Multi-Agent Research System | https://www.anthropic.com/engineering/multi-agent-research-system | 官方工程博客 |
| 3 | Cursor Blog - Agent Best Practices | https://cursor.com/blog/agent-best-practices | 官方博客 |
| 4 | LangChain Blog - Planning Agents | https://blog.langchain.com/planning-agents/ | 官方博客 |
| 5 | LangChain Blog - Building LangGraph | https://blog.langchain.com/building-langgraph/ | 官方博客 |
| 6 | Andrew Ng - Multi-Agent Collaboration | https://www.deeplearning.ai/the-batch/agentic-design-patterns-part-5-multi-agent-collaboration/ | 教育平台 |
| 7 | Comet - Multi-Agent Systems | https://www.comet.com/site/blog/multi-agent-systems/ | 技术博客 |
| 8 | Asana - Feasibility Study | https://asana.com/resources/feasibility-study | 项目管理平台 |
| 9 | DevChecklists - Design Technical Viability | https://devchecklists.com/en/checklist/design-technical-viability | 清单平台 |
| 10 | US Dept of Energy - Stage-Gate Review | https://www.energy.gov/cmei/articles/stage-gate-review-guide-industrial-technologies-program | 政府指南 |
| 11 | SevenCollab - PoC Best Practices | https://sevencollab.com/7-best-practices-for-proof-of-concept-poc-in-software-development/ | 技术博客 |
| 12 | Rina Artstain - Effective Design Documents | https://rinaarts.com/how-to-write-an-effective-design-document/ | 个人博客 |
| 13 | CrewAI GitHub | https://github.com/crewaiinc/crewAI | 开源项目 |
| 14 | IBM Think - BabyAGI | https://www.ibm.com/think/topics/babyagi | 技术百科 |

---

## 十、方法论反思

**做得好的**：
- 多视角并行调研（业界 + AI Agent + 可行性验证三个维度）
- 与具体架构问题紧密结合
- 每个建议都有业界案例支撑

**需改进的**：
- 大厂内部流程（Amazon、Meta、字节）公开资料有限，部分依赖二手信息
- AI Agent 多 Agent 协作领域的最佳实践仍在快速演进，结论可能需要定期更新
- 未能在本次调研中实际验证推荐方案的效果
