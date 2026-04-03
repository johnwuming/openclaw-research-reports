# R-001 调研结果：OpenClaw 2026 年最新功能更新

**调研时间：** 2026-03-28
**最新稳定版：** v2026.3.24（发布于 2026-03-25）

---

## 关键新功能

### 1. OpenAI 兼容 API 增强
Gateway 新增 `/v1/models`、`/v1/embeddings` 端点，并支持在 `/v1/chat/completions` 和 `/v1/responses` 中转发显式模型覆盖参数，大幅提升与 RAG 管线和第三方客户端的兼容性。
- **来源：** [GitHub Releases - v2026.3.24](https://github.com/openclaw/openclaw/releases)

### 2. Microsoft Teams 官方 SDK 集成
迁移至官方 Teams SDK，新增流式 1:1 回复、欢迎卡片、反馈/反思机制、打字指示器、AI 标签等 UX 最佳实践。同时支持消息编辑和删除。
- **来源：** [GitHub PR #51808](https://github.com/openclaw/openclaw/pull/51808)、[GitHub PR #49925](https://github.com/openclaw/openclaw/pull/49925)

### 3. 安全加固（CVE-2026-25253 修复 + 默认配置强化）
修复了 CVSS 9.8 的 Gateway 远程代码执行漏洞；新安装默认绑定 `127.0.0.1`、启用 auth、优先使用 Podman；ClawHub 集成 VirusTotal 扫描；新增 Skill 签名验证（Beta）。
- **来源：** [What's New in March 2026 - tenten.co](https://tenten.co/openclaw/en/docs/whats-new/march-2026)

### 4. Skills 一键安装与 Control UI 改版
Bundled skills（coding-agent、gh-issues、weather 等）新增一键安装配方；Control UI 新增 skills 状态筛选标签页（All / Ready / Needs Setup / Disabled）、详情弹窗、内联 Markdown 预览。
- **来源：** [GitHub PR #53411](https://github.com/openclaw/openclaw/pull/53411)

### 5. Memory System v2 预览
重大记忆系统升级预览：向量搜索替代纯文本匹配、记忆分级（公开/私有/敏感）、自动 PII 脱敏、跨 Agent 共享记忆（可选）。通过 `--experimental-memory-v2` 启用。
- **来源：** [What's New in March 2026 - tenten.co](https://tenten.co/openclaw/en/docs/whats-new/march-2026)

---

## 其他值得关注的更新

| 功能 | 说明 |
|------|------|
| Docker 容器支持 | `--container` 参数可直接在运行中的容器内执行命令 |
| Discord 自动线程命名 | LLM 异步生成线程标题 |
| Slack 交互式回复 | 自动渲染按钮/选择器，恢复富文本回复 |
| Nvidia GPU 加速 | 支持本地 LLM 推理、Whisper STT 加速（实验性） |
| 腾讯中文生态包 | 官方微信适配器、飞书适配器、中文 STT 模型（开源） |

---

## 参考链接

- [GitHub Releases](https://github.com/openclaw/openclaw/releases)
- [OpenClaw 官方文档](https://docs.openclaw.ai)
- [What's New March 2026](https://tenten.co/openclaw/en/docs/whats-new/march-2026)
- [v2026.3.24 Release Notes](https://openclaw-hub.com/releases/v2026.3.24/)
