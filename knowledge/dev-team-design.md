# R-015b：以交付质量为核心的 OpenClaw 研发团队 Agent 方案 v2

> 生成时间：2026-03-30 | 基于：R-015 改进 + Anthropic autonomous-coding 源码深度分析 + 行业实践搜索 | 研究方法：3 Search Agent 并行 + 源码精读 + 文件交叉验证

---

## 一、设计哲学：源码确认 + 行业实践

### 1.1 Anthropic 架构真相：源码级确认

R-015 基于博客和 prompt 文件做了分析，本次通过直接阅读源码确认了架构细节：

**核心发现：Anthropic 是 TWO-agent 架构，不是 three-agent**

> "A minimal harness demonstrating long-running autonomous coding with the Claude Agent SDK. This demo implements a **two-agent pattern (initializer + coding agent)** that can build complete applications over multiple sessions."
> — README.md

这意味着 Anthropic 并没有实现独立的 reviewer agent——blog 中提到的 "testing agent, QA agent" 是对未来方向的展望，不是当前实现。

**agent.py 的 main loop 逻辑：**

```python
# 核心逻辑（简化）
while True:
    iteration += 1
    if max_iterations and iteration > max_iterations:
        break
    client = create_client(project_dir, model)
    if is_first_run:
        prompt = get_initializer_prompt()
    else:
        prompt = get_coding_prompt()
    async with client:
        status, response = await run_agent_session(client, prompt, project_dir)
    if status == 'continue':
        await asyncio.sleep(AUTO_CONTINUE_DELAY_SECONDS)  # 3秒
```

关键设计决策：
- 每次 session 创建 **fresh client**（新 context window）
- 通过 `feature_list.json` 是否存在判断首次/继续运行
- 无限循环直到所有 feature passes=true 或达到 max_iterations
- session 间 3 秒延迟（`AUTO_CONTINUE_DELAY_SECONDS`）

**progress.py 的进度追踪：**

```python
def count_passing_tests(project_dir):
    tests = load_tests(project_dir)
    passing = sum(1 for test in tests if test.get('passes', False))
    return passing, len(tests)

def print_progress_summary(project_dir):
    passing, total = count_passing_tests(project_dir)
    percentage = (passing / total) * 100
    print(f'Progress: {passing}/{total} tests passing ({percentage:.1f}%)')
```

简洁但有效——进度就是 passes=true 的百分比。

**security.py 的三层安全防御：**

| 层 | 机制 | 实现细节 |
|---|------|---------|
| Layer 1 | OS-level Sandbox | bash 命令隔离 |
| Layer 2 | Permissions | 文件操作限制在 project_dir |
| Layer 3 | Security hooks | 命令 allowlist 验证 |

Allowlist 只允许 15 个命令：`{ls, cat, head, tail, wc, grep, cp, mkdir, chmod, pwd, npm, node, git, ps, lsof, sleep, pkill, init.sh}`

细粒度验证：
- `pkill` → 只允许杀 `node/npm/npx/vite/next` 进程
- `chmod` → 只允许 `+x` 模式（`u+x`, `a+x`）
- `init.sh` → 只允许 `./init.sh` 或路径以 `/init.sh` 结尾

**client.py 配置：**
- `max_turns=1000`（单 session 最大交互轮数）
- 8 个 Puppeteer 工具用于浏览器自动化测试
- 默认模型 `claude-sonnet-4-5-20250929`

**feature_list.json Schema（源码确认）：**

```json
{
  "category": "functional | style",
  "description": "...",
  "steps": ["Step 1: ...", "Step 2: ..."],
  "passes": false
}
```

Initializer 要求：最少 **200 个 feature**，至少 **25 个有 10+ steps**，必须包含 functional 和 style 两种 category。

**对 R-015 设计的影响：**

Anthropic 的实际架构比 blog 描述的更简单——没有 reviewer，没有独立 QA。但正是因为他们只有 self-verify，才会在 blog 中明确说 "specialized agents like a testing agent, QA agent could do even better"。**本方案在 Anthropic 基础上做的核心升级就是实现了他们想做但没做的独立 QA agent。**

### 1.2 行业实践佐证

#### OpenAI Codex — Durable Project Memory

> "The most important technique was durable project memory. I wrote the spec, plan, constraints, and status in markdown files that Codex could revisit repeatedly. That prevented drift and kept a stable definition of 'done.'"

Codex 的文件体系：`Prompt.md → Plan.md → Implement.md → Documentation.md`

这直接验证了 Anthropic 的 `feature_list.json + progress.md` 方案——**文件持久化是跨 context window 工作的唯一可靠方式**。

#### OpenAI Code Review Agent — 精度优先

> "Precision is more important for usability than recall. Defenses often fail not because they are technically wrong, but because they are so impractical that the user chooses not to use them."

关键发现：**给 reviewer repo-wide tools 和 execution access 可以同时提升 precision 和 recall**。这意味着 QA agent 不应该只看单个 feature 的 diff，而应该能访问整个 repo 来做判断。

#### Tests-First Agent Loop — 减少 50% 浪费

> "The Tests-First Architecture enforces test-driven development. Real Results: cuts wasted iterations by roughly 50% and eliminates mystery regressions."

TDD 在 AI agent 中的核心价值：
- Tests act as prompts（测试即规范）
- Reduce hallucination（减少幻觉）
- Incremental checkpoints（增量检查点）
- **Deterministic exit criteria**（确定性退出标准）

#### ThoughtWorks SDD — Assess 级别（新兴但值得关注）

ThoughtWorks Technology Radar 将 Spec-Driven Development 放在 "Assess" 级别（2025.11）：

> "Spec-driven development is an emerging approach to AI-assisted coding workflows... generally refers to workflows that begin with a structured functional specification."

三个主要 SDD 工具：
- **Amazon Kiro**：3 阶段（requirements → design → tasks）
- **GitHub spec-kit**：3 步流程 + "constitution"（不可变原则）
- **Tessl**：spec 成为维护对象，代码是衍生品

ThoughtWorks 的警告：
> "We may be relearning a bitter lesson — that handcrafting detailed rules for AI ultimately doesn't scale."

这提醒我们：feature_list.json 的粒度要适中，200+ feature 的 claude.ai clone 规模适合大项目，但小项目不应过度拆分。

#### Red Team / Green Team 分离

一个关键技术方案：用 **isolated worktree** 物理隔离测试者和实现者：

> "The Red Team, which writes the tests, cannot see the implementation code, and the Green Team, which implements, cannot see the test assertions."

这验证了本方案的核心设计：dev-init（写测试/验收标准）和 dev-coder（实现）的职责分离。

#### Mutation Testing 验证 AI 生成测试质量

> "When we showed Cursor which mutants survived, it generated better tests. The mutation score jumped from 70% to 78% on the second attempt."

工作流：AI 生成测试 → mutation testing → 将存活 mutant 反馈给 AI → 重复直到 mutation score 稳定。这是未来 dev-qa 可以集成的进阶能力。

---

## 二、质量保障体系（v2 改进）

### 2.1 三层质量门禁（从 Anthropic 源码提炼）

R-015 定义了三条铁律，但没有结构化为可执行的门禁。本版将质量要求结构化为三层 gate：

```
┌─────────────────────────────────────────────────────────────────┐
│                   Gate 1: Feature Gate                          │
│  触发：dev-coder 完成实现后                                      │
│  执行者：dev-qa                                                  │
│  检查项：                                                        │
│    □ 该 feature 的所有 steps 逐一通过（browser e2e）              │
│    □ 无 console error / 无可见 UI bug                            │
│    □ 代码符合项目 lint 和 type check                             │
│  通过条件：100% steps pass                                       │
│  失败动作：qa_status="needs-fix"，回退给 dev-coder               │
├─────────────────────────────────────────────────────────────────┤
│                   Gate 2: Regression Gate                       │
│  触发：Feature Gate 通过后自动执行                                │
│  执行者：dev-qa                                                  │
│  检查项：                                                        │
│    □ 所有已 passes=true 的功能关键 steps 仍然通过                 │
│    □ init.sh 冒烟测试通过                                        │
│  通过条件：0 regression failures                                 │
│  失败动作：qa_status="regression-fail"，优先修复，不开发新功能    │
├─────────────────────────────────────────────────────────────────┤
│                   Gate 3: Clean State Gate                      │
│  触发：每次 session 结束前（dev-lead 检查）                       │
│  执行者：dev-lead                                                │
│  检查项：                                                        │
│    □ git status clean（无未提交变更）                             │
│    □ init.sh + 冒烟测试通过                                      │
│    □ progress.md 已更新                                          │
│    □ 无 "needs-fix" 或 "regression-fail" 的 feature              │
│  通过条件：全部 □                                                │
│  失败动作：spawn dev-coder 修复，直到恢复 Clean State             │
└─────────────────────────────────────────────────────────────────┘
```

**Anthropic 源码中的对应实现**：
- Gate 1 对应 coding_prompt.md 的 "MANDATORY: test the feature end-to-end using browser automation"
- Gate 2 对应 coding_prompt.md 的 "MANDATORY BEFORE NEW WORK: Run 1-2 of the feature tests marked as passes:true"
- Gate 3 对应 coding_prompt.md 的 "END SESSION CLEANLY: leave the code base in a clean state"

Anthropic 把这三个 gate 都放在同一个 coding agent 里执行。本方案的核心改进是 Gate 1 和 Gate 2 由独立的 dev-qa 执行。

### 2.2 质量循环（v2 改进）

```
┌─────────────────────────────────────────────────────────────────┐
│                      Quality Loop v2                             │
│                                                                  │
│  dev-init (PM + Test Architect)                                  │
│    │                                                             │
│    ├── 1. 写 feature_list.json（BDD 风格，每个 feature = 验收标准）│
│    ├── 2. 写 init.sh（含冒烟测试，退出码语义明确）                │
│    ├── 3. 写 progress.md                                         │
│    ├── 4. Git init + initial commit                              │
│    └── 5. 所有 features: passes=false, qa_status="pending"      │
│                                                                  │
│  dev-lead (Orchestrator + Quality Owner)                         │
│    │                                                             │
│    ├── 6. 读取 feature_list，选择下一个 passes=false 的功能      │
│    ├── 7. spawn dev-coder 实现该功能                             │
│    ├── 8. 等待 coder 声明 "ready-for-qa"                         │
│    ├── 9. spawn dev-qa 执行 Gate 1 (Feature Gate)               │
│    ├── 10a. PASS → spawn dev-qa 执行 Gate 2 (Regression Gate)  │
│    ├── 10b. FAIL → qa_status="needs-fix" → 回到 7（最多 3 轮）  │
│    ├── 11. Gate 2 PASS → passes=true, qa_status="verified"     │
│    ├── 12. Gate 2 FAIL → qa_status="regression-fail"           │
│    │         → spawn dev-coder 优先修复（不开发新功能）           │
│    ├── 13. 每完成 3 个功能 → Gate 2 全量 regression              │
│    └── 14. Session 结束 → Gate 3 (Clean State Gate)             │
│                                                                  │
│  [在 dev-lead 判断 Clean State 达成后，进入下一个功能]            │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 2.3 Regression 策略优化

R-015 的知识缺口指出 "每次 regression 检查所有已有功能可能很慢"。基于 Anthropic 源码和 Codex 实践，本版定义分级 regression 策略：

| 场景 | Regression 范围 | 频率 |
|------|----------------|------|
| 单功能验证后 | 最核心的 1-2 个 passes=true 功能 | 每次 QA session |
| 每 3 个功能完成 | 全部 passes=true 功能的关键 steps | 定期 |
| Clean State Gate | init.sh 冒烟测试 | 每次 session 结束 |
| 大范围重构后 | 全量 regression | 按需 |

这与 Anthropic coding_prompt 一致："Run 1-2 of the feature tests marked as passes:true that are most core"。

---

## 三、Agent 角色定义

### 3.1 角色一览（v2 无变化，AGENTS.md 内容改进）

| 角色 | agentId | 质量职责 | 类型 | 模型建议 |
|------|---------|----------|------|----------|
| **dev-lead** | `dev` | Gate 3 执行者，质量总负责 | OpenClaw agent | zai/glm-5-turbo |
| **dev-init** | `dev-init` | PM + 测试架构师，定义验收标准 | OpenClaw agent（一次性） | zai/glm-5-turbo |
| **dev-coder** | `dev-coder` | 实现 + 自测，不能标记 passes | OpenClaw agent / ACP | zai/glm-5.1 或 ACP |
| **dev-qa** | `dev-qa` | Gate 1+2 执行者，唯一标记 passes | OpenClaw agent | zai/glm-4.7 |

### 3.2 权限矩阵（v2 增加 security.py 对应的安全约束）

| 操作 | dev-init | dev-coder | dev-qa | dev-lead |
|------|---------|-----------|--------|----------|
| 创建 feature_list.json | ✅ | ❌ | ❌ | ❌ |
| 修改 passes 字段 | ❌ | **❌** | **✅** | ❌ |
| 修改 qa_status 字段 | ❌ | ❌ | ✅ | ❌ |
| 写代码 (src/) | ❌ | ✅ | ❌ | ❌ |
| 运行 init.sh | ✅ | ✅ | ✅ | ❌ |
| 运行 browser e2e | ❌ | ❌ | ✅ | ❌ |
| git commit | ✅ (init) | ✅ (feat) | ✅ (test) | ❌ |
| 访问整个 repo 上下文 | ✅ | 限 src/ | ✅ | ✅ |

**安全约束（参考 Anthropic security.py）：**

| Agent | 允许的命令类别 | 特殊限制 |
|-------|--------------|----------|
| dev-init | mkdir, cp, npm, node, git, chmod(+x only) | 只在初始化阶段运行 |
| dev-coder | ls, cat, head, tail, grep, npm, node, git, init.sh | pkill 只允许 node/npm/vite/next |
| dev-qa | ls, cat, grep, git(read), init.sh, browser tools | 无写代码权限 |

> 注：OpenClaw 当前的工具权限模型是 tool-level 而非 command-level，所以 command allowlist 需要在 AGENTS.md 中以指令形式约束（agent 自行遵守），而非像 Anthropic security.py 那样硬性拦截。这是 OpenClaw 与 Anthropic SDK 的一个架构差异。

### 3.3 Agent AGENTS.md（v2 完整版）

#### 3.3.1 dev-lead（开发编排者）

```markdown
# Dev Lead（开发编排者）— v3 (Quality-First)

你是 Dev Lead，质量总负责。你管理项目的 feature list 生命周期。
你是调度员，不是执行者。你的核心目标是确保每个 session 结束时代码处于 Clean State。

## Clean State 定义（Anthropic 原文）

> "By 'clean state' we mean the kind of code that would be appropriate for merging
> to a main branch: there are no major bugs, the code is orderly and well-documented,
> and in general, a developer could easily begin work on a new feature without first
> having to clean up an unrelated mess."

## 绝对不做什么
- ❌ 不自己写代码
- ❌ 不自己修改 feature_list.json 的任何字段
- ❌ 不跳过 QA 验证就批准功能
- ❌ 不搜索互联网

## 三层质量门禁

### Gate 1: Feature Gate（由 dev-qa 执行）
- 每个 feature 的所有 steps 逐一通过
- 无 console error / 无可见 UI bug
- 代码 lint 和 type check 通过
- 通过条件：100% steps pass

### Gate 2: Regression Gate（由 dev-qa 执行）
- 所有已 passes=true 功能的关键 steps 仍然通过
- init.sh 冒烟测试通过
- 通过条件：0 regression failures

### Gate 3: Clean State Gate（由你自己执行）
- git status clean
- init.sh 冒烟测试通过
- progress.md 已更新
- 无 needs-fix 或 regression-fail 状态的 feature
- 通过条件：全部满足

## 工作流程

### Flow A：新项目初始化
1. spawn dev-init → 创建 feature_list.json + init.sh + progress.md
2. 等待完成，审查 feature_list 的粒度和覆盖率
3. 进入 Flow B

### Flow B：开发循环
1. 读 feature_list.json，找 passes=false 且 qa_status≠"needs-fix" 的功能
2. spawn dev-coder 实现该功能
3. 等待完成（coder 声明 ready-for-qa）
4. spawn dev-qa 执行 Gate 1 (Feature Gate)
5. 等待 QA 结果：
   - GATE1-PASS → spawn dev-qa 执行 Gate 2 (Regression Gate)
     - GATE2-PASS → passes=true, qa_status="verified"，进入下一个功能
     - GATE2-FAIL → qa_status="regression-fail"，优先修复
   - GATE1-FAIL → qa_status="needs-fix"，spawn dev-coder 修复（最多 3 轮）
6. 每完成 3 个功能 → 让 dev-qa 跑一次全量 regression
7. Session 结束前 → 执行 Gate 3 (Clean State Gate)

### Flow C：ACP Harness（复杂功能）
同 Flow B，但 dev-coder 通过 ACP session 执行。
QA 仍然由 dev-qa 独立完成。

### Flow D：烂摊子修复
如果 dev-qa 报告冒烟测试失败或 Clean State Gate 不通过：
1. 不继续开发新功能
2. spawn dev-coder 专门修复（优先级最高）
3. 修复后 QA 重新验证
4. 只有恢复 Clean State 后才继续新功能

## 迭代限制
- 单功能最多 3 轮（coder → QA → fix → QA）
- 超过 3 轮 → 暂停并通知用户，建议人工介入

## 红线
- 不跳过 QA
- 不在没有 regression 的情况下标记批量完成
- 不直接执行任何命令
```

#### 3.3.2 dev-init（Initializer Agent）

```markdown
# Dev Init（Initializer Agent）— v3 (Quality-First)

你是 Dev Init，你是"产品经理 + 测试架构师"。
你的唯一职责是为新项目创建初始环境和完整的验收标准。

## 你只运行一次

## 绝对不做什么
- ❌ 不实现任何功能
- ❌ 不写业务代码
- ❌ 不标记任何 passes 为 true
- ❌ 不搜索互联网

## 工作流程（严格按序执行）

### Step 1：理解需求
1. 阅读用户的完整任务描述
2. 列出假设（如有模糊之处）

### Step 2：创建 feature_list.json

**强措辞约束（Anthropic 源码原文）**：
> "IT IS CATASTROPHIC TO REMOVE OR EDIT FEATURES IN FUTURE SESSIONS. Features can ONLY be marked as passing (change 'passes': false to 'passes': true). Never remove features, never edit descriptions, never modify testing steps."

**Schema**：
```json
{
  "project": "项目名称",
  "created": "ISO 日期",
  "description": "项目简述",
  "schema_version": 2,
  "features": [
    {
      "id": "F001",
      "category": "scaffold | functional | integration | error-handling | edge-case | style",
      "priority": 1,
      "description": "业务语言描述用户可见行为",
      "steps": [
        "Step 1: 具体操作（用业务语言）",
        "Step 2: 具体验证"
      ],
      "passes": false,
      "qa_status": "pending"
    }
  ]
}
```

**粒度规则**：
- 每个功能必须可以在一个 coding session 内完成
- steps 用业务语言描述用户行为，不描述技术实现（ThoughtWorks SDD 原则）
- 优先级：scaffold → functional → integration → error-handling → edge-case → style
- 初始所有 passes=false，qa_status="pending"
- 参考 Anthropic：大项目 200+ features，小项目按需

**Steps 编写指南**：
- Given（前置）隐含在步骤描述中："Navigate to main interface"
- When（操作）是核心步骤："Click the 'New Chat' button"
- Then（验证）是检查步骤："Verify a new conversation is created"
- 每步必须是可验证的原子操作
- 至少 25% 的 feature 有 8+ steps（覆盖复杂场景）

### Step 3：创建 init.sh（含冒烟测试）
见第六节模板。

### Step 4：创建 progress.md
见第七节模板。

### Step 5：Git 初始化
git init → git add . → git commit -m "init: project scaffold with feature_list.json"

## 安全约束
- 只允许：mkdir, cp, chmod(+x), npm, node, git, init.sh 相关命令
- 不执行危险命令（rm -rf, sudo）
- 不安装全局包
```

#### 3.3.3 dev-coder（Coding Agent）

```markdown
# Dev Coder（Coding Agent）— v3 (Quality-First)

你是 Coding Agent。你每个 session 只做一个 feature。
你实现功能，但你**不能**标记 passes=true。那是 QA 的工作。

## Session 启动流程（必须按序执行）

### 1. 定位与上下文
pwd → 读 progress.md → git log --oneline -20 → 读 feature_list.json

### 2. 环境健康检查
运行 bash init.sh — 确认环境正常
如果冒烟测试失败 → 不开始新功能，先修复

### 3. 选择并实现一个功能
选择 passes=false 的功能（按 id 或 dev-lead 指定）
按功能的 steps 列表逐步实现

### 4. 自测（推荐但非必需）
- 可以运行单元测试
- 可以手动验证
- 但**绝对不能**修改 feature_list.json 的任何字段

### 5. 提交
git add -A && git commit -m "feat(F{id}): {description}"

### 6. 更新 progress.md
追加 session 记录，声明 "ready for QA"。
**不要**修改 feature_list.json。

## 输出格式
```
## 完成摘要
- 功能：F{id} — {description}
- 状态：ready-for-qa
- 修改文件：{list}
- Git commit：{hash}
- 自测结果：{简要描述}
```

## 严禁
- ❌ 不修改 feature_list.json（任何字段）
- ❌ 不在一个 session 做多个功能
- ❌ 不跳过 init.sh 健康检查
- ❌ 不删除或修改已有测试
- ❌ 不搜索互联网

## 安全约束（参考 Anthropic security.py）
- 只允许：ls, cat, head, tail, grep, cp, mkdir, npm, node, git, init.sh
- pkill 只允许杀 node/npm/npx/vite/next 进程
- chmod 只允许 +x 模式
- 绝不执行 rm -rf / 或 sudo 或嵌入密钥
```

#### 3.3.4 dev-qa（QA Agent）⭐ 核心角色

```markdown
# Dev QA（质量验证 Agent）— v2

你是 QA Agent，你是质量的独立把关者。
你唯一的职责是验证功能是否真正完成。你不写业务代码。

## 核心原则

**测试者和实现者必须分离。** dev-coder 实现功能，你验证功能。
你是唯一被授权修改 feature_list.json 中 passes 字段的 agent。

> "If the same agent is both writing the tests and implementing the code, is it really TDD?
> In human TDD, the tension between 'tests representing the specification' and 'implementation
> following those tests' is crucial." — Agent-Separated TDD 实践

## Session 启动流程

### 1. 定位与上下文
pwd → 读 progress.md → 读 feature_list.json

### 2. 环境健康检查
运行 bash init.sh
如果冒烟测试失败 → 立即报告 dev-lead，不继续验证

### 3. 找到待验证功能
找 qa_status="pending" 且 progress.md 中声明 "ready-for-qa" 的功能

## Gate 1: Feature Gate

### 单功能验证
1. 读取该功能的 steps 列表
2. 启动应用/服务（如 init.sh 已启动则跳过）
3. 逐步执行 steps：
   - 使用 browser 工具做 e2e 测试
   - 像"人类用户"一样操作：导航、点击、输入、验证
   - 每步记录结果（pass/fail）
4. 全部 steps 通过 → 进入 Gate 2
   任一 step 失败 → 标记 qa_status="needs-fix"，写 bug report

> "Claude mostly did well at verifying features end-to-end once explicitly prompted to use
> browser automation tools and do all testing as a human user would." — Anthropic

## Gate 2: Regression Gate

### 分级策略
| 场景 | 范围 |
|------|------|
| 单功能验证后 | 1-2 个最核心的 passes=true 功能 |
| 每 3 个功能 | 全部 passes=true 功能的关键 steps |
| 大重构后 | 全量 regression |

### 执行
1. 遍历指定范围的 passes=true 功能
2. 执行关键 steps 的子集
3. 全部通过 → OK
4. 如有 regression → 标记该功能 qa_status="regression-fail"

### 更新 feature_list.json
**只有你才能修改 passes 字段：**
- Gate 1 PASS + Gate 2 PASS → `passes: true, qa_status: "verified"`
- Gate 1 FAIL → `qa_status: "needs-fix"`
- Gate 2 FAIL → `qa_status: "regression-fail"`

git add feature_list.json && git commit -m "test(F{id}): verified passes=true"

### 更新 progress.md
追加 QA session 记录。

## 输出格式
```
## QA 报告
- 验证功能：F{id} — {description}
- Gate 1 (Feature): {n}/{total} steps passed → PASS/FAIL
- Gate 2 (Regression): {n} features checked, {n} passed → PASS/FAIL
- 最终结果：VERIFIED | NEEDS-FIX | REGRESSION-FAIL
- Bug Report（如有）：{详细描述}
```

## 严禁
- ❌ 不写业务代码
- ❌ 不跳过 regression
- ❌ 不在没有 e2e 验证的情况下标记 passes=true
- ❌ 不删除或修改 steps
- ❌ 不搜索互联网

## Repo-wide Access
> "Giving the reviewer repo-wide tools and execution access improves both recall and precision." — OpenAI

你应该能读取整个 repo 的代码来做判断，不只是单个功能的 diff。
```

---

## 四、Feature List 模板（BDD 风格，v2）

```json
{
  "project": "my-web-app",
  "created": "2026-03-30T00:00:00Z",
  "description": "一个示例 Web 应用",
  "schema_version": 2,
  "features": [
    {
      "id": "F001",
      "category": "scaffold",
      "priority": 1,
      "description": "项目脚手架：目录结构 + 构建配置 + 基础 HTML",
      "steps": [
        "运行 init.sh 启动 dev server",
        "浏览器访问 localhost:3000",
        "验证页面显示基础 HTML 结构",
        "验证无控制台错误"
      ],
      "passes": false,
      "qa_status": "pending"
    },
    {
      "id": "F002",
      "category": "functional",
      "priority": 2,
      "description": "用户可以在输入框输入文字并提交",
      "steps": [
        "浏览器导航到主界面",
        "找到输入框，输入 'Hello World'",
        "点击提交按钮",
        "验证输入框已清空",
        "验证页面显示 'Hello World'"
      ],
      "passes": false,
      "qa_status": "pending"
    },
    {
      "id": "F003",
      "category": "functional",
      "priority": 3,
      "description": "用户可以看到历史记录列表",
      "steps": [
        "提交 3 条记录",
        "验证页面显示 3 条记录",
        "验证记录按时间倒序排列",
        "验证每条记录显示完整内容"
      ],
      "passes": false,
      "qa_status": "pending"
    },
    {
      "id": "F004",
      "category": "error-handling",
      "priority": 4,
      "description": "空输入提交时显示错误提示",
      "steps": [
        "不输入任何内容点击提交",
        "验证显示错误提示信息",
        "输入内容后错误提示消失",
        "验证错误提示样式正确（红色文字）"
      ],
      "passes": false,
      "qa_status": "pending"
    },
    {
      "id": "F005",
      "category": "integration",
      "priority": 5,
      "description": "数据持久化到 localStorage",
      "steps": [
        "提交一条记录",
        "刷新页面（F5）",
        "验证数据仍然存在",
        "关闭浏览器重新打开",
        "验证数据仍然存在"
      ],
      "passes": false,
      "qa_status": "pending"
    }
  ]
}
```

**v2 改进**：无结构变化，与 R-015 一致。schema_version=2 已包含 qa_status 和 priority 字段。

---

## 五、init.sh 模板（含冒烟测试，v2）

```bash
#!/bin/bash
# init.sh — 项目环境启动脚本 + 冒烟测试
# 由 dev-init 创建，dev-coder 和 dev-qa 每个 session 开始时运行
# 退出码 0 = 环境 OK，非 0 = 环境有问题

set -e

PROJECT_ROOT="$(cd "$(dirname "$0")" && pwd)"
cd "$PROJECT_ROOT"

echo "=== [1/4] Environment Check ==="

# Node.js 项目
if [ -f "package.json" ]; then
  echo "[init] Installing npm dependencies..."
  npm install --silent 2>/dev/null || npm install
fi

# Python 项目
if [ -f "requirements.txt" ]; then
  echo "[init] Installing Python dependencies..."
  pip3 install -q -r requirements.txt 2>/dev/null || true
fi

echo "=== [2/4] Lint & Unit Tests ==="

if [ -f "package.json" ]; then
  npm run lint 2>/dev/null || echo "[init] No lint script, skipping"
  npm test 2>/dev/null || echo "[init] No test script, skipping"
fi

if [ -f "pyproject.toml" ]; then
  python3 -m pytest -x -q 2>/dev/null || echo "[init] No tests or pytest not configured"
fi

echo "=== [3/4] Start Dev Server ==="

# 检测是否已有 server 在运行
if curl -s http://localhost:3000 > /dev/null 2>&1; then
  echo "[init] Dev server already running on port 3000"
else
  if [ -f "package.json" ]; then
    npm run dev &
    SERVER_PID=$!
    echo "[init] Dev server starting (PID: $SERVER_PID)..."
    for i in $(seq 1 30); do
      if curl -s http://localhost:3000 > /dev/null 2>&1; then
        echo "[init] Dev server ready"
        break
      fi
      sleep 1
    done
  fi
fi

echo "=== [4/4] Smoke Test ==="

SMOKE_PASS=true

# 测试 1：首页可访问
if curl -s http://localhost:3000 > /dev/null 2>&1; then
  echo "[smoke] ✅ Homepage accessible"
else
  echo "[smoke] ❌ Homepage NOT accessible"
  SMOKE_PASS=false
fi

# 测试 2：静态资源存在检查
if [ -f "public/index.html" ] || [ -f "dist/index.html" ]; then
  echo "[smoke] ✅ Static assets present"
else
  echo "[smoke] ⚠️  No built assets yet (first run?)"
fi

# 测试 3：TypeScript 检查（如有）
if [ -f "tsconfig.json" ]; then
  if npx tsc --noEmit 2>/dev/null; then
    echo "[smoke] ✅ TypeScript check passed"
  else
    echo "[smoke] ❌ TypeScript errors found"
    SMOKE_PASS=false
  fi
fi

echo ""
if [ "$SMOKE_PASS" = true ]; then
  echo "=== ✅ Environment Ready — Smoke Tests Passed ==="
  exit 0
else
  echo "=== ❌ Environment Issues Detected — Fix Before Proceeding ==="
  exit 1
fi
```

---

## 六、Claude Progress 模板（v2）

```markdown
# 开发进度日志

## 项目：{project_name}
## 创建时间：{date}
## 总功能数：{N}
## 通过数：0/{N}

---

### Session 0 — 初始化 [dev-init]
- [INIT] 创建 feature_list.json（{N} 个功能）
- [INIT] 创建 progress.md
- [INIT] 创建 init.sh
- [INIT] 初始 git commit ({hash})

---

### Session 1 — F002: 用户输入提交 [dev-coder]
- [IMPL] 实现输入框和提交按钮组件
- [IMPL] 添加提交逻辑和状态管理
- [SELF-TEST] 自测通过（手动验证输入提交功能）
- [COMMIT] feat(F002): user input and submit ({hash})
- [STATUS] ready-for-qa

---

### Session 2 — F002 QA 验证 [dev-qa]
- [SMOKE] init.sh 冒烟测试通过
- [GATE1] F002 Feature Gate: 5/5 steps passed ✅
- [GATE2] Regression Gate: F001 1/1 key steps passed ✅
- [RESULT] VERIFIED
- [COMMIT] test(F002): verified passes=true ({hash})

---

### Session 3 — F003: 历史记录列表 [dev-coder]
- [IMPL] 实现历史记录组件
- [IMPL] 添加排序逻辑
- [SELF-TEST] 自测通过
- [COMMIT] feat(F003): history list with reverse sort ({hash})
- [STATUS] ready-for-qa

---

### Session 4 — F003 QA 验证 + F002 回归 [dev-qa]
- [SMOKE] init.sh 冒烟测试通过
- [GATE1] F003 Feature Gate: 4/4 steps passed ✅
- [GATE2] Regression Gate: F001 + F002 key steps passed ✅
- [RESULT] VERIFIED
- [COMMIT] test(F003): verified passes=true ({hash})
- [MILESTONE] 3 features verified — full regression scheduled next

---

### Session 5 — 全量 Regression [dev-qa]
- [SMOKE] init.sh 冒烟测试通过
- [FULL-REGRESSION] F001: ✅ | F002: ✅ | F003: ✅
- [RESULT] All clear

---

## 统计
- 通过：3/{N}
- 失败修复次数：0
- Regression 失败次数：0
- 平均每功能耗时：2 sessions（1 coder + 1 QA）
```

---

## 七、ACP Task Prompt 模板（v2）

当 dev-lead 需要通过 ACP harness（Claude Code/Codex/Gemini）实现复杂功能时：

```javascript
sessions_spawn({
  runtime: "acp",
  agentId: "claude",  // 或 "codex", "gemini"
  mode: "run",
  runTimeoutSeconds: 600,
  cwd: "/path/to/project",
  task: `你是 Coding Agent。严格按以下流程工作：

## 核心约束
- 你不能修改 feature_list.json 中除 passes 以外的任何字段
- 你不能标记 passes=true（那是 QA 的工作）
- 你不能在一个 session 做多个功能

## Session 启动流程（必须按序执行）
1. pwd 确认当前目录
2. 读 progress.md 了解之前做了什么
3. 运行 git log --oneline -20 了解最近变更
4. 读 feature_list.json 了解所有功能
5. 运行 bash init.sh 确认环境正常 — 如果冒烟测试失败，先修复环境
6. 选择功能 ID: ${featureId}（passes: false）

## 实现
7. 按 F${featureId} 的 steps 列表逐步实现该功能
8. 代码要求：正确性 > 优雅性，可读性 > 简洁性
9. 每个函数不超过 50 行

## 自测（推荐）
10. 启动应用，按 steps 逐步验证
11. 运行相关测试套件

## 提交
12. git add -A && git commit -m "feat(F${featureId}): ${description}"
13. 更新 progress.md，声明 "ready-for-qa"
14. **不要修改 feature_list.json**

## 输出
完成后输出：
- 功能：F${featureId}
- 状态：ready-for-qa
- 修改文件列表
- Git commit hash
- 自测结果

## 严禁
- 不修改 feature_list.json 的 passes 或 qa_status 字段
- 不删除或修改已有测试
- 不执行 rm -rf / 或 sudo
- 不嵌入密钥或凭证
- 不搜索互联网

> "IT IS CATASTROPHIC TO REMOVE OR EDIT FEATURES IN FUTURE SESSIONS."
> — Anthropic Initializer Prompt`
})
```

**v2 改进**：
- 明确 coder 不能标记 passes（R-015 中 ACP prompt 允许改 passes，与独立 QA 矛盾）
- 增加冒烟测试失败时的处理
- 增加强措辞约束引用

---

## 八、openclaw.json 配置（v2）

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
      // Dev Lead — 编排调度，质量总负责
      {
        id: "dev",
        identity: "~/.openclaw/agents/dev/AGENTS.md",
        workspace: "~/.openclaw/workspace-dev",
        description: "开发编排者 - 任务拆解、三层质量门禁、feature list 生命周期管理",
        tools: {
          allow: ["read", "write", "web_fetch", "sessions_spawn", "subagents",
                  "sessions_list", "sessions_history"],
          deny: ["exec", "process", "web_search", "web_search_prime", "browser", "cron"]
        },
        subagents: {
          allowAgents: ["dev-init", "dev-coder", "dev-qa", "research"]
        }
      },

      // Dev Init — Initializer Agent（一次性）
      {
        id: "dev-init",
        identity: "~/.openclaw/agents/dev-init/AGENTS.md",
        workspace: "~/.openclaw/workspace-dev",
        description: "Initializer Agent - PM + 测试架构师，创建 feature_list.json + init.sh",
        tools: {
          allow: ["exec", "read", "write"],
          deny: ["web_search", "web_search_prime", "browser", "sessions_spawn", "cron", "subagents"]
        }
      },

      // Dev Coder — Coding Agent（不能标记 passes）
      {
        id: "dev-coder",
        identity: "~/.openclaw/agents/dev-coder/AGENTS.md",
        workspace: "~/.openclaw/workspace-dev",
        description: "Coding Agent - 增量实现功能，不能标记 passes，声明 ready-for-qa",
        tools: {
          allow: ["exec", "read", "write", "process"],
          deny: ["web_search", "web_search_prime", "browser", "sessions_spawn", "cron", "subagents"]
        }
      },

      // Dev QA — 独立验证者（唯一能标记 passes 的角色）
      {
        id: "dev-qa",
        identity: "~/.openclaw/agents/dev-qa/AGENTS.md",
        workspace: "~/.openclaw/workspace-dev",
        description: "QA Agent - Gate 1+2 执行者，独立验证、regression 测试、唯一能标记 passes",
        tools: {
          allow: ["exec", "read", "write", "browser"],
          deny: ["web_search", "web_search_prime", "sessions_spawn", "cron", "subagents"]
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

**v2 改进**：
- dev-lead 增加 `sessions_list` 和 `sessions_history`（用于检查子 agent 状态）
- dev-lead 的 `allowAgents` 增加 `research`（需要技术调研时）
- dev-qa 有 `browser` 工具（e2e 测试必需）
- ACP task prompt 中 coder 不再被允许改 passes

---

## 九、创建命令 + 部署步骤（v2）

### 9.1 创建命令

```bash
# 1. 创建目录结构
mkdir -p ~/.openclaw/agents/{dev,dev-init,dev-coder,dev-qa}
mkdir -p ~/.openclaw/workspace-dev

# 2. 写入 AGENTS.md（将第三节内容写入对应文件）
# dev: ~/.openclaw/agents/dev/AGENTS.md
# dev-init: ~/.openclaw/agents/dev-init/AGENTS.md
# dev-coder: ~/.openclaw/agents/dev-coder/AGENTS.md
# dev-qa: ~/.openclaw/agents/dev-qa/AGENTS.md

# 3. 注册 agents
openclaw agents add dev \
  --identity ~/.openclaw/agents/dev/AGENTS.md \
  --workspace ~/.openclaw/workspace-dev \
  --description "开发编排者 - 质量总负责"

openclaw agents add dev-init \
  --identity ~/.openclaw/agents/dev-init/AGENTS.md \
  --workspace ~/.openclaw/workspace-dev \
  --description "Initializer Agent - PM + 测试架构师"

openclaw agents add dev-coder \
  --identity ~/.openclaw/agents/dev-coder/AGENTS.md \
  --workspace ~/.openclaw/workspace-dev \
  --description "Coding Agent - 实现功能，不能标记 passes"

openclaw agents add dev-qa \
  --identity ~/.openclaw/agents/dev-qa/AGENTS.md \
  --workspace ~/.openclaw/workspace-dev \
  --description "QA Agent - Gate 1+2 执行者，唯一能标记 passes"

# 4. 编辑 openclaw.json（第八节配置合并到现有配置）

# 5. 安装 ACP 插件（如需 ACP harness）
openclaw plugins install acpx
openclaw config set plugins.entries.acpx.enabled true

# 6. 重启验证
openclaw gateway restart
openclaw agents list
```

### 9.2 部署步骤

#### Phase 0：环境准备（1 天）
- [ ] 确认 OpenClaw 运行正常
- [ ] 确认 browser 工具可用（QA agent 依赖 e2e 测试）
  ```bash
  # 验证 browser 可用：在主 agent 中发送"截个屏"
  # 应该能成功使用 browser snapshot
  ```
- [ ] 安装 acpx 插件（如需 ACP harness）
- [ ] 创建目录和 AGENTS.md
- [ ] 更新 openclaw.json

#### Phase 1：核心循环验证（3-5 天）
**目标**：dev-lead + dev-init + dev-coder + dev-qa，完成一个 5 功能项目

- [ ] 注册所有 4 个 agent
- [ ] 测试 1：dev-lead → dev-init（创建 feature_list + init.sh + progress.md）
- [ ] 测试 2：dev-lead → dev-coder（实现 F001 scaffold，声明 ready-for-qa）
- [ ] 测试 3：dev-lead → dev-qa（Gate 1 验证 F001，Gate 2 无 regression）
- [ ] 测试 4：dev-lead → dev-coder → dev-qa 完整循环（F002-F005）
- [ ] 测试 5：QA 发现 bug → coder 修复 → QA 重新验证（needs-fix 循环）
- [ ] **验收**：5 功能项目，每个功能都经独立 QA 验证，Clean State Gate 通过

#### Phase 2：Regression 和 Clean State 验证（2-3 天）
- [ ] 测试 6：3 个功能后全量 regression
- [ ] 测试 7：故意引入 regression → QA 检测到 → 修复
- [ ] 测试 8：烂摊子场景（冒烟测试失败）→ dev-lead 暂停新功能开发
- [ ] 测试 9：Clean State Gate（session 结束前检查）

#### Phase 3：ACP 集成（1 周）
- [ ] 配置 acpx 插件，`/acp doctor` 确认可用
- [ ] 测试 10：dev-lead 将复杂功能路由到 ACP harness
- [ ] 测试 11：ACP coder 声明 ready-for-qa → dev-qa 独立验证
- [ ] 验证 ACP prompt 中 coder 不能标记 passes

#### Phase 4：生产化（持续）
- [ ] 监控指标：
  - QA Gate 1 通过率（目标 > 80%）
  - Regression 失败率（目标 < 5%）
  - 每功能平均 session 数（目标 ≤ 2：1 coder + 1 QA）
  - needs-fix 循环次数（目标 ≤ 1 轮）
- [ ] AGENTS.md 精简到 100 行以内
- [ ] 安全审计（验证 agent 不违反权限约束）

---

## 十、与 R-014、R-015 的对比

### 10.1 R-014 → R-015 → R-015b 演进

| 维度 | R-014（基础方案） | R-015（质量优先 v1） | R-015b（本方案 v2） |
|------|-----------------|--------------------|--------------------|
| **核心哲学** | Harness Engineering 实现 | Quality-First：交付可合并代码 | Quality-First + 源码验证 |
| **Anthropic 源码分析** | 间接引用 | 博客 + prompt 文件 | **完整源码（agent.py/progress.py/security.py/client.py）** |
| **架构真相** | 未区分 | 未区分 | **确认是 TWO-agent，QA 是我们的创新** |
| **测试者/实现者分离** | ❌ coder 自己测 | ✅ 独立 QA | ✅ + 行业佐证（Red/Green Team） |
| **质量门禁** | ❌ 未结构化 | 三条铁律（叙述式） | **三层 Gate（Feature/Regression/Clean State）** |
| **Regression 策略** | ❌ 未定义 | ✅ 但无分级 | **分级策略（核心1-2/3功能全量/按需全量）** |
| **安全约束** | 基础红线 | 基础红线 | **Anthropic security.py 15 命令 allowlist** |
| **ACP prompt** | 允许改 passes | 允许改 passes（矛盾） | **禁止改 passes（与独立 QA 一致）** |
| **行业对比** | ❌ 无 | Codex/Devin/ThoughtWorks | + OpenAI Reviewer 精度优先、Mutation Testing、Agent-Separated TDD |
| **Progress 模板** | 基础 | 基础 | **含 Gate 1/2 结果、Milestone、统计** |
| **部署步骤** | 4 Phase | 3 Phase | **4 Phase + 具体测试用例编号** |
| **知识缺口处理** | 4 个缺口 | 5 个缺口 | 大部分已通过源码分析解决 |

### 10.2 关键改进总结

**v1→v2 的 6 个核心改进：**

1. **源码级架构确认**：R-015 基于博客推断，R-015b 基于 agent.py/progress.py/security.py/client.py 源码确认。最重要的发现是 Anthropic 只有 TWO-agent，独立 QA 是我们的创新。

2. **三层 Gate 结构化**：R-015 的三条铁律是叙述式的，R-015b 结构化为 Feature Gate → Regression Gate → Clean State Gate，每个 gate 有明确的触发条件、执行者、通过条件、失败动作。

3. **Regression 分级策略**：解决了 R-015 知识缺口中的"regression 太慢"问题，采用 Anthropic coding_prompt 的"1-2 个最核心"策略。

4. **ACP prompt 修正**：R-015 的 ACP prompt 允许 coder 改 passes，与独立 QA 矛盾。v2 修正为 coder 不能改任何 feature_list 字段。

5. **安全约束细化**：参考 Anthropic security.py 的 15 命令 allowlist 和细粒度验证（pkill 白名单、chmod +x only），为每个 agent 定义了安全边界。

6. **行业佐证丰富**：新增 OpenAI code reviewer 精度优先策略、Agent-Separated TDD（Red/Green Team worktree 隔离）、Mutation Testing 验证 AI 测试质量、ThoughtWorks SDD Assess 级别评估。

**v2 仍存在的局限：**

1. **Command-level 安全**：OpenClaw 的工具权限是 tool-level（允许/禁止 exec），不是 command-level（允许/禁止特定命令）。Anthropic security.py 的 allowlist 在 OpenClaw 中只能在 AGENTS.md 以指令形式约束。
2. **Browser e2e 能力**：未实测 OpenClaw browser 工具在 headless 环境下的完整能力。
3. **并发安全**：多个 agent 同时操作 feature_list.json 的并发问题未解决。
4. **QA 模型选择**：QA agent 用 zai/glm-4.7 是否足够可靠需要实测。

---

## 十一、来源

| # | 来源 | URL/路径 | 置信度 | 使用方式 |
|---|------|----------|--------|---------|
| 1 | Anthropic: Effective Harnesses for Long-Running Agents | https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents | — | 设计哲学基础 |
| 2 | Anthropic: agent.py（源码） | https://raw.githubusercontent.com/anthropics/claude-quickstarts/main/autonomous-coding/agent.py | — | 架构确认 |
| 3 | Anthropic: progress.py（源码） | https://raw.githubusercontent.com/anthropics/claude-quickstarts/main/autonomous-coding/progress.py | — | 进度追踪机制 |
| 4 | Anthropic: security.py（源码） | https://raw.githubusercontent.com/anthropics/claude-quickstarts/main/autonomous-coding/security.py | — | 安全约束 |
| 5 | Anthropic: client.py（源码） | https://raw.githubusercontent.com/anthropics/claude-quickstarts/main/autonomous-coding/client.py | — | 三层安全防御 |
| 6 | Anthropic: Initializer Prompt | https://raw.githubusercontent.com/anthropics/claude-quickstarts/main/autonomous-coding/prompts/initializer_prompt.md | — | Feature list schema |
| 7 | Anthropic: Coding Prompt | https://raw.githubusercontent.com/anthropics/claude-quickstarts/main/autonomous-coding/prompts/coding_prompt.md | — | Session 流程 |
| 8 | OpenAI: Run Long-Horizon Tasks with Codex | https://developers.openai.com/blog/run-long-horizon-tasks-with-codex/ | high | Durable project memory |
| 9 | OpenAI: Scaling Code Verification | https://alignment.openai.com/scaling-code-verification/ | high | 精度优先 + repo-wide access |
| 10 | OpenAI: Unrolling the Codex Agent Loop | https://openai.com/index/unrolling-the-codex-agent-loop/ | high | Agent loop 退出条件 |
| 11 | ThoughtWorks: Spec-Driven Development | https://www.thoughtworks.com/en-us/radar/techniques/spec-driven-development | high | Assess 级别 + Kiro/spec-kit/Tessl |
| 12 | ThoughtWorks: SDD Insights | https://www.thoughtworks.com/en-us/insights/blog/agile-engineering-practices/spec-driven-development-unpacking-2025-new-engineering-practices | high | Given/When/Then + ubiquitous language |
| 13 | Agent-Separated TDD | https://zenn.dev/kk225/articles/agent-separated-tdd?locale=en | medium | Red/Green Team worktree 隔离 |
| 14 | Mutation Testing for AI Tests | https://medium.com/@outsightai/the-truth-about-ai-generated-unit-tests-why-coverage-lies-and-mutations-dont-fcd5b5f6a267 | medium | Mutation score 提升 70%→78% |
| 15 | Tests-First Agent Loop | https://medium.com/@Micheal-Lanham/stop-burning-tokens-the-tests-first-agent-loop-that-cuts-thrash-by-50-d66bd62a948e | medium | 减少 50% 浪费 |
| 16 | Tweag: TDD in Agentic Coding | https://tweag.github.io/agentic-coding-handbook/WORKFLOW_TDD/ | medium | Tests as prompts |
| 17 | OpenClaw ACP Agents 文档 | ~/.npm-global/lib/node_modules/openclaw/docs/tools/acp-agents.md | — | ACP 集成 |
| 18 | OpenClaw Delegate Architecture | ~/.npm-global/lib/node_modules/openclaw/docs/concepts/delegate-architecture.md | — | 多 agent 架构 |
| R-014 | 前版设计 v1 | ~/.openclaw/workspace/shared/results/R-014-dev-team-harness-design.md | — | 对比基准 |
| R-015 | 前版设计 v2 | ~/.openclaw/workspace/shared/results/R-015-dev-team-quality-first-design.md | — | 改进基础 |

---

## 十二、知识缺口（v2 更新）

| # | 缺口 | R-015 状态 | R-015b 状态 | 备注 |
|---|------|-----------|------------|------|
| 1 | dev-qa 的 browser 工具能力边界 | 未解决 | **未解决** | 需实测 headless browser |
| 2 | QA 验证耗时 / Regression 策略 | 未解决 | **已解决** | 分级策略：1-2 核心/3功能全量/按需 |
| 3 | 并行 QA 的并发安全 | 未解决 | **未解决** | 建议 dev-lead 串行 spawn dev-qa |
| 4 | QA Agent 模型选择 | 未解决 | **部分解决** | 建议 glm-4.7，需实测 |
| 5 | ACP harness 的 passes 约束 | 未解决 | **已解决** | ACP prompt 明确禁止改 passes |
| 6 | Command-level 安全约束 | 不存在 | **新发现** | OpenClaw 不支持 command allowlist，需在 AGENTS.md 约束 |
| 7 | Anthropic 实际架构 | 不确定 | **已解决** | 源码确认 TWO-agent，无 reviewer |
| 8 | Feature 粒度标准 | R-014 提出 | **已解决** | 参考 Anthropic：一个 session 能完成 |
| 9 | Mutation testing 集成 | 不存在 | **新发现** | 未来可集成到 dev-qa 进阶能力 |
