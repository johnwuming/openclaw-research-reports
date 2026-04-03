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
