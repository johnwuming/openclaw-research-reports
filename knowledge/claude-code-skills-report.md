# R-007: ClawHub 技能市场中 Claude Code 相关技能深度调研报告

> 调研日期：2026-03-29 | 研究员：Research Lead (subagent) | 版本：v1

---

## 一、核心发现

### 1.1 ClawHub 上已发现的 Claude Code 相关技能

通过 ClawHub 搜索（clawhub.ai）和 GitHub 交叉验证，共发现以下 Claude Code 相关技能：

| # | 技能名称 | 作者 | ClawHub URL | 功能概述 |
|---|---------|------|-------------|---------|
| 1 | **claude-skill** | feiskyer | clawhub.ai/feiskyer/claude-skill | 调用 Claude/Claude Code 执行编码任务（代码实现、审查等） |
| 2 | **claude-connect** | tunaissacoding | aiclawskills.com/skills/claude-connect | 将 Claude 订阅 OAuth 令牌桥接到 Clawdbot，自动刷新 |
| 3 | **openclaw-skill-claude-code** | noncelogic | github.com/noncelogic/openclaw-skill-claude-code | 异步持久化 Job Manager，解耦编码任务与 agent 超时 |
| 4 | **claude-code-skill** | Enderfga | clawhub.ai/Enderfga/claude-code-skill | Claude Code Agent，集成文件系统和 GitHub MCP 服务器 |
| 5 | **claude-code** | hw10181913 | clawhub.ai/hw10181913/claude-code | 将 Claude Code 能力集成到 OpenClaw，提供编码工作流 |
| 6 | **claude-code-control** | (unknown) | clawhub.ai/skills/claude-code-control | 在可见终端窗口中驱动 Claude Code |
| 7 | **claude-code-task** | (unknown) | clawhub.ai/skills/claude-code-task | 异步启动 Claude Code，结果推送到 Telegram/WhatsApp |
| 8 | **claude-code-cli** | AtelyPham | clawhub.ai/AtelyPham/claude-code-cli | 通过后台进程（PTY/headless）委派编码任务给 Claude Code CLI |
| 9 | **claude-code-mastery** | cheenu1092-oss | clawhub.ai/cheenu1092-oss/claude-code-mastery | Claude Code 全面掌控，含设置脚本、开发团队 subagent、自学习系统 |
| 10 | **skill-stats** | leo-306 | clawhub.ai/leo-306/skill-stats | 跨 Claude Code 和 OpenClaw 的技能使用统计分析 |

此外，**feiskyer/claude-code-settings**（GitHub，非 ClawHub 独立技能）是一个大型集合，包含多个子技能：
- **codex-skill** — 将任务交给 OpenAI Codex CLI 执行
- **autonomous-skill** — 长时间运行任务自动化（双 agent 模式）
- **nanobanana-skill** — 图像生成
- **kiro-skill** — Kiro 工作流
- **spec-kit-skill** — Spec-Kit 规范驱动开发
- **youtube-transcribe-skill** — YouTube 字幕提取

### 1.2 重点技能详细分析

#### noncelogic/openclaw-skill-claude-code（最推荐的方案）
- **核心功能**：使用 `@anthropic-ai/claude-agent-sdk` 启动持久化、分离的编码任务
- **解决的问题**：
  - Agent turn 有超时限制（如 900s），编码任务可能 20+ 分钟
  - OpenClaw 重启会杀死子进程
  - 通用 wrapper 无法区分"正在思考"和"API 429/503"
- **架构**：`SKILL.md → node scripts/run.mjs → 分离进程 → Claude Agent SDK → jobs/<id>/` 写入状态
- **命令**：`start`, `status`, `logs`, `result`, `list`, `kill`
- **依赖**：`ANTHROPIC_API_KEY`（必须）、Node.js
- **平台**：Linux/macOS 均可（无 macOS 特定依赖）✅

#### tunaissacoding/claude-connect（OAuth 桥接）
- **核心功能**：将 Claude CLI 的 OAuth 令牌桥接到 Clawdbot
- **工作流程**：从 macOS Keychain 读取 → 写入 `auth-profiles.json` → 每 2 小时自动刷新
- **依赖**：macOS Keychain、Claude CLI、launchd
- **平台限制**：**仅 macOS**（依赖 Keychain 和 launchd）❌ Linux 不可用
- **⚠️ 安全风险**：直接操作 OAuth 令牌，需高度信任

#### feiskyer/claude-skill（ClawHub 版）
- **核心功能**：当用户要求使用 Claude/Claude Code 时触发（实现功能、审查代码等）
- **安装**：`openclaw skill install feiskyer/claude-skill`
- **平台**：跨平台 ✅

#### feiskyer/claude-code-settings（GitHub 大型集合）
- **核心功能**：面向 Claude Code 的完整开发工作流配置集
- **亮点子技能**：
  - **codex-skill**：将 Claude 设计的方案交给 Codex/GPT-5 实现
  - **autonomous-skill**：双 agent 模式（Initializer + Executor），跨 session 自动继续
- **安装方式**：Claude Code Plugin 或 `npx skills add`
- **额外依赖**：LiteLLM proxy（可选，用于 GitHub Copilot 代理）
- **平台**：跨平台 ✅

#### Enderfga/claude-code-skill
- 集成文件系统和 GitHub MCP 服务器
- **安全警告**：允许访问任意文件系统路径 ⚠️

#### AtelyPham/claude-code-cli
- 通过 PTY（交互模式）或 headless pipe 模式运行 Claude Code CLI
- 后台进程管理
- 平台：跨平台 ✅

---

## 二、技能对比矩阵

| 维度 | noncelogic/claude-code | claude-connect | feiskyer/claude-skill | claude-code-cli | claude-code-mastery |
|------|----------------------|----------------|----------------------|-----------------|-------------------|
| **核心定位** | 异步 Job Manager | OAuth 桥接 | Claude 调用触发器 | CLI 后台委派 | 全面掌控套件 |
| **持久化** | ✅ PID 追踪+磁盘 | ❌ | ❌ | 部分 | ✅ |
| **跨平台** | ✅ Linux+Mac | ❌ 仅 macOS | ✅ | ✅ | ✅ |
| **Agent SDK** | ✅ 官方 SDK | ❌ | ❌ | ❌ | 部分 |
| **复杂度** | 中 | 低 | 低 | 中 | 高 |
| **安全性** | 中（需 API Key） | ⚠️ 高风险（OAuth） | 中 | 中 | ⚠️ 高权限 |

---

## 三、OpenClaw 原生 ACP 协议 vs 技能方案

### 3.1 ACP 协议概述

OpenClaw 内置 ACP（Agent Client Protocol）支持，可直接通过 `sessions_spawn({ runtime: "acp" })` 启动 Claude Code、Codex、Cursor 等外部编码 harness。

**ACP 核心能力**：
- 直接 spawn Claude Code/Codex 等 ACPX harness 作为外部 session
- 支持 one-shot (`mode: "run"`) 和 persistent (`mode: "session"`) 模式
- 支持绑定到 Discord thread、Telegram conversation
- `/acp spawn codex --bind here` 一条命令即可启动

### 3.2 何时用原生 ACP vs 技能

| 场景 | 推荐方案 | 原因 |
|------|---------|------|
| 简单的"让 Claude Code 做这个任务" | **原生 ACP** | 内置支持，无需安装 |
| 长时间编码任务（20+ 分钟） | **noncelogic/claude-code 技能** | ACP session 会因超时断开 |
| 需要跨 OpenClaw 重启保持任务 | **技能方案** | ACP session 不持久化 |
| OAuth 令牌桥接（macOS） | **claude-connect 技能** | ACP 不处理认证桥接 |
| 多模型协作（Claude 设计+Codex 实现） | **feiskyer 技能集** | 需要复杂编排 |
| 开发团队 subagent 编排 | **claude-code-mastery 或 feiskyer** | 需要预定义角色 |

### 3.3 ACP 的局限性（来源：shashikantjagtap.net 深度分析）

- OpenClaw ACP 是 **bridge（桥接）**，不是 **full ACP agent**
- 缺少完整的双向通信能力
- 缺少结构化输出（diff、execution plan）
- 不支持 session 持久化跨重启
- 对比原生 ACP agent（如 Zed 内置），功能有明显差距

---

## 四、实际安装可行性（Linux 环境）

| 技能 | 可安装 | 备注 |
|------|--------|------|
| noncelogic/openclaw-skill-claude-code | ✅ | 需要 Node.js + ANTHROPIC_API_KEY |
| feiskyer/claude-skill | ✅ | 标准 ClawHub 安装 |
| feiskyer/claude-code-settings | ✅ | 通过 npx skills 或 Claude Code Plugin |
| tunaissacoding/claude-connect | ❌ | 依赖 macOS Keychain + launchd |
| AtelyPham/claude-code-cli | ✅ | 需要 Claude Code CLI 已安装 |
| Enderfga/claude-code-skill | ✅（有安全风险） | 任意文件系统访问 |
| hw10181913/claude-code | ✅ | 标准 ClawHub 安装 |
| claude-code-mastery | ✅ | 功能最全但复杂度高 |

---

## 五、其他相关技能

### 5.1 Codex 相关
- **feiskyer/codex-settings**（GitHub）— OpenAI Codex 的设置和提示集合
- feiskyer/claude-code-settings 中的 **codex-skill** 子技能 — 将任务从 Claude 交给 Codex

### 5.2 通用编码技能
- **skill-creator**（chindden）— 创建新技能的元技能
- **skill-stats**（leo-306）— 跨 Claude Code 和 OpenClaw 的技能使用统计
- **PixVerse Skills** — 支持 Claude Code、Cursor、Codex 的共享技能目录

### 5.3 ClawHub 生态规模
- ClawHub 总计 10,000+ 技能（2026 年 3 月）
- VoltAgent/awesome-openclaw-skills 收录了 5,400+ 经过筛选分类的技能
- 技能提交量从 2026 年 1 月中旬的每天 <50 增长到 2 月初的每天 500+

---

## 六、安全风险分析

### 6.1 ClawHub 供应链安全事件

**Snyk ToxicSkills 研究**（2026 年 2 月）— 对 3,984 个技能的审计发现：
- **36.82%（1,467 个）** 存在至少一个安全漏洞
- **13.4%（534 个）** 存在严重安全问题
- **76 个确认的恶意 payload**：凭据窃取、后门安装、数据泄露
- **8 个恶意技能在发布时仍公开可获取**

**攻击手段**：
- Typosquatting（仿冒知名技能名）
- Post-install 脚本植入
- Base64 编码的恶意 payload
- 外部网站引用隐藏恶意代码（新型手法）
- 操纵 ClawHub 排名算法使恶意技能升至 #1

**其他事件**：
- 341 个恶意技能分发 Atomic Stealer（macOS/Windows 凭据窃取）
- 1,184 个非法技能的供应链攻击
- 恶意评论引导用户下载 infostealer
- ClawHub 排名操纵漏洞（Silverfort 披露）

### 6.2 技能安全评估

| 风险等级 | 技能 | 原因 |
|---------|------|------|
| 🟢 低 | feiskyer/claude-skill | 知名开发者，ClawHub 验证 |
| 🟢 低 | feiskyer/claude-code-settings | 开源 GitHub，社区活跃 |
| 🟡 中 | noncelogic/openclaw-skill-claude-code | 需要 API Key，但有源码可审计 |
| 🔴 高 | claude-connect | 直接操作 OAuth 令牌 |
| 🔴 高 | Enderfga/claude-code-skill | 任意文件系统访问 |
| 🔴 高 | 不明作者的 claude-code-task/control | 无验证来源 |

### 6.3 安全建议

1. **优先使用原生 ACP** 而非第三方技能，减少攻击面
2. **审计 SKILL.md** — 检查是否有 base64 编码内容、外部 URL 引用、可疑的 exec 指令
3. **使用 VirusTotal 扫描** — OpenClaw 内置支持
4. **仅安装知名作者** 的技能（feiskyer 等有 GitHub 历史的）
5. **避免需要 OAuth/Keychain 操作的技能**
6. **定期更新和清理** 未使用的技能

---

## 七、知识缺口

1. **ClawHub 评分/审核机制**：ClawHub 是否已实施代码签名或安全审核流程，尚无明确公开信息
2. **claude-code-control 和 claude-code-task 的详细内容**：未能获取完整 SKILL.md
3. **Claude Code Channels**：Anthropic 推出的 Channels 功能（类似 OpenClaw 的 Telegram 集成）对 ClawHub 生态的影响尚未明确
4. **ACP 协议演进**：OpenClaw 是否计划将 ACP bridge 升级为 full ACP agent

---

## 八、来源列表

1. [ClawHub 官网](https://clawhub.ai/) — 技能注册表
2. [noncelogic/openclaw-skill-claude-code - GitHub](https://github.com/noncelogic/openclaw-skill-claude-code) — 技能详情
3. [feiskyer/claude-code-settings - GitHub](https://github.com/feiskyer/claude-code-settings) — 技能集合
4. [Claude Connect - AIClawSkills](https://aiclawskills.com/skills/claude-connect/) — OAuth 桥接详情
5. [Snyk ToxicSkills 研究](https://snyk.io/blog/toxicskills-malicious-ai-agent-skills-clawhub/) — 安全审计
6. [OpenClaw ACP Agents 文档](https://docs.openclaw.ai/tools/acp-agents) — ACP 协议
7. [ACP Protocol Gaps 分析](https://shashikantjagtap.net/openclaw-acp-what-coding-agent-users-need-to-know-about-protocol-gaps/) — ACP 局限性
8. [VoltAgent/awesome-openclaw-skills](https://github.com/VoltAgent/awesome-openclaw-skills) — 技能索引
9. [DataCamp ClawHub Guide](https://www.datacamp.com/blog/best-clawhub-skills) — 生态概览
10. [The Hacker News - 341 Malicious Skills](https://thehackernews.com/2026/02/researchers-find-341-malicious-clawhub.html) — 安全事件
11. [Silverfort - ClawHub 排名操纵](https://www.silverfort.com/blog/clawhub-vulnerability-enables-attackers-to-manipulate-rankings-to-become-the-number-one-skill/) — 漏洞披露
12. [Reddit r/openclaw - 技能审核](https://www.reddit.com/r/openclaw/comments/1r3vnzj/finally_figured_out_how_to_actually_vet_clawhub/) — 社区讨论

---

## 九、方法论反思

**做得好**：
- 多源交叉验证（ClawHub、GitHub、安全研究报告、社区讨论）
- 发现了超出预期的 9+ 个 Claude Code 相关技能（用户仅列出 3 个）
- 安全风险部分基于 Snyk 专业审计报告

**需改进**：
- 部分 ClawHub 技能（claude-code-control、claude-code-task）未获取到完整 SKILL.md 内容
- 未实际安装测试任何技能（仅基于文档分析）
- ACP 与技能方案的对比缺少实际 benchmark 数据

---

*报告生成时间：2026-03-29 07:21 CST*
