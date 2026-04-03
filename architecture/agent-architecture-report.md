# OpenClaw 多 Agent 架构完整报告

> 生成时间：2026-04-03 23:15（修订版）
> 范围：~/.openclaw 下所有 agent 配置、workspace 文件、设计文档
> 完整提示词方案：[agent-config-scheme.md](agent-config-scheme.md)

---

## 一、架构总览

### 1.1 设计意图

**用角色隔离保证质量，用分层调度管理复杂度。**

架构分为两层：
- **主 Agent（main）**：唯一用户入口，负责意图识别和任务路由
- **专业团队**：研究团队（Research）和开发团队（Dev），各自由 Lead Agent 内部调度子 Agent

核心设计原则：
1. **唯一入口** — 所有用户交互只通过 main Agent
2. **做审分离** — 实现者和验证者必须是不同 Agent（来自 Anthropic 实践）
3. **工具最小权限** — 每个 Agent 只拥有完成职责所需的最小工具集
4. **文件交互** — 团队间通过共享目录（shared/）传递任务和结果

### 1.2 Agent 关系图

```
                        Telegram / Web UI
                              │
                              ▼
                    ┌─────────────────┐
                    │   Main Agent     │ ← 小朱桑（主 Agent）
                    │   kimi/k2p5     │    唯一用户入口
                    │   allowAgents:  │
                    │   research, dev  │
                    └──┬────────────┬─┘
                       │            │
            sessions_spawn    sessions_spawn
                       │            │
          ┌────────────┘            └────────────┐
          ▼                                      ▼
┌──────────────────┐                ┌──────────────────┐
│ Research Team    │                │ Dev Team         │
│ (研究团队)       │                │ (开发团队)       │
│                  │                │                  │
│ research (Lead)  │                │ dev (Lead)       │
│ ├─ search        │                │ ├─ dev-init      │
│ ├─ reviewer      │                │ ├─ dev-coder     │
│ └─ citation      │                │ └─ dev-qa        │
│                  │                │                  │
│ Workspace:       │                │ Workspace:       │
│ workspace-research│               │ workspace-dev    │
└──────────────────┘                └──────────────────┘
          │                                      │
          └──────────┬───────────────────────────┘
                     ▼
            shared/ (共享目录)
         ├── tasks/       ← 任务下发
         ├── results/     ← 结果提交
         └── knowledge/   ← 共享知识库
```

---

## 二、Agent 详细说明

> 每个Agent的完整提示词见 [agent-config-scheme.md](agent-config-scheme.md) 对应章节。
> 以下为定位、职责摘要和协作关系分析。

---

### 2.1 Main Agent（小朱桑）

**完整提示词**：→ [agent-config-scheme.md §一](agent-config-scheme.md)

| 项 | 值 |
|---|---|
| 定位 | 唯一用户入口、任务路由器、轻量兜底处理器 |
| 模型 | kimi/k2p5 |
| 可调度 | research, dev |
| 工具 | 全部可用 |
| Workspace | workspace/ |

**提示词核心要点**：
- 人格：小朱桑，朱桑的数字分身，记性好、逻辑好、爱学习、严谨乐观
- 路由规则：调研 → research，开发 → dev，❌ 不自己深度搜索或写代码
- 开发分派纪律：只传需求描述，不指定 dev-lead 步骤，不直接 spawn dev-coder
- 心跳：白天主动检查，深夜静默

---

### 2.2 Research Lead（研究主管）

**完整提示词**：→ [agent-config-scheme.md §二](agent-config-scheme.md)

| 项 | 值 |
|---|---|
| 定位 | 研究团队调度员，拆解任务、调度子 Agent、审核整合结果 |
| 模型 | kimi/k2p5 |
| 可调度 | search, reviewer, citation |
| 工具 | web_search, web_fetch, browser, read, write, sessions_spawn, webSearchPrime |
| 禁止 | exec, process, edit |
| Workspace | workspace-research/ |

**提示词核心要点**：
- 6 Phase 工作流：规划 → 并行搜索 → 收集去重 → 双 Reviewer → 迭代判断 → 收敛
- 搜索策略：webSearchPrime（首选）→ browser（深度）→ web_search（兜底）
- 迭代最多 3 轮，avg ≥ 7 才收敛
- 最终报告写入 shared/results/，返回主 Agent 的是 3-5 句摘要

---

### 2.3 Search Agent（搜索研究员）

**完整提示词**：→ [agent-config-scheme.md §三](agent-config-scheme.md)

| 项 | 值 |
|---|---|
| 定位 | 纯信息收集者，不做判断 |
| 模型 | kimi/k2p5 |
| 可调度 | 无（叶子节点） |
| 工具 | web_search, web_fetch, browser, read, write, webSearchPrime |
| 禁止 | exec, process, sessions_spawn |
| Workspace | workspace-search/ |

**提示词核心要点**：
- 严格 JSON 输出，JSON 外不输出任何文字
- 最多 15 次工具调用，连续 2 次无新结果停止
- 置信度分级：high/medium/low，有明确标准

---

### 2.4 Reviewer Agent（审核员）

**完整提示词**：→ [agent-config-scheme.md §四](agent-config-scheme.md)

| 项 | 值 |
|---|---|
| 定位 | 独立质量审核者，不做搜索不做报告 |
| 模型 | kimi/k2p5 |
| 可调度 | 无（叶子节点） |
| 工具 | read, write（最受限） |
| 禁止 | exec, web_search, web_fetch, browser, sessions_spawn |
| Workspace | workspace-reviewer/ |

**提示词核心要点**：
- 两种模式：accuracy（0-10分/条）和 completeness（四维度评分）
- 交叉验证规则：2+ 独立来源不限分，仅厂商自报上限 5 分
- 防锚定效应，通过阈值 ≥ 7

---

### 2.5 Citation Agent（引用处理员）

**完整提示词**：→ [agent-config-scheme.md §五](agent-config-scheme.md)

| 项 | 值 |
|---|---|
| 定位 | 引用标准化 + URL 可访问性验证 |
| 模型 | kimi/k2p5 |
| 可调度 | 无（叶子节点） |
| 工具 | web_fetch, read, write |
| 禁止 | exec, web_search, browser, sessions_spawn |
| Workspace | workspace-citation/ |

**提示词核心要点**：
- 去重 → 验证（web_fetch）→ 标准化 → 分类
- 输出 broken_links + stats
- source_type 五类：paper/official_doc/tech_blog/news/other

---

### 2.6 Dev Lead（开发主管）— 新版 v3

**完整提示词**：→ [agent-config-scheme.md §六](agent-config-scheme.md)

| 项 | 值 |
|---|---|
| 定位 | 质量总负责，调度开发流程，确保 Clean State |
| 模型 | kimi/k2p5 |
| 可调度 | dev-init, dev-coder, dev-qa |
| 工具 | read, write, web_fetch, sessions_spawn, exec（受限） |
| 禁止 | process, web_search, browser, cron |
| Workspace | workspace-dev/ |

**提示词核心要点**：
- 三层质量门禁：Gate 1（功能验证）→ Gate 2（回归验证）→ Gate 3（Clean State）
- PRODUCT.md 作为单一真相源，与代码和 feature_list.json 三方同步
- 四种 Flow：A（初始化）/ B（开发循环）/ C（ACP Harness）/ D（烂摊子修复）
- Git 纪律：改代码前必须 checkpoint commit
- 单功能最多 3 轮迭代

---

### 2.7 Dev Init（初始化 Agent）— 新版 v3

**完整提示词**：→ [agent-config-scheme.md §七](agent-config-scheme.md)

| 项 | 值 |
|---|---|
| 定位 | 产品经理 + 测试架构师，只运行一次 |
| 模型 | kimi/k2p5 |
| 可调度 | 无（叶子节点） |
| 工具 | exec, read, write |
| 禁止 | web_search, browser, sessions_spawn |
| Workspace | workspace-dev/（共享） |

**提示词核心要点**：
- 创建 PRODUCT.md + feature_list.json + init.sh + progress.md
- feature_list.json 不可删除/编辑已有 feature（CATASTROPHIC 约束）
- steps 用 BDD 风格业务语言，至少 25% 的 feature 有 8+ steps
- 含完整 init.sh 模板（环境检查 → lint → 启动服务 → 冒烟测试）

---

### 2.8 Dev Coder（编码 Agent）— 新版 v3

**完整提示词**：→ [agent-config-scheme.md §八](agent-config-scheme.md)

| 项 | 值 |
|---|---|
| 定位 | 纯编码执行者，每 session 一个 feature |
| 模型 | kimi/k2p5 |
| 可调度 | 无（叶子节点） |
| 工具 | exec, read, write, process |
| 禁止 | web_search, browser, sessions_spawn |
| Workspace | workspace-dev/（共享） |

**提示词核心要点**：
- 启动流程：pwd → progress.md → git log → feature_list.json → init.sh
- 改代码前 checkpoint commit，改完后正式 commit
- **绝对不能**修改 feature_list.json（CATASTROPHIC）
- 代码质量标准：正确性 > 优雅性，函数不超过 50 行
- 安全约束白名单：ls, cat, grep, npm, node, git, init.sh, python3
- "You have unlimited time" — Anthropic Coding Prompt 原则

---

### 2.9 Dev QA（质量验证 Agent）— 新版 v3

**完整提示词**：→ [agent-config-scheme.md §九](agent-config-scheme.md)

| 项 | 值 |
|---|---|
| 定位 | 独立 QA 把关者，唯一可改 passes 字段 |
| 模型 | kimi/k2p5 |
| 可调度 | 无（叶子节点） |
| 工具 | exec, read, write, browser |
| 禁止 | web_search, sessions_spawn |
| Workspace | workspace-dev/（共享） |

**提示词核心要点**：
- v3 验证策略：curl + dump-dom + browser snapshot + 代码审查（禁止 screenshot）
- 验证闭环（Devin Autofix）：dev-coder 必须自验证，QA 独立复现
- "不能只审查代码就说通过"重复强调 4 次（铁律）
- 双 Gate：Feature Gate（逐 step）+ Regression Gate（分级策略）

---

## 三、两团队协作机制对比分析

### 3.1 协作模式总览

| 维度 | Research Team | Dev Team |
|------|--------------|----------|
| **协作模式** | 探索-验证-收敛循环 | 流水线+门禁 |
| **Lead 角色** | 规划者 + 数据整合者 | 调度者 + 质量守门人 |
| **子 Agent 关系** | 并行（多 search 同时跑） | 串行（init→coder→qa 依次） |
| **迭代机制** | 全局迭代（最多 3 轮，基于质量分） | 单功能迭代（最多 3 轮，基于 QA 结果） |
| **质量保障** | 双 Reviewer（accuracy + completeness） | 双 Gate（Feature + Regression） |
| **输出物** | 研究报告（shared/results/） | 代码 + feature_list.json + PRODUCT.md |
| **跨团队通信** | 无（research 不 spawn dev） | dev-lead 可 spawn research（⚠️ 配置矛盾） |

### 3.2 Research Team：探索-验证-收敛

```
需求 ──→ Phase 1: 规划（拆子问题）
            │
            ▼
       Phase 2: 并行搜索 ──┬── Search A ──→ findings A
                          ├── Search B ──→ findings B
                          └── Search C ──→ findings C
            │
            ▼
       Phase 3: 去重 + 知识库更新
            │
            ▼
       Phase 4: 双 Reviewer ──┬── Reviewer A（accuracy）──→ 逐条评分
                              └── Reviewer B（completeness）──→ 维度评分
            │
            ▼
       Phase 5: 质量判断 ──→ avg ≥ 7? ──是──→ Phase 6
                              │
                             否
                              │
                          回到 Phase 2（最多 3 轮）
            │
            ▼
       Phase 6: 收敛 ──→ Citation 验证 ──→ 写报告 ──→ 返回摘要
```

**协作特点**：
1. **扇出-聚合模式**：Lead 扇出 N 个 Search Agent，聚合后扇出 2 个 Reviewer
2. **数据驱动迭代**：基于数值化质量评分（0-10）决定是否继续
3. **做审完全分离**：Search 不做判断，Reviewer 不做搜索，Citation 不做判断
4. **知识累积**：每次迭代更新 knowledge-base.json 和 gaps.json
5. **子 Agent 间无通信**：Search A 和 Search B 不知道彼此存在，由 Lead 整合

### 3.3 Dev Team：流水线+门禁

```
需求 ──→ Flow A（dev-init）──→ PRODUCT.md + feature_list.json + init.sh + progress.md
                                    │
                                    ▼
                              ┌─── Flow B 循环 ───┐
                              │                    │
                              │  读 feature_list   │
                              │  找 passes=false   │
                              │       │            │
                              │       ▼            │
                              │  dev-coder 实现    │
                              │       │            │
                              │       ▼            │
                              │  dev-qa Gate 1     │
                              │       │            │
                              │    PASS/FAIL       │
                              │    │      │        │
                              │    │   needs-fix   │
                              │    │      │        │
                              │    │   ←──┘(≤3轮)  │
                              │    │               │
                              │    ▼               │
                              │  dev-qa Gate 2     │
                              │       │            │
                              │    PASS/FAIL       │
                              │    │      │        │
                              │  verified regression-fail ──→ 优先修复
                              │    │               │
                              │    ▼               │
                              │  更新 PRODUCT.md   │
                              │       │            │
                              │  下一个功能 ───────┘
                              │
                              └─── Gate 3（Clean State）──→ 结束
```

**协作特点**：
1. **串行流水线**：init → coder → qa，严格顺序，不可跳步
2. **门禁驱动**：每步有明确的 pass/fail 判定，失败回退有明确动作
3. **做审完全分离**：coder 不能改 passes，qa 不能写业务代码
4. **状态机管理**：feature_list.json 是全局状态机（pending → needs-fix → verified）
5. **Git 纪律**：每次改动前 checkpoint，确保可回滚

### 3.4 协作机制深度对比

| 分析维度 | Research | Dev | 评价 |
|---------|----------|-----|------|
| **并行度** | 高（Search 并行） | 低（Coder 串行） | ✅ 合理：搜索天然可并行，编码有依赖 |
| **迭代粒度** | 全局（整批 findings） | 单功能（逐个 feature） | ✅ 合理：研究需要全局视角，开发需要逐个稳定 |
| **迭代上限** | 3 轮全局 | 3 轮/功能 | ✅ 合理：防止无限循环 |
| **失败处理** | major_revision（重新规划） | needs-fix → 回退 coder | ✅ 两级处理 |
| **状态持久化** | knowledge-base.json + gaps.json | feature_list.json + progress.md + PRODUCT.md | ✅ 都有结构化状态 |
| **质量度量** | 数值评分（0-10）+ 阈值 | 二值（pass/fail） | 🟡 Research 更精细，Dev 更实用 |
| **Lead 权限** | 只调度，不做搜索 | 只调度 + exec 做 Gate 3 检查 | ⚠️ Dev Lead 混合了调度和执行 |
| **子 Agent 独立性** | 完全独立（不同 workspace） | 共享 workspace | ⚠️ 共享 workspace 有并发风险 |
| **跨团队协作** | 无 | 可 spawn research | ❌ 实际不可行（工具集不兼容） |

---

## 四、提示词匹配情况分析

### 4.1 版本对照表

| Agent | 实际加载（workspace） | 更新版（agents/xxx/agent/） | 是否一致 |
|-------|---------------------|---------------------------|---------|
| main | workspace/AGENTS.md + SOUL.md | — | ✅ 唯一版本 |
| research | workspace-research/AGENTS.md + SOUL.md | 无 AGENTS.md | 🟡 SOUL.md 搜索策略旧版 |
| search | workspace-search/AGENTS.md + SOUL.md | 无 AGENTS.md | ✅ 一致 |
| reviewer | workspace-reviewer/AGENTS.md + SOUL.md | 无 AGENTS.md | ✅ 一致 |
| citation | workspace-citation/AGENTS.md + SOUL.md | 无 AGENTS.md | ✅ 一致 |
| dev | **workspace-dev/AGENTS.md（默认模板！）** | agents/dev/agent/AGENTS.md（v3 新版） | 🔴 **严重不一致** |
| dev-init | workspace-dev/AGENTS.md（共享默认模板） | agents/dev-init/agent/AGENTS.md（v3 新版） | 🔴 **严重不一致** |
| dev-coder | workspace-dev/AGENTS.md（共享默认模板） | agents/dev-coder/agent/AGENTS.md（v3 新版） | 🔴 **严重不一致** |
| dev-qa | workspace-dev/AGENTS.md（共享默认模板） | agents/dev-qa/agent/AGENTS.md（v3 新版） | 🔴 **严重不一致** |

### 4.2 提示词与工具配置匹配度

| Agent | 提示词要求 | 工具配置 | 匹配度 |
|-------|----------|---------|--------|
| main | 全工具 | 全工具 | ✅ 100% |
| research | 不执行命令 | deny exec, process, edit | ✅ 100% |
| search | 不执行命令，不 spawn | deny exec, process, sessions_spawn | ✅ 100% |
| reviewer | 只读+写（纯判断） | allow read, write | ✅ 100% |
| citation | web_fetch 验证 URL | allow web_fetch, read, write | ✅ 100% |
| dev | "exec 仅用于 git/init.sh Gate 3" | **deny exec** | 🔴 **0%** |
| dev | 不搜索互联网 | deny web_search | ✅ |
| dev | 可 spawn research | allowAgents 含 research | 🟡 可行但工具不兼容 |
| dev-init | mkdir, cp, chmod, npm, node, git | allow exec, read, write | ✅ |
| dev-coder | ls, cat, grep, npm, node, git, init.sh | allow exec, read, write, process | ✅ |
| dev-qa | curl, dump-dom, browser snapshot | allow exec, read, write, browser | ✅ |

### 4.3 提示词内部一致性

| Agent | 检查项 | 结果 |
|-------|--------|------|
| research | AGENTS.md 搜索策略 vs SOUL.md 搜索策略 | 🟡 不一致：AGENTS.md 是 webSearchPrime 优先，SOUL.md 是 web_search 优先 |
| dev | Gate 3 要求 exec vs 工具 deny exec | 🔴 矛盾 |
| dev | "不直接写业务代码" vs "exec 仅用于 git/init.sh" | ✅ 不矛盾（exec 用于检查非编码） |
| dev-qa | 禁止 browser screenshot vs allow browser | ✅ 不矛盾（browser ≠ screenshot） |
| dev-coder | "不修改 feature_list.json" vs 无 edit 工具 | ✅ 一致（连工具都没有） |
| dev-qa | "唯一可改 passes" vs allow exec + write | ✅ 一致（write 可覆盖文件） |
| dev-init | "只运行一次" | ⚠️ 无机制强制，靠提示词约束 |

### 4.4 协作流程完整性

| 流程 | 涉及 Agent | 提示词覆盖 | 配置支持 | 完整度 |
|------|-----------|-----------|---------|--------|
| 研究任务端到端 | main → research → search → reviewer → citation | ✅ 全覆盖 | ✅ | 95%（SOUL.md 搜索策略需同步） |
| 新项目初始化 | main → dev → dev-init | ✅ 全覆盖 | ✅ | 90%（dev-init AGENTS.md 未被加载） |
| 功能开发循环 | dev → dev-coder → dev-qa | ✅ 全覆盖 | 🟡 | 70%（dev-lead 缺 exec，AGENTS.md 未被加载） |
| QA 回归测试 | dev-qa | ✅ 全覆盖 | ✅ | 90%（AGENTS.md 未被加载） |
| Clean State Gate | dev-lead | 🟡 提示词要求 exec | 🔴 exec 被 deny | 0%（**不可执行**） |
| 跨团队协作 | dev → research | 🟡 dev 可 spawn research | 🔴 工具集不兼容 | 0%（**不可行**） |

---

## 五、结论与建议

### 5.1 核心结论

1. **提示词设计质量很高**：两个团队的提示词都体现了成熟的工程实践（做审分离、门禁、迭代限制、结构化状态），借鉴了 Anthropic、OpenAI、Devin 的最佳实践。

2. **但实际运行时大部分 Dev 提示词未生效**：workspace-dev/AGENTS.md 是 OpenClaw 默认模板，Dev 团队的 v3 Quality-First 提示词（含 PRODUCT.md、Git 纪律、验证闭环等核心设计）从未被加载。**这是当前最严重的问题。**

3. **研究团队协作机制更成熟**：探索-验证-收敛循环设计合理，并行搜索 + 双 Reviewer + Citation 验证，数据驱动迭代，提示词和配置基本匹配。

4. **开发团队协作机制设计合理但无法落地**：流水线 + 门禁 + 状态机设计很好，但 dev-lead 缺 exec（Gate 3 不可执行）、AGENTS.md 未加载（所有流程规范未生效）。

5. **两团队协作模式差异合理**：Research 用扇出-聚合（适合信息收集），Dev 用串行流水线（适合代码开发）。但跨团队通道（dev → research）在当前配置下不可行。

### 5.2 优先修复建议（按紧急度排序）

#### P0 — 立即修复（影响基本可用性）

**1. 同步 Dev 团队提示词到 workspace-dev/**
```
agents/dev/agent/AGENTS.md    → workspace-dev/AGENTS.md
agents/dev-coder/agent/AGENTS.md → workspace-dev/AGENTS.md（需合并或独立）
agents/dev-qa/agent/AGENTS.md    → workspace-dev/AGENTS.md（需合并或独立）
agents/dev-init/agent/AGENTS.md  → workspace-dev/AGENTS.md（需合并或独立）
```
> ⚠️ 注意：Dev 团队 4 个 agent 共享 workspace-dev，AGENTS.md 只能有一份。需要确认 OpenClaw 是否支持 `agentDir` 覆盖 workspace 下的 AGENTS.md。如果支持，则不需要合并，只需修正 agentDir 路径。

**2. 给 dev-lead 添加受限 exec**
```json
// openclaw.json → agents.list → dev
"tools": {
  "allow": ["read", "write", "web_fetch", "sessions_spawn", "subagents",
            "sessions_list", "sessions_history", "exec"],
  "deny": ["process", "web_search", "web_search_prime", "browser", "cron"]
}
```

**3. 修正 dev 的 agentDir**
```json
"agentDir": "/home/noname/.openclaw/agents/dev/agent"  // 加 /agent 后缀
```

#### P1 — 短期修复（影响质量）

**4. 同步 Research SOUL.md 搜索策略**
将 SOUL.md 中的搜索策略从 `web_search → browser` 更新为 `webSearchPrime → browser → web_search`，与 AGENTS.md 保持一致。

**5. 移除 dev 的 research 子 agent 权限**
```json
"subagents": { "allowAgents": ["dev-init", "dev-coder", "dev-qa"] }
// 移除 "research"
```

**6. 清理冗余 fallbacks**
```json
// 所有 dev 系列，将 fallbacks 改为 [] 或不同模型
"fallbacks": []
```

#### P2 — 中期优化

**7. 定制 workspace-dev SOUL.md**
当前使用 OpenClaw 默认模板，建议设置开发团队人格（区别于 main agent 的小朱桑人格）。

**8. 定制 search/reviewer/citation TOOLS.md**
当前全部使用默认模板，可添加团队专属备忘（如 research 常用搜索模板）。

**9. 清理历史文件**
workspace 下的草稿文件（multi-team-feasibility.md 等 10+ 个）可归档到 `workspace/archive/`。

**10. 删除 codex 空壳目录**
`agents/codex/` 无配置无提示词，可安全删除。

### 5.3 待确认事项

1. **OpenClaw 加载优先级**：当 `agentDir` 和 workspace 下都有 AGENTS.md 时，哪个优先？需要确认后决定是同步文件还是修正 agentDir。
2. **Dev 团队共享 workspace 的 AGENTS.md 问题**：4 个 agent 共享一个 AGENTS.md 文件，但它们各自有不同的提示词。如果 OpenClaw 不支持 per-agent AGENTS.md 覆盖，需要重新设计文件组织方式。
3. **browser 与 exec 的依赖关系**：search 和 research allow 了 browser 但 deny exec，需确认 browser 工具是否内部需要 exec。

---

## 六、工具权限矩阵

| 工具 | main | research | search | reviewer | citation | dev | dev-init | dev-coder | dev-qa |
|------|------|----------|--------|----------|----------|-----|----------|-----------|--------|
| read | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| write | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| edit | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ |
| exec | ✅ | ❌ | ❌ | ❌ | ❌ | ⚠️建议加 | ✅ | ✅ | ✅ |
| process | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ✅ | ❌ |
| web_search | ✅ | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ |
| web_fetch | ✅ | ✅ | ✅ | ❌ | ✅ | ✅ | ❌ | ❌ | ❌ |
| browser | ✅ | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ | ✅ |
| sessions_spawn | ✅ | ✅ | ❌ | ❌ | ❌ | ✅ | ❌ | ❌ | ❌ |
| webSearchPrime | ✅ | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ |

## 七、模型配置矩阵

| Agent | Primary | Fallbacks | 建议 |
|-------|---------|-----------|------|
| main | kimi/k2p5 | 全局默认 | ✅ |
| research | kimi/k2p5 | 全局默认 | ✅ |
| search | kimi/k2p5 | 全局默认 | ✅ |
| reviewer | kimi/k2p5 | 全局默认 | 🟡 可用更强模型 |
| citation | kimi/k2p5 | 全局默认 | ✅ |
| dev | kimi/k2p5 | kimi/k2p5（冗余） | 清理 |
| dev-init | kimi/k2p5 | kimi/k2p5（冗余） | 清理 |
| dev-coder | kimi/k2p5 | kimi/k2p5（冗余） | 清理 |
| dev-qa | kimi/k2p5 | kimi/k2p5（冗余） | 清理 |

**全局默认**（agents.defaults）：primary zai/glm-5，fallbacks kimi/k2p5 / zai/glm-5-turbo / zai/glm-5.1 / zai/glm-5v-turbo，subagents model kimi/k2p5，maxConcurrent 8，maxSpawnDepth 3

---

*报告完毕。完整提示词配置方案见 [agent-config-scheme.md](agent-config-scheme.md)。*
