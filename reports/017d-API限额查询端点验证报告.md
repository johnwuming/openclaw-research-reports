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
