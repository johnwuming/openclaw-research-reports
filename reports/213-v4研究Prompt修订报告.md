# R-012：v4 研究 Prompt 修订报告

> 日期：2026-03-29
> 任务：基于 meta-review 发现和 AGENTS.md 实际部署差异，修订 v4 研究 prompt

---

## 一、数据来源

1. `/home/noname/.openclaw/workspace/research-team-prompts-v4.md`（v4 原版）
2. `/home/noname/.openclaw/workspace/research-team-v4-improvements.md`（5 个改进方向详细分析）
3. 各 agent AGENTS.md：research、search、reviewer、citation
4. meta-review 评分：6.5/10，主要改进方向：robustness hardening > Writer Agent split > JSON schema

## 二、v4 方案文档 vs 实际 AGENTS.md 不同步发现

| # | 不同步项 | v4 方案文档 | 实际 AGENTS.md | 影响 |
|---|---------|-----------|---------------|------|
| 1 | 搜索策略优先级 | Search Agent 写"先用 web_search" | 已改为 webSearchPrime 优先 | 🔴 搜索质量/成本直接影响 |
| 2 | agent_metadata 字段 | v4 prompt 有但 Search Agent prompt 部分描述不完整 | AGENTS.md 有完整说明 | 🟡 Lead 解析可能遗漏信息 |
| 3 | Reviewer 实现方式 | 两个独立 prompt（Reviewer A/B） | 合并为单个 AGENTS.md 用 review_type 区分 | 🟢 实际部署更灵活，prompt 保持分开合理 |
| 4 | Phase 0 bad-answers.json | 包含此步骤 | AGENTS.md 中无此步骤 | 🟡 未实际使用，已移除 |
| 5 | 模型分配描述 | "用户在 GUI 上配置" | AGENTS.md 硬编码了模型名 | 🟡 措辞已统一为"workspace 配置" |

## 三、修订内容

### 3.1 Robustness Hardening（meta-review 最高优先级）

**Reviewer A 增加 3 条系统性检测硬规则：**
- 所有 findings 来自同一来源 → score 整体降 2 分
- 所有 findings 都是 low confidence → 建议重新搜索
- overall_quality < 5 → recommendation 必须是 "major_revision"

### 3.2 搜索策略同步（R-008 建议 + AGENTS.md 对齐）

**Search Agent prompt 完全重写搜索策略部分：**
- 首选：webSearchPrime（智谱 MCP），含完整参数说明
- 深度：Browser（百度/Google）
- 兜底：web_search（DuckDuckGo）
- 已知 URL：web_fetch

### 3.3 输出规范统一

所有 5 个 agent prompt 统一措辞："不要输出任何其他内容"（替代原"不要有任何其他内容"）

### 3.4 配置描述准确化

- spawn 参数：改为"让子 agent 使用其 workspace 配置的默认模型"
- 模型分配：改为"由各 agent workspace 自行配置"
- 不硬编码任何模型名

### 3.5 简化

- Phase 0 去掉 bad-answers.json（未实际使用）
- 模型分配建议合入配置参考 section

## 四、未采纳的 meta-review 建议

| 建议 | 原因 |
|------|------|
| Writer Agent split | 当前 Lead Agent 收敛阶段已足够，增加 Writer Agent 会增加复杂度和延迟 |
| JSON Schema 强制验证 | OpenClaw sub-agent 不支持 provider-native Structured Outputs，只能靠 prompt + 容错 |

## 五、输出文件

- ✅ 已更新：`/home/noname/.openclaw/workspace/research-team-prompts-v4.md`（v4.1 修订版）
- ✅ 本报告：`/home/noname/.openclaw/workspace/shared/results/R-012-v4-prompt-revision.md`
