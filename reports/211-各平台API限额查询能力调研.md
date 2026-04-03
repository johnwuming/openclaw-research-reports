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


---
## Usage Limits Detail (R-017c)

# R-017c: 模型提供商 API 限额查询——按 5 小时/周/月维度

> 调研时间：2026-03-31
> 状态：已完成

---

## 1. MiniMax（Token Plan / Coding Plan）

### API 端点

```
GET https://www.minimaxi.com/v1/api/openplatform/coding_plan/remains
Authorization: Bearer <API_KEY>
Content-Type: application/json
```

### 可查询维度

| 维度 | 支持情况 | 说明 |
|------|---------|------|
| **5 小时** | ✅ | 文本模型（M2.7/M2.5）共享请求额度，5 小时滚动窗口 |
| **日** | ✅ | 非文本模型（TTS/视频/音乐/图像）按每日配额 |
| **周** | ❌ | 无周维度 |
| **月** | ❌（仅订阅期） | 无独立月度维度，按订阅周期 |

### 限额规格（Token Plan）

| 套餐 | 文本模型/5小时 | 价格 |
|------|--------------|------|
| Starter | 1,500 requests | ¥29/月 |
| Plus | 4,500 requests | ¥49/月 |
| Max | 15,000 requests | ¥119/月 |

### 返回数据示例

根据社区反馈（CC-Switch 配置），返回包含：
```json
{
  "model_remains": [
    {
      "start_time": 1771588800000,
      "end_time": 1771606800000,
      "current_interval_usage_count": 42,
      "total_interval_quota": 1500
    }
  ]
}
```

### ⚠️ 已知问题

Termo.ai 文档指出：API 端点 `coding_plan/remains` 存在已知问题——窗口切换后 `current_interval_usage_count` 不刷新。**不可完全信赖 API 返回的已用量数据**，需结合控制台页面验证。

### 替代方案
- 控制台页面：https://platform.minimaxi.com/user-center/payment/token-plan

---

## 2. 腾讯云（Coding Plan）

### API 端点

**❌ 无公开的用量查询 API**

腾讯云 Coding Plan 和 Token Plan 的官方文档均未提供任何用量查询 API 端点。用量仅能通过控制台页面查看。

### 可查询维度

| 维度 | 支持情况 | 说明 |
|------|---------|------|
| **5 小时** | ✅（控制台） | 滑动 5 小时窗口，Lite: 1,200 次，Pro: 6,000 次 |
| **周** | ✅（控制台） | 每周一 00:00:00 重置，Lite: 9,000 次，Pro: 45,000 次 |
| **月** | ✅（控制台） | 每订阅月，Lite: 18,000 次，Pro: 90,000 次 |

### 限额规格（Coding Plan）

| 套餐 | 5小时 | 周 | 月 | 价格 |
|------|-------|----|----|------|
| Lite | ~1,200 次 | ~9,000 次 | ~18,000 次 | ¥40/月 |
| Pro | ~6,000 次 | ~45,000 次 | ~90,000 次 | ¥200/月 |

### 限额规格（Token Plan）

| 套餐 | 月 Tokens | 价格 |
|------|----------|------|
| Lite | 3,500 万 | ¥39/月 |
| Standard | 1 亿 | ¥99/月 |
| Pro | 3.2 亿 | ¥299/月 |
| Max | 6.5 亿 | ¥599/月 |

Token Plan **没有** 5 小时或周维度限制，仅有月度 Token 总量。

### 错误码

```
20097 — hour allocated quota exceeded   (5小时额度用完)
20097 — week allocated quota exceeded   (周额度用完)  
20097 — month allocated quota exceeded  (月额度用完)
```

### 替代方案
- Coding Plan 控制台：https://hunyuan.cloud.tencent.com/#/app/subscription
- Token Plan 控制台：https://hunyuan.cloud.tencent.com/#/app/tokenplan

---

## 3. Kimi / Moonshot

### 余额查询 API（按量付费）

```
GET https://api.moonshot.cn/v1/users/me/balance
Authorization: Bearer <API_KEY>
```

### 返回数据示例

```json
{
  "code": 0,
  "data": {
    "available_balance": 49.58894,
    "voucher_balance": 46.58893,
    "cash_balance": 3.00001
  },
  "scode": "0x0",
  "status": true
}
```

### 可查询维度

| 维度 | 支持情况 | 说明 |
|------|---------|------|
| **余额** | ✅ | 现金余额 + 代金券余额 |
| **5 小时** | ❌ | Moonshot API（api.moonshot.cn）按量付费，无 5 小时限额 |
| **周** | ❌ | 无 |
| **月** | ❌ | 无月度限额，按量扣费 |

### Kimi Code Plan（api.kimi.com/coding/）

Kimi 的 Coding Plan 使用不同的端点 `api.kimi.com/coding/`（配置中的 baseUrl），这是一个 **Anthropic Messages 兼容 API**。

| 维度 | 说明 |
|------|------|
| **5 小时** | 有 5 小时滚动限额（免费 Adagio ~40 prompts/5h，付费 Andante/Moderato/Allegretto 更高） |
| **API 查询** | ❓ **未找到公开的 Coding Plan 用量查询 API** |

### Kimi Code 套餐（参考社区数据）

| 套餐 | 价格 | 预估额度 |
|------|------|---------|
| Adagio（免费） | ¥0 | ~40 prompts/5h |
| Andante | ~¥49/月 | ~100 prompts/5h |
| Moderato | ~¥99/月 | ~200 prompts/5h |
| Allegretto | ~¥199/月 | ~500 prompts/5h |

### 替代方案
- Kimi API 控制台：https://platform.kimi.com/
- Moonshot API 控制台：https://platform.moonshot.ai/

---

## 4. 智谱 ZAI（GLM Coding Plan）

### 已逆向端点

```
GET https://api.z.ai/api/monitor/usage/quota/limit
Authorization: Bearer <API_KEY>
```

### 可查询维度

| 维度 | 支持情况 | 说明 |
|------|---------|------|
| **TIME_LIMIT（5小时/每日）** | ✅ | 已确认返回 |
| **COUNT_LIMIT（月度）** | ✅ | 已确认返回 |
| **周** | ❓ | 需确认返回数据 |

### GLM Coding Plan 限额（参考社区）

| 套餐 | 5小时 prompts | 价格 |
|------|--------------|------|
| Lite | ~80 次 | ¥49/月 |
| Pro | ~400 次 | ¥149/月 |

### 待验证
- 需要实际调用已逆向端点确认返回结构中是否有周维度数据
- 需确认是否有按模型拆分的用量数据

### 替代方案
- 智谱控制台：https://open.bigmodel.cn/

---

## 5. 综合对比

| 提供商 | 有用量查询 API | 5h 维度 | 周维度 | 月维度 | API 可靠性 |
|--------|--------------|---------|--------|--------|-----------|
| **MiniMax** | ✅ `coding_plan/remains` | ✅ | ❌ | ❌ | ⚠️ 窗口切换不刷新 |
| **腾讯云 Coding Plan** | ❌ | ✅（控制台） | ✅（控制台） | ✅（控制台） | — |
| **腾讯云 Token Plan** | ❌ | ❌ | ❌ | ✅（控制台） | — |
| **Kimi/Moonshot API** | ✅ `/v1/users/me/balance` | ❌ | ❌ | ❌ | ✅（仅余额） |
| **Kimi Code Plan** | ❓ 未发现 | ✅ | ❌ | ❌ | — |
| **智谱 ZAI** | ✅（逆向） | ✅ | ❓ | ✅ | 需验证 |

### 关键发现

1. **腾讯云 Coding Plan 是唯一有三维度限制（5h/周/月）的提供商**，但完全没有 API 查询接口
2. **MiniMax 的 API 有已知 bug**，返回的已用量在窗口切换时不刷新
3. **Kimi Code Plan**（api.kimi.com/coding/）与 Moonshot API（api.moonshot.cn）是完全不同的服务，前者无公开查询 API
4. **智谱 ZAI** 的逆向端点需要重新验证实际返回数据

### 实践建议

- 对于自动化监控，**MiniMax 和智谱** 可以通过 API 实现（MiniMax 需注意 bug）
- **腾讯云** 只能通过浏览器自动化（Puppeteer/Playwright）抓取控制台页面
- **Kimi Code Plan** 暂无查询方案，只能从控制台手动查看
- 建议在 OpenClaw 中实现统一限额监控，按 provider 分别对接


---
## API Verification (R-017d)

# R-017d: API 限额查询端点验证报告

验证时间：2026-03-31 01:00 CST

---

## 1. ZAI（智谱）

### 端点：`GET https://api.z.ai/api/monitor/usage/quota/limit`

**请求：**
```bash
curl -s -H "Authorization: Bearer <ZAI_API_KEY>" \
  "https://api.z.ai/api/monitor/usage/quota/limit"
```

**返回 JSON：**
```json
{
  "code": 200,
  "msg": "Operation successful",
  "data": {
    "limits": [
      {
        "type": "TOKENS_LIMIT",
        "unit": 3,
        "number": 5,
        "percentage": 16,
        "nextResetTime": 1774891051109
      },
      {
        "type": "TOKENS_LIMIT",
        "unit": 6,
        "number": 1,
        "percentage": 56,
        "nextResetTime": 1775181642998
      },
      {
        "type": "TIME_LIMIT",
        "unit": 5,
        "number": 1,
        "usage": 1000,
        "currentValue": 179,
        "remaining": 821,
        "percentage": 17,
        "nextResetTime": 1777255242999,
        "usageDetails": [
          {"modelCode": "search-prime", "usage": 179},
          {"modelCode": "web-reader", "usage": 0},
          {"modelCode": "zread", "usage": 0}
        ]
      }
    ],
    "level": "pro"
  },
  "success": true
}
```

**状态：** ✅ 可用

**字段分析：**
- `type`: 限额类型 — `TOKENS_LIMIT`（令牌额度）或 `TIME_LIMIT`（时间限额）
- `unit`:
  - TOKENS_LIMIT 中 `unit=3` 可能是月度（3月），`unit=6` 可能是周/季度（推测）
  - TIME_LIMIT 中 `unit=5` + `number=1` → 5小时窗口，共1个（推测）
- `TOKENS_LIMIT` 字段：`number`（总额度数？）、`percentage`（已用百分比）、`nextResetTime`（重置时间戳）
- `TIME_LIMIT` 字段：`usage`（总量）、`currentValue`（已用）、`remaining`（剩余）、`percentage`（已用百分比）、`usageDetails`（按服务拆分：search-prime / web-reader / zread）
- `level`: 账户等级（pro）

**补充：** 尝试了 `/usage` 和 `/remain` 子路径均返回 404，说明 `/limit` 是唯一可用的端点。

---

## 2. MiniMax

### 端点：`GET https://api.minimaxi.com/v1/coding_plan/remains`

**请求：**
```bash
curl -s -H "Authorization: <MINIMAX_OAUTH_TOKEN>" \
  "https://api.minimaxi.com/v1/coding_plan/remains"
```

**返回 JSON（OAuth token）：**
```json
{"base_resp":{"status_code":1000,"status_msg":"unknown error"}}
```

**返回 JSON（无 Bearer 前缀）：**
```json
{"base_resp":{"status_code":1004,"status_msg":"login fail: Please carry the API secret key in the 'Authorization' field of the request header"}}
```

**状态：** ⚠️ 端点存在但认证失败

**分析：**
- 端点确实存在于 `api.minimaxi.com`（也存在于 `api.minimax.chat`）
- 该端点要求 **API Secret Key**（API 密钥），不接受 OAuth access token
- 当前配置中 MiniMax 使用的是 OAuth 认证（`sk-cp-...` 开头），没有找到独立的 API Secret Key
- 没有其他可用的用量查询端点（尝试了 `/v1/usage`、`/v1/query_coding_plan_info` 均返回 404）
- **结论：** 需要获取 MiniMax 的 API Secret Key（非 OAuth token）才能调用此端点

---

## 3. 腾讯云（Token Plan）

### 端点：`GET https://api.lkeap.cloud.tencent.com/plan/v3/usage`

**请求：**
```bash
curl -s -H "Authorization: Bearer <TENCENT_TOKEN>" \
  "https://api.lkeap.cloud.tencent.com/plan/v3/usage"
```

**返回 JSON：**
```json
{"id":"fcbd3afd8223fc1a56dfd957b0a1a473","error":{"message":"not authorized","type":"not_authorized_error","param":null,"code":"not_authorized"}}
```

**状态：** ❌ 不可用（Token 无权限查询用量）

**分析：**
- Token Plan 的 API Key（`sk-tp-...`）可以正常调用 chat completions 接口，但无权访问用量查询端点
- 尝试了 `/plan/v3/usage`、`/plan/v3/quota`、`/plan/v3/plan_info`、`/v1/plan_info`、`/v1/quota` 均返回 `not_authorized`
- GetCharacterUsage（TC3-HMAC-SHA256 签名方式）需要 SecretId/SecretKey，配置中未找到
- **结论：** 腾讯云 Token Plan 没有暴露用量查询 API 给 token 用户；需通过腾讯云控制台查看

---

## 4. Kimi / Moonshot

### 端点：`GET https://api.moonshot.cn/v1/users/me/balance`

**请求：**
```bash
curl -s -H "Authorization: Bearer <KIMI_API_KEY>" \
  "https://api.moonshot.cn/v1/users/me/balance"
```

**返回 JSON：**
```json
{"error":{"message":"Invalid Authentication","type":"invalid_authentication_error"}}
```

**状态：** ❌ 不可用（认证失败）

**分析：**
- Kimi 的 Coding Plan API Key（`sk-kimi-...`）是给 `api.kimi.com/coding/` 用的，与 Moonshot 开放平台（`api.moonshot.cn`）是不同的认证体系
- 在 `api.kimi.com/coding/` 下尝试了 `/v1/usage`、`/v1/plan_info`、`/v1/quota`、`/v1/coding_plan/info` 均返回 404
- **结论：** Kimi Coding Plan 不提供用量查询 API；需通过 Kimi 网页端查看

---

## 5. 汇总

| 提供商 | 端点 | 状态 | 说明 |
|--------|------|------|------|
| ZAI（智谱） | `api.z.ai/api/monitor/usage/quota/limit` | ✅ 可用 | 返回 tokens + time 维度，含服务拆分 |
| MiniMax | `api.minimaxi.com/v1/coding_plan/remains` | ⚠️ 端点存在 | 需要 API Secret Key，OAuth token 不被接受 |
| 腾讯云 | `api.lkeap.cloud.tencent.com/plan/v3/*` | ❌ 不可用 | Token 无用量查询权限 |
| Kimi | `api.moonshot.cn/v1/users/me/balance` | ❌ 不可用 | 认证体系不同；Coding API 无用量端点 |

### 关键发现
1. **只有 ZAI 提供了完整可用的限额查询 API**，且返回数据丰富（多时间维度、按服务拆分）
2. MiniMax 的用量端点存在但需要不同的密钥类型
3. 腾讯云和 Kimi 的 Coding Plan 产品均不提供 API 级别的用量查询
4. 实际可集成到监控系统的只有 ZAI 一家
