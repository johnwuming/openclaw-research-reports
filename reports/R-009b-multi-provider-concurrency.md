# R-009b: 多厂商 Coding Plan 并发限制评估

**日期**: 2026-03-29  
**主题**: OpenClaw 多 agent 并行场景下，腾讯 Token Plan / MiniMax / Kimi / 智谱 ZAI 的限流风险

---

## 一、各厂商并发限制详情

### 1. 腾讯云 Token Plan（tencent-token-plan）

**什么是 Token Plan**: 腾讯云大模型 Token Plan 是面向 OpenClaw 等编程工具场景的订阅套餐，一次订阅可自由切换多个模型（HY 2.0、GLM-5、Kimi-K2.5、MiniMax-M2.5、Hunyuan-T1 等）。

**套餐档位**:

| 套餐 | 月 Token 额度 | 价格 |
|------|-------------|------|
| Lite | 3,500万 | ¥39/月 |
| Standard | 1亿 | ¥99/月 |
| Pro | 3.2亿 | ¥299/月 |
| Max | 6.5亿 | ¥599/月 |

**并发限制**: 
- 并发速率与套餐等级相关，平台动态调整，基本原则 Max > Pro > Standard > Lite
- **无明确 RPM/TPM 数值公开**，仅有"速率限制与套餐等级相关"的描述
- 高峰期动态限流，资源紧张时会降低并发

**转发第三方模型**: Token Plan 转发 GLM-5 / Kimi-K2.5 / MiniMax-M2.5 时，**并发限制由腾讯云控制**（不是原始厂商），因为请求经由腾讯云网关。但腾讯官方注明"Kimi-K2.5 当前资源负载较高，高峰时段可能触发请求限频"。

**使用限制**: 严禁 API 调用（自动化脚本、自定义后端、批量调用），仅限交互式 AI 工具。

**关键风险**: Token Plan 本质是 Coding Plan，有月 Token 总量上限。v4 研究流程 9 agent 并发 × 每次请求约 2-5K tokens = 峰值约 20-45K tokens/轮。一次完整研究任务可能消耗 200-500 万 tokens，Pro 套餐（3.2亿）约可支撑 60-150 次完整研究。

---

### 2. MiniMax（minimax-portal）

**API 速率限制**（官方文档 2026年3月）:

| 用户类型 | RPM | TPM |
|---------|-----|-----|
| 免费用户 | 20 | 1,000,000 |
| 充值用户 | **500** | **20,000,000** |

**关键数据**:
- 充值用户 RPM=500 意味着每分钟可发 500 个请求，远超 v4 并发需求（9 agent）
- TPM=2000万，按每个请求 5K tokens 计算，可支撑约 4000 并发请求/分钟
- 主账号+子账号共享限额

**MiniMax Coding Plan**:
- Starter ¥29/月，每 5 小时 40 次 prompt，无每周/月度限额
- 使用 M2.5 模型（非最新 M2.7）
- 仅限交互式工具使用

**v4 风险评估**: 如果 OpenClaw 的 minimax-portal 配置使用的是**充值 API Key**（非 Coding Plan），500 RPM / 2000万 TPM 完全够用，9 agent 并发无压力。如果是 Coding Plan Key，则受 40 prompts/5h 限制，会严重不足。

---

### 3. Kimi（Moonshot）

**API 速率限制**: 基于累计充值金额分级，具体数值在文档中以表格呈现（渲染未获取到完整数据），已知：
- 分并发数、RPM、TPM、TPD 四个维度
- 集群负载高时会临时限流
- 代金券不计入累计充值

**Kimi Coding Plan**:
- Andante 档 ¥49/月，按 Token 计量（2026.1.28 切换）
- 限时 3 倍额度活动（可能已结束）
- 支持 k2p5 / k2.5
- 工具适配少，非指定工具使用可能封号

**v4 风险评估**: Kimi 作为 fallback 使用时，并发压力不大。但 Coding Plan 的 Token 计量模式在高并发下可能快速耗尽额度。

---

### 4. 智谱 ZAI（zai）

**API 速率限制**（官方文档 2026年3月）:
- 按并发请求数限制（非 RPM），不同模型独立限额
- 与用户权益等级相关
- 高峰期（工作日白天、每天 15:00-18:00）动态限流
- 错误码 1302 = 用户速率限制，1305 = 平台过载

**GLM Coding Plan**:

| 套餐 | 价格 | 额度 | 推荐项目数 |
|------|------|------|-----------|
| Lite | $3/月 | 120 prompts/5h | 1个项目 |
| Pro | $15/月 | 600 prompts/5h | 1-2个项目 |
| Max | 更高 | 更高 | 2+项目 |

**GLM-5 特殊规则**: 高峰期按 3 倍抵扣，非高峰期按 2 倍抵扣额度。GLM-5-Turbo 非高峰期 1 倍优惠至 2026 年 4 月底。

**v4 风险评估**: Research agent（Lead + Search ×4）= 5 agent 同时用 ZAI，Pro 套餐建议 1-2 个项目。v4 峰值 5 agent ≈ 相当于 2.5 个项目并发，已超出 Pro 推荐上限。高峰期 3 倍抵扣会加速耗尽额度。

---

## 二、当前配置限流风险评估

### 峰值并发矩阵

| Agent | 数量 | 厂商 | 主要限制 |
|-------|------|------|---------|
| Lead | 1 | ZAI GLM-5-turbo | Coding Plan Pro 600 prompts/5h |
| Search | 4-5 | MiniMax M2.7-hs | API 500 RPM / Coding Plan 40/5h |
| Reviewer | 2 | 腾讯 Token Plan GLM-5 | 动态并发，Token 总量 |
| Citation | 1 | 腾讯 Token Plan GLM-5 | 同上 |

**总峰值**: 9 agent 同时请求

### 风险等级

| 厂商 | 风险 | 说明 |
|------|------|------|
| MiniMax (API) | 🟢 低 | 500 RPM 远超需求 |
| MiniMax (Coding Plan) | 🔴 高 | 40 prompts/5h 严重不足 |
| 腾讯 Token Plan | 🟡 中 | 无明确 RPM，动态调整，高峰期可能限流 |
| 智谱 ZAI | 🟡 中 | 5 agent 并发超出 Pro 推荐（1-2项目），高峰期 3x 抵扣 |
| Kimi (fallback) | 🟢 低 | 仅 fallback 使用，并发压力小 |

---

## 三、优化方案

### 1. 当前配置的优势：天然分散限流

当前配置已将不同 agent 分配到不同厂商，这是一个很好的策略：
- Search → MiniMax（RPM 充裕）
- Reviewer/Citation → 腾讯 Token Plan（独立限额池）
- Lead/Research → 智谱 ZAI（独立限额池）
- Fallback → Kimi（兜底）

**每个厂商只需承受部分并发压力**，总限流风险被有效分散。

### 2. 腾讯 Token Plan 转发 GLM-5 的限流归属

Token Plan 转发 GLM-5 时，**限流由腾讯云控制**，不受智谱原始厂商限制。这意味着：
- Reviewer/Citation 的并发请求走腾讯 Token Plan 的限额池
- 不会消耗智谱 ZAI 的 Coding Plan 额度
- 两个限额池完全独立 ✅

### 3. 推荐模型分配策略

| Agent | 推荐厂商 | 理由 |
|-------|---------|------|
| Main/Lead | ZAI GLM-5-turbo | 中文理解好，Agent 优化 |
| Search | MiniMax M2.7-hs | RPM=500 充裕，速度快 |
| Reviewer | 腾讯 Token Plan GLM-5 | 独立限额池，强推理模型 |
| Citation | 腾讯 Token Plan GLM-5 | 轻量任务，不限流 |
| Fallback | Kimi k2p5 | 长上下文，兜底 |

### 4. 具体优化建议

1. **确认 MiniMax 使用 API Key 而非 Coding Plan Key** — 这是最大的风险点。如果 minimax-portal 配置的是 Coding Plan Key（40 prompts/5h），v4 流程完全无法运行。

2. **智谱 ZAI 控制并发** — 建议将 Lead agent 的搜索延迟到 Search agent 返回后再发起新任务，避免 Lead + Search 同时向 ZAI 发请求。或者考虑 Lead 也使用 MiniMax/腾讯 Token Plan。

3. **腾讯 Token Plan 注意 Token 消耗** — Reviewer 2 + Citation 1 = 3 agent，每次研究任务约消耗 50-100 万 tokens。Pro 套餐 3.2 亿约支撑 30-60 次完整研究。

4. **高峰期避让** — 智谱 ZAI 高峰期（15:00-18:00 UTC+8）GLM-5 按 3 倍抵扣，建议重研究任务安排在非高峰期。

5. **Fallback 链路验证** — 确保 kimi/k2p5 fallback 配置正确，当主厂商限流时能无缝切换。

---

## 四、知识缺口

1. 腾讯 Token Plan 的**具体 RPM/并发连接数**未公开，仅有"动态调整"描述
2. Kimi API 的**具体限速数值**（按充值金额分级的详细表格）未获取到
3. 腾讯 Token Plan 转发第三方模型时，**底层路由机制**（是否有优先级/排队）不明确
4. MiniMax M2.7-highspeed 是否与 M2.5 共享限额，还是独立计数

---

## 五、来源列表

1. [腾讯云 Token Plan 套餐概览](https://cloud.tencent.com/document/product/1772/129449) — 官方文档
2. [MiniMax 速率限制](https://platform.minimaxi.com/docs/guides/rate-limits) — 官方文档
3. [智谱 AI 速率限制](https://docs.bigmodel.cn/cn/api/rate-limit) — 官方文档
4. [Kimi API 充值与限速](https://platform.moonshot.cn/docs/pricing/limits) — 官方文档
5. [Coding Plan 能当 API 用吗？](https://help.apiyi.com/coding-plan-api-restrictions-openai-codex-exception.html) — 各家限制对比
6. [2026年国内主流AI Coding Plan套餐全对比](https://www.cnblogs.com/wzxNote/p/19648084) — 横向评测
7. [GLM-5.1正式开放！智谱编程套餐全解析](https://unifuncs.com/s/Dm5egcKj) — UniFuncs

---

## 六、总结

当前 OpenClaw 配置通过多厂商分散策略已有效降低限流风险。**最大风险点是 MiniMax 使用 Coding Plan Key（40 prompts/5h）vs API Key（500 RPM）**。如果确认 minimax-portal 使用充值 API Key，v4 流程的 9 agent 并发在各厂商限额内均可安全运行。腾讯 Token Plan 转发 GLM-5 时限流由腾讯控制，不消耗智谱原始额度，两个限额池完全独立。建议避免在智谱高峰期（15:00-18:00）运行高并发研究任务，并确保 fallback 链路可用。
