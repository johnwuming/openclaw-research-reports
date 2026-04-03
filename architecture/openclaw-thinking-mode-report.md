# OpenClaw 思考模式（Thinking）调研报告

> 调研日期：2026-04-01 | 信息来源：[OpenClaw 官方文档](https://docs.openclaw.ai/tools/thinking)

---

## 1. Thinking 配置选项

OpenClaw 支持以下 thinking 级别（通过 `/think`、`/t`、`/thinking` 指令设置）：

| 级别 | 别名/映射 | 说明 |
|------|----------|------|
| `off` | — | 关闭思考 |
| `minimal` | "think" | 最小思考 |
| `low` | "think hard" | 轻度思考（**reasoning-capable 模型的默认值**） |
| `medium` | "think harder" | 中等思考 |
| `high` | "ultrathink" / `highest` / `max` | 最大思考预算 |
| `xhigh` | "ultrathink+" / `x-high` / `extra-high` | 超高思考（**仅 GPT-5.2 + Codex 模型**） |
| `adaptive` | — | 由提供商自动管理推理预算（**仅 Anthropic Claude 4.6**） |

### 简写指令
- `/t <level>` = `/think:<level>` = `/thinking <level>`
- 仅发送 `/t medium`（指令独立消息）会设为会话默认
- 在消息中内联 `/t high` 仅对当前消息生效

---

## 2. Thinking 对 API 调用的影响

### 参数传递
- 思考级别解析后传递给模型运行时（Pi agent），映射为各提供商的推理参数
- **Anthropic Claude 4.6**：未显式设置时默认为 `adaptive`（由 Anthropic 自行决定推理预算）
- **Z.AI (`zai/*`)**：仅支持二值 `on`/`off`，任何非 `off` 级别映射为 `on`（等同于 `low`）
- **Moonshot (`moonshot/*`)**：`off` → `thinking: { type: "disabled" }`，非 `off` → `thinking: { type: "enabled" }`；启用时仅接受 `tool_choice: auto|none`

### Token 消耗差异
- **思考模式会产生额外 token 消耗**（推理 token），级别越高消耗越大
- `off`：零推理 token，最快最便宜
- `low`：轻度推理，适合日常任务
- `medium`/`high`：显著增加 token 用量，适合复杂推理任务
- `high`（ultrathink）：最大预算，token 消耗最高
- 推理 token 通常按提供商的"输出 token"费率计费（与普通输出 token 同价）

### 查看消耗
- `/usage full`：每条回复后显示 token 用量
- `/status`：显示当前会话的输入/输出 token 和预估费用

---

## 3. 开启 vs 关闭的质量差异

### 开启思考（`low` 及以上）
- ✅ 复杂推理、数学、代码调试、多步骤规划质量显著提升
- ✅ 减少幻觉，回答更准确
- ✅ 对"深层分析"类任务（如研究报告、架构设计）效果明显
- ❌ 响应延迟增加（模型需要先推理再输出）
- ❌ Token 成本增加（推理 token 按输出价格计费）

### 关闭思考（`off`）
- ✅ 响应速度最快
- ✅ Token 成本最低
- ✅ 适合简单问答、格式转换、翻译等不需要深度推理的任务
- ❌ 复杂任务可能出现推理错误或表面化回答

### 实践建议
| 场景 | 推荐级别 |
|------|---------|
| 日常闲聊 / 简单问答 | `off` |
| 常规任务（翻译、格式化） | `off` 或 `minimal` |
| 编码 / 调试 | `low` ~ `medium` |
| 深度研究 / 架构设计 | `medium` ~ `high` |
| 数学证明 / 极端复杂问题 | `high` |

---

## 4. 配置位置与优先级

### 优先级（从高到低）

1. **消息内联指令**：`帮我分析一下 /t high 这个问题` → 仅对当前消息生效
2. **会话覆盖**：发送独立指令 `/t medium` → 设为当前会话默认，可通过 `/t off` 清除
3. **Per-agent 默认**：`agents.list[].thinkingDefault`（openclaw.json）
4. **全局默认**：`agents.defaults.thinkingDefault`（openclaw.json）
5. **兜底**：Claude 4.6 → `adaptive`；其他推理模型 → `low`；非推理模型 → `off`

### 配置示例（openclaw.json）

```json
{
  "agents": {
    "defaults": {
      "thinkingDefault": "medium"
    },
    "list": [
      {
        "id": "research",
        "thinkingDefault": "high"
      },
      {
        "id": "chat",
        "thinkingDefault": "off"
      }
    ]
  }
}
```

### Reasoning 可见性（独立控制）
除了 thinking 级别，还有一个独立的 `/reasoning` 指令控制推理过程是否**可见**：
- `off`（默认）：不显示推理过程
- `on`：作为单独消息前缀 `Reasoning:` 发送
- `stream`（仅 Telegram）：流式显示推理过程，最终回复不含推理

**注意**：`/reasoning` 控制的是"是否展示"，不是"是否启用"。实际思考由 `/think` 级别控制。

---

## 5. 模型兼容性

| 提供商 | 支持情况 |
|--------|---------|
| **Anthropic Claude 4.6** | 完整支持，默认 `adaptive` |
| **OpenAI GPT-5.2 / Codex** | 完整支持，包括 `xhigh` |
| **Z.AI (`zai/*`)** | 仅二值 `on`/`off`，非 `off` 均映射为 `low` |
| **Moonshot (`moonshot/*`)** | 二值 enabled/disabled，启用时 `tool_choice` 受限 |
| **其他非推理模型** | 不支持，自动回退到 `off`；设置非 `off` 级别时行为取决于提供商（通常被忽略） |

### 不支持时的表现
- 如果模型不支持推理（thinking），设置非 `off` 级别**不会报错**，OpenClaw 会忽略该参数
- 对于只支持二值的提供商（Z.AI、Moonshot），OpenClaw 自动降级映射

---

## 总结

OpenClaw 的 thinking 系统设计得比较灵活：
1. **7 个级别**覆盖从关闭到超高推理的全范围
2. **4 层优先级**（内联 → 会话 → agent → 全局）提供精细控制
3. **各提供商适配**：自动处理不同提供商的差异（二值映射、adaptive 模式等）
4. **成本与质量权衡**：简单任务用 `off`，复杂任务按需提升
5. **推理可见性独立**：`/reasoning` 和 `/think` 是两个独立维度
