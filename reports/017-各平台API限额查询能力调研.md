# R-017: Model Usage Dashboard — 各平台 API 限额查询能力调研

**日期**: 2026-03-31
**主题**: 调研当前在用模型的 API 限额查询能力，设计统一 Model Usage Dashboard

---

## 一、当前在用的 Provider/Model 清单

| Provider | 模型 | 用途 | 认证方式 |
|----------|------|------|---------|
| **ZAI (智谱)** | glm-5, glm-5-turbo, glm-5.1, glm-4.7, glm-4.7-flash, glm-4.7-flashx, glm-4.6, glm-4.6v, glm-4.5, glm-4.5-air, glm-4.5-flash, glm-4.5v | Main/Research agent 主力 | API Key (Coding Plan) |
| **MiniMax (minimax-portal)** | MiniMax-M2.7, MiniMax-M2.7-highspeed | Search agent | OAuth 门户 |
| **腾讯云 Token Plan** | tc-code-latest, hunyuan-2.0-instruct/thinking, hunyuan-t1, hunyuan-turbos, minimax-m2.5, kimi-k2.5, glm-5 | Reviewer/Citation/Fallback | API Key (sk-tp-xxx, Bearer Token) |
| **Kimi** | kimi-code (k2p5) | Fallback | API Key (Coding Plan) |

---

## 二、各平台 API 限额查询能力对比

### 对比总表

| 能力 | 智谱 ZAI | MiniMax | 腾讯云 Token Plan | Kimi (Moonshot) |
|------|---------|---------|------------------|----------------|
| **余额查询 API** | ❌ 无公开 API | ⚠️ 仅 Coding Plan 有 | ❌ 无 API | ✅ `GET /v1/users/me/balance` |
| **用量查询 API** | ⚠️ 仅 Claude Code 插件 | ✅ `GET /v1/api/openplatform/coding_plan/remains` | ❌ 仅控制台 | ❌ 无 |
| **响应头速率限制** | ❌ 未文档化 | ❌ 未文档化 | ❌ 未文档化 | ❌ 未文档化 |
| **控制台查询** | ✅ open.bigmodel.cn | ✅ platform.minimaxi.com | ✅ 腾讯云控制台 | ✅ platform.moonshot.cn |
| **第三方插件** | ✅ glm-plan-usage (Claude Code) | ❌ | ❌ | ❌ |
| **认证方式** | API Key (JWT) | Bearer Token | Bearer Token (sk-tp-xxx) | Bearer Token |

### 详细分析

#### 1. 智谱 AI (ZAI)

- **余额/用量 API**: 官方 API 文档（llms.txt 索引）中**不包含**任何 billing/usage/balance 端点
- **替代方案**: `glm-plan-usage` Claude Code 插件可查询 Coding Plan 配额，但底层 API 端点未公开
  - 插件源码: [zai-org/zai-coding-plugins](https://github.com/zai-org/zai-coding-plugins)
  - 命令: `/glm-plan-usage:usage-query`
- **可行路径**: 逆向插件源码提取底层 HTTP 端点，或直接爬取 open.bigmodel.cn 控制台页面
- **置信度**: 🟡 中 — 底层端点存在但未公开

#### 2. MiniMax

- **Coding Plan 用量 API**: ✅ 有
  - 端点: `GET https://www.minimaxi.com/v1/api/openplatform/coding_plan/remains`
  - 认证: `Authorization: Bearer <API Key>`
  - 返回: `{ "model_remains": [{ "start_time": 1771588800000, ... }] }`
  - 文档: [MiniMax Token Plan FAQ](https://platform.minimaxi.com/docs/token-plan/faq)
- **按量付费余额**: 仅控制台查看，无 API
- **限额模式**: 文本模型 5h 滚动窗口，非文本模型每日配额
- **置信度**: 🟢 高

#### 3. 腾讯云 Token Plan

- **用量查询**: ❌ 无 API，仅控制台（Token Plan > 我的套餐）可查看本周期总用量及近30天每日明细
- **相关 API**: 知识引擎平台有 `DescribeTokenUsage`（lke.tencentcloudapi.com），但**不确定**是否可查 Token Plan 用量
- **API Key**: 专属格式 `sk-tp-xxx`，Bearer Token 认证
- **可行路径**: 爬取腾讯云控制台页面（需登录态），或探索腾讯云 API 3.0 的资源查询接口
- **置信度**: 🟡 中

#### 4. Kimi (Moonshot)

- **余额查询 API**: ✅ 有
  - 端点: `GET https://api.moonshot.cn/v1/users/me/balance`
  - 认证: `Authorization: Bearer $MOONSHOT_API_KEY`
  - 返回: `{ "available_balance": X, "voucher_balance": X, "cash_balance": X }`（单位：人民币元）
  - 文档: [Kimi API Balance](https://platform.kimi.com/docs/api/balance)
- **Coding Plan 用量**: ⚠️ `/v1/users/me/balance` 返回的是现金/代金券余额，**Coding Plan 基于 Token 计量，可能不在该接口中体现**
- **流式 usage**: 每个 choice 结束数据块包含 prompt_tokens/completion_tokens/total_tokens
- **置信度**: 🟡 中（Coding Plan 适用性未确认）

---

## 三、技术方案设计

### 方案概览

由于 4 个平台中只有 MiniMax 和 Kimi 有（部分）查询 API，统一 Dashboard 需采用**混合数据采集策略**。

### 数据采集层

| Provider | 采集方式 | 可靠性 | 实现复杂度 |
|----------|---------|--------|-----------|
| MiniMax | REST API (`coding_plan/remains`) | 🟢 高 | 低 |
| Kimi | REST API (`/v1/users/me/balance`) | 🟢 高 | 低 |
| 智谱 ZAI | 逆向插件端点 或 爬控制台 | 🟡 中 | 中-高 |
| 腾讯云 Token Plan | 爬控制台 或 腾讯云 API 3.0 | 🟡 中 | 高 |

### 架构设计

```
┌─────────────────────────────────────────┐
│         Model Usage Dashboard           │
│         (Web UI / md-viewer)            │
└──────────────┬──────────────────────────┘
               │
┌──────────────▼──────────────────────────┐
│          Data Aggregation Layer          │
│         (Node.js / Python script)        │
├──────────┬──────────────────────────────┤
│ Provider │ Adapter                       │
├──────────┼──────────────────────────────┤
│ MiniMax  │ REST API → coding_plan/remains│
│ Kimi     │ REST API → /users/me/balance  │
│ ZAI      │ 逆向插件API 或 Puppeteer 爬取 │
│ Tencent  │ Puppeteer 爬取控制台          │
└──────────┴──────────────────────────────┘
```

### 实现策略

#### Phase 1：API 可用的 Provider（MiniMax + Kimi）

**MiniMax Adapter**:
```bash
curl -s 'https://www.minimaxi.com/v1/api/openplatform/coding_plan/remains' \
  -H 'Authorization: Bearer <API_KEY>' \
  -H 'Content-Type: application/json'
```
返回 `model_remains` 数组，包含各模型剩余额度和时间窗口。

**Kimi Adapter**:
```bash
curl -s 'https://api.moonshot.cn/v1/users/me/balance' \
  -H 'Authorization: Bearer <API_KEY>'
```
返回现金/代金券余额。需额外确认 Coding Plan 额度是否在此接口中体现。

#### Phase 2：逆向 ZAI 插件端点

1. 克隆 `zai-org/zai-coding-plugins` 仓库
2. 分析 `glm-plan-usage` 插件源码
3. 提取底层 HTTP 请求的 URL、headers、认证方式
4. 封装为 adapter

#### Phase 3：腾讯云 Token Plan

腾讯云控制台爬取需要：
- 腾讯云账号登录态（cookie/session）
- Puppeteer 或类似 headless browser
- 解析"我的套餐"页面

或探索腾讯云 API 3.0 的 `DescribeResourceUsage` 等接口。

### 展示层设计

**推荐方案**: 集成到现有 md-viewer（5173 端口），新增 `/dashboard` 路由。

**展示指标**:

| 指标 | MiniMax | Kimi | ZAI | 腾讯云 |
|------|---------|------|-----|--------|
| 剩余额度 | ✅ prompts/5h | ✅ 余额(元) | ⚠️ 待确认 | ⚠️ Token 剩余 |
| 已用 Token | ✅ | ❌ | ⚠️ | ✅ |
| 时间窗口重置 | ✅ 5h 滚动 | ❌ | ⚠️ | 月度 |
| RPM 限制 | ❌ | ❌ | ❌ | ❌ |
| 套餐等级 | Coding Plan | Coding Plan | Coding Plan | Token Plan |

**Dashboard 布局（手机友好）**:

```
┌──────────────────────────────┐
│     Model Usage Dashboard    │
│     Last updated: 5 min ago  │
├──────────────────────────────┤
│ 🟢 MiniMax M2.7-hs          │
│ ████████████░░░░  78% 剩余   │
│ 5h 窗口重置: 2h 30m 后       │
├──────────────────────────────┤
│ 🟢 Kimi (Moonshot)          │
│ 余额: ¥42.50                 │
│ Coding Plan: Andante         │
├──────────────────────────────┤
│ 🟡 智谱 ZAI (GLM)           │
│ Pro Plan: 600 prompts/5h     │
│ 已用: ~320 (53%)             │
├──────────────────────────────┤
│ 🟡 腾讯云 Token Plan         │
│ Pro: 3.2亿 Token/月          │
│ 已用: 1.2亿 (37%)            │
│ 周期重置: 4月25日             │
├──────────────────────────────┤
│        [🔄 刷新数据]          │
└──────────────────────────────┘
```

### 刷新与缓存策略

| 策略 | 值 | 理由 |
|------|-----|------|
| 缓存时间 | 5 分钟 | MiniMax 5h 窗口不需要秒级刷新 |
| 手动刷新 | 支持 | 用户可随时触发 |
| 自动刷新 | 5 分钟间隔 | Dashboard 打开时自动刷新 |
| 并发采集 | 4 provider 并行 | 总采集时间 ≤ 10s |

---

## 四、实现路线图

### Step 1: ZAI 插件逆向（优先级最高）

ZAI 是主力 provider，需要先确认其底层 API。

```bash
git clone https://github.com/zai-org/zai-coding-plugins.git
# 分析源码，提取 usage query 的 HTTP 端点
```

### Step 2: 编写数据采集脚本

推荐 Node.js（与 OpenClaw 技术栈一致）：

```javascript
// adapters/minimax.js
// adapters/kimi.js
// adapters/zai.js (待逆向后)
// adapters/tencent.js (待确认后)
```

### Step 3: Dashboard 页面

- 单 HTML 文件，内嵌 CSS + JS
- Fetch 采集脚本输出（JSON）
- 渐进式加载：先显示已获取的 provider

### Step 4: 集成到 md-viewer

在 md-viewer 的 express 应用中添加 `/dashboard` 路由。

---

## 五、知识缺口

1. **ZAI 插件底层端点**: glm-plan-usage 插件的具体 HTTP 端点未公开，需逆向源码
2. **Kimi Coding Plan 额度**: `/v1/users/me/balance` 是否包含 Coding Plan Token 额度未确认
3. **腾讯云 DescribeTokenUsage**: 知识引擎平台的 API 是否可查 Token Plan 数据未确认
4. **API 响应头**: 所有平台均未文档化响应头中的速率限制信息，需实际调用测试
5. **MiniMax 按量付费余额**: 充值用户的余额查询 API 未找到

---

## 六、来源列表

1. [智谱 AI glm-plan-usage 插件](https://docs.bigmodel.cn/cn/coding-plan/extension/usage-query-plugin) — 官方文档
2. [zai-org/zai-coding-plugins](https://github.com/zai-org/zai-coding-plugins) — GitHub
3. [智谱 AI API 文档索引](https://docs.bigmodel.cn/llms.txt) — 官方
4. [MiniMax Token Plan FAQ](https://platform.minimaxi.com/docs/token-plan/faq) — 官方文档
5. [MiniMax Coding Plan remains API](https://linux.do/t/topic/1631036) — LINUX DO 社区
6. [腾讯云 Token Plan 快速开始](https://cloud.tencent.com/document/product/1772/129450) — 官方文档
7. [腾讯云 Token Plan 常见问题](https://cloud.tencent.com/document/product/1772/129451) — 官方文档
8. [腾讯云 DescribeTokenUsage API](https://cloud.tencent.com/document/product/1759/111063) — 官方文档
9. [Kimi API Balance](https://platform.kimi.com/docs/api/balance) — 官方文档
10. [Kimi API 迁移指南](https://platform.moonshot.cn/docs/guide/migrating-from-openai-to-kimi) — 官方文档
11. [R-009b 多厂商并发限制评估](./R-009b-multi-provider-concurrency.md) — 内部研究

---

## 七、结论

4 个平台中，仅 **MiniMax 有完整的 Coding Plan 用量查询 API**（`coding_plan/remains`），**Kimi 有余额查询 API**（`/v1/users/me/balance`）。智谱 ZAI 的用量数据可通过逆向 `glm-plan-usage` 插件获取（存在未公开端点）。腾讯云 Token Plan 完全依赖控制台，无 API，是最难自动化的 provider。建议先逆向 ZAI 插件端点，再实现 Dashboard。Dashboard 可集成到现有 md-viewer（5173 端口），采用 5 分钟缓存的渐进式加载方案。
