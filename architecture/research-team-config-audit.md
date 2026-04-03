# 研究团队配置审计报告

> 2026-03-29 | 基于 OpenClaw 文档深度研究

## 核心发现：独立 Agent vs Subagent 的 Context 注入差异

### 关键事实

当主 Agent 通过 `sessions_spawn(agentId: "search")` 调度独立 agent 时：

| 维度 | 行为 |
|------|------|
| **Runtime** | subagent runtime（不是该 agent 的 main session） |
| **Session key** | `agent:search:subagent:<uuid>` |
| **Workspace** | ✅ 使用该 agent 的 workspace（`~/.openclaw/workspace-search`） |
| **Model** | ✅ 使用该 agent 的 model 配置（`zai/glm-5-turbo`） |
| **Tools** | ✅ 使用该 agent 的 tools allow/deny 配置 |
| **Prompt mode** | ⚠️ `minimal`（subagent 模式） |
| **AGENTS.md** | ✅ 注入（subagent 默认注入） |
| **TOOLS.md** | ✅ 注入（subagent 默认注入） |
| **SOUL.md** | ❌ **不注入**（subagent 只注入 AGENTS.md + TOOLS.md） |
| **IDENTITY.md** | ❌ 不注入 |
| **USER.md** | ❌ 不注入 |
| **HEARTBEAT.md** | ❌ 不注入 |
| **MEMORY.md** | ❌ 不注入 |
| **Skills** | ❌ 不注入（minimal mode 省略） |

### 这意味着什么

**独立 agent 的 SOUL.md 在被 spawn 为 subagent 时不会被看到！** 

所以当前的架构设计有一个关键缺陷：
- search/reviewer/citation 的 SOUL.md（详细的 prompt）**不会被注入**
- 只有 AGENTS.md + TOOLS.md 被注入
- subagent 的行为完全由 task 参数定义

## 解决方案

### 方案 A：将 prompt 写入 AGENTS.md（推荐 ✅）

因为 AGENTS.md 会被注入到 subagent context，把核心 prompt 放在这里：

```
workspace-search/
├── AGENTS.md    ← 核心角色定义 + 输出格式 + 行为规范（SOUL.md 的内容搬到这里）
├── TOOLS.md    ← 工具使用说明（可选）
└── SOUL.md     ← 保留，用于该 agent 的 main session（如果需要直接交互）
```

**优点：** 简单直接，利用现有注入机制
**缺点：** AGENTS.md 会增大 context，但搜索 agent 的 prompt 本来就不长

### 方案 B：在 Lead 的 task 参数中传递完整 prompt

Research Lead 在 spawn 子 agent 时，把完整 prompt 模板写在 task 里。

**优点：** 灵活，可按任务动态调整
**缺点：** Lead 的 SOUL.md 会非常长，增加 token 消耗

### 方案 C：使用 attachments 传递 prompt 文件

Lead 把 prompt 文件作为 attachment 传给子 agent，子 agent 在 AGENTS.md 中被告知先读取 attachment。

**优点：** Lead 的 SOUL.md 保持简洁
**缺点：** 子 agent 需要 read 工具 + 额外的工具调用

### 推荐：方案 A + C 混合

- **AGENTS.md**：核心角色定义 + 输出格式规范（保持精简，< 2000 字）
- **attachments**：任务特定的指令（搜索关键词、已有知识等）
- **Lead 的 task**：简洁的调度指令 + 引用 attachment

## 完整配置方案

### 1. Search Agent

**AGENTS.md**（核心 prompt，会被注入）：
```
# Search Agent（搜索研究员）

你是 Search Agent。唯一任务：搜索 → 阅读 → 提取事实。

## 绝对规则
- ❌ 不做质量判断、不做总结报告
- ❌ 不重复搜索相同关键词
- ❌ 不在 JSON 外输出任何文字
- ✅ 最多 15 次工具调用，连续 2 次无新结果则停止

## 输出格式
严格 JSON：
{"findings":[{"claim":"...","evidence":"...","source_url":"...","confidence":"high|medium|low","source_name":"..."}],"visited_urls":[],"used_queries":[],"gaps_found":[],"agent_metadata":{"tool_calls_used":N,"blocked_reasons":[]}}

置信度：high=一手来源+数据，medium=可信二手，low=个人博客
```

**tools 配置**：
```json
{
  "allow": ["web_search", "web_fetch", "browser", "read"],
  "deny": ["exec", "process", "sessions_spawn", "cron"]
}
```

### 2. Reviewer Agent

**AGENTS.md**（核心 prompt）：
```
# Reviewer Agent（审核员）

你是 Reviewer Agent。根据 task 中的 review_type 执行审核。

## 审核模式
- "accuracy"：评估事实准确性（来源可靠性+时效性+可验证性+无偏见，0-10分）
- "completeness"：评估覆盖完整性（视角+深度+缺口+连贯性，0-10分）

## 绝对规则
- ❌ 不做搜索、不做汇总报告
- ❌ 不要被前一个 finding 的评分影响

## 输出格式
严格 JSON（根据 review_type 不同格式不同，详见 Lead 传入的指令）
```

**tools 配置**：
```json
{
  "allow": ["read"],
  "deny": ["exec", "process", "web_search", "web_fetch", "browser", "sessions_spawn", "cron", "write"]
}
```

### 3. Citation Agent

**AGENTS.md**（核心 prompt）：
```
# Citation Agent（引用处理员）

你是 Citation Agent。标准化引用格式、验证来源可访问性。

## 绝对规则
- ❌ 不判断事实对错、不改写正文、不做搜索

## 输出格式
严格 JSON：
{"citations":[{"id":"c1","ref":"[1]","title":"...","source_type":"paper|official_doc|tech_blog|news|other","url":"...","accessible":true,"used_by":["f1"]}],"broken_links":[],"stats":{"total":N,"unique":N,"broken":N}}
```

**tools 配置**：
```json
{
  "allow": ["web_fetch", "read"],
  "deny": ["exec", "process", "web_search", "browser", "sessions_spawn", "cron", "write"]
}
```

### 4. Research Lead Agent

**AGENTS.md**（核心调度逻辑）：
```
# Research Lead（研究主管）

你是 Lead Researcher。你是调度员，不是执行者。

## 团队
- search agent: 搜索研究员（agentId: "search"）
- reviewer agent: 审核员（agentId: "reviewer"）
- citation agent: 引用处理员（agentId: "citation"）

## 工作流程
Phase 1 规划 → Phase 2 并行 Search → Phase 3 去重 → Phase 4 双 Reviewer → Phase 5 迭代判断 → Phase 6 收敛

## 调度方式
用 sessions_spawn({ agentId: "search"|"reviewer"|"citation", model: "...", task: "..." }) 调度子 agent。
子 agent 的 task 要清晰、具体、包含输出格式要求。
```

**SOUL.md**（不会被 subagent 注入，但 Lead 自己是 subagent 所以也不会看到！）

等等——**Research Lead 自己也是被 main agent spawn 的 subagent**！所以 Lead 的 SOUL.md 也不会被注入！

这意味着 **Research Lead 的核心 prompt 也必须写在 AGENTS.md 里**。

### 5. Main Agent → Research Lead 的 Context 问题

同样的问题：main agent spawn research 时，research agent 作为 subagent 只看到 AGENTS.md + TOOLS.md。

所以完整方案是：

```
workspace-research/
├── AGENTS.md    ← Lead 的完整 prompt（角色、工作流、调度模板、JSON 解析策略）
├── SOUL.md     ← 保留（research 的 main session 用，目前没有 main session）
└── TOOLS.md    ← 可选

workspace-search/
├── AGENTS.md    ← Search 的完整 prompt
├── SOUL.md     ← 保留
└── TOOLS.md    ← 可选

workspace-reviewer/
├── AGENTS.md    ← Reviewer 的完整 prompt
├── SOUL.md     ← 保留
└── TOOLS.md    ← 可选

workspace-citation/
├── AGENTS.md    ← Citation 的完整 prompt
├── SOUL.md     ← 保留
└── TOOLS.md    ← 可选
```

## Depth 问题

```
main (depth 0) → research (depth 1) → search (depth 2)
```

- maxSpawnDepth=2 ✅ 允许 depth 1 spawn depth 2
- depth 1 (research) 需要有 sessions_spawn 工具 ✅ 已在 tools.allow 中添加
- depth 2 (search) 不需要 spawn ✅

## 需要修改的文件

1. `workspace-research/AGENTS.md` — 写入 Lead 完整 prompt
2. `workspace-search/AGENTS.md` — 写入 Search 完整 prompt  
3. `workspace-reviewer/AGENTS.md` — 写入 Reviewer 完整 prompt
4. `workspace-citation/AGENTS.md` — 写入 Citation 完整 prompt
5. 各 SOUL.md 可以保留（用于 main session 交互），但核心 prompt 必须在 AGENTS.md 中
