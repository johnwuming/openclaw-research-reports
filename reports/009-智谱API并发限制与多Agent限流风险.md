# R-009：智谱 API 并发限制 + OpenClaw 多 Agent 并行限流风险

**日期**：2026-03-29  
**方法论**：4 Search Agent → 双 Reviewer（accuracy 7.2 / completeness 7.0）→ 收敛

---

## 一、核心发现

### 1.1 智谱 API 并发限制

**关键事实：智谱不公开具体并发数值。**

- 通用 API 并发限制按 **模型 × 用户权益等级** 划分，具体数值需登录控制台 `bigmodel.cn/usercenter/proj-mgmt/rate-limits` 查看 [\[1\]](https://docs.bigmodel.cn/cn/api/rate-limit)
- "并发数"指同一时刻正在处理中的请求数量（非 RPM/TPM，而是 concurrent connections）[\[1\]](https://docs.bigmodel.cn/cn/api/rate-limit)
- 通用 API 用户可通过控制台申请提升并发；**Coding Plan 用户不可申请调整** [\[1\]](https://docs.bigmodel.cn/cn/api/rate-limit)

### 1.2 Coding Plan 并发与额度

| 套餐 | 建议同时项目 | 额度/5h | 额度/周 | 支持模型 |
|------|-------------|---------|---------|----------|
| Lite | 1 个 | ~80 次 | ~400 次 | GLM-5.1, GLM-5-Turbo, GLM-4.x |
| Pro | 1-2 个 | ~400 次 | ~2000 次 | 同上 + GLM-5 |
| Max | 2+ 个 | ~1600 次 | ~8000 次 | 全部 |

- GLM-5.1 / GLM-5 / GLM-5-Turbo 作为高阶模型，按 **高峰期 3 倍、非高峰期 2 倍** 系数消耗额度（限时：GLM-5.1 和 GLM-5-Turbo 非高峰期 1 倍抵扣至 2026 年 4 月底）[\[2\]](https://docs.bigmodel.cn/cn/coding-plan/overview)

### 1.3 限流错误码与重试

| 错误码 | 含义 | 触发场景 |
|--------|------|----------|
| **1302** | 用户速率限制 | 单账户请求过快 |
| **1305** | 平台服务过载 | 全局流量过大，与单一账户无关 |
| **HTTP 429** | APIReachLimitError | SDK 层面限流 |
| **HTTP 503** | APIServerFlowExceedError | 服务器过载 |

- 智谱官方 Python SDK 支持 `max_retries` 参数（示例默认 3 次），基于 httpx 重试机制 [\[3\]](https://github.com/MetaGLM/zhipuai-sdk-python-v4)
- **未找到官方 exponential backoff 的明确建议**

### 1.4 历史限流事件（2026 Q1）

- 2026 年 1-2 月，GLM-5 需求激增导致严重限流，部分用户 GLM-4.7/4.6 并发被降至 1（社区反馈，非官方确认）[\[4\]](https://www.zhihu.com/question/1997433388664131775)
- 智谱官方承认"并发访问量突破了既有规划上限"，启动算力合伙人计划 [\[5\]](https://news.qq.com/rain/a/20260216A05SUL00)
- **启示**：智谱的并发能力在高峰期可能不稳定，需要防御性设计

---

## 二、OpenClaw 并发控制机制

### 2.1 Agent 级别排队

- 每个 agent session 通过 **session lane** 串行化请求（同一 session 不会并发）
- Sub-agent 通过全局 **`subagent` lane** 排队，但不同 sub-agent session 之间是 **并行** 的 [\[6\]](https://docs.openclaw.ai/tools/subagents)
- **关键缺口：未找到 `maxConcurrent` 配置的文档**，OpenClaw 似乎不限制同时运行的 sub-agent 数量

### 2.2 Model Failover 机制

OpenClaw 处理模型失败的两阶段机制：

1. **Auth Profile 轮换**：同一 provider 内切换 API key（仅在 429/rate_limit/quota 响应时触发）
2. **Model Fallback**：切换到 `agents.defaults.model.fallbacks` 中的下一个模型

Auth Profile Cooldown 指数退避：**1min → 5min → 25min → 1h（上限）** [\[7\]](https://docs.openclaw.ai/concepts/model-failover)

Session Stickiness：同一 session 会 pin 住 auth profile 直到 reset/compaction/cooldown [\[7\]](https://docs.openclaw.ai/concepts/model-failover)

### 2.3 缺失的机制

- ❌ 无内置的请求级 rate limiter（不会主动限速发往模型的请求）
- ❌ 无显式的 sub-agent 并发上限配置（或文档未记载）
- ✅ 有 auth profile 轮换应对 429

---

## 三、当前研究团队并发风险分析

### 3.1 v4 流程峰值并发

```
Phase 2（探索）：
  Lead 1 + Search Agent × 4 = 5 个同时活跃 session
  → 每个 Search Agent 内部可能多次调用模型（思考+工具+总结）
  → 估计峰值：5-8 个并发模型请求

Phase 4（验证）：
  Lead 1 + Reviewer × 2 = 3 个同时活跃 session
  → 峰值：3-5 个并发模型请求

Phase 6（收敛）：
  Lead 1 + Citation 1 = 2 个
  → 峰值：2-3 个并发模型请求
```

**最危险的是 Phase 2**：4 个 Search Agent 同时启动，每个 agent 的 loop 内有 3-5 次模型调用（思考→工具→思考→工具→输出），若时间重叠，可能在短时间内产生 **15-20 个并发请求**。

### 3.2 智谱 Coding Plan 风险评估

**如果使用 Coding Plan（Pro 套餐）**：
- 建议同时 1-2 个项目 = 极低并发容忍
- 每 5h 约 400 次，v4 流程一次完整研究消耗约 20-30 次模型调用
- **风险等级：🔴 高** — Phase 2 的 4 个并行 Search Agent 几乎必然触发限流

**如果使用通用 API（Token Plan）**：
- 具体并发数取决于权益等级，未公开
- 但根据 2026 Q1 限流事件，即使是付费用户在高峰期也可能被降至极低并发
- **风险等级：🟡 中高** — 并行 agent 数量可能超出默认并发限制

### 3.3 OpenClaw 自身的保护

- Auth profile 轮换可以缓解（如果有多个 API key）
- 但 **每个 sub-agent 有独立的 session**，pin 不同的 auth profile，不会自动共享限流状态
- 指数退避（1min→5min→25min→1h）在频繁限流时会导致 agent 长时间阻塞

---

## 四、竞品并发限制对比

| 提供商 | 限流模式 | 免费层大致限制 | 备注 |
|--------|----------|---------------|------|
| **智谱** | 模型×权益等级，并发数 | 未公开 | Coding Plan 极度保守 |
| **DeepSeek** | 动态调整，不设固定上限 | 无固定值 | 10min 超时断连 [\[\8\]](https://api-docs.deepseek.com/zh-cn/quick_start/rate_limit) |
| **阿里云 Qwen** | RPM + TPM 双维度 | qwen-flash: RPM 30K, TPM 10M | 快照版本严格得多（RPM 60）[\[9\]](https://help.aliyun.com/zh/model-studio/rate-limit) |
| **Moonshot/Kimi** | 并发+RPM+TPM+TPD 四维度 | RPM ≈ 3（未验证） | 按充值金额分级 [\[\10\]](https://platform.moonshot.cn/docs/pricing/limits) |

**结论**：阿里云 Qwen 的限制最宽松，DeepSeek 不设固定上限但动态限流不可预测，智谱和 Kimi 较严格。

---

## 五、实践建议

### 5.1 立即可做（配置层面）

1. **配置多个智谱 API key**：在 OpenClaw 的 model provider 配置中添加多个 auth profiles，利用 auth profile 轮换分担限流压力
2. **设置 model fallbacks**：将 `zai/glm-5-turbo` 的 fallback 设为 `zai/glm-4.7` 或其他低成本模型
   ```yaml
   agents:
     defaults:
       model:
         fallbacks:
           - zai/glm-5-turbo
           - zai/glm-4.7
   ```
3. **考虑限制并行 Search Agent 数量**：从 4 个降到 2-3 个，虽牺牲速度但大幅降低限流风险

### 5.2 中期优化（架构层面）

4. **错峰策略**：将深度研究任务安排在非高峰期（智谱非高峰期消耗系数更低、并发更宽松）
5. **混合模型策略**：Search Agent 用 glm-5-turbo（便宜），Reviewer 用 glm-5.1（准确），分散不同模型的并发压力
6. **引入外部 rate limiter**：如果 OpenClaw 不提供内置限流，可在 gateway hook（`before_model_resolve`）中实现简单的令牌桶

### 5.3 长期方案

7. **关注 OpenClaw `maxConcurrent` 配置**：持续跟踪文档更新，等待官方支持 sub-agent 并发上限
8. **备用 provider**：将阿里云 Qwen 作为 fallback provider，利用其宽松的 RPM 限制（30K）作为安全网
9. **监控 Coding Plan 额度消耗**：v4 一次完整研究（4 Search + 2 Reviewer + 1 Citation）约消耗 20-40 次 prompt，Pro 套餐每 5h 仅 400 次

---

## 六、知识缺口

1. **智谱通用 API（Token Plan）各权益等级的具体并发数** — 需登录控制台查看，无法从外部获取
2. **OpenClaw `maxConcurrent` 是否存在** — 文档中未找到，可能未实现或命名不同
3. **智谱 429 响应是否包含 `Retry-After` header** — 未确认
4. **OpenClaw sub-agent 实际触发限流时的行为** — 无实战案例数据
5. **GLM-5.1 在通用 API 中的定价和并发** — 信息较少

---

## 七、来源列表

1. [智谱 API 限流文档](https://docs.bigmodel.cn/cn/api/rate-limit)
2. [Coding Plan 概览](https://docs.bigmodel.cn/cn/coding-plan/overview)
3. [智谱 Python SDK GitHub](https://github.com/MetaGLM/zhipuai-sdk-python-v4)
4. [知乎：GLM-4.7/4.6 并发被砍](https://www.zhihu.com/question/1997433388664131775)
5. [腾讯网：智谱 GLM-5 流量超预期](https://news.qq.com/rain/a/20260216A05SUL00)
6. [OpenClaw Sub-Agents 文档](https://docs.openclaw.ai/tools/subagents)
7. [OpenClaw Model Failover 文档](https://docs.openclaw.ai/concepts/model-failover)
8. [DeepSeek Rate Limit 文档](https://api-docs.deepseek.com/zh-cn/quick_start/rate_limit)
9. [阿里云百炼限流文档](https://help.aliyun.com/zh/model-studio/rate-limit)
10. [Moonshot API 限流文档](https://platform.moonshot.cn/docs/pricing/limits)

---

## 八、方法论反思

**做得好**：
- 4 个 Search Agent 覆盖了智谱限流、错误码、竞品对比、OpenClaw 机制四个维度
- 双 Reviewer 机制识别了 f16（Kimi RPM≈3）的低可信度和 f12（DeepSeek 不限并发）的绝对化表述

**需改进**：
- 智谱具体并发数值是核心问题但无法从外部获取，这是结构性限制
- OpenClaw 本地文档路径不可用（文件不存在），只能依赖远程文档
- 竞品信息对核心问题的直接帮助有限
