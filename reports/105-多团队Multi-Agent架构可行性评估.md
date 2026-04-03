# 多团队 Multi-Agent 架构可行性评估

> 在 OpenClaw 上为不同团队（调研/开发/运营）配置独立的 multi-agent，
> 团队间隔离、文件交互、所有用户交互只通过主 Agent。
> 2026-03-28

---

## 结论：✅ 完全可行，OpenClaw 原生支持

OpenClaw 的多 agent 架构**天然支持**无名描述的所有需求，不需要 hack 或 workaround。

---

## 一、需求 vs OpenClaw 能力对照

| 需求 | OpenClaw 能力 | 状态 |
|------|-------------|------|
| 不同团队用不同 agent | `openclaw agents add` 创建隔离 agent，独立 workspace + session | ✅ 原生 |
| 每个 agent 可配不同模型 | `agents.list[].model` 覆盖全局默认 + sessions_spawn 的 model 参数 | ✅ 原生 |
| 团队间互相隔离 | 独立 workspace、session store、auth profiles、skills | ✅ 原生 |
| 通过文件交互 | 共享目录 + workspace 绝对路径访问 + JSON handoff 文件 | ✅ 原生 |
| 所有用户交互通过主 Agent | 主 Agent 是唯一入口，通过 sessions_spawn/sessions_send 调度子 agent | ✅ 原生 |
| 不同团队不同 tools | 8 层 tool policy 优先级，per-agent allow/deny | ✅ 原生 |
| 沙箱隔离（可选） | Docker/SSH/OpenShell，session/agent/shared 三级 scope | ✅ 原生 |
| 定时任务 per-team | Cron job 支持 isolated session，不同 agent 可配不同 HEARTBEAT.md | ✅ 原生 |

---

## 二、推荐架构

```
                        Telegram / Web UI
                              │
                              ▼
                    ┌─────────────────┐
                    │   Main Agent     │ ← 唯一用户入口
                    │   (主 Agent)      │    GLM-5.1
                    │   接收用户消息     │
                    │   理解意图        │
                    │   路由到对应团队   │
                    └──┬───┬───┬──────┘
                       │   │   │
            sessions_spawn / sessions_send
                       │   │   │
          ┌────────────┘   │   └────────────┐
          ▼                ▼                 ▼
┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│ Research Team│  │  Dev Team    │  │ Ops Team     │
│ (调研团队)   │  │ (开发团队)   │  │ (运营团队)   │
│              │  │              │  │              │
│ Model:       │  │ Model:       │  │ Model:       │
│ GLM-5.1      │  │ GLM-5-turbo  │  │ GLM-5-turbo  │
│              │  │              │  │              │
│ Tools:       │  │ Tools:       │  │ Tools:       │
│ web_search   │  │ exec, read   │  │ browser      │
│ web_fetch    │  │ write, edit  │  │ web_fetch    │
│ browser      │  │ git          │  │ read         │
│              │  │              │  │              │
│ Workspace:   │  │ Workspace:   │  │ Workspace:   │
│ ~/research/  │  │ ~/dev/       │  │ ~/ops/       │
└──────────────┘  └──────────────┘  └──────────────┘
       │                │                 │
       └────────────────┼─────────────────┘
                        │
                  共享文件目录
               ~/shared/
          ├── tasks/       ← 主 Agent 下发的任务
          ├── results/     ← 团队提交的结果
          └── knowledge/   ← 共享知识库
```

---

## 三、具体配置方案

### 3.1 创建团队 Agent

```bash
# 创建三个团队 agent
openclaw agents add research   # 调研团队
openclaw agents add dev        # 开发团队
openclaw agents add ops        # 运营团队
```

每个 agent 自动获得：
- 独立 workspace（~/research、~/dev、~/ops）
- 独立 SOUL.md（团队人格）
- 独立 session store
- 独立 auth profiles

### 3.2 openclaw.json 配置

```json5
{
  agents: {
    defaults: {
      model: { primary: "zai/glm-5.1" },
      subagents: {
        maxConcurrent: 8,
        maxChildrenPerAgent: 5,
        runTimeoutSeconds: 600,
        model: "zai/glm-5-turbo"
      }
    },
    list: [
      {
        // 调研团队 — 强模型 + 全搜索能力
        id: "research",
        model: { primary: "zai/glm-5.1" },
        tools: {
          allow: ["web_search", "web_fetch", "browser", "read", "write", "sessions_spawn"],
          deny: ["exec"]  // 不允许执行命令
        }
      },
      {
        // 开发团队 — 中模型 + 代码能力
        id: "dev",
        model: { primary: "zai/glm-5-turbo" },
        tools: {
          allow: ["exec", "read", "write", "edit", "browser"],
          deny: ["sessions_spawn"]  // 不能再 spawn 子 agent
        }
      },
      {
        // 运营团队 — 中模型 + 浏览器
        id: "ops",
        model: { primary: "zai/glm-5-turbo" },
        tools: {
          allow: ["web_fetch", "browser", "read", "write"],
          deny: ["exec", "sessions_spawn"]
        }
      }
    ]
  }
}
```

### 3.3 共享文件交互

```bash
# 创建共享目录
mkdir -p ~/shared/{tasks,results,knowledge}

# 主 Agent 通过 sessions_spawn 传文件给子 agent
# 方法 1：attachments 参数（内联小文件）
sessions_spawn({
  task: "根据 shared/tasks/research-001.md 执行调研",
  agentId: "research",
  attachments: [{
    name: "research-001.md",
    content: "调研 OpenClaw 多 agent 架构...",
    encoding: "utf8"
  }]
})

# 方法 2：共享目录（大文件/持久化）
# 主 Agent 写入 → 子 Agent 读取
write("~/shared/tasks/research-001.md", taskContent)
sessions_spawn({
  task: "读取 ~/shared/tasks/research-001.md 并执行",
  agentId: "research"
})
# 子 Agent 写入结果
# → ~/shared/results/research-001-output.md
```

### 3.4 主 Agent 作为唯一入口

主 Agent 的 SOUL.md 中明确：

```markdown
## 团队路由规则
你是唯一与用户交互的 Agent。所有任务通过你路由：

- 调研类任务 → sessions_spawn(agentId="research", ...)
- 开发类任务 → sessions_spawn(agentId="dev", ...)
- 运营类任务 → sessions_spawn(agentId="ops", ...)
- 跨团队协调 → 你自己协调多个团队的结果

❌ 不要让用户直接与子团队 Agent 交互
❌ 不要将子团队的内部信息直接暴露给用户（先整合）
```

---

## 四、隔离级别分析

| 隔离维度 | 实现方式 | 强度 |
|---------|---------|------|
| **Context（上下文）** | 独立 session store | 🔒 强 — 完全隔离 |
| **Identity（身份）** | 独立 SOUL.md、auth profiles | 🔒 强 — 完全隔离 |
| **Tools（工具）** | per-agent allow/deny 8 层策略 | 🔒 强 — 精细控制 |
| **Model（模型）** | per-agent model override | 🔒 强 — 完全隔离 |
| **Files（文件）** | workspace 默认隔离，绝对路径可跨 | ⚠️ 中 — 需 discipline 或 sandbox |
| **Network（网络）** | 无隔离（共享网络） | ⚠️ 中 — 正常，同一台机器 |
| **Process（进程）** | 同一 gateway 进程 | ⚠️ 中 — 正常，sub-agent 是隔离 session |

### 如需更强隔离

启用 Docker 沙箱：
```json5
{
  agents: {
    list: [
      {
        id: "dev",
        sandbox: { mode: "agent", backend: "docker" }
      }
    ]
  }
}
```

---

## 五、与行业方案对比

| 维度 | OpenClaw | CrewAI | LangGraph | AutoGen |
|------|---------|--------|-----------|---------|
| 团队隔离 | ✅ 独立 agent | ✅ 独立 Agent config | ⚠️ 需自行实现 | ⚠️ 需自行实现 |
| 统一入口 | ✅ 主 Agent + bindings | ✅ Hierarchical manager | ✅ Supervisor | ✅ Group chat manager |
| 文件交互 | ✅ 共享目录 + attachments | ✅ Shared state (Pydantic) | ✅ Shared state | ✅ Message passing |
| 不同模型 | ✅ per-agent model | ✅ per-agent LLM | ✅ per-node model | ✅ per-agent config |
| Tool 隔离 | ✅ 8 层 policy | ✅ per-agent tools | ⚠️ 需自行实现 | ⚠️ 需自行实现 |
| 沙箱 | ✅ Docker/SSH | ❌ | ❌ | ❌ |
| 用户交互 | ✅ Telegram/Web/Discord | ❌ 纯代码 | ❌ 纯代码 | ❌ 纯代码 |
| 运行方式 | 常驻 daemon | 按需脚本 | 按需脚本 | 按需脚本 |

**OpenClaw 是唯一同时支持：团队隔离 + 统一入口 + 文件交互 + 沙箱 + 用户交互界面（Telegram）的方案。**

---

## 六、风险与建议

### 风险

1. **文件系统非硬沙箱** — 绝对路径可跨 workspace 访问。如果需要严格隔离，启用 Docker sandbox。
2. **DuckDuckGo 反爬** — 搜索能力受限，建议配置智谱 Web Search API 或 Groq 作为备选。
3. **子 agent 输出格式不稳定** — GLM-5-turbo 偶尔输出非 JSON，需 3 层容错解析。
4. **Prompt injection 跨 agent** — 子 agent 的文件输出可能包含恶意内容，主 Agent 应做内容审核。

### 建议

1. **渐进式实施**：先建 1 个团队（如调研），验证流程后再扩展
2. **共享目录规范**：定义文件命名约定和目录结构（tasks/、results/、knowledge/）
3. **主 Agent 做内容审核**：子 agent 返回的结果经主 Agent 整合后再给用户
4. **定期安全审计**：`openclaw security audit --deep`

---

## 七、实施路线图

### Phase 1：验证（1 天）
- [ ] 创建 research agent
- [ ] 配置独立 model + tools
- [ ] 主 Agent 通过 sessions_spawn 调度
- [ ] 验证文件交互

### Phase 2：扩展（2-3 天）
- [ ] 创建 dev + ops agent
- [ ] 配置共享文件目录
- [ ] 主 Agent 路由逻辑
- [ ] 每个 agent 的 SOUL.md 定制

### Phase 3：强化（1 周）
- [ ] Docker sandbox（可选）
- [ ] Cron job per-team
- [ ] 安全审计
- [ ] 性能监控

---

## 来源

- OpenClaw Docs: multi-agent.md, session-tool.md, delegate-architecture.md, sandboxing, security, cron-jobs, multi-agent-sandbox-tools, skills
- CrewAI Docs: crews, agents, flows
- LangGraph Tutorials: agent_supervisor
- AutoGen Core README
- Anthropic: Building Effective Agents
- OWASP LLM Top 10
- OpenAI Function Calling Docs
