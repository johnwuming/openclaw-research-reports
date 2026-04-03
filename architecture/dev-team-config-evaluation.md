# OpenClaw 开发团队配置对比评估报告

## 1. 当前配置（从 openclaw.json 读取）

### Dev 团队架构
| Agent ID | 角色 | 模型 | 子Agent | 主要工具 |
|----------|------|------|---------|----------|
| **dev** | 主开发调度 | kimi/k2p5 | dev-init, dev-coder, dev-qa, research | exec, read, write, web_fetch, sessions_spawn |
| **dev-init** | 初始化 | kimi/k2p5 | 无 | exec, read, write |
| **dev-coder** | 编码 | kimi/k2p5 | 无 | exec, read, write, process |
| **dev-qa** | 测试 | kimi/k2p5 | 无 | exec, read, write, browser |

### 关键特性
- 所有 dev 系列 agents 共享同一个 workspace (`/home/noname/.openclaw/workspace-dev`)
- dev 作为父 agent 可以调度子 agents，init/coder/qa 之间不可相互调用
- 使用相同的模型配置 (kimi/k2p5 + zai/glm-5.1 fallback)

---

## 2. 原始设计参考（基于公开调研文档）

### 火山引擎官方推荐的开发团队架构
根据 https://developer.volcengine.com/articles/7617825213975920703，标准 AI 开发团队包含：

| 角色 | 职责 |
|------|------|
| **PM** | 产品经理，需求分析、任务拆解 |
| **UI** | UI 设计师 |
| **FE** | 前端开发工程师 |
| **BE** | 后端开发工程师 |
| **TE** | 测试工程师 |

### 阿里云教程中的分层架构
根据 https://developer.aliyun.com/article/1720648：
- 1个调度 Agent (main)
- N个功能 Agent (如 mr市场研究员、pm产品经理、dev开发工程师)

---

## 3. 出入点对比

### ✅ 符合原始设计的部分

| 方面 | 当前配置 | 原始设计 | 评价 |
|------|----------|----------|------|
| **分层架构** | dev 作为父调度 + 子agent分工 | 调度Agent + 功能Agent | ✅ 符合 |
| **角色隔离** | 独立 agentDir (dev-init, dev-coder, dev-qa) | 每个角色独立Agent | ✅ 符合 |
| **工具权限** | 根据角色限制工具 (dev-qa有browser, dev-coder有process) | 根据角色分配工具 | ✅ 符合 |
| **workspace 共享** | dev系列共享 workspace-dev | 团队共享工作空间 | ✅ 符合 |

### ❌ 偏差与缺失

| 偏差项 | 当前配置 | 原始设计 | 偏差原因 | 影响 |
|--------|-----------|-----------|----------|------|
| **角色数量** | 3个子agent (init/coder/qa) | 5个角色 (PM/UI/FE/BE/TE) | 简化需求或成本考虑 | 中等 - 无法处理产品/UI需求 |
| **PM 角色缺失** | 无 | 产品经理负责需求拆解 | 未配置 | 高 - 无法进行需求分析 |
| **前端/后端分离** | 统一的 coder | FE + BE 分离 | 未细分 | 中 - 代码规模大时难以管理 |
| **UI 角色** | 无 | UI设计 | 未配置 | 高 - 无法处理界面设计任务 |
| **dev-init 定位** | 初始化 | TE（测试） | 命名混淆 | 低 - 功能上可覆盖 |

---

## 4. 建议

### 短期（保持现状）
- 当前配置适合简单的"编码 + 测试"工作流
- 如果只涉及后端开发，当前配置基本够用

### 长期（向原始设计对齐）
如需完整开发团队能力，建议扩展：
1. 添加 `pm` agent (产品经理)
2. 拆分 `dev-coder` → `dev-fe` + `dev-be`
3. 考虑添加 `dev-ui` (UI设计)

---

## 5. 结论

**当前配置偏离原始设计约 40%**，主要缺失：
- 产品经理角色
- 前端/后端分离
- UI 设计角色

如果用户需求仅限"初始化 → 编码 → 测试"三步流程，当前配置基本符合设计。否则建议按原始调研文档扩展团队角色。