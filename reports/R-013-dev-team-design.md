# R-013：OpenClaw Dev Team 架构设计方案

> 生成时间：2026-03-29 | 方法：v4 深度研究流程（3 Search Agent + 双 Reviewer + 1 轮迭代）
> 研究基础：R-007（Claude Code 技能调研）、R-011（单体 dev agent 设计）、R-006（OpenClaw vs Claude Code 对比）

---

## 一、架构总览

### 1.1 设计哲学

**Humans steer. Agents execute.** — 借鉴 OpenAI Harness Engineering 实践 [1]，开发团队的核心理念是：

- **Dev Lead** 是人类工程师的 agent 化身，负责「做什么」（任务拆解、代码审查、质量把关）
- **Harness Worker** 是外部编码 agent（Claude Code/Codex），负责「怎么做」（实现、测试、调试）
- **Dev Lead 不写代码**，正如 Research Lead 不做搜索

### 1.2 架构图

```
                            ┌──────────────────┐
                            │    main agent     │
                            │   (全局路由调度)   │
                            └────────┬─────────┘
                    ┌────────────────┼────────────────┐
                    ▼                                 ▼
         ┌──────────────────┐              ┌──────────────────┐
         │  research (lead) │              │   dev (lead)     │
         │  (调研编排)       │              │  (开发编排)       │
         └────────┬─────────┘              └────────┬─────────┘
                  │                                  │
       ┌──────────┼──────────┐           ┌───────────┼───────────┐
       ▼          ▼          ▼           ▼                       ▼
  ┌────────┐┌────────┐┌────────┐  ┌─────────────┐      ┌──────────────┐
  │ search ││reviewer││citation│  │  dev-coder   │      │ dev-harness  │
  │(搜索)  ││(审核)  ││(引用)  │  │ (轻量编码)   │      │(ACP harness) │
  └────────┘└────────┘└────────┘  │  OpenClaw    │      │ Claude Code  │
                                    │  原生 agent  │      │ /Codex/Gemini│
                                    └─────────────┘      └──────┬───────┘
                                                               │
                                                    ┌──────────┼──────────┐
                                                    ▼          ▼          ▼
                                               ┌────────┐┌────────┐┌────────┐
                                               │claude  ││ codex  ││ gemini │
                                               │ code   ││        ││  CLI   │
                                               └────────┘└────────┘└────────┘
```

### 1.3 角色一览

| 角色 | 类型 | agentId | 职责 | Runtime |
|------|------|---------|------|---------|
| **dev-lead** | OpenClaw agent | `dev` | 任务拆解、代码审查、质量把关、协作编排 | subagent |
| **dev-coder** | OpenClaw agent | `dev-coder` | 轻量编码（脚本、配置、小修改） | subagent |
| **dev-harness** | ACP bridge | `dev-harness` | 重量编码（通过 ACP 调用外部 harness） | acp |

> **设计决策**：不设独立的 dev-tester agent。测试由 dev-coder 或 harness worker 直接执行，dev-lead 审查结果。理由：Azure 建议「只在单 agent 不可靠时才引入 multi-agent」[11]，测试执行可靠性高，不需要独立 agent。

---

## 二、各 Agent 角色详细设计

### 2.1 dev-lead（开发编排者）

**定位**：借鉴 Anthropic 的 planner 角色 [7] 和 OpenAI 的「人类工程师」角色 [2]。不写代码，只做决策和审查。

**核心职责**：
1. 接收开发任务，拆解为子任务
2. 判断子任务复杂度，路由到 dev-coder 或 dev-harness
3. 审查代码变更（diff review），决定是否需要迭代
4. 管理 ACP session 生命周期
5. 与 research team 协作（需要技术调研时 spawn research agent）

**工具权限**：
```
tools.allow: ["read", "write", "web_fetch", "sessions_spawn", "subagents"]
tools.deny:  ["exec", "process", "web_search", "web_search_prime", "browser", "cron"]
```

> dev-lead 不直接执行命令（不做 exec），所有执行委派给 dev-coder 或 dev-harness。

**subagents 配置**：
```json5
subagents: {
  allowAgents: ["dev-coder", "dev-harness", "research"]
}
```

**AGENTS.md**：

```markdown
# Dev Lead（开发编排者）— v1

你是 Dev Lead，负责 orchestrating 开发任务。你是调度员，不是执行者。

## 绝对不做什么
- ❌ 不自己写代码（交给 dev-coder 或 dev-harness）
- ❌ 不自己执行命令（没有 exec 权限）
- ❌ 不自己做调研（交给 research agent）

## 工作流程

### Step 1：任务分析
1. 读取任务描述，理解目标
2. 读取相关文件（read 工具），了解现有代码结构
3. 判断复杂度：
   - **简单**（改配置、写脚本、小 bug fix）→ spawn dev-coder
   - **中等**（新功能、重构、需要外部工具链）→ spawn dev-harness
   - **需要调研**（不确定技术方案）→ 先 spawn research agent

### Step 2：任务拆解
将任务拆成可独立完成的子任务，每个子任务包含：
- 目标（做什么）
- 约束（不做什么）
- 验收标准（怎么算完成）
- 上下文（相关文件路径、已有代码片段）

### Step 3：执行与审查
1. spawn 执行 agent（dev-coder 或 dev-harness）
2. 等待结果
3. 审查代码变更（read diff 文件或 output）
4. 如质量不达标，给出具体修改意见，重新 spawn（最多 3 轮）

### Step 4：验证
1. spawn dev-coder 运行测试和 lint
2. 检查结果
3. 如测试失败，分析原因，决定是修复还是回退

### Step 5：交付
- 总结：做了什么、改了哪些文件
- 测试结果
- 已知问题和后续建议

## ACP Harness 使用指南

### 何时用 dev-harness（ACP）
- 需要完整的 Claude Code / Codex 体验（IDE 级别的代码理解）
- 大规模重构（涉及 10+ 文件）
- 需要 agent 自主探索代码库的任务
- 预计执行时间 > 10 分钟的任务

### spawn 模式选择
- **mode: "run"**（one-shot）：独立子任务，完成后自动关闭
- **mode: "session" + thread: true**：需要多轮交互的复杂任务

### 超时设置
- 简单任务：runTimeoutSeconds: 300（5 分钟）
- 中等任务：runTimeoutSeconds: 600（10 分钟）
- 复杂任务：runTimeoutSeconds: 900（15 分钟，参考 issue #38419）
- 超长任务：使用 fire-and-forget + 轮询结果

### Harness 不可用时回退
1. 检测 ACP 错误（AcpRuntimeError）
2. 回退到 dev-coder（纯 OpenClaw agent）
3. 在结果中标注「harness 不可用，使用降级方案」

## 代码审查标准
- 代码是否符合项目风格
- 是否有明显的 bug 或安全漏洞
- 是否有适当的错误处理
- 是否有必要的测试
- 是否遵循 AGENTS.md 中的红线规则

## 红线
- 不直接执行任何命令
- 不跳过代码审查直接交付
- 最多迭代 3 轮（防止无限循环 [33]）
- 不确定的技术决策标注 [NEEDS_REVIEW]
```

---

### 2.2 dev-coder（轻量编码者）

**定位**：纯 OpenClaw 原生 agent，执行轻量编码任务。继承 R-011 中的单体 dev agent 设计，但定位更窄——只做 dev-lead 委派的子任务。

**核心职责**：
1. 执行代码编写（脚本、配置、小修改）
2. 运行测试、lint、构建
3. Git 操作
4. 读取和分析代码

**工具权限**：
```
tools.allow: ["exec", "read", "write", "edit", "web_fetch", "process"]
tools.deny:  ["web_search", "web_search_prime", "browser", "sessions_spawn", "cron", "subagents"]
```

**AGENTS.md**：

```markdown
# Dev Coder（轻量编码者）— v1

你是 Dev Coder，执行 dev-lead 委派的编码子任务。你是执行者，不是决策者。

## 职责
- 代码编写（Python、Shell、Node.js、TypeScript 等）
- 代码编辑和重构
- 测试编写和执行
- Git 操作
- 依赖管理

## 不做什么
- ❌ 不搜索互联网
- ❌ 不创建子 agent
- ❌ 不做架构决策（由 dev-lead 决定）
- ❌ 不做定时任务

## 工作流程
1. 阅读任务描述和约束
2. 读取相关现有文件
3. 实现（遵循项目代码风格）
4. 自测（运行测试、lint）
5. 输出结构化结果

## 代码质量标准
- 正确性 > 优雅性
- 可读性 > 简洁性
- 不引入新依赖（除非明确要求）
- 函数不超过 50 行
- 错误处理不忽略异常

## 安全红线
- ❌ 绝不执行 rm -rf / 或等效危险命令
- ❌ 绝不修改系统级配置文件
- ❌ 绝不在代码中嵌入密钥或凭证
- ❌ 绝不使用 sudo

## 输出格式
```
## 完成摘要
- 任务：<描述>
- 状态：<已完成 | 部分完成 | 失败>
- 修改文件：<列表>

## 验证结果
- <测试>：<通过/失败>

## 问题
- <如有>
```
```

---

### 2.3 dev-harness（ACP Harness Bridge）

**定位**：不是独立 agent，而是 dev-lead 通过 `sessions_spawn({ runtime: "acp" })` 创建的 ACP session。dev-lead 充当 bridge，将子任务翻译为 ACP prompt。

**支持的 harness**（通过 acpx 后端）[ACP docs]：
- `claude` — Claude Code
- `codex` — OpenAI Codex
- `gemini` — Gemini CLI
- `opencode` — OpenCode
- `pi` — Pi
- `kimi` — Kimi

**spawn 模式**：

```javascript
// 简单子任务（one-shot）
sessions_spawn({
  runtime: "acp",
  agentId: "claude",
  mode: "run",
  runTimeoutSeconds: 600,
  task: "在 src/utils.ts 中添加一个 debounce 函数...",
  cwd: "/path/to/project"
})

// 复杂子任务（persistent + thread）
sessions_spawn({
  runtime: "acp",
  agentId: "codex",
  mode: "session",
  thread: true,
  runTimeoutSeconds: 900,
  task: "重构 src/api/ 目录，将 REST 端点迁移到 tRPC...",
  cwd: "/path/to/project"
})

// 恢复之前的 session
sessions_spawn({
  runtime: "acp",
  agentId: "claude",
  resumeSessionId: "<previous-session-id>",
  task: "继续之前的工作，修复剩余的测试失败..."
})
```

**ACP 生命周期管理**：

| 阶段 | 操作 | 超时 |
|------|------|------|
| 创建 | `sessions_spawn({ runtime: "acp" })` | runTimeoutSeconds |
| 监控 | 结果 auto-announce（push-based） | — |
| 续命 | `resumeSessionId` 恢复 | 新的 timeout |
| 取消 | `/acp cancel <session-key>` | — |
| 关闭 | `/acp close <session-key>` | — |

**权限配置**（openclaw.json）：
```json5
plugins: {
  entries: {
    acpx: {
      enabled: true,
      config: {
        permissionMode: "approve-all",      // ACP session 无 TTY，需自动审批
        nonInteractivePermissions: "deny"   // 权限不足时降级而非崩溃
      }
    }
  }
}
```

---

## 三、与 Research Team 的协作流程

### 3.1 交互模式

```
用户: "帮我实现一个 OAuth2 登录功能"

main agent → 判断需要开发 + 调研
  ├─ spawn research agent → 调研 OAuth2 最佳实践、库选择
  │   └─ 返回调研报告（R-xxx-report.md）
  │
  └─ spawn dev-lead（传入调研结果）
       ├─ 基于调研结果拆解任务
       ├─ spawn dev-harness(claude) → 实现核心逻辑
       ├─ spawn dev-coder → 写测试 + 配置
       └─ 审查 + 验证 → 交付
```

### 3.2 dev-lead 调用 research agent

dev-lead 配置了 `subagents.allowAgents: ["dev-coder", "dev-harness", "research"]`，可直接 spawn research agent：

```javascript
// dev-lead 遇到技术不确定性时
sessions_spawn({
  agentId: "research",
  runTimeoutSeconds: 600,
  task: "调研主题：Node.js 中 tRPC vs GraphQL 的性能对比\n子问题：在高并发场景下哪种方案更适合？\n输出 JSON 格式..."
})
```

### 3.3 协作边界

| 场景 | 谁发起 | 谁执行 |
|------|--------|--------|
| 开发前技术调研 | dev-lead | research team |
| 开发中 API 文档查询 | dev-coder | 自己（web_fetch 已知 URL） |
| 代码审查发现设计问题 | dev-lead | 标注 [NEEDS_RESEARCH]，由 main agent 决定 |
| 开发完成后文档更新 | dev-lead | dev-coder |

---

## 四、失败处理与回退

### 4.1 分级服务能力 [29]

| 级别 | 能力 | 触发条件 |
|------|------|----------|
| **FULL** | dev-lead + dev-coder + ACP harness（全部） | 正常运行 |
| **DEGRADED** | dev-lead + dev-coder（无 harness） | ACP 不可用、acpx 插件故障 |
| **MINIMAL** | dev-lead 仅审查（无执行能力） | dev-coder 和 harness 均不可用 |
| **OFFLINE** | 返回错误，建议人工介入 | 所有 agent 不可用 |

### 4.2 错误传播防控

多 agent 系统的复合错误率极高（10 步流水线 95% 准确率时整体仅 59% [25]）。防控策略：

1. **闭环验证**：每步结果经 dev-lead 审查后再传递（可拦截 96.4% 错误 [27]）
2. **Context reset**：长任务中使用 context reset（清空 + 结构化 handoff）而非 compaction（原地摘要），避免自条件效应 [8][26]
3. **最大迭代次数**：代码审查最多 3 轮，防止无限循环 [33]

### 4.3 Evaluator-Reflect-Refine Loop [30]

代码质量不达标时的迭代机制：

```
dev-harness 产出代码
  → dev-lead 审查（evaluator）
    → 不达标：给出具体修改意见
      → dev-harness 修订（reflect-refine）
        → 重新审查
          → 达标 或 达到 3 轮上限
```

### 4.4 具体失败场景与处理

| 失败场景 | 检测方式 | 处理策略 |
|----------|----------|----------|
| ACP harness 超时 | runTimeoutSeconds 到期 | 检查是否有部分结果，用 resumeSessionId 续命或降级到 dev-coder |
| ACP 权限错误 | AcpRuntimeError | 调整 permissionMode 或降级 |
| dev-coder 执行失败 | exec 返回非零退出码 | 分析错误，重试一次或升级到 dev-harness |
| 代码审查不通过 | dev-lead 审查判断 | 给修改意见，最多 3 轮 |
| 所有 agent 不可用 | spawn 失败 | 返回 OFFLINE，建议人工介入 |
| harness 限流（429） | API 响应 | 指数退避重试 [32]，最多 3 次 |

---

## 五、openclaw.json 配置片段

```json5
{
  // === ACP 全局配置 ===
  acp: {
    enabled: true,
    dispatch: { enabled: true },
    backend: "acpx",
    defaultAgent: "claude",           // 默认使用 Claude Code
    allowedAgents: ["claude", "codex", "gemini", "opencode", "pi", "kimi"],
    maxConcurrentSessions: 8,
    stream: {
      coalesceIdleMs: 300,
      maxChunkChars: 1200,
    },
    runtime: {
      ttlMinutes: 120,                // ACP session 2 小时 TTL
    },
  },

  // === Agent 列表 ===
  agents: {
    list: [
      // --- Dev Lead ---
      {
        id: "dev",
        identity: "~/.openclaw/agents/dev/AGENTS.md",
        workspace: "~/.openclaw/workspace-dev",
        description: "开发编排者 - 任务拆解、代码审查、质量把关、ACP harness 调度",
        tools: {
          allow: ["read", "write", "web_fetch", "sessions_spawn", "subagents"],
          deny: ["exec", "process", "web_search", "web_search_prime", "browser", "cron"]
        },
        subagents: {
          allowAgents: ["dev-coder", "dev-harness", "research"]
        }
      },

      // --- Dev Coder ---
      {
        id: "dev-coder",
        identity: "~/.openclaw/agents/dev-coder/AGENTS.md",
        workspace: "~/.openclaw/workspace-dev",   // 共享 workspace
        description: "轻量编码者 - 代码编写、测试执行、Git 操作",
        tools: {
          allow: ["exec", "read", "write", "edit", "web_fetch", "process"],
          deny: ["web_search", "web_search_prime", "browser", "sessions_spawn", "cron", "subagents"]
        }
        // 叶节点，不配置 subagents
      },

      // --- Dev Harness（ACP bridge，不需要独立 AGENTS.md）---
      {
        id: "dev-harness",
        description: "ACP Harness Bridge - 由 dev-lead 通过 sessions_spawn 调用",
        runtime: {
          type: "acp",
          acp: {
            backend: "acpx",
            mode: "run",              // 默认 one-shot
            cwd: "~/.openclaw/workspace-dev"
          }
        }
      },

      // --- 更新 main agent ---
      // {
      //   id: "main",
      //   subagents: {
      //     allowAgents: ["research", "dev"]  // 新增 "dev"
      //   }
      // }
    ]
  },

  // === ACP 插件配置 ===
  plugins: {
    entries: {
      acpx: {
        enabled: true,
        config: {
          permissionMode: "approve-all",
          nonInteractivePermissions: "deny"
        }
      }
    }
  }
}
```

---

## 六、创建命令

```bash
# === 1. 创建目录结构 ===
mkdir -p ~/.openclaw/agents/dev
mkdir -p ~/.openclaw/agents/dev-coder
mkdir -p ~/.openclaw/workspace-dev
mkdir -p ~/.openclaw/workspace-dev/docs        # 知识库目录（参考 OpenAI 实践 [4]）

# === 2. 注册 agents ===

# Dev Lead
openclaw agents add dev \
  --identity ~/.openclaw/agents/dev/AGENTS.md \
  --workspace ~/.openclaw/workspace-dev \
  --description "开发编排者 - 任务拆解、代码审查、质量把关"

# Dev Coder
openclaw agents add dev-coder \
  --identity ~/.openclaw/agents/dev-coder/AGENTS.md \
  --workspace ~/.openclaw/workspace-dev \
  --description "轻量编码者 - 代码编写、测试执行、Git 操作"

# Dev Harness（ACP bridge）
openclaw agents add dev-harness \
  --description "ACP Harness Bridge - Claude Code/Codex/Gemini"

# === 3. 手动编辑 openclaw.json ===
# 添加上面的完整配置片段（tools, subagents, acp, plugins）
nano ~/.openclaw/openclaw.json

# === 4. 安装 ACP 插件 ===
openclaw plugins install acpx
openclaw config set plugins.entries.acpx.enabled true

# === 5. 验证 ===
openclaw gateway restart
/acp doctor                                    # 检查 ACP 后端健康
openclaw agents list                           # 确认 agents 注册成功

# === 6. 写入 AGENTS.md 文件 ===
# 将上面 2.1 和 2.2 的 AGENTS.md 内容分别写入对应目录
```

---

## 七、渐进式部署路线图

### Phase 0：准备（1-2 天）

- [x] 安装 acpx 插件
- [x] 配置 ACP 全局设置
- [ ] 创建 workspace 目录结构
- [ ] 写入 AGENTS.md 文件
- [ ] `/acp doctor` 确认 ACP 可用

### Phase 1：最小可用（3-5 天）

**目标**：dev-lead + dev-coder 组合，不依赖 ACP

- [ ] 注册 dev 和 dev-coder agent
- [ ] 更新 main agent 的 `subagents.allowAgents`
- [ ] 测试：让 main agent 将开发任务路由到 dev-lead
- [ ] 测试：dev-lead spawn dev-coder 执行简单编码任务
- [ ] 测试：dev-lead 审查 dev-coder 的输出并决定是否迭代

**验收标准**：能完成一个简单的编码任务（如写一个 Python 脚本）

### Phase 2：ACP 集成（1 周）

**目标**：接入 ACP harness，实现 FULL 级别服务

- [ ] 配置 acpx 插件（permissionMode, nonInteractivePermissions）
- [ ] 测试：dev-lead spawn ACP session（Claude Code）
- [ ] 测试：one-shot 模式（mode: "run"）
- [ ] 测试：超时处理（runTimeoutSeconds: 900）
- [ ] 测试：harness 不可用时降级到 dev-coder

**验收标准**：能通过 ACP harness 完成一个中等复杂度的编码任务

### Phase 3：多 Harness 并行（1 周）

**目标**：同时使用多个 harness，按任务特性选择

- [ ] 配置多个 harness（claude, codex, gemini）
- [ ] 实现智能路由（简单任务→codex，复杂→claude）
- [ ] 测试：并行 spawn 两个 harness 执行不同子任务
- [ ] 测试：resumeSessionId 续命
- [ ] 测试：thread-bound persistent session

**验收标准**：能并行执行 2 个子任务，结果正确

### Phase 4：Research 协作（3-5 天）

**目标**：dev-lead 与 research team 无缝协作

- [ ] 配置 dev-lead 的 `subagents.allowAgents` 包含 "research"
- [ ] 测试：dev-lead spawn research agent 获取技术背景
- [ ] 测试：dev-lead 将调研结果传递给 dev-harness
- [ ] 端到端测试：从调研到实现的完整流程

**验收标准**：能完成一个需要调研+实现的端到端任务

### Phase 5：生产化（持续）

- [ ] 监控指标：任务成功率、平均迭代次数、token 消耗
- [ ] 错误归因：实现简化版 CHIEF 因果图 [31]
- [ ] AGENTS.md 精简：控制在 100 行以内 [4]，详细知识库放 docs/
- [ ] Workspace 隔离：评估是否需要 git worktree 隔离 [5]
- [ ] 安全审计：验证权限边界和红线执行情况

---

## 八、知识缺口

1. **ACP steer 支持**：当前 ACP runtime sessions 不支持 subagents tool 的 steer [19]，无法在任务执行中途调整方向。需关注 issue #43496 的进展。
2. **ACP 默认超时**：没有配置级默认 runTimeoutSeconds，只能在每个 spawn 调用中传参 [20]。建议在 AGENTS.md 中硬编码推荐值。
3. **多 harness 文件冲突**：并行 harness 操作同一仓库时的文件锁机制未找到文档。
4. **Token 消耗预算**：多 agent 架构下的 token 消耗缺乏量化估算，建议 Phase 5 监控。
5. **acpx 并发限制**：8 session 并发限制是否可配置未确认。

---

## 九、来源列表

| # | 来源 | URL |
|---|------|-----|
| 1 | OpenAI Harness Engineering | https://openai.com/index/harness-engineering/ |
| 2 | OpenAI Harness Engineering（角色转变） | https://openai.com/index/harness-engineering/ |
| 3 | OpenAI Agent-to-Agent Review | https://openai.com/index/harness-engineering/ |
| 4 | OpenAI AGENTS.md 实践 | https://openai.com/index/harness-engineering/ |
| 5 | OpenAI Worktree 隔离 | https://openai.com/index/harness-engineering/ |
| 6 | OpenAI Observability Stack | https://openai.com/index/harness-engineering/ |
| 7 | Anthropic 三 Agent 架构 | https://www.anthropic.com/engineering/harness-design-long-running-apps |
| 8 | Anthropic Context Reset | https://www.anthropic.com/engineering/harness-design-long-running-apps |
| 9 | Claude Code Agent Teams | https://dev.to/uenyioha/porting-claude-codes-agent-teams-to-opencode-4hol |
| 10 | OpenCode JSONL Inbox | https://dev.to/uenyioha/porting-claude-codes-agent-teams-to-opencode-4hol |
| 11 | Azure 5 种编排模式 | https://learn.microsoft.com/en-us/azure/architecture/ai-ml/guide/ai-agent-design-patterns |
| 12 | Azure Multi-Agent 建议 | https://learn.microsoft.com/en-us/azure/architecture/ai-ml/guide/ai-agent-design-patterns |
| 13 | Orchestrator-Worker 模式 | https://arize.com/blog/orchestrator-worker-agents/ |
| 14 | Danau5tin Multi-Agent Coding | https://github.com/Danau5tin/multi-agent-coding-system |
| 15 | Gerred 层级架构 | https://gerred.github.io/building-an-agentic-system/second-edition/part-iv-advanced-patterns/chapter-10-multi-agent-orchestration.html |
| 17 | acpx 命名 Session | https://github.com/openclaw/acpx/blob/main/README.md |
| 18 | acpx 长任务管理 | https://github.com/openclaw/acpx/blob/main/README.md |
| 19 | ACP Steer 不支持 (Issue #43496) | https://github.com/openclaw/openclaw/issues/43496 |
| 20 | Codex 超时问题 (Issue #38419) | https://github.com/openclaw/openclaw/issues/38419 |
| 22 | sessions_send Fire-and-Forget | https://docs.openclaw.ai/concepts/session-tool |
| 23 | Simon Willison 并行 Agent 模式 | https://simonwillison.net/2025/Oct/5/parallel-coding-agents/ |
| 25 | 多 Agent 失败率 | https://www.zartis.com/the-compounding-errors-problem-why-multi-agent-systems-fail-and-the-architecture-that-fixes-it/ |
| 26 | 自条件效应 | https://www.zartis.com/the-compounding-errors-problem-why-multi-agent-systems-fail-and-the-architecture-that-fixes-it/ |
| 27 | 闭环验证 96.4% | https://www.zartis.com/the-compounding-errors-problem-why-multi-agent-systems-fail-and-the-architecture-that-fixes-it/ |
| 28 | Agent 协调成本指数增长 | https://galileo.ai/blog/why-multi-agent-systems-fail |
| 29 | 优雅降级分级 | https://docs.praison.ai/docs/best-practices/graceful-degradation |
| 30 | Evaluator Reflect-Refine Loop | https://docs.aws.amazon.com/prescriptive-guidance/latest/agentic-ai-patterns/evaluator-reflect-refine-loop-patterns.html |
| 31 | CHIEF 失败归因框架 | https://arxiv.org/html/2602.23701v1 |
| 33 | Agent 无限循环 | Medium (Komal Baparmar) |
| 34 | AWS 工具冗余 | https://aws.amazon.com/blogs/architecture/build-resilient-generative-ai-agents/ |
| ACP | OpenClaw ACP Agents 文档 | https://docs.openclaw.ai/tools/acp-agents |
| R-007 | Claude Code 技能调研 | /home/noname/.openclaw/workspace/shared/results/R-007-claude-code-skills-report.md |
| R-011 | 单体 Dev Agent 设计 | /home/noname/.openclaw/workspace/shared/results/R-011-dev-agent-design.md |
| R-006 | OpenClaw vs Claude Code 对比 | /home/noname/.openclaw/workspace/shared/results/R-006-dev-team-comparison.md |

---

## 十、方法论反思

**做得好的**：
- 3 个 Search Agent 覆盖了架构模式、ACP 集成、失败处理三个核心维度
- OpenAI Harness Engineering 博客提供了高价值的一手实践参考
- 双 Reviewer 机制发现了 Anthropic 三 agent 架构的表述不准确（原文主要是双 agent）
- 结合已有 R-007/R-011/R-006 成果填补了 Reviewer 指出的 gaps

**需要改进的**：
- 工具权限和配置的深度不足（依赖 R-011 的既有设计，未做独立搜索）
- ACP steer 不支持（issue #43496）是架构设计的硬约束，应该在 Phase 1 就考虑 workaround
- 缺少多 harness 并行的实际测试数据（所有建议基于文档和理论）
- Token 成本估算缺失（影响部署决策）
