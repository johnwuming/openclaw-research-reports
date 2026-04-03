# 深度研究团队 v4 — 改进方案

> 基于 v3 实践 + 5 个改进方向的深度调研
> 2026-03-28

---

## v3 实践暴露的问题

| 问题 | 严重程度 | v3 中的表现 |
|------|---------|-----------|
| JSON 输出不稳定 | 🔴 高 | 4/5 sub-agent 正常输出 JSON，1 个需要容错 |
| 无循环检测 | 🟡 中 | Search Agent 有步数限制但无重复检测 |
| Review Agent 偏见 | 🟡 中 | 只有单个 reviewer，存在 LLM 固有偏见 |
| 中文搜索弱 | 🟡 中 | DuckDuckGo 中文结果差，被反爬限流 |
| Agent 间信息传递粗糙 | 🟡 中 | 只通过 task prompt 传递，缺少结构化 schema |
| 无进度追踪 | 🟡 中 | Lead 不知道 Search Agent 是否在重复劳动 |

---

## 改进一：JSON 输出稳定性（🔴→🟢）

### 根本原因
OpenClaw sub-agent 不支持 provider-native Structured Outputs（那是 API 层的能力）。我们只能通过 prompt + 容错来提升稳定性。

### 改进措施

**1. Lead Agent 端：容错解析层**
```python
# Lead Agent 收到 sub-agent 结果后的处理流程
def parse_subagent_result(raw_text):
    # 1. 尝试直接解析 JSON
    try:
        return json.loads(extract_json(raw_text))
    except:
        pass
    
    # 2. 尝试修复常见错误（尾逗号、注释、markdown包裹）
    try:
        return json.loads(repair_json(raw_text))
    except:
        pass
    
    # 3. 让 LLM 修复（最后一次机会）
    try:
        fixed = llm_extract_json(raw_text)
        return json.loads(fixed)
    except:
        # 4. 标记为失败，spawn 替换
        return {"_parse_failed": True, "raw": raw_text}
```

**2. Search Agent prompt 改进**
```
# 在 prompt 末尾增加强制 JSON 输出保障

## 输出保障（最重要！）
你的最后一次工具调用必须是输出 JSON 结果。
如果前面的搜索都没有结果，也必须输出空的 findings 数组。
绝对不要在 JSON 后面追加文字说明。

格式要求：
- JSON 必须以 { 开头，以 } 结尾
- 不要用 ```json ``` 包裹
- 不要在 JSON 前后写任何解释文字
- 如果无法输出 JSON，输出：{"findings":[],"visited_urls":[],"gaps_found":["无法完成搜索"]}
```

**3. 简化 schema**
```
# 原方案：嵌套对象 + 多种类型 → 容易出错
# 改进方案：扁平化 + 固定字段

findings 数组中每个元素只有 5 个固定字段：
- claim: string（必填）
- evidence: string（必填）
- source_url: string（必填）
- confidence: "high"|"medium"|"low"（必填，三选一）
- source_name: string（可选，为空则从 URL 提取）

不要添加额外字段。
```

---

## 改进二：循环检测与防死循环（🟡→🟢）

### 改进措施

**1. Search Agent：内置去重状态**
```
# Search Agent prompt 增加

## 循环防护（必须遵守）
- 维护一个 visited_urls 列表，每次访问前检查是否已访问
- 维护一个 used_queries 列表，每次搜索前检查是否已搜过相同关键词
- 如果连续 2 次搜索返回无结果或与之前结果重复，立即停止搜索并输出已有 findings
- 最多 15 次工具调用（含搜索、web_fetch、browser 操作），到达上限立即输出
```

**2. Lead Agent：进度追踪**
```
# Lead Agent 在 spawn Search Agent 时传递

## 进度追踪
收到 Search Agent 结果后：
1. 检查 findings 数组是否为空 → 空 = 可能被限流或方向错误
2. 检查 gaps_found 是否与上一轮相同 → 相同 = 无新发现，考虑换方向
3. 检查 visited_urls 与之前轮次的 overlap → 高 overlap = 方向枯竭
4. 连续两轮无新 facts → 标记为死路，不再探索此方向
```

**3. Review Agent：检测系统性问题**
```
# Review Agent 增加

## 系统性问题检测
- 如果所有 findings 都来自同一来源（单一来源偏见）→ score 整体降 2 分
- 如果所有 findings 都是 low confidence → 建议重新搜索而非继续
- 如果发现 contradictions → 这是正面信号（说明搜索足够广）
- 如果 overall_quality < 5 → recommendation 必须是 "major_revision"
```

---

## 改进三：质量控制升级（🟡→🟢）

### 改进措施

**1. 双 Reviewer 交叉验证**
```
# 原 v3：单个 Review Agent
# v4：两个独立 Review Agent，不同 prompt 角度

Reviewer A（准确性审查员）：
- 专注：事实准确性、来源可靠性、数据可验证性
- 不关心：覆盖完整性、写作质量

Reviewer B（完整性审查员）：
- 专注：覆盖是否全面、是否有明显遗漏角度、缺口是否关键
- 不关心：单个事实的准确性

Lead Agent 综合 A 和 B 的评分取平均。
```

**2. Reviewer 隐藏来源偏见**
```
# Reviewer A prompt 改进

## 减少偏见
- 你不知道这些 findings 来自哪个 Search Agent
- 你不知道其他 Reviewer 的评分
- 对每个 finding 独立评分，不要被前一个的评分影响（锚定效应）
- 优先质疑而非确认（确认偏差是人类和 LLM 的共同弱点）
```

**3. 多源交叉验证要求**
```
# Review Agent 评分标准调整

## 交叉验证加分
- 如果一个 claim 有 2+ 个独立来源确认 → confidence 自动升级
- 如果一个 claim 只有一个来源且是厂商自报 → score 上限 5
- 如果一个 claim 与常识矛盾但有一手来源 → 给 7 分但标记"需人工确认"
```

---

## 改进四：中文搜索方案（🟡→🟢）

### 关键发现：智谱有原生 Web Search API

智谱提供 Web Search API（search_pro 引擎），支持：
- 意图增强检索（query 拆解 + 多轮对话搜索）
- 多搜索引擎（自研 + 搜狗/夸克）
- 域名过滤、时间范围过滤、摘要长度控制
- 结构化输出（标题/URL/摘要/网站名称）
- MCP Server（可接入 Cursor 等）
- **我们已经配置了智谱 API key，可以直接用！**

### 改进措施

**1. 搜索分层策略**
```
# 搜索工具选择优先级（改进版）

中文研究主题：
1. 智谱 Web Search API（通过 MCP 或直接 API 调用）→ 中文最优
2. 浏览器搜索百度/搜狗 → 备选
3. DuckDuckGo → 最后手段

英文研究主题：
1. DuckDuckGo（web_search）→ 免费快速
2. 浏览器搜索 Google → 深度搜索
3. 智谱 Web Search → 备选（支持英文但非最优）

通用：
- web_fetch 抓取已知 URL → 任何语言
- 浏览器深度阅读 → JS 渲染页面
```

**2. Search Agent prompt 增加语言策略**
```
## 搜索语言策略
- 判断研究主题的主要语言
- 中文主题优先搜索中文来源（百度/知乎/36kr/CSDN）
- 英文主题优先搜索英文来源（Google Scholar/arXiv/GitHub）
- 关键数据尝试中英文各搜一次，交叉验证
```

**3. 智谱 MCP 集成（后续）**
```
# 智谱提供 MCP Server，可接入 OpenClaw
# URL: https://open.bigmodel.cn/api/mcp-broker/proxy/web-search/mcp
# 需要 Authorization header

# OpenClaw 配置（待验证）
{
  "tools": {
    "mcp": {
      "zhipu-search": {
        "command": "...",  // MCP server 启动命令
        "args": ["--authorization", "YOUR_KEY"]
      }
    }
  }
}
```

---

## 改进五：Agent 间信息传递优化（🟡→🟢）

### 关键发现

Anthropic 的核心建议：
- **Prompt Chaining**：每个 LLM 调用处理前一个的输出，中间加 programmatic gate
- **Sectioning 并行化**：独立子任务并行，每个 agent 专注一个方面
- **保持简洁**：从直接 API 调用开始，只在需要时加复杂度

### 改进措施

**1. 结构化任务描述（取代自由文本 prompt）**
```
# 原 v3：Lead 用自然语言描述任务
"搜索以下问题并提取事实..."

# v4：结构化任务 JSON

{
  "task_type": "search",
  "research_topic": "整体研究主题（给 agent 上下文）",
  "sub_question": "这个 agent 负责的具体子问题",
  "search_hints": [
    "建议搜索的关键词 1",
    "建议搜索的关键词 2"
  ],
  "source_hints": [
    "建议优先查看的域名/来源类型"
  ],
  "output_schema": {
    "type": "array",
    "items": {
      "claim": "string",
      "evidence": "string", 
      "source_url": "string",
      "confidence": "high|medium|low"
    }
  },
  "constraints": {
    "max_tool_calls": 15,
    "max_findings": 20,
    "language": "zh-CN"
  }
}
```

**2. Lead → Search Agent 传递精炼上下文**
```
# 不要传递全部历史，只传递：
1. 整体研究主题（1 句话）
2. 这个 agent 负责的子问题（1 句话）
3. 已知的关联信息（2-3 条 facts，不是全部）
4. 其他 agent 正在搜索的方向（避免重复）
5. 明确的输出格式和约束
```

**3. Search Agent 输出中增加元数据**
```
# Search Agent 输出增加

{
  "agent_metadata": {
    "agent_id": "search-q1",
    "tool_calls_used": 12,
    "queries_used": ["keyword1", "keyword2"],
    "blocked_reasons": ["DuckDuckGo rate limited at call 8"]
  },
  "findings": [...]
}

这样 Lead 可以：
- 判断 agent 是否充分利用了预算
- 识别限流问题
- 避免其他 agent 使用相同的关键词
```

---

## 改进总览

| 改进 | v3 状态 | v4 状态 | 核心变化 |
|------|--------|--------|---------|
| JSON 稳定性 | 🔴 无保障 | 🟢 容错解析 + 简化 schema + 强制输出 | 3 层防护 |
| 循环检测 | 🟡 步数限制 | 🟢 去重 + 进度追踪 + 死路标记 | 多维检测 |
| 质量控制 | 🟡 单 reviewer | 🟢 双 reviewer 交叉验证 | 消除偏见 |
| 中文搜索 | 🟡 DuckDuckGo | 🟢 智谱 Web Search API + 分层策略 | 原生中文 |
| 信息传递 | 🟡 自由文本 | 🟢 结构化任务 JSON + 元数据 | 可靠交接 |
| 来源多样性 | 🟡 无要求 | 🟢 多源交叉验证要求 | 质量保障 |

---

## 更新后的提示词变更摘要

### Search Agent 主要变更
1. 增加 `visited_urls` 和 `used_queries` 去重要求
2. 增加 `agent_metadata` 输出字段
3. 增加搜索语言策略指引
4. 简化 output schema（5 个固定字段）
5. 增加强制 JSON 输出保障段落
6. 连续 2 次无结果 → 立即停止

### Review Agent 主要变更
1. 拆分为 Reviewer A（准确性）和 Reviewer B（完整性）
2. 增加偏见防护指令（隐藏来源、独立评分）
3. 增加多源交叉验证加分规则
4. 增加系统性问题检测（单一来源偏见、全低 confidence）
5. `< 5 分必须 major_revision` 的硬规则

### Lead Researcher 主要变更
1. 增加 JSON 容错解析层
2. 使用结构化任务 JSON 而非自由文本
3. 传递精炼上下文而非全部历史
4. 双 Reviewer 综合评分
5. 增加 progress tracking（检查 overlap、死路检测）
6. 搜索工具按语言分层选择

---

## 成本影响

| 变更 | 成本变化 |
|------|---------|
| 双 Reviewer | +1 次 GLM-5.1 调用（~20k token） |
| JSON 容错解析 | 无额外成本（Lead Agent 本地处理） |
| 智谱 Web Search | 取决于 API 定价（可能免费额度） |
| 结构化任务描述 | 略增 prompt token（~500 token/agent） |
| **总增加** | **约 20k token/次研究（仍为免费模型）** |

---

## 下一步

1. ✅ 更新 `research-team-prompts.md` 中的提示词（v4 版本）
2. ⬜ 配置智谱 Web Search API
3. ⬜ 实现 Lead Agent 的 JSON 容错解析逻辑
4. ⬜ 用同一主题做 v3 vs v4 对比测试
5. ⬜ 评估智谱中文搜索 vs DuckDuckGo 的效果差异
