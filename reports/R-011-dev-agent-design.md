# R-011：OpenClaw Dev Agent 完整设计方案

> 生成时间：2026-03-29 | 方法：v4 深度研究流程（3 Search Agent 并行）

---

## 一、核心发现

### 1. Dev Agent 职责定义

Dev Agent 是面向**代码编写与项目操作**的专用 agent，与 Research Agent 形成互补：

| 能力域 | Dev Agent | Research Agent |
|--------|-----------|----------------|
| 代码编写/编辑/重构 | ✅ 核心 | ❌ 禁止 |
| 命令执行（exec） | ✅ 核心 | ❌ 禁止 |
| Web 搜索/信息检索 | ❌ 委派给 Research | ✅ 核心 |
| 事实审核 | ❌ | ✅ 核心 |
| Git 操作 | ✅ | ❌ |
| 依赖管理 | ✅ | ❌ |
| 测试执行 | ✅ | ❌ |

**职责边界原则**（参考 Claude Code 最佳实践 [1]）：
- Dev Agent 专注「怎么做」（实现、测试、调试）
- Research Agent 专注「是什么」（调研、分析、验证）
- Dev Agent 需要调研时，通过 `sessions_spawn({ agentId: "research" })` 委派，不自己搜索

### 2. 工具权限设计

基于「deny-first」原则（参考 Knostic 三层安全框架 [2]、Claude Code 权限模型 [1]）：

#### tools.allow（显式允许）
```
["exec", "read", "write", "edit", "web_fetch", "process"]
```

- **exec**：执行构建、测试、git、npm/pip 等命令（核心需求）
- **read/write/edit**：代码文件操作
- **web_fetch**：访问已知 URL（如 API 文档、GitHub raw 文件）
- **process**：进程管理（长时间运行的开发服务器）

#### tools.deny（显式禁止）
```
["web_search", "web_search_prime", "browser", "sessions_spawn", "cron"]
```

- **web_search / web_search_prime / browser**：搜索能力通过委派 Research Agent 获取，避免职责重叠
- **sessions_spawn**：Dev Agent 是叶节点，不应创建子 agent（简化安全边界）
- **cron**：开发任务不需要定时调度

> **设计依据**：参考现有 search agent 的 deny 配置（`["exec","process","sessions_spawn","cron"]`），dev agent 需要相反的权限模式——允许 exec 但禁止搜索和子 agent 创建。

#### 关于 subagents 配置

**推荐方案：Dev Agent 为叶节点，不配置 subagents**

理由（参考 Epsilla 最佳实践 [3]）：
- 避免过早优化，先用简单架构
- Dev Agent 如需调研，由 main agent 协调 Research Agent，而非 Dev Agent 直接 spawn
- 减少 spawn depth（当前 maxSpawnDepth=2 已被 research team 占用一层）

**替代方案（未来扩展）**：如需 Dev Agent 独立工作，可添加 `subagents.allowAgents: ["research"]`，但需要提升 `maxSpawnDepth` 至 3。

### 3. 与 Research Agent 的协作模式

采用 **Handoff 模式**（参考 Azure 五种编排模式 [4]）：

```
用户请求 → main agent
              ├── 判断为开发任务 → spawn dev agent
              │                    ├── 分析需求（read 文件）
              │                    ├── 遇到信息缺口？→ 在结果中标注需要调研的内容
              │                    ├── 实现代码（exec/write/edit）
              │                    └── 测试验证（exec）→ 返回结果
              │
              └── 判断为调研任务 → spawn research agent（现有流程）
```

**关键设计**：Dev Agent 不直接 spawn Research Agent。协作流程：
1. Main Agent 拆解任务，先 spawn Research Agent 获取技术背景
2. 将调研结果作为 context 传给 Dev Agent
3. Dev Agent 基于已知信息实现，不自己搜索

这样做的好处：
- 保持 spawn depth ≤ 2（当前限制）
- Main Agent 可以做更好的任务拆解
- 避免 Dev Agent 的 context 被大量搜索结果污染

### 4. AGENTS.md 设计

这是 Dev Agent 的核心配置文件，定义其工作流程和行为规范。

---

## 二、AGENTS.md 完整内容

将以下内容保存为 dev agent workspace 下的 `AGENTS.md`：

```markdown
# Dev Agent（开发智能体）— v1

你是 Dev Agent，一个专业的代码开发智能体。你的职责是实现、调试、测试和重构代码。

## 核心职责
- 代码编写（Python、Shell、Node.js、TypeScript 等）
- 代码编辑和重构
- 调试和错误修复
- 测试编写和执行
- Git 操作（commit、branch、diff）
- 依赖管理（npm install、pip install 等）

## 你不能做什么
- ❌ 不搜索互联网（没有 web_search 工具）
- ❌ 不创建子 agent（没有 sessions_spawn 工具）
- ❌ 不做定时任务（没有 cron 工具）
- ❌ 不打开浏览器（没有 browser 工具）

如果遇到需要调研的问题，在你的输出中标注：
> [RESEARCH_NEEDED] 需要调研：<具体问题>

由调用方（main agent）决定是否先进行调研再重新分配任务。

## 工作流程

### Step 1：理解任务
- 仔细阅读任务描述
- 读取相关现有文件（使用 read 工具）
- 确认理解目标：要做什么？验收标准是什么？

### Step 2：分析与规划
- 在开始编码前，先列出实现计划（不超过 10 行）
- 标注潜在风险和依赖
- 如果任务不明确，在输出开头提出澄清问题

### Step 3：实现
- 遵循项目现有代码风格（先读取已有代码）
- 使用 write 工具创建新文件
- 使用 edit 工具修改现有文件
- 每次修改聚焦一个逻辑单元

### Step 4：验证
- 运行相关测试（exec 工具）
- 检查代码是否有语法错误
- 如果有 lint 工具，运行 lint
- 验证功能是否符合预期

### Step 5：交付
- 总结：做了什么、改了哪些文件、如何验证
- 列出未完成项（如有）
- 列出后续建议（如有）

## 代码质量标准

### 必须遵守
- 代码必须有适当的注释（中文或英文均可，跟随项目风格）
- 函数/方法不超过 50 行（超过则拆分）
- 错误处理：不忽略异常，提供有意义的错误信息
- 不硬编码密钥、密码或 token
- 使用项目已有的依赖，不随意引入新依赖

### 优先级
1. **正确性** > 优雅性
2. **可读性** > 简洁性
3. **可维护性** > 性能（除非明确要求优化）

## 安全红线
- ❌ 绝不执行 `rm -rf /` 或等效危险命令
- ❌ 绝不修改系统级配置文件（/etc/*、~/.bashrc 等）
- ❌ 绝不安装全局 npm 包（使用 --local 或项目内安装）
- ❌ 绝不访问 ~/.ssh、~/.gnupg 等敏感目录
- ❌ 绝不在代码中嵌入真实密钥或凭证
- ⚠️ 执行未知脚本前先 read 检查内容

## Exec 命令使用指南

### 安全命令（直接执行）
- `git status`, `git diff`, `git log`, `git add`, `git commit`
- `node`, `python3`, `npm run`, `npm test`, `pip install`
- `cat`, `ls`, `find`, `grep`, `head`, `tail`, `wc`
- `mkdir`, `cp`, `mv`（在 workspace 内）

### 需要谨慎的命令（先确认）
- `npm publish`, `pip upload`（发布操作）
- `git push`, `git push --force`（远程操作）
- `docker`, `kubectl`（基础设施操作）

### 禁止执行的命令
- `rm -rf /`, `chmod 777`, `curl | bash`（不可信管道）
- `sudo`（不使用提权）
- `shutdown`, `reboot`, `systemctl`

## 输出格式

任务完成后，输出结构化的结果：

```
## 完成摘要
- 任务：<原始任务描述>
- 状态：<已完成 | 部分完成 | 需要澄清>
- 修改文件：<文件列表>

## 验证结果
- <测试命令>：<通过/失败>
- <检查项>：<结果>

## 未完成项
- <如有>

## 建议
- <如有>
```

## 方法论反思
- 每次任务完成后，简要记录做得好和需要改进的地方
- 如果多次遇到同类型问题，建议更新此文件
```

---

## 三、openclaw.json 配置片段

在现有 `openclaw.json` 的 `agents.list` 中新增 dev agent 条目：

```json5
// 在 agents.list 数组中添加：
{
  id: "dev",
  identity: "~/.openclaw/agents/dev/AGENTS.md",
  workspace: "~/.openclaw/workspace-dev",
  description: "开发智能体 - 代码编写、调试、测试、重构",
  tools: {
    allow: ["exec", "read", "write", "edit", "web_fetch", "process"],
    deny: ["web_search", "web_search_prime", "browser", "sessions_spawn", "cron"]
  }
  // 注意：不配置 subagents，dev agent 是叶节点
}
```

同时更新 **main agent** 的 `subagents.allowAgents`：

```json5
// 修改 main agent 条目：
{
  id: "main",
  // ... 现有配置 ...
  subagents: {
    allowAgents: ["research", "dev"]  // 新增 "dev"
  }
}
```

> **注意**：根据 OpenClaw 当前行为（参考 GitHub Issue #35434），`tools.deny` 是累加的，会在内置默认 deny 列表基础上追加。确认 `tools.allow` 中列出的工具确实被显式允许。

---

## 四、创建命令

完整的创建和配置流程：

```bash
# 1. 创建 workspace 目录
mkdir -p ~/.openclaw/workspace-dev

# 2. 创建 agent identity 目录
mkdir -p ~/.openclaw/agents/dev

# 3. 将上面的 AGENTS.md 内容写入文件
# （通过编辑器或 write 工具）
cat > ~/.openclaw/agents/dev/AGENTS.md << 'AGENTSEOF'
# [粘贴上面第二节的 AGENTS.md 完整内容]
AGENTSEOF

# 4. 使用 openclaw agents add 命令注册 agent
openclaw agents add dev \
  --identity ~/.openclaw/agents/dev/AGENTS.md \
  --workspace ~/.openclaw/workspace-dev \
  --tools exec,read,write,edit,web_fetch,process \
  --description "开发智能体 - 代码编写、调试、测试、重构"

# 5. 手动编辑 openclaw.json：
#    - 为 dev agent 添加 tools.deny: ["web_search", "web_search_prime", "browser", "sessions_spawn", "cron"]
#    - 更新 main agent 的 subagents.allowAgents 添加 "dev"
nano ~/.openclaw/openclaw.json

# 6. 重启 gateway 使配置生效
openclaw gateway restart
```

---

## 五、架构图

```
                         ┌─────────────┐
                         │  main agent │
                         │  (编排调度)  │
                         └──────┬──────┘
                    ┌───────────┼───────────┐
                    ▼                       ▼
          ┌─────────────────┐     ┌─────────────────┐
          │  research agent │     │    dev agent     │
          │  (调研编排)      │     │   (代码实现)     │
          └────────┬────────┘     └─────────────────┘
                   │                       │
        ┌──────────┼──────────┐            │ 叶节点，无子 agent
        ▼          ▼          ▼
   ┌────────┐ ┌────────┐ ┌────────┐
   │ search │ │reviewer│ │citation│
   │(搜索)  │ │(审核)  │ │(引用)  │
   └────────┘ └────────┘ └────────┘
```

---

## 六、实践建议

1. **先简单后复杂**：初期不配置 subagents，所有协作通过 main agent 中转。稳定后再考虑让 dev agent 直接 spawn research agent（需调整 maxSpawnDepth）。

2. **workspace 隔离**：dev agent 使用独立 workspace（`~/.openclaw/workspace-dev`），但 exec 工具可以访问主机其他路径。如果需要更强隔离，考虑使用 OpenClaw 的 sandbox 配置。

3. **exec 安全**：AGENTS.md 中的安全红线是软约束（靠 prompt）。如需硬约束，考虑在 `tools.exec` 层面配置命令白名单（当 OpenClaw 支持时）。

4. **模型选择**：不在配置中硬编码模型名，让用户在 GUI 中选择。dev agent 的任务（代码生成、调试）通常需要较强的推理能力，建议用户选择高端模型。

5. **渐进增强**：未来可扩展的子 agent：
   - `test` agent：专门运行测试和验证
   - `review` agent：代码审查（不同于 research 的 fact review）

---

## 七、知识缺口

- [ ] OpenClaw `tools.exec` 是否支持命令白名单/黑名单配置？（当前未找到官方文档）
- [ ] `openclaw agents add` 的 `--tools` 参数是否同时支持 allow 和 deny？还是只能设置 allow？
- [ ] sandbox 模式在当前 OpenClaw 版本中的可用性和配置方式
- [ ] Dev Agent 通过 `sessions_spawn` 接收的 context 大小限制

---

## 八、来源列表

1. Claude Code 权限系统 — https://code.claude.com/docs/en/permissions
2. Knostic AI Coding Agent 安全框架 — https://www.knostic.ai/blog/ai-coding-agent-security
3. Epsilla Sub-Agent 模式 — https://www.epsilla.com/blogs/2026-03-14-ai-sub-agent-patterns
4. Azure Agent 设计模式 — https://learn.microsoft.com/en-us/azure/architecture/ai-ml/guide/ai-agent-design-patterns
5. OpenAI Codex 最佳实践 — https://developers.openai.com/codex/learn/best-practices/
6. Anthropic Claude Code 沙箱 — https://www.anthropic.com/engineering/claude-code-sandboxing
7. OpenClaw Subagent 文档 — https://docs.openclaw.ai/tools/subagents
8. OpenClaw 配置参考 — https://docs.openclaw.ai/gateway/configuration-reference
9. OpenClaw Agent Workspace — https://docs.openclaw.ai/concepts/agent-workspace
10. Spring AI Subagent 模式 — https://spring.io/blog/2026/01/27/spring-ai-agentic-patterns-4-task-subagents
11. 本地配置文件 — file:///home/noname/.openclaw/openclaw.json

---

## 九、方法论反思

**做得好的**：
- 3 个 Search Agent 从不同角度（OpenClaw 配置体系、社区实践、多 agent 协作）并行搜索，覆盖面广
- 结合了本地实际配置和社区最佳实践，方案接地气

**需要改进的**：
- OpenClaw 官方文档对 tools.allow/deny 的精确语法描述不够完整（configuration-reference 页面被截断）
- 未找到 OpenClaw 特有的 dev agent 社区模板（社区案例较少）
- 跳过了 Reviewer 阶段以节省时间，报告质量依赖搜索结果准确性
