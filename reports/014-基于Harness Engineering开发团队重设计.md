# R-014：基于 Anthropic Harness Engineering 3 组件框架的 OpenClaw 开发团队重设计

> 生成时间：2026-03-29 | 基础：Anthropic "Effective Harnesses for Long-Running Agents" | 改进自：R-013

---

## 一、核心洞察：Anthropic 3 组件框架

Anthropic 的长任务 agent 方案解决的是**跨 context window 的连续工作**问题。核心设计：

| 组件 | 职责 | 关键产物 |
|------|------|----------|
| **Initializer Agent** | 首次 session 设置项目环境 | `feature_list.json` + `progress.md` + `init.sh` + 初始 git commit |
| **Coding Agent** | 每个 session 做一个 feature，增量进步 | 代码变更 + git commit + 更新 progress |
| **Feature List** | JSON 格式的功能清单，coding agent 只改 `passes` 字段 | `feature_list.json` |

**4 个失败模式及对策**：

| 问题 | Initializer 对策 | Coding Agent 对策 |
|------|-----------------|-------------------|
| 过早宣布完成 | 写完整 feature list | 每次只做一个功能，测试后才标记 pass |
| 留下 bug | 创建 git repo + progress file | session 开始读 progress + git log + 跑测试 |
| 未验证就标记完成 | 设 feature list | 端到端测试后才标记 pass |
| 不知道怎么启动项目 | 写 init.sh | session 开始读 init.sh |

---

## 二、架构图：3 组件 → OpenClaw Agent 映射

```
┌─────────────────────────────────────────────────────────────────┐
│                      main agent（全局路由）                       │
│  识别开发任务 → spawn dev-lead                                    │
└──────────────────────────┬──────────────────────────────────────┘
                           │
              ┌────────────▼────────────┐
              │       dev-lead           │
              │  (开发编排 = Harness 调度器) │
              │  不写代码，只拆解+审查+调度    │
              │  管理 feature list 生命周期   │
              └───┬─────────┬─────────┬──┘
                  │         │         │
     ┌────────────▼──┐  ┌──▼────────┐ ┌▼──────────────┐
     │ dev-init      │  │ dev-coder │ │ dev-harness   │
     │ (Initializer) │  │ (Coding   │ │ (ACP Coding   │
     │               │  │  Agent)   │ │  Agent)       │
     │ 1次性运行：    │  │ OpenClaw  │ │ Claude Code   │
     │ → feature_list│  │ 原生 agent│ │ /Codex/Gemini │
     │ → progress.md │  │           │ │               │
     │ → init.sh     │  │ exec+read │ │ 通过 ACP 协议  │
     │ → git init    │  │ +write    │ │ 调用          │
     └───────────────┘  └───────────┘ └───────────────┘
                            │                │
                            └───────┬────────┘
                                    ▼
                         ┌────────────────────┐
                         │ 共享 Project Dir    │
                         │ (git repo)         │
                         ├────────────────────┤
                         │ feature_list.json  │ ← Initializer 创建
                         │ progress.md        │ ← Initializer 创建，Coding 更新
                         │ init.sh            │ ← Initializer 创建
                         │ src/               │ ← Coding Agent 修改
                         └────────────────────┘

    可选协作：
    dev-lead → spawn research agent（需要技术调研时）
```

**关键映射关系**：

| Anthropic 组件 | OpenClaw 实现 | 说明 |
|---------------|--------------|------|
| Initializer Agent | `dev-init`（一次性 OpenClaw subagent） | dev-lead 在项目首次启动时 spawn |
| Coding Agent | `dev-coder` 或 `dev-harness`（ACP） | 按任务复杂度选择 |
| Feature List | `feature_list.json`（共享目录） | 文件系统持久化，不依赖 agent 记忆 |
| Progress File | `progress.md`（共享目录） | 同上 |

---

## 三、Agent 角色定义（4 个 Agent）

### 设计决策：为什么是 4 个而不是 R-013 的 3 个？

R-013 方案没有 Initializer Agent，将初始化混在 dev-lead 中。Anthropic 的核心洞察是 **Initializer 必须是独立的、有专门 prompt 的 session**，因为它的职责与后续编码完全不同。因此新增 `dev-init`。

### Agent 角色一览

| 角色 | agentId | 类型 | 职责 |
|------|---------|------|------|
| **dev-lead** | `dev` | OpenClaw agent | 编排调度、代码审查、质量把关 |
| **dev-init** | `dev-init` | OpenClaw agent (一次性) | Initializer Agent — 创建 feature list + progress + init.sh |
| **dev-coder** | `dev-coder` | OpenClaw agent | Coding Agent（轻量）— 脚本、配置、小功能 |
| **dev-harness** | — | ACP session | Coding Agent（重量）— 通过 Claude Code/Codex 执行 |

> **dev-harness 不是注册 agent**，而是 dev-lead 通过 `sessions_spawn({ runtime: "acp" })` 动态创建的 session。

---

## 四、各 Agent 完整 AGENTS.md

### 4.1 dev-lead（开发编排者）

```markdown
# Dev Lead（开发编排者）— v2 (Harness Engineering)

你是 Dev Lead，负责 orchestrating 开发任务。你是调度员，不是执行者。
你管理项目的 feature list 生命周期，决定何时初始化、何时编码、何时验证。

## 绝对不做什么
- ❌ 不自己写代码
- ❌ 不自己执行命令（没有 exec 权限）
- ❌ 不自己修改 feature_list.json 的 passes 字段

## 工作流程

### Flow A：新项目（Initializer 模式）
1. 接收用户的开发任务
2. 判断项目目录中是否存在 `feature_list.json`
3. 如果不存在 → spawn dev-init 创建初始环境
4. 等待 dev-init 完成
5. 进入 Flow B

### Flow B：继续开发（Coding 模式）
1. 读取 `feature_list.json`，统计未完成功能
2. 选择下一个未完成功能（按 category 顺序：functional → integration → edge-case）
3. 判断复杂度：
   - **简单**（< 50 行改动）→ spawn dev-coder
   - **复杂**（多文件/需要深度推理）→ spawn dev-harness (ACP)
4. 等待结果
5. 审查代码变更（read git diff）
6. 如不达标，给修改意见，重新 spawn（最多 3 轮）
7. 验证通过 → 让 coding agent 更新 feature_list.json 的 passes 字段

### Flow C：需要调研
1. spawn research agent 获取技术背景
2. 将调研结果写入 `docs/research/` 目录
3. 基于调研结果回到 Flow B

## Feature List 管理规则
- 只能由 dev-init 创建
- coding agent 只能改 `passes` 字段（false → true）
- dev-lead 不直接修改，只负责读取和调度
- 每次调度前必须重新读取（不缓存）

## Harness 不可用时回退
1. ACP spawn 失败 → 降级到 dev-coder
2. dev-coder 也失败 → 返回 OFFLINE，建议人工介入
3. 结果中标注降级信息

## 与 Research Team 协作
- subagents.allowAgents 包含 "research"
- 需要 API 文档查询时用 web_fetch（已知 URL）
- 不确定技术方案时 spawn research agent

## 代码审查标准
- 是否符合 feature_list.json 中该功能的描述和 steps
- 是否有适当的错误处理
- 是否有必要的测试
- 不确定处标注 [NEEDS_REVIEW]

## 红线
- 不直接执行命令
- 不跳过审查
- 最多迭代 3 轮
- 不确定的技术决策标注 [NEEDS_REVIEW]
```

### 4.2 dev-init（Initializer Agent）

```markdown
# Dev Init（初始化 Agent）— v1

你是 Dev Init，基于 Anthropic Harness Engineering 的 Initializer Agent 设计。
你的唯一职责是为新项目创建初始环境，让后续 Coding Agent 能有效工作。

## 你只运行一次
当项目目录中没有 `feature_list.json` 时，dev-lead 会 spawn 你。
完成后你不会再被调用。

## 绝对不做什么
- ❌ 不实现任何功能（只做环境设置）
- ❌ 不写业务代码
- ❌ 不修改 feature_list.json 中的 passes 为 true

## 工作流程（严格按序执行）

### Step 1：理解需求
1. 阅读用户的完整任务描述
2. 如有模糊之处，在输出中列出假设（dev-lead 会审查）

### Step 2：创建 feature_list.json
在项目根目录创建 `feature_list.json`，格式如下：
```json
{
  "project": "项目名称",
  "created": "ISO 日期",
  "description": "项目简述",
  "features": [
    {
      "id": "F001",
      "category": "functional",
      "description": "用户可以点击新建按钮创建空白对话",
      "steps": [
        "导航到主界面",
        "点击新建对话按钮",
        "验证创建了新对话",
        "验证侧边栏显示新对话"
      ],
      "passes": false
    }
  ]
}
```

**要求**：
- 功能必须细化到可以在一个 session 内完成（避免 agent 试图一次性做太多）
- 初始所有 features 的 passes 必须为 false
- 功能数量至少覆盖核心需求（参考 Anthropic 实践：一个 claude.ai clone 有 200+ features）
- 优先级：functional → integration → error-handling → edge-case → polish

### Step 3：创建 progress.md
```markdown
# 开发进度日志

## 项目：{project_name}
## 创建时间：{date}

### Session 0 — 初始化
- [INIT] 创建 feature_list.json（{N} 个功能）
- [INIT] 创建 progress.md
- [INIT] 创建 init.sh
- [INIT] 初始 git commit
```

### Step 4：创建 init.sh
```bash
#!/bin/bash
# 项目启动脚本 — Coding Agent 在每个 session 开始时运行此脚本

# 1. 安装依赖（如有 package.json/requirements.txt）
# 2. 启动开发服务器（后台）
# 3. 等待服务器就绪
# 4. 运行基础测试确认环境正常

set -e

echo "=== Init: Checking environment ==="

# 示例：Node.js 项目
if [ -f "package.json" ]; then
  npm install --silent 2>/dev/null || true
fi

# 示例：Python 项目
if [ -f "requirements.txt" ]; then
  pip3 install -q -r requirements.txt 2>/dev/null || true
fi

echo "=== Init: Environment ready ==="
```

### Step 5：Git 初始化
1. `git init`（如果还不是 git repo）
2. `git add .`
3. `git commit -m "init: project scaffold with feature_list.json and init.sh"`

## 输出格式
```
## 初始化完成
- 项目：{name}
- 功能数：{N}
- Git commit：{hash}
- 文件创建：feature_list.json, progress.md, init.sh

## 功能概览
- functional: {n} 个
- integration: {n} 个
- error-handling: {n} 个
- edge-case: {n} 个
```

## 安全红线
- 不执行危险命令（rm -rf、sudo 等）
- 不安装全局 npm 包
- 不修改系统配置
```

### 4.3 dev-coder（Coding Agent — OpenClaw 原生）

```markdown
# Dev Coder（Coding Agent）— v2 (Harness Engineering)

你是 Coding Agent，基于 Anthropic Harness Engineering 设计。
你每个 session 只做一个 feature，完成后更新进度文件。

## Session 启动流程（必须按序执行）

### 1. 定位
确认当前工作目录（pwd）

### 2. 读进度文件
读取 `progress.md` — 了解之前做了什么

### 3. 读 Git 日志
运行 `git log --oneline -20` — 了解最近的变更

### 4. 读 Feature List
读取 `feature_list.json` — 了解所有功能和状态

### 5. 选择功能
选择 **一个** passes=false 的功能（按 id 顺序，或 dev-lead 指定的 id）

### 6. 环境健康检查
运行 `bash init.sh` — 确认环境正常

### 7. 实现
按功能的 steps 列表逐步实现

### 8. 端到端验证
**必须端到端验证**，不能只跑单元测试：
- 启动应用/服务
- 按功能描述的 steps 手动验证
- 运行相关测试套件
- 所有验证通过后才标记 passes=true

### 9. 提交
- `git add -A`
- `git commit -m "feat(F{id}): {description}"`
- 更新 `progress.md`

### 10. 更新 Feature List
**只改 passes 字段**：将已完成功能的 passes 改为 true
**严禁**删除功能、修改 steps、修改 description

## Session 结束输出
```
## 完成摘要
- 功能：F{id} — {description}
- 状态：已完成 | 部分完成 | 失败
- 修改文件：{list}
- Git commit：{hash}

## 验证结果
- {step}: ✅/❌
- 测试：{pass}/{total} passed

## 问题
- {如有}
```

## 不做什么
- ❌ 不搜索互联网
- ❌ 不创建子 agent
- ❌ 不做架构决策
- ❌ 不修改 feature_list.json 中除 passes 以外的字段
- ❌ 不在一个 session 做多个功能

## 代码质量标准
- 正确性 > 优雅性
- 可读性 > 简洁性
- 错误处理不忽略异常
- 函数不超过 50 行

## 安全红线
- ❌ 绝不执行 rm -rf / 或等效命令
- ❌ 绝不使用 sudo
- ❌ 绝不在代码中嵌入密钥或凭证
- ❌ 绝不修改系统级配置文件
```

---

## 五、Feature List 管理方案

### 5.1 位置与格式

```
<project-root>/
├── feature_list.json    ← 功能清单（Initializer 创建）
├── progress.md          ← 进度日志（Initializer 创建，Coding 更新）
├── init.sh              ← 环境启动脚本（Initializer 创建）
├── src/                 ← 源代码
└── tests/               ← 测试
```

### 5.2 权限矩阵

| 操作 | dev-init | dev-coder | dev-lead | dev-harness (ACP) |
|------|---------|-----------|----------|-------------------|
| 创建 feature_list.json | ✅ | ❌ | ❌ | ❌ |
| 修改 passes 字段 | ❌ | ✅ | ❌ | ✅ |
| 修改其他字段 | ❌ | ❌ | ❌ | ❌ |
| 读取 | ✅ | ✅ | ✅ | ✅ |
| 创建 progress.md | ✅ | ❌ | ❌ | ❌ |
| 追加 progress.md | ❌ | ✅ | ✅ (审查) | ✅ |
| 创建 init.sh | ✅ | ❌ | ❌ | ❌ |

### 5.3 feature_list.json 模板

```json
{
  "project": "my-web-app",
  "created": "2026-03-29T00:00:00Z",
  "description": "一个示例 Web 应用",
  "features": [
    {
      "id": "F001",
      "category": "scaffold",
      "description": "项目脚手架：目录结构 + 构建配置 + 基础 HTML",
      "steps": [
        "创建 src/, public/, tests/ 目录",
        "配置构建工具（vite/webpack）",
        "创建 index.html 入口",
        "运行 dev server 确认页面加载"
      ],
      "passes": false
    },
    {
      "id": "F002",
      "category": "functional",
      "description": "用户可以在输入框输入文字并提交",
      "steps": [
        "渲染输入框和提交按钮",
        "输入文字后点击提交",
        "验证提交后输入框清空",
        "验证提交的内容显示在页面上"
      ],
      "passes": false
    },
    {
      "id": "F003",
      "category": "functional",
      "description": "用户可以看到历史记录列表",
      "steps": [
        "提交多条记录",
        "验证历史记录按时间倒序显示",
        "验证每条记录显示完整内容"
      ],
      "passes": false
    },
    {
      "id": "F004",
      "category": "error-handling",
      "description": "空输入提交时显示错误提示",
      "steps": [
        "不输入任何内容点击提交",
        "验证显示错误提示信息",
        "输入内容后错误提示消失"
      ],
      "passes": false
    },
    {
      "id": "F005",
      "category": "integration",
      "description": "数据持久化到 localStorage",
      "steps": [
        "提交数据",
        "刷新页面",
        "验证数据仍然存在"
      ],
      "passes": false
    }
  ]
}
```

---

## 六、增量进度管理

### 6.1 progress.md 模板

```markdown
# 开发进度日志

## 项目：my-web-app
## 创建时间：2026-03-29

---

### Session 0 — 初始化 [dev-init]
- [INIT] 创建 feature_list.json（5 个功能）
- [INIT] 创建 progress.md
- [INIT] 创建 init.sh
- [INIT] 初始 git commit (abc1234)

---

### Session 1 — F002: 用户输入提交 [dev-coder]
- [DONE] 实现输入框和提交按钮组件
- [DONE] 添加提交逻辑和状态管理
- [DONE] 端到端验证通过
- [DONE] git commit (def5678): feat(F002): user input and submit
- [PASS] F002 passes: false → true

---

### Session 2 — F003: 历史记录列表 [dev-coder]
- [DONE] 实现历史记录组件
- [DONE] 添加排序逻辑
- [DONE] 端到端验证通过
- [DONE] git commit (ghi9012): feat(F003): history list with reverse sort
- [PASS] F003 passes: false → true
```

### 6.2 Git Commit 策略

| 场景 | 格式 | 示例 |
|------|------|------|
| 初始化 | `init: {描述}` | `init: project scaffold with feature list` |
| 功能完成 | `feat(F{id}): {描述}` | `feat(F002): user input and submit` |
| Bug 修复 | `fix(F{id}): {描述}` | `fix(F003): sort order in history list` |
| 回退 | `revert: {原因}` | `revert: F004 broken by previous commit` |

### 6.3 Session 间上下文传递

**核心原则：不依赖 agent 记忆，所有状态通过文件传递。**

```
Session N 结束时的文件状态：
├── feature_list.json  → 包含 F001..F003 passes=true, F004..F005 passes=false
├── progress.md        → 包含 Session 0..N 的完整日志
├── init.sh            → 不变
├── src/               → 包含 F001..F003 的代码
└── .git/              → 包含所有 commit 历史

Session N+1 开始时：
1. 读 progress.md → 知道上次做到哪了
2. 读 git log → 知道代码变更历史
3. 读 feature_list.json → 知道下一个未完成功能
4. 运行 init.sh → 确认环境
5. 开始工作
```

---

## 七、测试验证机制

### 7.1 端到端验证流程

```
Coding Agent 的验证步骤：
1. 运行 init.sh（环境健康检查）
2. 实现功能代码
3. 运行单元测试（如有）
4. 运行集成测试（如有）
5. 手动/脚本验证（按 feature 的 steps 列表）：
   a. 启动应用
   b. 逐步执行 steps
   c. 每步确认通过
6. 全部通过 → 标记 passes=true
7. 任一失败 → 不标记，在 progress.md 中记录失败原因
```

### 7.2 init.sh 完整模板

```bash
#!/bin/bash
# init.sh — 项目环境启动脚本
# 由 Initializer Agent 创建，Coding Agent 每个 session 开始时运行

set -e

PROJECT_ROOT="$(cd "$(dirname "$0")" && pwd)"
cd "$PROJECT_ROOT"

echo "=== Environment Check ==="

# Node.js 项目
if [ -f "package.json" ]; then
  echo "[init] Installing npm dependencies..."
  npm install --silent 2>/dev/null || npm install
  echo "[init] Running lint..."
  npm run lint 2>/dev/null || echo "[init] No lint script, skipping"
  echo "[init] Running tests..."
  npm test 2>/dev/null || echo "[init] No test script, skipping"
fi

# Python 项目
if [ -f "requirements.txt" ]; then
  echo "[init] Installing Python dependencies..."
  pip3 install -q -r requirements.txt
fi
if [ -f "pyproject.toml" ]; then
  echo "[init] Running pytest..."
  python3 -m pytest -x -q 2>/dev/null || echo "[init] No tests or pytest not configured"
fi

# 通用检查
echo "[init] Git status:"
git status --short

echo "=== Environment Ready ==="
```

---

## 八、ACP Harness 集成方案

### 8.1 任务路由决策

```
dev-lead 收到任务
  │
  ├── 功能简单（< 50行，单文件） → dev-coder (OpenClaw 原生)
  │     优点：快速、低成本、无外部依赖
  │
  ├── 功能复杂（多文件、需要深度推理） → dev-harness (ACP)
  │     优点：Claude Code 有更好的代码理解能力
  │     spawn: sessions_spawn({ runtime: "acp", agentId: "claude", mode: "run" })
  │
  └── 超长任务（20+ 分钟） → dev-harness (ACP, persistent)
        spawn: sessions_spawn({ runtime: "acp", agentId: "claude",
                               mode: "session", thread: true })
```

### 8.2 ACP Coding Agent 的特殊处理

ACP harness（如 Claude Code）不读 OpenClaw 的 AGENTS.md，所以需要通过 task prompt 传递 Harness Engineering 流程：

```javascript
// dev-lead spawn ACP coding agent 时的 task 模板
sessions_spawn({
  runtime: "acp",
  agentId: "claude",    // 或 "codex", "gemini"
  mode: "run",
  runTimeoutSeconds: 600,
  cwd: "/path/to/project",
  task: `你是 Coding Agent。严格按以下流程工作：

## Session 启动流程
1. pwd 确认当前目录
2. 读 progress.md 了解之前做了什么
3. 运行 git log --oneline -20 了解最近变更
4. 读 feature_list.json 了解所有功能
5. 选择功能 ID: ${featureId}（passes: false）
6. 运行 bash init.sh 确认环境正常
7. 按功能的 steps 实现该功能
8. 端到端验证（启动应用，按 steps 逐步验证）
9. 全部通过后，修改 feature_list.json 中 F${featureId} 的 passes 为 true
10. git add -A && git commit -m "feat(F${featureId}): ${description}"
11. 更新 progress.md

## 严禁
- 不要修改 feature_list.json 中除 passes 以外的任何字段
- 不要在一个 session 做多个功能
- 不要跳过端到端验证就标记 passes=true`
})
```

### 8.3 降级方案

| 级别 | 能力 | 触发条件 |
|------|------|----------|
| **FULL** | dev-lead + dev-coder + ACP harness | 正常运行 |
| **DEGRADED** | dev-lead + dev-coder（无 ACP） | ACP 不可用 |
| **MINIMAL** | dev-lead 仅审查 | dev-coder 和 harness 均不可用 |
| **OFFLINE** | 返回错误，建议人工 | 所有 agent 不可用 |

---

## 九、openclaw.json 配置片段

```json5
{
  // ACP 全局配置
  acp: {
    enabled: true,
    dispatch: { enabled: true },
    backend: "acpx",
    defaultAgent: "claude",
    allowedAgents: ["claude", "codex", "gemini", "opencode", "pi", "kimi"],
    maxConcurrentSessions: 8,
    stream: {
      coalesceIdleMs: 300,
      maxChunkChars: 1200,
    },
    runtime: {
      ttlMinutes: 120,
    },
  },

  agents: {
    list: [
      // Dev Lead — 编排调度
      {
        id: "dev",
        identity: "~/.openclaw/agents/dev/AGENTS.md",
        workspace: "~/.openclaw/workspace-dev",
        description: "开发编排者 - 任务拆解、代码审查、feature list 管理、ACP 调度",
        tools: {
          allow: ["read", "write", "web_fetch", "sessions_spawn", "subagents"],
          deny: ["exec", "process", "web_search", "web_search_prime", "browser", "cron"]
        },
        subagents: {
          allowAgents: ["dev-init", "dev-coder", "research"]
        }
      },

      // Dev Init — Initializer Agent
      {
        id: "dev-init",
        identity: "~/.openclaw/agents/dev-init/AGENTS.md",
        workspace: "~/.openclaw/workspace-dev",
        description: "Initializer Agent - 创建 feature_list.json、progress.md、init.sh",
        tools: {
          allow: ["exec", "read", "write"],
          deny: ["web_search", "web_search_prime", "browser", "sessions_spawn", "cron", "subagents"]
        }
        // 一次性 agent，不需要 subagents
      },

      // Dev Coder — Coding Agent (OpenClaw 原生)
      {
        id: "dev-coder",
        identity: "~/.openclaw/agents/dev-coder/AGENTS.md",
        workspace: "~/.openclaw/workspace-dev",
        description: "Coding Agent (OpenClaw) - 增量实现功能、测试、git 操作",
        tools: {
          allow: ["exec", "read", "write", "process"],
          deny: ["web_search", "web_search_prime", "browser", "sessions_spawn", "cron", "subagents"]
        }
      },
    ]
  },

  // ACP 插件
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

## 十、创建命令

```bash
# 1. 创建目录结构
mkdir -p ~/.openclaw/agents/dev
mkdir -p ~/.openclaw/agents/dev-init
mkdir -p ~/.openclaw/agents/dev-coder
mkdir -p ~/.openclaw/workspace-dev

# 2. 注册 agents
openclaw agents add dev \
  --identity ~/.openclaw/agents/dev/AGENTS.md \
  --workspace ~/.openclaw/workspace-dev \
  --description "开发编排者"

openclaw agents add dev-init \
  --identity ~/.openclaw/agents/dev-init/AGENTS.md \
  --workspace ~/.openclaw/workspace-dev \
  --description "Initializer Agent"

openclaw agents add dev-coder \
  --identity ~/.openclaw/agents/dev-coder/AGENTS.md \
  --workspace ~/.openclaw/workspace-dev \
  --description "Coding Agent (OpenClaw)"

# 3. 写入 AGENTS.md（将第四节的完整内容写入对应文件）

# 4. 编辑 openclaw.json（添加第九节的配置）

# 5. 安装 ACP 插件
openclaw plugins install acpx
openclaw config set plugins.entries.acpx.enabled true

# 6. 验证
openclaw gateway restart
/acp doctor
openclaw agents list
```

---

## 十一、部署步骤（从零开始）

### Phase 0：环境准备（1 天）
- [ ] 确认 OpenClaw 已安装且运行正常
- [ ] 安装 acpx 插件，`/acp doctor` 确认 ACP 可用
- [ ] 创建目录结构和 AGENTS.md 文件
- [ ] 更新 openclaw.json 配置

### Phase 1：最小可用（3-5 天）
**目标**：dev-lead + dev-init + dev-coder，不依赖 ACP

- [ ] 注册所有 3 个 agent
- [ ] 测试：给 main agent 一个开发任务 → 路由到 dev-lead
- [ ] 测试：dev-lead 检测无 feature_list.json → spawn dev-init
- [ ] 测试：dev-init 创建 feature_list.json + progress.md + init.sh + git init
- [ ] 测试：dev-lead 读 feature_list → spawn dev-coder 做一个功能
- [ ] 测试：dev-coder 按 session 启动流程工作，完成后更新 passes

**验收**：能完成一个 5 功能的小项目的完整初始化→编码→验证循环

### Phase 2：ACP 集成（1 周）
- [ ] 配置 acpx 插件
- [ ] 测试：dev-lead 将复杂功能路由到 ACP harness
- [ ] 测试：ACP harness 的 task prompt 包含完整的 session 启动流程
- [ ] 测试：ACP harness 正确更新 feature_list.json
- [ ] 测试：harness 不可用时降级到 dev-coder

### Phase 3：Research 协作（3-5 天）
- [ ] 配置 dev-lead 的 allowAgents 包含 "research"
- [ ] 测试：dev-lead 在技术不确定时 spawn research agent
- [ ] 端到端测试：调研→初始化→编码→验证

### Phase 4：生产化（持续）
- [ ] 监控：任务成功率、平均每功能耗时、token 消耗
- [ ] AGENTS.md 精简到 100 行以内
- [ ] 安全审计

---

## 十二、与 R-013 方案的对比和改进

| 维度 | R-013（旧方案） | R-014（本方案） | 改进说明 |
|------|---------------|---------------|----------|
| **Initializer** | ❌ 没有，混在 dev-lead 中 | ✅ 独立 dev-init agent | Anthropic 核心洞察：初始化必须是独立 session |
| **Feature List** | ❌ 没有机制 | ✅ feature_list.json + 严格权限 | 解决"过早宣布完成"问题 |
| **Progress File** | ❌ 没有机制 | ✅ progress.md + 结构化日志 | 解决"跨 session 上下文丢失"问题 |
| **Session 启动流程** | ❌ 未定义 | ✅ 6 步标准流程 | 每次开始都读 progress + git log + feature list |
| **增量进度** | ❌ 无约束 | ✅ 每次 session 只做一个功能 | 防止 agent 试图一次性做太多 |
| **端到端验证** | ❌ 只跑测试 | ✅ 按 steps 列表逐步验证 | 必须端到端通过才标记 pass |
| **Agent 数量** | 3（dev-lead + dev-coder + dev-harness） | 4（+ dev-init） | 多 1 个但职责更清晰 |
| **ACP prompt** | 只传任务描述 | 传完整的 session 启动流程 | ACP harness 也能遵循 Harness Engineering 流程 |
| **文件权限** | 未定义 | 明确的权限矩阵 | 防止误操作 feature list |

### 关键改进总结

1. **解决了 Anthropic 识别的 4 个失败模式**：过早完成、留下 bug、未验证标记、不知如何启动
2. **新增 Initializer Agent**：将项目初始化从 dev-lead 中独立出来，与 Anthropic 实践对齐
3. **结构化进度管理**：feature_list.json + progress.md + git history 三重状态追踪
4. **严格的 Coding Agent session 流程**：6 步启动流程确保每个 session 都有完整上下文
5. **ACP harness 兼容**：通过 task prompt 将流程传递给外部 harness，不依赖 OpenClaw AGENTS.md

---

## 十三、知识缺口

1. **ACP harness 的 feature_list.json 操作**：Claude Code 等外部 harness 是否可靠地遵循"只改 passes"的约束？需要实际测试。
2. **并行 Coding Agent**：多个 dev-coder 或 ACP session 同时操作同一个 feature_list.json 时的并发安全未解决。
3. **Feature 粒度**：多大的功能算"一个 feature"？太小会导致过多 session，太大会导致 context 不足。需要 dev-init 的 prompt 中加入粒度指引。
4. **init.sh 适应性**：不同项目类型（前端/后端/全栈/CLI 工具）的 init.sh 模板需要不同，是否需要 dev-init 自动识别项目类型？

---

## 十四、来源

| # | 来源 | URL |
|---|------|-----|
| 1 | Anthropic: Effective Harnesses for Long-Running Agents | https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents |
| 2 | Anthropic: Claude Agent SDK Quickstart (autonomous-coding) | https://github.com/anthropics/claude-quickstarts/tree/main/autonomous-coding |
| 3 | Claude 4 Prompting Guide: Multi-context Window Workflows | https://docs.claude.com/en/docs/build-with-claude/prompt-engineering/claude-4-best-practices |
| 4 | OpenClaw ACP Agents 文档 | /home/noname/.npm-global/lib/node_modules/openclaw/docs/tools/acp-agents.md |
| R-013 | OpenClaw Dev Team 架构设计方案 | /home/noname/.openclaw/workspace/shared/results/R-013-dev-team-design.md |
| R-007 | ClawHub Claude Code 技能调研 | /home/noname/.openclaw/workspace/shared/results/R-007-claude-code-skills-report.md |
| R-006 | OpenClaw vs Claude Code 对比 | /home/noname/.openclaw/workspace/shared/results/R-006-dev-team-comparison.md |
