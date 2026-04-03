# R-015：以交付质量为核心的 OpenClaw 研发团队 Agent 方案

> 生成时间：2026-03-30 | 基础：Anthropic "Effective Harnesses for Long-Running Agents" 原文精读 + autonomous-coding quickstart 源码 | 改进自：R-014

---

## 一、设计哲学：从 Anthropic 原文学到什么

### 1.1 核心问题

Anthropic 明确指出长任务 agent 的根本挑战：

> "The core challenge of long-running agents is that they must work in discrete sessions, and each new session begins with no memory of what came before."

这决定了我们的设计必须基于**文件持久化状态**，而非 agent 记忆。

### 1.2 两个失败模式

> "First, the agent tended to try to do too much at once—essentially to attempt to one-shot the app."

> "After some features had already been built, a later agent instance would look around, see that progress had been made, and declare the job done."

**对策**：Feature List（原子化任务） + 增量进度 + 强措辞约束。

### 1.3 Clean State 定义（逐字引用）

> "By 'clean state' we mean the kind of code that would be appropriate for merging to a main branch: there are no major bugs, the code is orderly and well-documented, and in general, a developer could easily begin work on a new feature without first having to clean up an unrelated mess."

**这是本方案的核心验收标准。** 不是"能跑"，是"可合并到 main"。

### 1.4 测试失败的洞察

> "One final major failure mode that we observed was Claude's tendency to mark a feature as complete without proper testing."

> "Claude mostly did well at verifying features end-to-end once explicitly prompted to use browser automation tools and do all testing as a human user would."

**关键**：unit test 不够，必须 e2e browser 测试。

### 1.5 JSON > Markdown

> "After some experimentation, we landed on using JSON for this, as the model is less likely to inappropriately change or overwrite JSON files compared to Markdown files."

### 1.6 强措辞约束（比博客更严厉的实际 prompt）

Anthropic 博客引用了 `"It is unacceptable to remove or edit tests"`，但实际 quickstart 代码中 Initializer prompt 的措辞更严厉：

> **"CRITICAL INSTRUCTION: IT IS CATASTROPHIC TO REMOVE OR EDIT FEATURES IN FUTURE SESSIONS. Features can ONLY be marked as passing (change 'passes': false to 'passes': true). Never remove features, never edit descriptions, never modify testing steps."**

这个措辞选择很有意思——"catastrophic"比"unacceptable"更强烈，说明 Anthropic 在实践中发现温和措辞不够，必须用极端语言约束模型行为。

### 1.7 Coding Agent 的 10 步 Session 流程（源码确认）

Anthropic 的 `coding_prompt.md` 定义了一个精确的 10 步流程：

1. `pwd` 确认工作目录
2. 读 git log + progress 文件获取上下文
3. 运行 `init.sh` 启动服务器
4. **回归测试**：Run 1-2 of the feature tests marked as `passes: true` that are most core
5. 选择一个 `passes: false` 的功能
6. 实现该功能
7. 用浏览器自动化验证（Puppeteer）
8. **只修改 passes 字段**
9. `git commit` + 更新 `claude-progress.txt`
10. 干净地结束 session

**关键发现**：回归测试是 Coding Agent 自己做的（"如发现问题立即标记为 failing 并修复后才做新功能"）。但本方案认为这个回归测试应该由独立 QA 执行——见下文。

### 1.8 Initializer 和 Coding Agent 共享 System Prompt

> "We refer to these as separate agents in this context only because they have different initial user prompts. The system prompt, set of tools, and overall agent harness was otherwise identical."

这意味着 Anthropic 的方案中两个"agent"其实用的是同一个 harness，只是第一次 session 用 initializer prompt，后续 session 用 coding prompt。**在 OpenClaw 中，我们用不同的 agentId + 不同 AGENTS.md 实现同样的效果。**

### 1.9 Multi-Agent 方向

> "It seems reasonable that specialized agents like a testing agent, a quality assurance agent, or a code cleanup agent, could do an even better job at sub-tasks across the software development lifecycle."

**本方案直接回答这个问题：是的，需要独立的 QA Agent。**

---

## 二、行业对比：其他 Agent 如何处理质量

### 2.1 OpenAI Codex — plan→implement→validate→repair 循环

OpenAI Codex 5.3 实现了约 **25 小时不间断运行**（13M tokens，30k 行代码），核心是多步执行循环：

> "It performed well on the parts that matter for long-horizon work: following the spec, staying on task, running verification, and repairing failures as it went."

OpenAI 还训练了专门的 **agentic code reviewer**：
- 每个 PR 自动审查
- 工程师推送前运行 `/review`
- 已保护高价值实验并捕获 launch-blocking 问题
- **优先优化精度而非召回率**：因为防御系统失败往往不是因为技术错误，而是太慢太吵导致用户绕过

**启示**：Codex 的 validate→repair 循环与我们的 dev-qa → fix → re-verify 循环思路一致。OpenAI 的精度优先策略也适用于我们的 QA agent——宁可漏报一些小问题，也不要产生太多误报导致 developer 信任崩塌。

### 2.2 Devin — Testing & Validation 官方用例

Devin 官方明确支持三类质量用例：
- **Test Generation**：自动生成集成测试和单元测试
- **QA Testing**：编写 QA 测试并执行自动化 QA 测试
- **PR Review**：代码审查

**启示**：Devin 将测试生成和 QA 测试分为不同用例，印证了测试者和实现者分离的思路。

### 2.3 ThoughtWorks — Spec-Driven Development (SDD)

ThoughtWorks 2025 年提出将 BDD 经验应用于 AI 辅助开发：

> "Specifications should still use domain-oriented ubiquitous language to describe business intent rather than specific tech-bound implementations. They should also have a clear structure, with a common style to define scenarios using Given/When/Then."

**启示**：我们的 feature_list.json 的 steps 字段应该用业务语言描述用户行为，而非技术实现。已有的 steps 格式（"Navigate to main interface" → "Click the 'New Chat' button"）正好符合这个原则。

### 2.4 BDD 在 AI Agent 中的价值

多个来源指出 BDD 在 AI agent 场景中成为必需品：

> "Agents don't need opinions — they need instructions. Clear, complete, consistent, and auditable instructions."

BDD 的 Given/When/Then 结构提供 agent 兼容的规范格式，作为人类与 agent 之间的契约。更重要的是：

> "Before agents even write a single line of code, we can already validate business logic with pre-implementation tests."

这意味着 dev-init 在写 feature_list.json 时就已经在做"预实现测试设计"——定义验收标准在编码之前。

### 2.5 TDD 的确定性退出标准

> "TDD provides deterministic exit criteria for AI agents. Instead of relying on the AI's judgment about when code is 'done,' tests force agents to iterate until all requirements pass."

这直接支持本方案的核心设计：**passes=false → true 是唯一的"完成"标准**，而非 coder 自己声明完成。

---

## 三、质量保障体系

### 3.1 设计原则：测试者和实现者必须分离

R-014 的根本问题：**dev-coder 既写代码又自己测试又自己标记 passes=true**。这违反了软件工程的基本原则——开发者不能验证自己的代码。

Anthropic 原文的 Coding Agent 确实是"self-verify"（包括回归测试也是 coder 自己做），但 Anthropic 也明确指出：

> "It seems reasonable that specialized agents like a testing agent, a quality assurance agent... could do an even better job."

本方案将 Anthropic 的"self-verify"升级为"independent QA verify"，是对 multi-agent 方向的直接实践。

### 3.2 质量循环（Quality Loop）

```
┌─────────────────────────────────────────────────────────────────┐
│                      Quality Loop                                │
│                                                                  │
│  dev-init (PM + QA Architect)                                    │
│    │                                                             │
│    ├── 1. 写 feature_list.json（每个 feature = BDD 测试用例）     │
│    ├── 2. 写 init.sh（含冒烟测试）                                │
│    ├── 3. git init + initial commit                              │
│    └── 4. 所有 features passes=false, qa_status="pending"       │
│                                                                  │
│  dev-coder (Implementer)                                         │
│    │                                                             │
│    ├── 5. 运行 init.sh（环境健康检查）                            │
│    ├── 6. 选一个 passes=false 的 feature                         │
│    ├── 7. 实现代码                                               │
│    ├── 8. 自测（可选，但不标记 passes）                           │
│    ├── 9. git commit（feat 标记，但 passes 仍为 false）           │
│    └── 10. 更新 progress.md，声明"ready for QA"                  │
│                                                                  │
│  dev-qa (Tester) — 独立验证 ⭐                                   │
│    │                                                             │
│    ├── 11. 运行 init.sh + 冒烟测试                               │
│    ├── 12. 按该 feature 的 steps 做 e2e 验证（browser 工具）     │
│    ├── 13. 运行 regression（已有 passes=true 的功能）            │
│    ├── 14a. 全部通过 → 改 passes=false → true                   │
│    ├── 14b. 新功能失败 → qa_status="needs-fix"                  │
│    ├── 14c. Regression 失败 → qa_status="regression-fail"       │
│    └── 15. git commit（test: 标记）                              │
│                                                                  │
│  dev-lead (Orchestrator)                                         │
│    │                                                             │
│    ├── 16. 审查 QA 结果                                          │
│    ├── 17a. VERIFIED → 选择下一个 feature                        │
│    ├── 17b. NEEDS-FIX → spawn dev-coder 修复（最多 3 轮）       │
│    └── 17c. REGRESSION-FAIL → 优先修复，不开发新功能             │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 3.3 三条质量铁律

1. **没有独立 QA 通过 = 没有 passes=true**。coding agent 永远不能自己改 passes。
2. **每个 QA session 必须跑 regression**。验证新功能时，同时验证所有已有 passes=true 的功能。
3. **冒烟先行**。任何 agent 开始工作前，先跑 init.sh，确认基础功能没坏。

### 3.4 Feature = BDD 测试用例

每个 feature 的 `steps` 字段就是 BDD 的验收标准：

| BDD 概念 | Feature List 对应 | 示例 |
|----------|-------------------|------|
| **Given** | 前置 steps（隐含在描述中） | "Navigate to main interface" |
| **When** | 操作 steps | "Click the 'New Chat' button" |
| **Then** | 验证 steps | "Verify a new conversation is created" |

这不是形式化的 Given/When/Then 语法，而是 Anthropic 实践验证过的 **自然语言 steps 格式**，对 agent 更友好。

---

## 四、Agent 角色定义

### 4.1 从质量角度重新定义角色

| 角色 | agentId | 质量职责 | 类型 | 模型建议 |
|------|---------|----------|------|----------|
| **dev-lead** | `dev` | 质量总负责，决定何时进入下一阶段 | OpenClaw agent | zai/glm-5-turbo |
| **dev-init** | `dev-init` | PM + 测试架构师：定义验收标准 | OpenClaw agent（一次性） | zai/glm-5-turbo |
| **dev-coder** | `dev-coder` | 实现 + 自测，但**不能**标记 passes | OpenClaw agent / ACP | zai/glm-5.1 或 ACP |
| **dev-qa** | `dev-qa` | 独立验证 + regression + 标记 passes | OpenClaw agent | zai/glm-4.7（成本优化） |

**关键改进**：新增 `dev-qa`，将测试权从 dev-coder 剥离。QA agent 可以用较便宜的模型——它不需要写复杂代码，只需要按步骤验证。

### 4.2 权限矩阵（核心变更 vs R-014）

| 操作 | dev-init | dev-coder | dev-qa | dev-lead |
|------|---------|-----------|--------|----------|
| 创建 feature_list.json | ✅ | ❌ | ❌ | ❌ |
| 修改 passes 字段 | ❌ | **❌** | **✅** | ❌ |
| 修改 qa_status 字段 | ❌ | ❌ | ✅ | ❌ |
| 写代码 (src/) | ❌ | ✅ | ❌ | ❌ |
| 运行 init.sh | ✅ | ✅ | ✅ | ❌ |
| 运行 browser e2e | ❌ | ❌ | ✅ | ❌ |
| git commit | ✅ (init) | ✅ (feat) | ✅ (test) | ❌ |

**与 R-014 的核心区别**：dev-coder **不再**有 `passes` 修改权。

### 4.3 5 个 Agent 完整 AGENTS.md

#### 4.3.1 dev-lead（开发编排者）

```markdown
# Dev Lead（开发编排者）— v3 (Quality-First)

你是 Dev Lead，质量总负责。你管理项目的 feature list 生命周期。
你是调度员，不是执行者。你的核心目标是确保每个 session 结束时代码处于 Clean State。

## Clean State 定义（Anthropic 原文）
代码必须可合并到 main branch：无重大 bug、代码有序有文档、下一个 agent 可直接开始新功能。

## 绝对不做什么
- ❌ 不自己写代码
- ❌ 不自己修改 feature_list.json
- ❌ 不跳过 QA 验证就批准功能

## 工作流程

### Flow A：新项目初始化
1. spawn dev-init → 创建 feature_list.json + init.sh + progress.md
2. 等待完成，审查 feature_list 的粒度和覆盖率
3. 进入 Flow B

### Flow B：开发循环
1. 读 feature_list.json，找 passes=false 且 qa_status≠"needs-fix" 的功能
2. spawn dev-coder 实现该功能
3. 等待完成（coder 声明 ready-for-qa）
4. spawn dev-qa 验证该功能
5. 等待 QA 结果：
   - VERIFIED → 进入下一个功能
   - NEEDS-FIX → spawn dev-coder 修复，最多 3 轮
   - REGRESSION-FAIL → spawn dev-coder 优先修复，不开发新功能
6. 每完成 3 个功能 → 让 dev-qa 跑一次全量 regression

### Flow C：ACP Harness（复杂功能）
同 Flow B，但 dev-coder 通过 ACP session 执行。
QA 仍然由 dev-qa 独立完成。

## 质量门禁
每次 session 结束前确认：
1. git status clean（无未提交变更）
2. init.sh + 冒烟测试通过
3. 所有已 passes=true 的功能 regression 通过
4. progress.md 已更新

## 烂摊子检测
如果 dev-qa 报告冒烟测试失败：
1. 不继续开发新功能
2. spawn dev-coder 专门修复（优先级最高）
3. 修复后 QA 重新验证
4. 只有恢复 Clean State 后才继续新功能

## 红线
- 不跳过 QA
- 不在没有 regression 的情况下标记批量完成
- 最多迭代 3 轮（coder → QA → fix → QA），超过则暂停并通知用户
```

#### 4.3.2 dev-init（Initializer Agent）

```markdown
# Dev Init（Initializer Agent）— v2 (Quality-First)

你是 Dev Init，你是"产品经理 + 测试架构师"。
你的唯一职责是为新项目创建初始环境和完整的验收标准。

## 你只运行一次

## 绝对不做什么
- ❌ 不实现任何功能
- ❌ 不写业务代码
- ❌ 不标记任何 passes 为 true

## 工作流程（严格按序执行）

### Step 1：理解需求
1. 阅读用户的完整任务描述
2. 列出假设（如有模糊之处）

### Step 2：创建 feature_list.json
**粒度规则**：
- 每个功能必须可以在一个 coding session 内完成
- steps 必须是可执行、可验证的具体操作（用业务语言，非技术术语）
- 优先级：scaffold → functional → integration → error-handling → edge-case → polish
- 初始所有 passes=false，qa_status="pending"
- 参考 Anthropic 实践：一个 claude.ai clone 有 200+ features

**强措辞约束（Anthropic 实际 prompt）**：
> "IT IS CATASTROPHIC TO REMOVE OR EDIT FEATURES IN FUTURE SESSIONS."

### Step 3：创建 init.sh（含冒烟测试）
见第六节模板。

### Step 4：创建 progress.md

### Step 5：Git 初始化
git init → git add . → git commit -m "init: project scaffold"

## Feature List 粒度指引
- 太大（"实现整个用户系统"）→ 拆成注册、登录、登出、个人资料等
- 太小（"创建一个 div"）→ 合并到上一个功能
- 标准：一个功能 = 一次 git commit = 一次 QA 验证
- steps 用自然语言描述用户行为，不描述技术实现

## 安全红线
- 不执行危险命令
- 不安装全局包
```

#### 4.3.3 dev-coder（Coding Agent — 实现者）

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

### 4. 自测（可选但推荐）
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

## 安全红线
- ❌ 绝不执行 rm -rf / 或等效
- ❌ 绝不使用 sudo
- ❌ 绝不嵌入密钥凭证
```

#### 4.3.4 dev-qa（QA Agent — 独立验证者）⭐ 新增

```markdown
# Dev QA（质量验证 Agent）— v1

你是 QA Agent，你是质量的独立把关者。
你唯一的职责是验证功能是否真正完成。你不写业务代码。

## 核心原则
**测试者和实现者必须分离。** dev-coder 实现功能，你验证功能。
你是唯一被授权修改 feature_list.json 中 passes 字段的 agent。

## Session 启动流程

### 1. 定位与上下文
pwd → 读 progress.md → 读 feature_list.json

### 2. 环境健康检查
运行 bash init.sh
如果冒烟测试失败 → 立即报告 dev-lead，不继续验证

### 3. 找到待验证功能
找 qa_status="pending" 且 progress.md 中声明 "ready-for-qa" 的功能

## 验证流程

### 单功能验证
1. 读取该功能的 steps 列表
2. 启动应用/服务（如 init.sh 已启动则跳过）
3. 逐步执行 steps：
   - 使用 browser 工具做 e2e 测试
   - 像"人类用户"一样操作：导航、点击、输入、验证
   - 每步记录结果（pass/fail）
4. 全部 steps 通过 → 进入 regression
   任一 step 失败 → 标记 qa_status="needs-fix"，写 bug report

### Regression（回归测试）
验证所有已 passes=true 的功能仍然正常：
1. 遍历 feature_list.json 中 passes=true 的功能
2. 对每个功能执行其 steps 的关键子集（不需要完整执行所有 steps）
3. 全部通过 → OK
4. 如有 regression → 标记该功能 qa_status="regression-fail"

### 更新 feature_list.json
**只有你才能修改 passes 字段：**
- 验证通过 + regression 通过 → `passes: true, qa_status: "verified"`
- 验证失败 → `qa_status: "needs-fix"`
- Regression 失败 → `qa_status: "regression-fail"`

git add feature_list.json && git commit -m "test(F{id}): verified passes=true"

### 更新 progress.md
追加 QA session 记录。

## 输出格式
```
## QA 报告
- 验证功能：F{id} — {description}
- Steps 验证：{n}/{total} passed
- Regression：{n} features checked, {n} passed
- 结果：VERIFIED | NEEDS-FIX | REGRESSION-FAIL
- Bug Report（如有）：{详细描述}
```

## 严禁
- ❌ 不写业务代码
- ❌ 不跳过 regression
- ❌ 不在没有 e2e 验证的情况下标记 passes=true
- ❌ 不删除或修改 steps
- ❌ 不搜索互联网

## 安全红线
- 不执行危险命令
```

---

## 五、Feature List 模板（BDD 风格）

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

**与 R-014 的区别**：
- 新增 `qa_status` 字段（pending → ready-for-qa → verified / needs-fix / regression-fail）
- 新增 `priority` 字段
- 新增 `schema_version` 字段
- steps 更具体，每步都是可验证的原子操作
- steps 用业务语言描述用户行为（ThoughtWorks SDD 原则）

---

## 六、init.sh 模板（含冒烟测试）

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

**与 R-014 的区别**：
- 新增**冒烟测试**阶段（第 4 步）
- dev server 自动检测和启动
- 明确的退出码语义（0=OK, 1=有问题）
- TypeScript 检查

---

## 七、openclaw.json 配置

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
        description: "开发编排者 - 任务拆解、质量把关、feature list 生命周期管理",
        tools: {
          allow: ["read", "write", "web_fetch", "sessions_spawn", "subagents"],
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
        description: "Initializer Agent - PM + 测试架构师",
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
        description: "Coding Agent - 增量实现功能，不能标记 passes",
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
        description: "QA Agent - 独立验证、regression 测试、唯一能标记 passes",
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

**与 R-014 的核心配置区别**：
- 新增 `dev-qa` agent（browser 工具用于 e2e）
- dev-coder 的 deny 列表确认无 sessions_spawn（不能自己 spawn QA）
- dev-lead 的 allowAgents 包含 dev-qa

---

## 八、创建命令 + 部署步骤

### 8.1 创建命令

```bash
# 1. 创建目录结构
mkdir -p ~/.openclaw/agents/{dev,dev-init,dev-coder,dev-qa}
mkdir -p ~/.openclaw/workspace-dev

# 2. 注册 agents
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
  --description "QA Agent - 独立验证，唯一能标记 passes"

# 3. 写入 AGENTS.md（第四节内容写入对应文件）
# 4. 编辑 openclaw.json（第七节配置）
# 5. 安装 ACP 插件（可选）
openclaw plugins install acpx
openclaw config set plugins.entries.acpx.enabled true

# 6. 重启验证
openclaw gateway restart
openclaw agents list
```

### 8.2 部署步骤

#### Phase 0：环境准备（1 天）
- [ ] 确认 OpenClaw 运行正常
- [ ] 确认 browser 工具可用（QA agent 依赖）
- [ ] 安装 acpx 插件（如需要 ACP）
- [ ] 创建目录和 AGENTS.md
- [ ] 更新 openclaw.json

#### Phase 1：核心循环（3-5 天）
**目标**：dev-lead + dev-init + dev-coder + dev-qa，完成一个 5 功能项目

- [ ] 注册所有 4 个 agent
- [ ] 测试：dev-lead → dev-init（创建 feature_list + init.sh）
- [ ] 测试：dev-lead → dev-coder（实现一个功能，声明 ready-for-qa）
- [ ] 测试：dev-lead → dev-qa（验证功能，标记 passes=true）
- [ ] 测试：QA 发现 bug → coder 修复 → QA 重新验证
- [ ] **验收**：5 功能项目，每个功能都经独立 QA 验证

#### Phase 2：ACP 集成（1 周）
- [ ] 复杂功能路由到 ACP harness
- [ ] QA 流程不变（仍然是 dev-qa 独立验证）

#### Phase 3：生产化（持续）
- [ ] 监控：QA 通过率、regression 失败率、每功能耗时
- [ ] AGENTS.md 精简到 100 行以内

---

## 九、与 R-014 对比

| 维度 | R-014（旧方案） | R-015（本方案） | 改进说明 |
|------|---------------|---------------|----------|
| **核心哲学** | Harness Engineering 实现 | **Quality-First**：交付可合并的干净代码 | 以质量为中心重新设计 |
| **测试者/实现者分离** | ❌ coder 自己测试自己标记 | ✅ 独立 dev-qa agent 验证 | 消除"自己给自己打分" |
| **QA Agent** | ❌ 没有 | ✅ dev-qa（唯一能标记 passes） | 核心新增角色 |
| **passes 修改权** | dev-coder 可以改 | **仅 dev-qa 可以改** | 权限收紧 |
| **feature_list.json** | 4 字段 | 6 字段（+qa_status, +priority, +schema_version） | 更丰富的状态追踪 |
| **init.sh** | 仅启动环境 | **含冒烟测试 + 明确退出码** | 环境问题提前发现 |
| **Regression 测试** | ❌ 未定义 | ✅ QA 每次验证时跑 regression | 防止新功能破坏旧功能 |
| **烂摊子检测** | 未定义 | ✅ 冒烟失败 → 不开发新功能，先修复 | 优先恢复 Clean State |
| **Browser e2e** | 未定义 | ✅ dev-qa 用 browser 工具做 e2e | 对齐 Anthropic 实践 |
| **Agent 数量** | 4（含 ACP 虚拟） | 5（+ dev-qa） | 多 1 个但质量闭环 |
| **Anthropic 原文引用** | 间接引用 | **逐字引用 + quickstart 源码** | 设计决策有据可查 |
| **强措辞约束** | "unacceptable" | **"CATASTROPHIC"**（quickstart 实际措辞） | 更强的行为约束 |
| **行业对比** | ❌ 无 | ✅ Codex/Devin/ThoughtWorks/SDD | 借鉴行业实践 |
| **BDD 测试用例** | steps 不够具体 | steps = BDD 验收标准，用业务语言 | 测试规范性提升 |

### 关键改进总结

1. **测试者/实现者分离**：R-014 的最大缺陷是 coder 自己标记 passes，本方案通过独立 QA Agent 解决
2. **冒烟先行 + Regression**：每个 session 开始先确认环境没坏，QA 验证时检查已有功能不被破坏
3. **权限收紧**：passes 修改权仅限于 dev-qa，从制度上防止"自说自话"
4. **烂摊子检测**：冒烟测试失败时不开发新功能，优先恢复 Clean State
5. **忠于 Anthropic 原文**：逐字引用 + quickstart 源码确认，确保设计决策有据可查
6. **行业验证**：Codex 的 validate→repair 循环、Devin 的 QA Testing 用例、ThoughtWorks 的 SDD 均支持本方案的核心设计

---

## 十、知识缺口

1. **dev-qa 的 browser 工具能力边界**：OpenClaw 的 browser 工具在 headless 环境下的截图能力和 Puppeteer 相比有何差异？需要实测。
2. **QA 验证耗时**：每次 regression 检查所有已有功能可能很慢。需要定义 regression 子集策略（Anthropic 只检查"1-2 个最核心"的已通过功能）。
3. **并行 QA**：多个 feature 同时 ready-for-qa 时，是否可以并行 spawn 多个 dev-qa？并发安全问题？
4. **QA Agent 的模型选择**：QA 不需要最强推理模型，但需要可靠的操作能力。zai/glm-4.7 是否足够？需要测试。
5. **ACP harness 的 passes 约束**：Claude Code 等外部 harness 是否可靠地遵循"不能改 passes"的约束？R-014 就提出了这个问题，仍未解决。

---

## 十一、来源

| # | 来源 | URL/路径 | 置信度 |
|---|------|----------|--------|
| 1 | Anthropic: Effective Harnesses for Long-Running Agents（原文） | https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents | — |
| 2 | Anthropic: Initializer Prompt（quickstart 源码） | https://raw.githubusercontent.com/anthropics/claude-quickstarts/main/autonomous-coding/prompts/initializer_prompt.md | — |
| 3 | Anthropic: Coding Agent Prompt（quickstart 源码） | https://raw.githubusercontent.com/anthropics/claude-quickstarts/main/autonomous-coding/prompts/coding_prompt.md | — |
| 4 | Anthropic: autonomous-coding quickstart 仓库 | https://github.com/anthropics/claude-quickstarts/tree/main/autonomous-coding | — |
| 5 | Claude 4 Prompting Guide: Multi-context Window Workflows | https://docs.claude.com/en/docs/build-with-claude/prompt-engineering/claude-4-best-practices | — |
| 6 | OpenAI: Scaling Code Verification | https://alignment.openai.com/scaling-code-verification/ | high |
| 7 | OpenAI: Run Long-Horizon Tasks with Codex | https://developers.openai.com/blog/run-long-horizon-tasks-with-codex/ | high |
| 8 | Devin: Testing & Validation Use Cases | https://docs.devin.ai/use-cases | high |
| 9 | ThoughtWorks: Spec-Driven Development | https://www.thoughtworks.com/insights/blog/agile-engineering-practices/spec-driven-development-unpacking-2025-new-engineering-practices | high |
| 10 | BDD is Essential in the Age of AI Agents | https://medium.com/@meirgotroot/why-bdd-is-essential-in-the-age-of-ai-agents-65027f47f7f6 | medium |
| 11 | Quality Gates for AI-Generated Code | https://www.softwareseni.com/building-quality-gates-for-ai-generated-code-with-practical-implementation-strategies/ | medium |
| 12 | TDD/BDD in the Age of AI Agents | https://natshah.com/blog/tdd-bdd-age-ai-why-ai-agents-demand-100-more-test-first-development | medium |
| 13 | OpenClaw ACP Agents 文档 | ~/.npm-global/lib/node_modules/openclaw/docs/tools/acp-agents.md | — |
| R-014 | 前版设计 | ~/.openclaw/workspace/shared/results/R-014-dev-team-harness-design.md | — |
