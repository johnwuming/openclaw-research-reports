# R-017b: 智谱 ZAI API 限额查询端点逆向

## API 端点

```
GET https://api.z.ai/api/monitor/usage/quota/limit
```

## 认证方式

```
Authorization: Bearer <API_KEY>
Accept: application/json
```

## 返回数据示例

```json
{
  "code": 200,
  "msg": "Operation successful",
  "success": true,
  "data": {
    "limits": [
      {
        "type": "TOKENS_LIMIT",
        "unit": 3,
        "number": 5,
        "percentage": 14,
        "nextResetTime": 1774891051109
      },
      {
        "type": "TOKENS_LIMIT",
        "unit": 6,
        "number": 1,
        "percentage": 55,
        "nextResetTime": 1775181642998
      },
      {
        "type": "TIME_LIMIT",
        "unit": 5,
        "number": 1,
        "usage": 1000,
        "currentValue": 165,
        "remaining": 835,
        "percentage": 16,
        "nextResetTime": 1777255242999,
        "usageDetails": [
          { "modelCode": "search-prime", "usage": 165 },
          { "modelCode": "web-reader", "usage": 0 },
          { "modelCode": "zread", "usage": 0 }
        ]
      }
    ],
    "level": "pro"
  }
}
```

## 字段说明

### 顶层
| 字段 | 含义 |
|------|------|
| `code` | 状态码，200 为成功 |
| `msg` | 状态描述 |
| `success` | 是否成功 |
| `data.level` | 订阅等级（`pro`） |

### limits[] 数组
| 字段 | 含义 |
|------|------|
| `type` | 限额类型：`TOKENS_LIMIT`（Token 配额）或 `TIME_LIMIT`（时间/次数配额） |
| `unit` | 时间窗口单位：`3`=小时，`5`=月，`6`=日 |
| `number` | 时间窗口数值（与 unit 组合：unit=3, number=5 → 5小时窗口） |
| `percentage` | 已用百分比（0-100） |
| `nextResetTime` | 下次重置时间（Unix 毫秒时间戳） |

### TIME_LIMIT 额外字段
| 字段 | 含义 |
|------|------|
| `usage` | 总配额 |
| `currentValue` | 已用量 |
| `remaining` | 剩余量 |
| `usageDetails[]` | 按模型/服务的用量明细（`modelCode` + `usage`） |

## 测试结果

当前账户状态（pro 等级）：
- **5小时 Token 配额**：已用 14%
- **每日 Token 配额**：已用 55%
- **每月 API 调用次数**：165/1000（剩余 835 次），其中 search-prime 占 165 次

## 源码位置

OpenClaw 插件实现：`/home/noname/.npm-global/lib/node_modules/openclaw/dist/provider-usage-D_y-rSPa.js`（第 581 行起）
