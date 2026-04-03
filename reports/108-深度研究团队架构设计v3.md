# 深度研究团队 — 架构设计 v3

## 核心思想

**用视角驱动广度，用缺口驱动深度，用独立验证保证质量，用结构化记忆跨越会话边界。**

研究本质上是一个信息压缩过程：从海量信息中提取、验证、组织出有价值的内容。
成败取决于解决三个核心问题：

1. **知道自己不知道什么** — 知识缺口发现
2. **验证自己以为知道的** — 独立审核
3. **不丢失已经知道的** — 上下文管理

---

## 统一框架：探索-验证-收敛循环

```
        ┌──────────────────────────────────────────────────┐
        │                                                  │
        ▼                                                  │
   ┌─────────┐      ┌──────────┐     通过      ┌────────┐ │
   │  探索    │ ───► │  验证     │ ───────────► │  收敛   │ │
   │ Diverge │      │ Validate │               │Converge│ │
   └─────────┘      └──────────┘               └───┬────┘ │
        ▲               │ 不通过                     │      │
        │               ▼                            ▼      │
        │        生成新缺口                     交付报告    │
        └──────────────────────────────────────────────────┘
```

---

### 阶段一：探索（Diverge）

目标：尽可能广地发现信息和缺口。

核心原则：**让 LLM 自由提问质量很差，需要外部约束驱动。**

三种约束组合使用：

**1. 视角约束（来自 STORM）**
- 先搜索类似主题的现有文章，提取不同角度
- 从每个角度出发拆解子问题
- 比"你觉得还需要研究什么"有效得多

**2. 缺口驱动（来自 Jina DeepResearch）**
- 维护结构化的"已知"和"未知"记录
- 每轮搜索后，针对具体缺口继续搜索
- 不让 agent 自由发挥，而是精确瞄准未解答的问题

**3. 递归深挖（来自 dzhng/deep-research）**
- 每层搜索返回两类内容：已发现的事实 + 值得深入的新方向
- 搜索是树遍历，不是线性扫描
- 每次递归带上累积的知识和原始目标

搜索策略的具体原则：
- **先宽后窄**：先用短查询探索全景，再逐步缩小焦点
- Agent 倾向于用过长过具体的查询导致结果太少，需要用 prompt 纠正
- 两层并行：lead agent 并行 spawn 子 agent；子 agent 内部也并行调用工具
- 并行度 3-5 个方向为宜

执行方式：**3-5 个 Search Agent 并行探索不同方向。**

---

### 阶段二：验证（Validate）

目标：质疑和修正探索阶段的产出。

核心原则：**agent 审核自己的工作不可靠，验证必须独立。**

这是所有项目最一致的发现：agent 会自信地表扬自己的工作，即使质量明显差。分离做和审是"强力杠杆"。

验证层的三个条件：
- **独立** — 验证者没有参与探索，没有沉没成本
- **结构化** — 逐条对照明确标准打分，不是"你觉得好不好"
- **有行动力** — 验证结果转化为具体的下一步动作

验证标准（每条通过/不通过，不给模糊空间）：
- 来源是否一手
- 时效性是否满足
- 多个来源是否一致
- 是否回答了原始问题
- 有没有明显遗漏的角度

评估方法论：
- LLM-as-judge：0.0-1.0 评分 + pass/fail，单次 LLM 调用比多 judge 更一致
- 评估维度：事实准确性、引用准确性、完整性、来源质量、工具效率
- 人类评估不可省略——能发现自动评估漏掉的系统性偏见（如偏好 SEO 农场）
- 从 20 个测试用例开始就够——早期改动效果大，小样本即可看出变化

验证不通过时，输出**精确的知识缺口**：
不是"再搜一遍"，而是"需要找到 XX 方面、来自 YY 时段、ZZ 类型的来源"。
下一轮探索才有明确的靶子。

---

### 阶段三：收敛（Converge）

目标：把验证过的信息组织成最终交付物。

收敛规则：
- 按大纲结构组织，不按搜索顺序堆叠
- 引用和正文分离处理（独立的 Citation Agent）
- 过滤掉未通过验证的信息，宁缺毋滥
- 引用标注：每个事实都有可追溯的来源

---

## 循环控制参数

| 参数 | 推荐值 | 说明 |
|------|--------|------|
| breadth | 3-5 | 每轮并行探索几个方向 |
| depth | 2-3 | 最多递归几轮 |
| quality_threshold | 7/10 | 验证评分低于此值打回重搜 |
| max_iterations | 3 | 探索-验证循环最多几轮 |

灭火机制：连续两轮没发现新信息时，强制终止探索，用已有材料产出报告。

按任务复杂度分级投入资源：
- 简单事实查找 = 1 agent + 3-10 次工具调用
- 直接比较 = 2-4 子 agent × 10-15 调用
- 深度研究 = 5-10 子 agent，各有明确分工
- 每次深度研究约 $0.4（o3-mini），约 5 分钟

---

## 跨会话记忆

长任务必跨上下文窗口。

关键发现：**Context Reset > Compaction**
完全重置上下文 + 结构化交接，优于在同一上下文中压缩。
原因：compaction 不能消除"上下文焦虑"（模型感觉快到上限就草率收尾）。
（注：此发现基于 Sonnet 4.5 的实验，更先进的模型可能不出现上下文焦虑。）

解决方案：**用结构化文件做交接，不依赖对话历史。**

核心交接文件（全部使用 JSON）：

**research-plan.json — 研究计划**
```json
{
  "topic": "原始研究问题",
  "created_at": "...",
  "perspectives": ["视角1", "视角2", "视角3"],
  "sub_questions": [
    {
      "id": "q1",
      "question": "子问题",
      "perspective": "来自哪个视角",
      "priority": "high",
      "status": "pending|exploring|reviewed|done"
    }
  ],
  "parameters": {
    "breadth": 4,
    "depth": 2,
    "max_iterations": 3,
    "quality_threshold": 7
  },
  "current_iteration": 0
}
```

**knowledge-base.json — 已验证知识**
```json
{
  "facts": [
    {
      "id": "f1",
      "claim": "事实陈述",
      "evidence": "原文摘录",
      "source_url": "https://...",
      "source_name": "来源名称",
      "source_type": "primary|secondary|unsourced",
      "confidence": "high|medium|low",
      "verified": true,
      "verified_at": "..."
    }
  ]
}
```

**gaps.json — 知识缺口**
```json
{
  "open_gaps": [
    {
      "id": "g1",
      "description": "需要找到 XX 方面的 YY 数据",
      "source_type_hint": "学术论文|行业报告|新闻报道",
      "time_range": "2024-2026",
      "priority": "high|medium|low",
      "attempts": 0
    }
  ],
  "closed_gaps": [
    {"id": "g0", "resolved_by": "f1", "closed_at": "..."}
  ],
  "dead_ends": [
    {"id": "g2", "reason": "搜索3轮无结果，标记为死路", "closed_at": "..."}
  ]
}
```

**bad-answers.json — 坏答案记录**
```json
{
  "rejected_answers": [
    {
      "attempt": "尝试回答的内容",
      "rejection_reason": "被拒绝的原因",
      "timestamp": "..."
    }
  ]
}
```

每次新会话启动固定流程：
1. 读 research-plan.json → 了解全局
2. 读 knowledge-base.json → 知道已经知道了什么
3. 读 gaps.json → 知道还缺什么
4. 读 bad-answers.json → 知道什么方向走不通
5. 从最高优先级的 open_gap 继续

---

## Agent 角色设计

```
用户："帮我研究 XXX"
         │
         ▼
┌─────────────────────────────────────────────────────────┐
│              Lead Researcher（主研究员）                   │
│                                                          │
│  ① 理解需求，搜索类似主题文章，提取多视角                  │
│  ② 基于视角拆解子问题，写入 research-plan.json            │
│  ③ 并行 spawn Search Agents 探索不同方向                  │
│  ④ 收到结果后 spawn Review Agent 验证                    │
│  ⑤ 根据验证结果更新 gaps.json，决定继续或收敛             │
│  ⑥ 收敛阶段：汇总验证过的知识，生成报告                    │
│                                                          │
│  ❌ 不做：搜索、验证质量、自评                             │
└────┬─────────────┬──────────────────┬───────────────────┘
     │             │                  │
     ▼             ▼                  ▼
┌──────────┐ ┌──────────┐     ┌──────────────────────┐
│Search #1 │ │Search #2 │ ... │   Review Agent       │
│          │ │          │     │   （独立审核员）       │
│ 视角 A   │ │ 视角 B   │     │                      │
│          │ │          │     │  逐条审核 findings    │
│ 搜索     │ │ 搜索     │     │  交叉验证一致性      │
│ 阅读     │ │ 阅读     │     │  标记可靠性等级      │
│ 提取事实 │ │ 提取事实 │     │  发现矛盾和缺口      │
│ 记录来源 │ │ 记录来源 │     │  输出精确的补充方向  │
│          │ │          │     │                      │
│ 发现缺口 │ │ 发现缺口 │     │  ❌ 不做：搜索、汇总  │
│ 记录方向 │ │ 记录方向 │     └──────────────────────┘
└──────────┘ └──────────┘
     │             │
     ▼             ▼
┌──────────────────────────────────────┐
│        Citation Agent（引用处理）      │
│  - 独立于报告撰写者处理引用           │
│  - 标准化引用格式                    │
│  - 验证来源可访问性                  │
│  - 去重和合并相同来源                │
└──────────────────────────────────────┘
```

5 个角色，职责不交叉：

| 角色 | 做什么 | 绝不做什么 |
|------|--------|-----------|
| **Lead Researcher** | 视角发现、拆解、调度、汇总报告 | 搜索、审核 |
| **Search Agent ×N** | 并行搜索、阅读、提取事实、发现缺口 | 审核、写报告 |
| **Review Agent** | 逐条审核、交叉验证、找缺口、打分 | 搜索、汇总 |
| **Citation Agent** | 引用格式化、来源验证、去重 | 搜索、写正文 |

Lead Agent 的关键技能——学会怎么分配任务：
- 每个子 agent 需要 4 样东西：明确目标、输出格式、工具/来源指引、任务边界
- 模糊指令如"研究 XX 短缺"会导致子 agent 重复工作或跑偏
- 子 agent 之间通过文件通信，不通过消息传递

---

## 工作流程详解

### Phase 1：规划（视角驱动）

```
用户问题 → Lead Researcher
  1. 搜索类似主题的现有文章（2-3篇）
  2. 从文章中提取不同视角/角度
  3. 基于视角拆解为 3-8 个子问题
  4. 写入 research-plan.json
  5. 初始化 gaps.json（每个子问题本身就是一个 gap）
```

### Phase 2：探索（并行 + 缺口驱动）

```
Lead → spawn Search Agent #1 (子问题A / 视角1)
     → spawn Search Agent #2 (子问题B / 视角2)
     → spawn Search Agent #3 (子问题C / 视角3)

每个 Search Agent：
  1. 收到明确的子任务 + 对应视角
  2. 搜索 → 阅读 → 推理 → 再搜索（循环）
  3. 提取事实 → 记录到 findings
  4. 发现新缺口 → 记录到 gaps
  5. 发现新方向 → 记录到 directions
  6. announce 回 Lead
```

### Phase 3：验证（独立审核）

```
Lead 收到所有 Search Agent 的 findings + gaps
Lead → spawn Review Agent

Review Agent：
  1. 逐条审核每个 finding（来源、时效、一致性、完整性）
  2. 标记：✅ 可靠 / ⚠️ 需补充 / ❌ 不可靠
  3. 交叉验证不同 findings 之间的矛盾
  4. 对比 research-plan.json 的子问题，检查覆盖度
  5. 合并 Search Agent 发现的新缺口，去重排序
  6. 整体评分（1-10）
  7. announce 回 Lead
```

### Phase 4：迭代决策

```
Lead 收到 Review Agent 的审核报告

如果评分 >= 阈值 且 无重大缺口：
  → 进入 Phase 5 收敛

如果评分 < 阈值 或 有重要缺口：
  → 更新 gaps.json（加入 Review Agent 发现的新缺口）
  → 更新 bad-answers.json（记录被拒绝的尝试）
  → 回到 Phase 2，spawn 新的 Search Agent 补充
  → 最多迭代 max_iterations 轮

灭火检查：
  → 连续两轮搜索没产生新 finding？
  → 强制进入 Phase 5
```

### Phase 5：收敛

```
Lead 综合所有 verified findings
  1. 按大纲组织结构
  2. 过滤掉未通过验证的信息
  3. spawn Citation Agent 处理引用
  4. 生成完整研究报告
  5. 发送给用户
```

---

## Review Agent 提示设计

Review Agent 必须被设计为严格的审稿人：

```
你是一个严格的研究审核员。你的工作是质疑、验证、挑战。

审核维度（每个 finding 逐条判断）：
1. 来源可靠性：一手 > 二手转述 > 无来源。无来源一律标 needs_verification
2. 时效性：过时数据标记 ⚠️，标明截止时间
3. 一致性：不同来源说法矛盾时必须标出，不能假装没看见
4. 完整性：重要角度被遗漏时必须指出
5. 偏见检测：来源有明显立场或商业利益时必须标注

回答质量标准：
- "大概""可能""据说"等模糊表述 → 不通过
- 只有单一来源支撑的断言 → 需补充
- 没有直接回答原始问题的 finding → 标记为偏题

你的风格：
- 宁可误伤不可漏判
- 绝不给出"整体还不错"这种模糊评价
- 绝不因为工作量大就降低标准
- 绝不默认信任任何单一来源

评分规则：
- 10 = 所有子问题充分回答，来源可靠，无矛盾
- 7-9 = 大部分回答了，有少量可接受的缺口
- 4-6 = 有重要缺口或可靠性问题
- 1-3 = 存在根本性问题，基本不可用
```

---

## 关键经验教训

### 已验证（有数据支撑）
- 多 agent 比单 agent 性能高 90.2%（Anthropic 内部评测，广度优先查询）
- Token 使用量解释 80% 性能差异（BrowseComp 公开基准）
- 自评不可靠，分离 generator 和 evaluator 是"强力杠杆"
- 并行化使复杂查询时间缩短 90%
- 升级模型比加 token 更有效
- Context Reset > Compaction（基于 Sonnet 4.5）

### 经验法则（多团队独立验证）
- 先宽后窄搜索策略
- 按复杂度分级投入资源
- 20 个测试用例开始评估就够
- 每个子 agent 需要 4 样东西：目标、格式、工具指引、边界

### 已知的局限性
- 90.2% 性能提升可能 cherry-pick 了适合并行的查询类型
- Context Reset > Compaction 只在 Sonnet 4.5 上验证，更先进模型可能不需要
- 多 agent 系统 token 消耗是聊天的 15 倍，需要高价值任务才划算
- 依赖性强的任务（如编码）不适合多 agent 并行

### 未解答的问题
- 并行度的精确最优值（为什么是 3-5？缺少消融实验）
- 多轮迭代的边际收益曲线（第 3 轮迭代还有多少提升？）
- 小模型（7B-14B）做子 agent 的可行性
- 搜索工具质量对结果的影响（DuckDuckGo vs Google vs Bing）
- 中文研究型 agent 的最佳实践（当前调研以英文项目为主）

---

## 成本估算

| 场景 | 子 agent 数 | 工具调用/agent | 预估 token | 预估成本（o3-mini） |
|------|------------|---------------|-----------|-------------------|
| 简单事实 | 1 | 5-10 | ~50K | ~$0.05 |
| 中等比较 | 2-4 | 10-15 | ~200K | ~$0.15 |
| 深度研究 | 5-10 | 15-30 | ~1M | ~$0.40 |

---

## OpenClaw 实现

### 配置 (openclaw.json)
```json5
{
  agents: {
    defaults: {
      subagents: {
        maxSpawnDepth: 2,
        maxConcurrent: 8,
        maxChildrenPerAgent: 5,
        runTimeoutSeconds: 1800
      }
    }
  }
}
```

### 执行方式
- Lead Researcher = 主 agent（我）
- Search Agent = `sessions_spawn`（并行，depth 1）
- Review Agent = `sessions_spawn`（独立会话）
- Citation Agent = `sessions_spawn`（独立会话）
- 每种角色的 system prompt 不同，工具权限不同

### 搜索工具依赖
- web_search：当前依赖 Kimi API key
- web_fetch：可直接使用，无额外依赖
- 缺少可靠的搜索 API 是当前最大的基础设施瓶颈

---

## 调研来源

本方案综合了以下项目的核心智慧：
- STORM（Stanford, NAACL 2024）：视角驱动提问、对话式收集、思维导图
- GPT-Researcher（~20k ⭐）：8 agent 流水线、独立 Reviewer+Revisor、Plan-and-Solve
- Jina DeepResearch（~8k ⭐）：gaps queue 缺口追踪、动作去重、Beast Mode
- dzhng/deep-research（~7k ⭐）：breadth×depth 递归搜索、知识传递
- Anthropic Research（闭源）：多 agent 并行（+90.2%）、独立 CitationAgent、LLM-as-judge
- Anthropic Harness：结构化 JSON 跨会话记忆、增量推进、Context Reset > Compaction、GAN 式 generator-evaluator

## 下一步

1. 解决搜索基础设施（API key / 搜索工具）
2. 写每个角色的详细 prompt
3. 用一个具体研究题目做端到端测试
4. 根据测试结果迭代
5. 回答未解问题：并行度消融实验、迭代边际收益、小模型子 agent 可行性
