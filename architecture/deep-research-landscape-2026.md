# 深度研究 Agent 项目全景调研 — 2026.03.28

> 通过 GitHub + Google 搜索系统调研，涵盖所有高星开源项目及闭源竞品。

---

## 一、项目总览

| # | 项目 | ⭐ | 出品方 | 定位 | 多 Agent | 自训练模型 |
|---|------|-----|--------|------|----------|-----------|
| 1 | **khoj-ai/khoj** | 33.7k | Khoj | 通用 AI 助手（含深度研究） | ✅ 自定义 agents | ❌ |
| 2 | **assafelovic/gpt-researcher** | 26.1k | 社区 | 专用研究 agent | ✅ 8 agent 流水线 | ❌ |
| 3 | **virattt/dexter** | 19.8k | 社区 | 金融研究 agent | ❌ 单 agent | ❌ |
| 4 | **dzhng/deep-research** | 18.6k | 社区 | 通用深度研究 | ✅ breadth×depth 递归 | ❌ |
| 5 | **Alibaba-NLP/DeepResearch** | 18.6k | 阿里通义 | 通用深度研究 | ❌ 单 agent | ✅ 30B-A3B |
| 6 | **arc53/DocsGPT** | 17.8k | 社区 | 文档问答 | ❌ | ❌ |
| 7 | **MiroMindAI/MiroThinker** | 8.5k | MiroMind | 通用深度研究 | ❌ 单 agent | ✅ 30B/235B |
| 8 | **langchain-ai/open_deep_research** | ~6k | LangChain | 通用深度研究 | ✅ LangGraph | ❌ |
| 9 | **blisspixel/deepr** | ~4k | 社区 | 多 provider 研究路由 | ✅ 持久化专家 | ❌ |
| 10 | **Salesforce EDR** | ~4k | Salesforce | 企业深度研究 | ✅ Master + 4 专用 | ❌ |

---

## 二、重点项目深度分析

### 1. khoj-ai/khoj (33.7k ⭐)
**不是纯研究 agent，是通用 AI 助手平台。** 含深度研究功能但不是核心。

- 自托管、支持本地/在线 LLM
- 自定义 agents（知识、人格、模型、工具）
- 自动化研究：个人新闻简报、智能通知
- 语义搜索（PDF、Markdown、Notion、Word）
- 多平台：Browser/Obsidian/Emacs/Desktop/Phone/WhatsApp

**对我们的启示：** agent 持久化知识 + 自动化调度的思路值得借鉴，但研究架构不深。

### 2. assafelovic/gpt-researcher (26.1k ⭐) — 已调研
8 agent 流水线：Planner → Searcher → Scraper → Reader → Writer → Reviewer → Revisor → Publisher。独立 Reviewer 消除自评偏见。Tavily 搜索 API。

### 3. virattt/dexter (19.8k ⭐)
**金融领域专用研究 agent。** "Think Claude Code, but for financial research."

- 单 agent，任务规划 + 自我反思
- 实时市场数据（收入表、资产负债表、现金流量表）
- 内置循环检测和步数限制（防失控）
- 支持 OpenAI/Anthropic/Google/xAI/OpenRouter/Ollama
- Exa 搜索 API + Financial Datasets API
- 评估套件：LLM-as-judge + LangSmith 追踪
- JSONL scratchpad 完整记录工具调用链
- WhatsApp 集成

**对我们的启示：**
- **Scratchpad 模式**：每步操作完整记录，方便调试和审计
- **循环检测**：agent 常见陷阱，必须有
- **领域专用 agent 比通用 agent 更实用**

### 4. dzhng/deep-research (18.6k ⭐) — 已调研
breadth×depth 递归搜索。先用广度搜索建立知识基础，再用深度搜索填充细节。知识在递归层级间传递。

### 5. Alibaba-NLP/DeepResearch (18.6k ⭐) — 通义深度研究
**自训练 30B-A3B 模型（仅激活 3.3B），专为信息搜索任务设计。**

- SOTA 性能：HLE、BrowseComp、BrowseComp-ZH、WebWalkerQA、SimpleQA
- 完全自动化合成数据管线：agentic pre-training + SFT + RL
- 持续预训练：保持模型新鲜度
- 端到端 RL：Group Relative Policy Optimization（GRPO）
- 两种推理模式：ReAct（评测用）+ IterResearch Heavy Mode（最大性能）
- 依赖：Serper（搜索）、Jina（网页阅读）、SandboxFusion（代码沙箱）

**对我们的启示：**
- **自训练模型是终极解法**，但门槛高（需要算力 + 数据管线）
- **IterResearch 模式**：test-time scaling，通过更多推理步骤提升质量
- **搜索工具栈**：Serper（Google 搜索）+ Jina（网页解析）的组合很成熟

### 6. MiroMindAI/MiroThinker (8.5k ⭐)
**BrowseComp 88.2 分（SOTA），开源深度研究 agent。**

- v1.7：74.0% BrowseComp, 75.3% BrowseComp-ZH, 42.9% HLE-Text
- 256K 上下文窗口，最多 300-400 次工具调用
- **Interactive Scaling**：第三个性能维度（除了模型大小和上下文长度）
  → 训练 agent 处理更深更频繁的 agent-环境交互
- 30B 和 235B 两个版本
- 在线服务：dr.miromind.ai

**对我们的启示：**
- **Interactive Scaling 是关键洞察**：不是靠更大的模型，而是靠训练模型更好地使用工具
- **300-400 次工具调用**：远超其他项目的交互深度
- **BrowseComp 基准**：目前最权威的深度研究评测

### 7. langchain-ai/open_deep_research (~6k ⭐)
**LangChain 官方的开源深度研究 agent。**

- 基于 LangGraph，支持多 provider（OpenAI、Anthropic、Gemini、Groq、Ollama）
- 多模型分工：summarization（4.1-mini）+ research（4.1）+ compression（4.1）+ final report（4.1）
- Tavily 搜索 API + MCP 兼容
- Deep Research Bench 评测：RACE 0.4344（#6 排名）
- 可视化：LangGraph Studio UI
- 完整评估管线：100 道博士级题目，LLM-as-judge

**对我们的启示：**
- **模型分工策略**：不同任务用不同能力的模型（省钱 + 效果好）
- **LangGraph Studio 可视化**：调试 multi-agent 工作流很有价值
- **Deep Research Bench**：中英双语各 50 题，22 个领域

### 8. blisspixel/deepr (~4k ⭐)
**多 provider 研究路由 + 持久化专家 agent。**

- Auto 模式：按复杂度自动路由（简单 $0.01，深度 $0.50-$2.00）
- **持久化专家**：跨会话积累知识，追踪置信度，自主发现知识缺口
- 批量处理：50 个查询一次跑
- MCP 接口：其他 agent 可调用
- 多 provider 自动故障转移
- 完整审计日志：路由决策、成本、来源信任度

**对我们的启示：**
- **持久化专家模式**：专家不只是一次性工具，而是持续学习的团队成员
- **按复杂度路由**：不是所有问题都需要深度研究，简单问题快速回答
- **成本透明**：每个决策的成本都记录

### 9. Salesforce Enterprise Deep Research (EDR) (~4k ⭐)
**企业级多 agent 深度研究系统。**

- Master Planning Agent + 4 个专用搜索 Agent（通用/学术/GitHub/LinkedIn）
- MCP 生态：NL2SQL、文件分析、企业工作流
- 可视化 Agent（数据洞察）
- Reflection 机制：检测知识缺口 + 更新研究方向
- Human-in-the-loop 实时引导
- DeepResearchBench #1 排名
- EDR-200 数据集：201 条完整研究轨迹

**对我们的启示：**
- **领域专用搜索 agent**：学术搜 Google Scholar，代码搜 GitHub，人脉搜 LinkedIn
- **Reflection + Human-in-the-loop**：发现缺口时可以请人类指导
- **完整轨迹数据集**：EDR-200 是训练/分析研究 agent 的宝贵资源

---

## 三、关键洞察总结

### 共性模式
1. **搜索 + 阅读 + 综合** 三步流水线是所有项目的共同基础
2. **迭代/递归搜索**：几乎所有项目都支持多轮搜索
3. **自我反思**：检查已收集信息是否充分
4. **结构化输出**：最终报告 + 引用

### 差异化策略
| 策略 | 代表项目 | 优势 |
|------|---------|------|
| 多 agent 流水线 | GPT-Researcher, Salesforce EDR | 并行 + 专业分工 |
| 自训练模型 | 通义 DeepResearch, MiroThinker | 更好的工具使用能力 |
| 持久化专家 | Deepr | 跨会话知识积累 |
| 按复杂度路由 | Deepr | 成本效益最优 |
| Interactive Scaling | MiroThinker | 更深的工具交互 |
| 领域专用 | Dexter（金融）, EDR（企业） | 更精准的搜索 |

### 对我们 v3 方案的验证

我们的 "探索-验证-收敛" 框架的设计方向**完全正确**：

| 我们的设计 | 行业验证 |
|-----------|---------|
| 独立 Review Agent | GPT-Researcher 独立 Reviewer ✅ |
| Gaps queue 缺口追踪 | Jina DeepResearch gaps queue ✅ |
| 多视角探索 | STORM 视角驱动 ✅ |
| 结构化 JSON 跨会话记忆 | Anthropic Harness ✅ |
| 上下文重置 > 压缩 | Anthropic Context Engineering ✅ |
| 多模型分工 | LangChain open_deep_research ✅ |
| 按复杂度路由 | Deepr ✅ |

### 尚未覆盖但值得加入的

1. **Interactive Scaling（MiroThinker）**：训练 agent 处理更多工具调用 → 我们通过 prompt engineering 模拟
2. **领域专用搜索 Agent（EDR）**：学术/GitHub/LinkedIn 分工 → 研究团队可配置
3. **持久化专家（Deepr）**：跨研究任务积累知识 → 后续版本考虑
4. **Scratchpad 审计（Dexter）**：完整记录每步操作 → 加入知识库 JSON
5. **循环检测（Dexter）**：防 agent 陷入死循环 → 必须实现

---

## 四、评测基准参考

| 基准 | 测试内容 | 语言 | 项目参考 |
|------|---------|------|---------|
| **BrowseComp** | 开放式网页搜索研究 | 英/中 | MiroThinker, 通义 |
| **HLE (Humanity's Last Exam)** | 最难人类考试 | 英 | MiroThinker, 通义 |
| **Deep Research Bench** | 100 道博士级研究题 | 英/中 | LangChain, EDR |
| **GAIA** | 通用 AI 助手评估 | 英 | MiroThinker |
| **Frames** | 事实核查 | 英 | 通义 |
| **LiveResearchBench** | 实时研究 | 英 | EDR (#1) |
| **EDR-200** | 201 条完整研究轨迹 | 英 | EDR |

---

## 五、推荐行动

1. **立即可做**：用 OpenClaw 内置 browser + web_fetch 实现搜索能力，不需要额外 API key
2. **Prompt 参考**：LangChain open_deep_research 的模型分工策略 + GPT-Researcher 的 Reviewer prompt
3. **评测参考**：Deep Research Bench（免费、双语、有 LLM-as-judge 评估脚本）
4. **长期方向**：Interactive Scaling + 持久化专家 + 领域专用搜索 agent
