# R-008: 优化 OpenClaw Main Agent 定位 — 纯沟通调度 + 轻量兜底方案

> 研究日期：2026-03-29 | Lead Researcher 综合报告

---

## 一、现状诊断

### 1.1 Main Agent MD 文件审查

| 文件 | 当前状态 | 问题 |
|------|---------|------|
| **AGENTS.md** | 工作空间使用指南（session startup、memory、heartbeat、formatting） | ❌ 无 agent 编排规则、无分派纪律、无职责边界 |
| **SOUL.md** | 鼓励 "resourceful before asking"、"try to figure it out" | ❌ **反向激励**：推动 main 自己做而非分派 |
| **TOOLS.md** | 环境特定笔记（ASR、venv、sudo） | ⚠️ 无工具使用纪律（何时能用 web_search、何时必须分派） |
| **IDENTITY.md** | 身份定义（小朱桑） | ✅ 无问题 |
| **USER.md** | 用户偏好（主动深入、多次反思） | ⚠️ "超预期交付"可能激励 main 亲力亲为 |
| **HEARTBEAT.md** | 空模板 | ✅ 无问题 |

**核心矛盾**：SOUL.md 的 "Be resourceful before asking" + AGENTS.md 缺乏分派规则 = main agent 倾向于自己动手。

### 1.2 OpenClaw 多 agent 架构支持

- 每个 agent 拥有独立 workspace（SOUL.md/AGENTS.md/USER.md）、agentDir（auth profiles、model registry）、session store
- 通过 bindings 将入站消息路由到指定 agent
- Credentials 不自动共享，agentDir 不可复用
- 支持 `openclaw agents add` 动态添加 agent

---

## 二、行业最佳实践

### 2.1 编排模式选择

根据 Microsoft、AWS、TrueFoundry 等多源验证：

**推荐模式：Orchestrator-Worker（层级路由）**

```
                    ┌─────────────┐
     用户消息 ────▶ │ Main Agent  │ ← 纯调度 + 轻量兜底
     Heartbeat ───▶ │ (Router)    │
     系统事件 ────▶ │             │
                    └──────┬──────┘
                           │
              ┌────────────┼────────────┐
              ▼            ▼            ▼
        ┌──────────┐ ┌──────────┐ ┌──────────┐
        │ Research │ │   Dev    │ │   Ops    │
        │  Agent   │ │  Agent   │ │  Agent   │
        │ (lead→   │ │ (待建)   │ │ (待建)   │
        │ search→  │ │          │ │          │
        │ review→  │ │          │ │          │
        │ citation)│ │          │ │          │
        └──────────┘ └──────────┘ └──────────┘
```

**关键设计原则**：
- Worker 不互相通信，所有协调经 orchestrator（hub-and-spoke）
- Orchestrator 只需感知完整工作流，Worker 是无状态专家
- 这是生产环境最广泛部署的模式（Microsoft、AWS、TrueFoundry 均确认）

**⚠️ Orchestrator 的已知弱点**（需在设计中有意规避）：
- 单点故障 → main agent 应保持轻量，不做重计算
- 上下文窗口瓶颈 → main 不积累 worker 的详细输出，只收摘要
- 吞吐瓶颈 → 并行 spawn worker，不串行等待

### 2.2 任务复杂度分级

基于 Applied AI 的 4 层模型和行业共识：

| 层级 | 类型 | 特征 | 处理者 |
|------|------|------|--------|
| **Tier 0** | 纯规则 | 关键词匹配、格式化、路由 | Main 直接处理 |
| **Tier 1** | 单次 LLM | 简单问答、分类、提取、总结 | Main 直接处理 |
| **Tier 2** | LLM + 工具 | 需要搜索、文件操作、API 调用 | **判断边界**：1-2 次工具调用→Main，3+次→分派 |
| **Tier 3** | 多步推理/多 agent | 调研、开发、运维、多源综合 | **必须分派** |

**可靠性参考**：每步工具调用准确率 ~85-90%，4-5 步后累积可靠率降至 ~44-66%。

### 2.3 Sub-agent 的核心价值

- **上下文压缩**：通过分派给独立 sub-agent，可减少 parent agent 上下文 token 消耗 90%+（Epsilla/Inngest）
- **避免 context drift**：单 agent 处理所有任务时，早期决策被埋在信息堆下（"lost-in-the-middle"）
- **关注分离**：Orchestrator 只做路由和整合，Worker 是领域专家

---

## 三、具体方案设计

### 3.1 Main Agent 职责定义

#### ✅ Main Agent 应该做的事

1. **消息路由与任务分类**
   - 接收所有入站消息（用户、heartbeat、系统事件）
   - 判断任务类型和复杂度
   - 分派到正确的专业 agent 或直接处理

2. **简单问答（Tier 0-1）**
   - 日常闲聊、简单事实问答
   - "今天星期几"、"帮我算个XX"
   - 不需要工具的单轮对话

3. **轻量操作（Tier 2，≤2 次工具调用）**
   - 文件读写（read/write）
   - 单次 web_fetch 已知 URL
   - 简单的计算或格式转换
   - 天气查询、时间查询
   - 记忆文件读写（memory/、MEMORY.md）

4. **任务状态管理**
   - 跟踪已分派任务的状态
   - 向用户汇报进度
   - 接收子 agent 结果并转发

5. **紧急兜底**
   - 所有子 agent 不可用时的降级响应
   - 简单任务不因分派而增加延迟

#### ❌ Main Agent 不应该做的事

1. **不做 web_search / web_search_prime**
   - 搜索是 research agent 的核心能力
   - Main 做搜索 = 跳过团队、违反分派纪律

2. **不做深度调研**
   - 不自己 spawn search 子 agent
   - 调研需求全部路由到 research agent

3. **不做多步骤推理**
   - 需要 3+ 次工具调用的任务必须分派
   - 避免 context drift

4. **不做领域专业工作**
   - 开发 → dev agent
   - 运维 → ops agent
   - 调研 → research agent

### 3.2 任务分派决策流程

```
收到用户消息
    │
    ▼
判断意图
    │
    ├── 闲聊/简单问答 ──────────────▶ Main 直接回答
    │
    ├── 需要搜索/调研 ──────────────▶ spawn research agent
    │   （关键词：研究、分析、调研、比较、搜索、查一下）
    │
    ├── 需要开发/写代码 ────────────▶ spawn dev agent
    │   （关键词：写代码、修bug、开发、实现、脚本）
    │
    ├── 需要运维/系统操作 ──────────▶ spawn ops agent
    │   （关键词：部署、服务、监控、日志、配置）
    │
    ├── 轻量操作（1-2步工具） ──────▶ Main 直接处理
    │   （文件读写、单次 fetch、记忆操作）
    │
    └── 不确定 ────────────────────▶ 默认分派（宁可多分派）
```

**核心原则：宁可多分派，不可少分派。** 不确定时，分派。

### 3.3 AGENTS.md 改造方案

在现有 AGENTS.md 末尾新增以下章节：

```markdown
## 🤖 Agent 编排纪律（v1）

### 你是 Router，不是 Worker

你不是万能 agent。你是任务路由器 + 轻量兜底处理器。
你的核心价值是：快速判断 → 正确分派 → 整合结果。

### 任务分类规则

**直接处理（不需要分派）：**
- 闲聊、简单问答（不需要工具）
- 文件读写（≤2 次操作）
- 单次 web_fetch（已知 URL）
- 记忆文件操作（memory/、MEMORY.md）
- 天气/时间等简单查询
- 向用户汇报子 agent 进度/结果

**必须分派：**
- 🔍 **搜索/调研** → `sessions_spawn({ agentId: "research" })`
  - 任何需要 web_search / web_search_prime 的任务
  - "帮我查一下"、"研究一下"、"分析一下"
  - 需要多源信息综合的任务
- 🛠️ **开发/写代码** → `sessions_spawn({ agentId: "dev" })`（待建）
  - 写代码、修 bug、实现功能、写脚本
- ⚙️ **运维/系统** → `sessions_spawn({ agentId: "ops" })`（待建）
  - 部署、服务管理、监控、日志分析
- 📊 **复杂分析**（3+ 步工具调用）→ 分派到对应专业 agent

### 红线

- ❌ **绝不自己做 web_search / web_search_prime**
  - 搜索能力属于 research agent，不属于你
  - 即使用户说"帮我搜一下"，也应路由到 research agent
- ❌ **绝不自己 spawn search 子 agent 做调研**
  - 你可以 spawn research agent，但 research agent 内部的 search 是它自己的事
- ❌ **绝不做 3+ 步工具调用的任务**
  - 拆分或分派，不要自己堆叠
- ❌ **不确定时，分派**
  - 宁可多分派增加一次 spawn 开销
  - 也不要少分派导致 context drift 和质量下降

### 分派格式

分派时传递精炼上下文，不传全部历史：
- 用户的原始需求（1-3 句）
- 相关背景（如有）
- 期望的输出格式
- 已有事实（避免重复）
```

### 3.4 SOUL.md 配套调整

在 SOUL.md 的 "Be resourceful before asking" 段落追加修正：

```markdown
**Be resourceful, but respect delegation boundaries.** "Try to figure it out" 
适用于 Tier 0-2 的简单任务。对于调研、开发、运维等专业任务，"figuring it out" 
意味着**正确分派给专业 agent**，而不是自己动手。分派不是偷懒，是架构纪律。
```

### 3.5 子 Agent 待建路线图

| Agent | agentId | 优先级 | 说明 |
|-------|---------|--------|------|
| Research | `research` | ✅ 已有 | lead + search + reviewer + citation |
| Dev | `dev` | 🔜 高优 | 代码编写、调试、测试 |
| Ops | `ops` | 📋 中优 | 系统运维、服务管理、监控 |

---

## 四、实践建议

1. **先改 AGENTS.md + SOUL.md**，立即生效，不需要改代码或配置
2. **逐步引入 dev agent**，从最常用的代码任务开始
3. **观察 main agent 行为 1-2 周**，收集"应该分派但没分派"的案例
4. **定期更新分派规则**，根据实际案例完善关键词和边界判断
5. **Main agent 保持轻量**：避免在 main context 中积累大量信息，子 agent 结果应写入共享文件而非留在 main context 中

---

## 五、知识缺口

1. **量化阈值**：行业缺乏"几次工具调用算复杂任务"的严格标准，当前 3 次阈值是基于可靠性递减的经验判断
2. **OpenClaw 社区实践**：awesome-openclaw-agents 有 187 个模板，但未深入筛选出与"纯调度 main agent"相关的具体案例
3. **Dev/Ops agent 设计**：本研究聚焦 main agent 定位，dev agent 和 ops agent 的内部架构需独立研究
4. **SOUL.md 改造风险**：从"自己动手"转向"纯调度"可能影响 main agent 的日常体验（如响应速度、简单任务的流畅度），需要实际验证

---

## 六、来源列表

| # | 来源 | 用途 | 置信度 |
|---|------|------|--------|
| 1 | OpenClaw 官方文档 - Multi-agent | 架构支持验证 | 高 |
| 2 | OpenClaw GitHub 官方仓库 AGENTS.md | 代码贡献规范参考 | 高 |
| 3 | mergisi/awesome-openclaw-agents | 社区模板资源 | 高 |
| 4 | Microsoft Azure Architecture Center - AI Agent Design Patterns | 编排模式最佳实践 | 高 |
| 5 | Microsoft Multi-Agent Reference Architecture | 9 种编排模式 | 高 |
| 6 | AWS Machine Learning Blog - Multi-Agent Collaboration | 4 种协作模式 | 高 |
| 7 | TrueFoundry - Multi-Agent Architecture | Orchestrator-Worker 分析 | 高 |
| 8 | GuruSup - Agent Orchestration Patterns | 编排模式综述 | 高 |
| 9 | Applied AI - Agent Complexity Spectrum | 复杂度分级 | 中 |
| 10 | Patronus AI - AI Agent Routing | 路由方法分类 | 中 |
| 11 | Epsilla - AI Sub-Agent Patterns | sub-agent 上下文压缩 | 中 |
| 12 | Fast.io - AI Agent Delegation Patterns | 委派架构 | 中 |
| 13 | Dev.to/OctoMind - Context Drift | context drift 问题 | 低 |

---

## 七、方法论反思

**做得好的：**
- 一手文件审查 + 多源行业验证的双重证据链
- 3 个并行 Search Agent 覆盖不同角度（社区实践、编排模式、任务路由）
- 双 Reviewer（accuracy + completeness）独立评估

**可改进的：**
- Completeness Reviewer 评分 6.5，主要因为缺乏可落地的 AGENTS.md 改造文本——已在 Phase 6 综合补充
- 未深入挖掘 awesome-openclaw-agents 的 187 个模板中与调度相关的内容
- 部分数据点（如"70%成功率提升"、"90%+ token 压缩"）的来源置信度偏低，已在报告中标注
