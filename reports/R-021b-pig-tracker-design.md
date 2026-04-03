# 猪周期自动化跟踪系统 — 技术设计文档

> 设计日期：2026-03-31 | 研究代号：R-021b | 基于：R-021 猪周期研究报告 + R-020c 数据源评估
> 状态：**设计阶段**，待 dev team 实施

---

## 目录

1. [系统架构设计](#1-系统架构设计)
2. [数据采集模块详细设计](#2-数据采集模块详细设计)
3. [存储模块设计](#3-存储模块设计)
4. [分析模块设计](#4-分析模块设计)
5. [输出模块设计](#5-输出模块设计)
6. [运维设计](#6-运维设计)
7. [实施路线图](#7-实施路线图)
8. [风险和限制](#8-风险和限制)
9. [附录](#9-附录)

---

## 1. 系统架构设计

### 1.1 整体架构

```
                          ┌─────────────────────┐
                          │   定时调度器 (cron)   │
                          │  日更/周更/月更/季更   │
                          └────────┬────────────┘
                                   │
                          ┌────────▼────────────┐
                          │   数据采集层          │
                          │   DataCollector      │
                          │  ┌───────────────┐   │
                          │  │ AKShare (免费) │   │
                          │  │ 网页抓取 (半自)│   │
                          │  │ 手动录入 (兜底)│   │
                          │  └───────────────┘   │
                          └────────┬────────────┘
                                   │ 写入
                          ┌────────▼────────────┐
                          │   存储层             │
                          │   DuckDB             │
                          │   pig_cycle.duckdb   │
                          └────────┬────────────┘
                                   │ 读取
                          ┌────────▼────────────┐
                          │   分析层             │
                          │   CycleAnalyzer      │
                          │  · 周期位置评分      │
                          │  · 趋势检测          │
                          │  · 异常检测          │
                          └────────┬────────────┘
                                   │
                    ┌──────────────┼──────────────┐
                    │              │              │
           ┌───────▼──────┐ ┌────▼─────┐ ┌──────▼───────┐
           │  报告生成层   │ │  告警层   │ │  可视化层    │
           │  ReportGen   │ │  Alerter │ │  md-viewer   │
           │  周报/月报    │ │ Telegram │ │  OpenClaw    │
           │  Markdown    │ │  推送    │ │  集成        │
           └──────────────┘ └──────────┘ └──────────────┘
```

### 1.2 各模块职责与接口

| 模块 | 职责 | 输入接口 | 输出接口 |
|------|------|---------|---------|
| **DataCollector** | 从各数据源采集生猪产业数据 | 无（被 cron 触发） | 写入 DuckDB |
| **DuckDB** | 存储时序数据，支持分析查询 | SQL INSERT/SELECT | SQL 查询结果 |
| **CycleAnalyzer** | 评分模型、趋势判断、异常检测 | 从 DuckDB 读取 | JSON 分析结果 |
| **ReportGen** | 生成 Markdown 周报/月报 | 分析结果 + 原始数据 | `.md` 文件 |
| **Alerter** | 阈值告警推送 | 分析结果 | Telegram 消息 |
| **md-viewer** | 在 OpenClaw 中展示报告 | Markdown 文件路径 | 用户可读页面 |

### 1.3 技术栈

| 组件 | 技术选择 | 理由 |
|------|---------|------|
| 语言 | Python 3.10+ | AKShare 生态、数据分析工具链 |
| 数据库 | DuckDB（单文件嵌入式） | 列式存储，时序分析性能好，零运维 |
| 数据采集 | AKShare + requests + BeautifulSoup | 免费、覆盖度足够 |
| 调度 | cron（系统级） | 简单可靠，无需额外服务 |
| 报告输出 | Markdown | 与 OpenClaw md-viewer 兼容 |
| 告警推送 | OpenClaw Telegram 集成 | 现有能力，无需额外开发 |
| 版本管理 | Git | 配置文件、脚本版本追踪 |

### 1.4 目录结构

```
pig-tracker/
├── config/
│   ├── config.yaml           # 全局配置（数据库路径、告警阈值等）
│   └── akshare_fields.yaml   # AKShare 接口字段映射
├── collector/
│   ├── __init__.py
│   ├── base.py               # BaseCollector 基类
│   ├── akshare_collector.py  # AKShare 数据采集
│   ├── web_scraper.py        # 网页抓取（发改委、华储网等）
│   └── manual_input.py       # 手动录入辅助脚本
├── storage/
│   ├── __init__.py
│   ├── schema.py             # DuckDB 建表语句
│   └── repository.py         # 数据读写操作封装
├── analyzer/
│   ├── __init__.py
│   ├── cycle_scorer.py       # 周期位置评分模型
│   ├── trend_detector.py     # 趋势检测（环比、移动平均）
│   └── alert_rules.py        # 告警规则引擎
├── reporter/
│   ├── __init__.py
│   ├── weekly_report.py      # 周报生成
│   ├── monthly_report.py     # 月报生成
│   └── templates/
│       ├── weekly.md.j2      # 周报 Jinja2 模板
│       └── monthly.md.j2     # 月报模板
├── alerter/
│   ├── __init__.py
│   └── telegram_alerter.py   # Telegram 推送
├── jobs/
│   ├── daily_collect.py      # 日度采集入口
│   ├── weekly_analyze.py     # 周度分析+报告入口
│   └── monthly_collect.py    # 月度采集入口
├── tests/
│   └── ...
├── pig_cycle.duckdb          # 数据库文件（.gitignore）
├── requirements.txt
└── README.md
```

---

## 2. 数据采集模块详细设计

### 2.1 指标与采集方案完整映射

基于 R-021 §2.1 的 15 个核心指标，逐一明确采集方案：

#### 指标 1：能繁母猪存栏量

| 项目 | 说明 |
|------|------|
| **重要性** | ⭐⭐⭐⭐⭐ 最核心指标，领先猪价 ~10 个月 |
| **AKShare** | `ak.futures_pig_spot()` 返回产能子集，含"能繁母猪存栏"字段 |
| **数据频率** | 月度（官方农业农村部数据，月中旬发布上月数据） |
| **AKShare 可靠性** | 中。接口存在但字段映射需实测验证，历史数据深度不确定 |
| **备选方案** | ① 农业农村部官网新闻发布会文字稿抓取 ② 重庆市农委等地方农业部门网站同步转载 ③ 手动从券商研报提取 |
| **增量策略** | 查询 DuckDB 中最新月份，只插入新月份。官方数据不会修订，无需回刷 |
| **错误处理** | AKShare 失败 → 记录日志 → 跳过本次（月度数据滞后容忍度高）→ 3 天后重试。若连续 3 次失败 → 触发 Telegram 通知运维人员 |

#### 指标 2：生猪存栏量

| 项目 | 说明 |
|------|------|
| **重要性** | ⭐⭐⭐⭐ |
| **AKShare** | `ak.futures_pig_spot()` 产能子集，含"生猪存栏"字段 |
| **数据频率** | 季度（国家统计局发布） |
| **备选方案** | 国家统计局官网数据发布库 |
| **增量策略** | 按季度去重，只插入新季度 |
| **错误处理** | 同指标 1 |

#### 指标 3：仔猪价格

| 项目 | 说明 |
|------|------|
| **重要性** | ⭐⭐⭐⭐ |
| **AKShare** | ⚠️ `futures_pig_rank()` 不含仔猪价格。需测试 `ak.futures_pig_spot()` 是否有仔猪字段 |
| **数据频率** | 日度/周度 |
| **备选方案** | ① 我的农产品网网页抓取（部分免费）② 博亚和讯 ③ 手动从行业微信订阅号截图提取 |
| **AKShare 不可用时** | 设为"需手动补充"字段，周报中标注数据缺失 |
| **增量策略** | 仔猪价格日度更新但波动相对平缓，建议采集频率为**周度**（每周一采集上周均价）。按日期去重 |

#### 指标 4：母猪价格

| 项目 | 说明 |
|------|------|
| **重要性** | ⭐⭐⭐ |
| **AKShare** | ❌ 不支持。涌益咨询付费数据 |
| **数据频率** | 周度 |
| **备选方案** | 券商研报定期引用涌益数据，可半自动从公开研报提取。**优先级低，Phase 3 后考虑** |
| **增量策略** | 手动录入，按周 |
| **错误处理** | 无数据时跳过，分析模型中该指标权重设为 0 |

#### 指标 5：猪粮比

| 项目 | 说明 |
|------|------|
| **重要性** | ⭐⭐⭐⭐ |
| **AKShare** | ✅ `ak.futures_pig_spot()` 含"猪粮比价"字段 |
| **数据频率** | 周度（国家发改委价格监测中心） |
| **备选方案** | 可用外三元猪价 ÷ 玉米价格自行计算 |
| **增量策略** | 按周去重，记录发布日期 |
| **错误处理** | AKShare 失败 → 用日度价格自行计算近似值 |

#### 指标 6：猪料比

| 项目 | 说明 |
|------|------|
| **重要性** | ⭐⭐⭐⭐ |
| **AKShare** | ❌ 不直接支持。卓创资讯付费数据 |
| **数据频率** | 周度 |
| **备选方案** | ① 用玉米+豆粕加权计算全价料成本 → 反推猪料比 ② 行业公开报告引用 |
| **增量策略** | 估算值可日度生成（依赖玉米、豆粕日度价格） |
| **错误处理** | 玉米/豆粕价格缺失时跳过 |

#### 指标 7：屠宰量/开工率

| 项目 | 说明 |
|------|------|
| **重要性** | ⭐⭐⭐ |
| **AKShare** | ❌ 不支持。卓创资讯付费数据 |
| **数据频率** | 周度 |
| **备选方案** | 中国期货业协会/行业公开文章偶尔引用。**Phase 2 后考虑手动补充** |
| **增量策略** | 手动录入 |

#### 指标 8：二次育肥入场情况

| 项目 | 说明 |
|------|------|
| **重要性** | ⭐⭐ |
| **AKShare** | ❌ 不支持。数据不透明 |
| **数据频率** | 周度 |
| **备选方案** | 通过出栏体重变化间接推断（体重↑ 暗示压栏/二次育肥）。出栏体重见指标 14 |
| **建议** | 不直接采集，通过出栏体重间接代理 |

#### 指标 9：饲料销量

| 项目 | 说明 |
|------|------|
| **重要性** | ⭐⭐⭐ |
| **AKShare** | ❌ 不支持 |
| **数据频率** | 月度 |
| **备选方案** | 饲料工业协会官网发布全国饲料产量月报 → 网页抓取 |
| **增量策略** | 按月去重 |
| **错误处理** | 跳过，权重设为 0 |

#### 指标 10：养殖利润/亏损幅度

| 项目 | 说明 |
|------|------|
| **重要性** | ⭐⭐⭐⭐⭐ |
| **AKShare** | ⚠️ 需验证。`ak.futures_pig_spot()` 可能包含"自繁自养利润"字段，但不确定 |
| **数据频率** | 周度 |
| **备选方案** | ① 用猪价 - 饲料成本估算 ② 涌益/卓创公开引用数据 |
| **估算公式** | `自繁自养利润 ≈ (猪价 - 料肉比×全价料价格) × 出栏体重 - 固定成本分摊`。其中料肉比 ≈ 2.8，固定成本约 150-200 元/头 |
| **增量策略** | 按周去重 |
| **错误处理** | AKShare 无数据 → 用估算公式生成近似值，报告中标注"估算值" |

#### 指标 11：能繁母猪淘汰量

| 项目 | 说明 |
|------|------|
| **重要性** | ⭐⭐⭐⭐ |
| **AKShare** | ❌ 不支持。涌益咨询付费数据 |
| **数据频率** | 月度 |
| **备选方案** | 通过能繁母猪存栏环比变化间接推算：`淘汰量 ≈ 上月存栏 - 本月存栏 + 后备转能繁` |
| **增量策略** | 依赖能繁母猪月度数据 |
| **建议** | 不直接采集，间接推算 |

#### 指标 12：冻肉库存

| 项目 | 说明 |
|------|------|
| **重要性** | ⭐⭐ |
| **AKShare** | ❌ 不支持 |
| **数据频率** | 周度 |
| **备选方案** | 华储网收储/放储公告 → 网页抓取；卓创库存容数据需付费 |
| **增量策略** | 抓取华储网公告，解析公告文本 |
| **错误处理** | 低优先级，缺失不影响核心分析 |

#### 指标 13：猪肉进出口数据

| 项目 | 说明 |
|------|------|
| **重要性** | ⭐ |
| **AKShare** | ✅ `ak.china_import_export()` 或海关相关接口 |
| **数据频率** | 月度 |
| **备选方案** | 海关总署在线查询平台 |
| **增量策略** | 按月去重 |
| **注意** | 进口占比 <5%，对分析影响有限，可降低优先级 |

#### 指标 14：出栏体重

| 项目 | 说明 |
|------|------|
| **重要性** | ⭐⭐⭐⭐ |
| **AKShare** | ❌ 不支持 |
| **数据频率** | 周度 |
| **备选方案** | 涌益/卓创公开引用。**通过上市公司月度出栏数据中的体重信息间接参考**（牧原月报含出栏体重） |
| **增量策略** | 手动录入 |
| **替代方案** | 监测牧原/温氏月度销售简报中的出栏体重数据 |

#### 指标 15：天气/疫病因素

| 项目 | 说明 |
|------|------|
| **重要性** | 不定期，极端情况重要 |
| **AKShare** | ❌ |
| **数据频率** | 不定期 |
| **备选方案** | 农业农村部疫病公告网页抓取；新闻搜索 |
| **建议** | 不自动采集，由人工判断后手动录入重大事件 |

### 2.2 采集能力总结

| 采集方式 | 覆盖指标 | 自动化程度 |
|---------|---------|-----------|
| **AKShare 全自动** | 外三元猪价、内三元猪价、玉米价格、豆粕价格、猪粮比、能繁母猪存栏、生猪存栏、生猪出栏、猪肉产量 | 高（核心指标基本覆盖） |
| **AKShare + 计算** | 猪料比（估算）、养殖利润（估算）、母猪淘汰量（推算）、二次育肥（间接） | 中 |
| **网页抓取** | 华储网收储公告、饲料工业协会饲料产量、海关进出口 | 中 |
| **手动录入** | 仔猪价格、屠宰开工率、冻肉库存、出栏体重、母猪价格 | 低 |

### 2.3 增量更新通用策略

```python
class BaseCollector:
    """所有采集器的基类，封装增量逻辑"""

    def get_latest_date(self, table_name: str, date_column: str = "date") -> Optional[str]:
        """查询表中最新日期"""
        result = self.conn.execute(
            f"SELECT MAX({date_column}) FROM {table_name}"
        ).fetchone()
        return result[0] if result and result[0] else None

    def upsert_data(self, table_name: str, df: pd.DataFrame, key_columns: list[str]):
        """插入新数据，忽略已存在的（按 key 去重）"""
        # 使用 DuckDB 的 INSERT ... ON CONFLICT DO NOTHING
        # 或先过滤掉已有日期的数据
        existing = self.conn.execute(
            f"SELECT {','.join(key_columns)} FROM {table_name}"
        ).fetchall()
        existing_set = set(existing)
        for _, row in df.iterrows():
            key = tuple(row[k] for k in key_columns)
            if key not in existing_set:
                # INSERT
                pass
```

### 2.4 错误处理策略

```
采集失败处理流程：

1. 单次采集失败
   → 记录 warning 日志（指标名、错误类型、时间戳）
   → 跳过该指标，不影响其他指标采集
   → 数据库中该指标该日期标记为 NULL

2. 同一指标连续 3 次失败
   → 记录 error 日志
   → 触发 Telegram 告警："⚠️ {指标名} 连续 3 次采集失败，需人工介入"
   → 在周报中标注该指标数据缺失

3. AKShare 接口整体不可用
   → 切换到降级模式：仅使用本地已有数据生成报告
   → Telegram 通知运维人员
   → 记录 incident 日志

4. 数据异常值检测
   → 采集后对比前一值：涨跌幅 >20% 触发 warning
   → 对比历史同期均值：偏差 >30% 标记为"待验证"
   → 异常值不自动修正，但在报告中标注
```

### 2.5 AKShare 请求策略

- **请求间隔**：≥3 秒（反爬策略）
- **重试**：最多 3 次，指数退避（3s, 9s, 27s）
- **超时**：单次请求 30 秒
- **User-Agent**：使用 AKShare 默认
- **错误日志**：记录完整的 API 调用参数和返回状态

---

## 3. 存储模块设计

### 3.1 设计决策：按频率分表 vs 统一时序表

**选择：按频率分表（4 张核心表 + 2 张辅助表）**

理由：
1. 不同频率的数据时间粒度不同（日/周/月/季），统一表会导致大量 NULL
2. 分表后 SQL 查询更简洁，无需按频率过滤
3. DuckDB 对多表 JOIN 性能足够好

### 3.2 表结构 Schema

#### 表 1：`daily_prices` — 日度价格数据

```sql
CREATE TABLE daily_prices (
    date         DATE NOT NULL,              -- 日期（主键）
    pig_waisan   DECIMAL(6,2),               -- 外三元猪价 元/kg
    pig_neisan   DECIMAL(6,2),               -- 内三元猪价 元/kg
    pig_tuza     DECIMAL(6,2),               -- 土杂猪猪价 元/kg
    corn         DECIMAL(6,2),               -- 玉米价格 元/kg
    soybean_meal DECIMAL(6,2),               -- 豆粕价格 元/kg
    piglet_price DECIMAL(6,2),               -- 仔猪价格 元/kg（可能为 NULL）
    source       VARCHAR DEFAULT 'akshare',  -- 数据来源
    updated_at   TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (date)
);

-- 索引：日期范围查询
CREATE INDEX idx_daily_date ON daily_prices (date);
```

#### 表 2：`weekly_indicators` — 周度指标

```sql
CREATE TABLE weekly_indicators (
    week_end_date    DATE NOT NULL,           -- 周截止日期（周六，主键）
    pig_corn_ratio   DECIMAL(6,4),            -- 猪粮比（官方）
    pig_feed_ratio   DECIMAL(6,4),            -- 猪料比（估算）
    profit_self      DECIMAL(8,2),            -- 自繁自养利润 元/头
    profit_buy_piglet DECIMAL(8,2),           -- 外购仔猪利润 元/头
    slaughter_rate   DECIMAL(6,2),            -- 屠宰开工率 %
    frozen_inv_rate  DECIMAL(6,2),            -- 冻肉库存容 %
    avg_weight       DECIMAL(6,2),            -- 出栏均重 kg
    sow_price        DECIMAL(6,2),            -- 母猪价格 元/kg（可能为 NULL）
    source           VARCHAR DEFAULT 'mixed',
    updated_at       TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (week_end_date)
);

CREATE INDEX idx_weekly_date ON weekly_indicators (week_end_date);
```

#### 表 3：`monthly_capacity` — 月度产能数据

```sql
CREATE TABLE monthly_capacity (
    month           DATE NOT NULL,            -- 月份（YYYY-MM-01，主键）
    breeding_sow    DECIMAL(8,1),             -- 能繁母猪存栏 万头
    pig_inventory   DECIMAL(8,1),             -- 生猪存栏 万头
    pig_slaughter   DECIMAL(8,1),             -- 生猪出栏 万头（月度可能为 NULL，季度有值）
    pork_production DECIMAL(8,1),             -- 猪肉产量 万吨
    feed_production DECIMAL(8,1),             -- 全国饲料产量 万吨（可能为 NULL）
    sow_mom_change  DECIMAL(6,2),             -- 能繁母猪环比变化 %
    sow_yoy_change  DECIMAL(6,2),             -- 能繁母猪同比变化 %
    source          VARCHAR DEFAULT 'akshare',
    updated_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (month)
);

CREATE INDEX idx_monthly_date ON monthly_capacity (month);
```

#### 表 4：`quarterly_macro` — 季度宏观数据

```sql
CREATE TABLE quarterly_macro (
    quarter         DATE NOT NULL,            -- 季度首日（YYYY-01-01/04-01/07-01/10-01，主键）
    gdp_agri_yoy    DECIMAL(6,2),            -- GDP农业分项同比 %
    cpi_pork_yoy    DECIMAL(6,2),            -- CPI猪肉分项同比 %
    pig_inventory   DECIMAL(8,1),            -- 季末生猪存栏 万头（国家统计局）
    pig_slaughter_q DECIMAL(8,1),            -- 季度生猪出栏 万头
    source          VARCHAR DEFAULT 'manual',
    updated_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (quarter)
);
```

#### 辅助表 5：`alerts` — 告警记录

```sql
CREATE TABLE alerts (
    id          INTEGER PRIMARY KEY DEFAULT nextval('alert_seq'),
    created_at  TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    alert_type  VARCHAR NOT NULL,             -- 告警类型标识
    indicator   VARCHAR NOT NULL,             -- 触发指标
    value       DECIMAL(10,2) NOT NULL,       -- 当前值
    threshold   DECIMAL(10,2) NOT NULL,       -- 阈值
    direction   VARCHAR NOT NULL,             -- 'above' | 'below'
    message     TEXT,                          -- 告警消息
    notified    BOOLEAN DEFAULT FALSE,        -- 是否已推送
    acknowledged BOOLEAN DEFAULT FALSE         -- 是否已确认
);

CREATE SEQUENCE alert_seq START 1;
CREATE INDEX idx_alerts_type ON alerts (alert_type);
CREATE INDEX idx_alerts_created ON alerts (created_at);
```

#### 辅助表 6：`collection_log` — 采集日志

```sql
CREATE TABLE collection_log (
    id          INTEGER PRIMARY KEY DEFAULT nextval('log_seq'),
    run_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    job_type    VARCHAR NOT NULL,             -- 'daily' | 'weekly' | 'monthly' | 'quarterly'
    indicator   VARCHAR NOT NULL,
    status      VARCHAR NOT NULL,             -- 'success' | 'partial' | 'failed'
    rows_added  INTEGER DEFAULT 0,
    error_msg   TEXT,
    duration_ms INTEGER                       -- 执行耗时
);

CREATE SEQUENCE log_seq START 1;
CREATE INDEX idx_log_run ON collection_log (run_at);
```

### 3.3 索引策略

- **主键索引**：每张表的日期/月份字段（DuckDB 自动创建）
- **额外索引**：仅 `alerts` 和 `collection_log` 需要按类型/时间查询
- **不建索引**：daily_prices 的价格字段（无需按价格范围查询）

### 3.4 数据保留策略

| 表 | 保留期限 | 理由 |
|------|---------|------|
| `daily_prices` | **10 年** | 猪周期 ~4-6 年，需至少 2 个完整周期历史 |
| `weekly_indicators` | **10 年** | 同上 |
| `monthly_capacity` | **永久** | 月度数据量极小（120 行/10 年），无清理必要 |
| `quarterly_macro` | **永久** | 同上 |
| `alerts` | **3 年** | 告警历史参考价值有限 |
| `collection_log` | **1 年** | 运维排查用，1 年足够 |

### 3.5 数据库文件管理

- **文件位置**：`pig-tracker/pig_cycle.duckdb`（单文件）
- **预估大小**：10 年日度数据约 3,650 行 × 8 列 ≈ 30KB（DuckDB 列式压缩后更小），总库 <10MB
- **备份**：每日 rsync 到备份目录（见 §6.3）
- **WAL 模式**：不需要（单进程写入）

---

## 4. 分析模块设计

### 4.1 周期阶段定义

| 阶段 | 编号 | 典型特征 | 评分区间 |
|------|------|---------|---------|
| **顶部** | P5 | 能繁母猪高位/持续增加、猪价高于成本线 30%+、利润丰厚、补栏积极 | 0-20 |
| **下降早期** | P4 | 能繁母猪仍在增长但增速放缓、猪价开始回落、利润收窄 | 20-35 |
| **下降中后期** | P3 | 能繁母猪开始去化、猪价低于成本线、行业亏损 | 35-50 |
| **底部去化** | P2 | 能繁母猪加速去化（环比 -1%+）、深度亏损、政策收储启动 | 50-65 |
| **底部确认** | P1 | 能繁母猪降至正常保有量以下、仔猪价格企稳回升、期货远月升水 | 65-75 |
| **上升期** | P0 | 能繁母猪持续低位、猪价回升突破成本线、利润转正、补栏恢复 | 75-100 |

> **注意**：评分越高表示周期位置越接近投资机会（底部确认→上升期），越低表示风险越大（顶部→下降期）。

### 4.2 多指标加权评分模型

```python
# analyzer/cycle_scorer.py

CYCLE_WEIGHTS = {
    "breeding_sow_level":     0.25,   # 能繁母猪绝对水平
    "breeding_sow_momentum":  0.15,   # 能繁母猪环比变化趋势
    "profit_level":           0.20,   # 养殖利润水平
    "pig_corn_ratio":         0.10,   # 猪粮比
    "avg_weight_trend":       0.08,   # 出栏体重趋势（压栏/抛售信号）
    "piglet_price_trend":     0.07,   # 仔猪价格趋势
    "futures_structure":      0.05,   # 期货期限结构（远月升/贴水）
    "slaughter_rate":         0.05,   # 屠宰开工率
    "frozen_inventory":       0.03,   # 冻肉库存水平
    "feed_production_trend":  0.02,   # 饲料产量趋势
}

class CycleScorer:
    """
    每个指标根据当前值映射到 0-100 的子评分，
    然后按权重加权求和得到综合评分。
    """

    def score_breeding_sow_level(self, current_sow: float) -> tuple[int, str]:
        """能繁母猪绝对水平 → 子评分"""
        if current_sow >= 4300:
            return (5, "能繁母猪严重过剩（>4300万头）")
        elif current_sow >= 4100:
            return (15, "能繁母猪高于正常保有量上沿")
        elif current_sow >= 3900:
            return (30, "能繁母猪处于绿色区域上沿，略高于正常保有量")
        elif current_sow >= 3700:
            return (65, "能繁母猪降至正常保有量以下，反转信号增强")
        elif current_sow >= 3500:
            return (80, "能繁母猪显著低于正常保有量，强反转信号")
        else:
            return (95, "能繁母猪严重去化，极强反转信号（但需警惕过度去化风险）")

    def score_breeding_sow_momentum(self, mom_3m_avg: float) -> tuple[int, str]:
        """能繁母猪 3 个月环比均值 → 子评分"""
        if mom_3m_avg > 1.0:
            return (5, "能繁母猪快速扩张")
        elif mom_3m_avg > 0:
            return (20, "能繁母猪缓慢增长")
        elif mom_3m_avg > -0.5:
            return (40, "能繁母猪微幅去化")
        elif mom_3m_avg > -1.0:
            return (65, "能繁母猪加速去化")
        else:
            return (85, "能繁母猪快速去化，底部特征明显")

    def score_profit_level(self, profit: float) -> tuple[int, str]:
        """自繁自养利润 元/头 → 子评分"""
        if profit > 500:
            return (5, "超高利润，产能扩张动力强")
        elif profit > 200:
            return (15, "利润丰厚，补栏积极")
        elif profit > 0:
            return (35, "微利，行业观望")
        elif profit > -100:
            return (50, "小幅亏损，去产能缓慢开始")
        elif profit > -300:
            return (70, "中度亏损，去产能加速")
        else:
            return (90, "深度亏损（>-300元/头），去产能剧烈")

    def score_pig_corn_ratio(self, ratio: float) -> tuple[int, str]:
        """猪粮比 → 子评分"""
        if ratio >= 9.0:
            return (5, "猪粮比≥9，过度上涨预警")
        elif ratio >= 6.0:
            return (20, "猪粮比正常偏高，行业盈利")
        elif ratio >= 5.5:
            return (40, "猪粮比接近盈亏平衡")
        elif ratio >= 5.0:
            return (60, "猪粮比低于平衡线，二级预警")
        elif ratio >= 4.5:
            return (75, "猪粮比≤5，一级预警，收储可能启动")
        else:
            return (90, "猪粮比严重失衡，深度亏损")

    # 其他指标类似实现...

    def compute_overall_score(self, data: dict) -> dict:
        """
        输入：各指标当前值
        输出：综合评分、阶段判定、各指标子评分明细
        """
        scores = {}
        signals = []

        scores["breeding_sow_level"], sig = self.score_breeding_sow_level(data["breeding_sow"])
        signals.append(sig)
        # ... 对每个指标计算子评分

        weighted_score = sum(
            scores[k] * CYCLE_WEIGHTS[k]
            for k in CYCLE_WEIGHTS
            if k in scores
        )

        # 归一化（如果有指标缺失，按实际权重归一化）
        total_weight = sum(CYCLE_WEIGHTS[k] for k in scores)
        if total_weight > 0:
            weighted_score = weighted_score / total_weight

        return {
            "overall_score": round(weighted_score, 1),
            "phase": self._map_phase(weighted_score),
            "phase_description": self._phase_description(weighted_score),
            "sub_scores": scores,
            "signals": signals,
            "missing_indicators": [k for k in CYCLE_WEIGHTS if k not in scores],
            "timestamp": datetime.now().isoformat(),
        }

    @staticmethod
    def _map_phase(score: float) -> str:
        if score < 20:  return "P5_TOP"
        elif score < 35: return "P4_EARLY_DECLINE"
        elif score < 50: return "P3_MID_DECLINE"
        elif score < 65: return "P2_BOTTOM_DERATING"
        elif score < 75: return "P1_BOTTOM_CONFIRM"
        else:            return "P0_RISING"
```

### 4.3 趋势检测

```python
# analyzer/trend_detector.py

class TrendDetector:
    """检测各指标的趋势方向和拐点"""

    def detect_sow_trend(self, months: int = 6) -> dict:
        """
        能繁母猪趋势检测
        返回：方向、速度、拐点信号
        """
        # 取最近 N 个月数据
        # 计算环比变化率序列
        # 判断：持续下降？加速下降？减速下降？反弹？
        # 3 个月移动平均交叉判断拐点
        pass

    def detect_profit_trend(self, weeks: int = 12) -> dict:
        """养殖利润趋势"""
        pass

    def detect_price_momentum(self, days: int = 30) -> dict:
        """猪价动量：30 日涨幅/跌幅"""
        pass
```

### 4.4 告警规则

```python
# analyzer/alert_rules.py

ALERT_RULES = [
    {
        "id": "sow_below_normal",
        "name": "能繁母猪跌破正常保有量",
        "indicator": "breeding_sow",
        "condition": "value < 3900",
        "severity": "HIGH",
        "message": "⚠️ 能繁母猪跌破 3900 万头正常保有量！产能去化确认，周期底部信号增强。",
        "cooldown_days": 30,  # 同一告警 30 天内不重复触发
    },
    {
        "id": "sow_mom_fast_decrease",
        "name": "能繁母猪快速去化",
        "indicator": "sow_mom_change",
        "condition": "value < -1.0",
        "severity": "MEDIUM",
        "message": "📊 能繁母猪环比降幅超 1%，去产能加速中。",
        "cooldown_days": 30,
    },
    {
        "id": "pig_price_below_10",
        "name": "猪价跌破 10 元/kg",
        "indicator": "pig_waisan",
        "condition": "value < 10.0",
        "severity": "HIGH",
        "message": "🔴 生猪价格跌破 10 元/kg！极端低位，历史性底部区域。",
        "cooldown_days": 7,
    },
    {
        "id": "pig_corn_ratio_alert_1",
        "name": "猪粮比一级预警",
        "indicator": "pig_corn_ratio",
        "condition": "value < 4.5",
        "severity": "HIGH",
        "message": "🔴 猪粮比跌破 4.5:1，一级预警！国家将加大收储力度。",
        "cooldown_days": 14,
    },
    {
        "id": "pig_corn_ratio_alert_2",
        "name": "猪粮比二级预警",
        "indicator": "pig_corn_ratio",
        "condition": "value < 5.0",
        "severity": "MEDIUM",
        "message": "⚠️ 猪粮比跌破 5:1，二级预警，国家可能启动收储。",
        "cooldown_days": 14,
    },
    {
        "id": "deep_loss",
        "name": "深度亏损",
        "indicator": "profit_self",
        "condition": "value < -300",
        "severity": "MEDIUM",
        "message": "⚠️ 自繁自养亏损超 300 元/头，深度亏损加速去产能。",
        "cooldown_days": 14,
    },
    {
        "id": "profit_turn_positive",
        "name": "养殖利润转正",
        "indicator": "profit_self",
        "condition": "value > 0 and previous_value <= 0",
        "severity": "HIGH",
        "message": "🟢 养殖利润由负转正！周期反转确认信号。",
        "cooldown_days": 30,
    },
    {
        "id": "piglet_price_rally",
        "name": "仔猪价格大涨",
        "indicator": "piglet_price",
        "condition": "wow_change > 10",  # 周环比涨幅 >10%
        "severity": "MEDIUM",
        "message": "📊 仔猪价格周环比大涨 >10%，补栏意愿恢复信号。",
        "cooldown_days": 14,
    },
    {
        "id": "policy_storage",
        "name": "国家收储启动",
        "indicator": "policy_event",
        "condition": "event_type == '收储'",
        "severity": "MEDIUM",
        "message": "📢 国家启动冻猪肉收储，政策底信号。",
        "cooldown_days": 30,
    },
]

class AlertEngine:
    """告警引擎：检查所有规则，触发匹配的告警"""

    def check_all(self, data: dict) -> list[dict]:
        """返回触发的告警列表"""
        triggered = []
        for rule in ALERT_RULES:
            if self._evaluate(rule["condition"], data):
                if not self._in_cooldown(rule):
                    triggered.append(rule)
        return triggered

    def _in_cooldown(self, rule: dict) -> bool:
        """检查该规则是否在冷却期内"""
        # 查询 alerts 表中最近一次同 id 告警时间
        pass
```

### 4.5 报告生成逻辑

#### 周报内容清单

1. **关键指标速览表**：本周 vs 上周 vs 上月同期 vs 去年同期
2. **周期位置评分**：综合评分、阶段判定、评分变动
3. **趋势分析**：
   - 能繁母猪去化趋势（最近 6 个月环比折线图描述）
   - 猪价走势（最近 3 个月）
   - 养殖利润走势
4. **告警事项**：本周触发的告警
5. **数据更新状态**：哪些指标有更新、哪些缺失
6. **简要投资建议**：基于当前阶段的模板化建议
7. **前瞻关注**：下周应关注的事件（如数据发布日期、政策会议）

#### 月报额外内容

1. 月度数据详细分析（能繁母猪、出栏量、饲料产量）
2. 期货期限结构分析
3. 上市公司月度出栏数据汇总
4. 政策动态梳理
5. 行业新闻摘要

---

## 5. 输出模块设计

### 5.1 周报 Markdown 模板

```markdown
# 🐷 猪周期周报 — {year}年第{week_num}周

> 生成时间：{generated_at}
> 周期阶段：**{phase_name}**（评分 {score}/100，{score_change}）

---

## 📊 关键指标速览

| 指标 | 本周 | 上周 | 变化 | 状态 |
|------|------|------|------|------|
| 外三元猪价 | {price} 元/kg | {prev_price} 元/kg | {price_change} | {profit_status} |
| 猪粮比 | {ratio}:1 | {prev_ratio}:1 | {ratio_change} | {warning_level} |
| 能繁母猪（月） | {sow} 万头 | — | {sow_mom} | {sow_zone} |
| 自繁自养利润 | {profit} 元/头 | {prev_profit} 元/头 | {profit_change} | {profit_status} |
| 出栏均重 | {weight} kg | {prev_weight} kg | {weight_change} | — |

## 🔄 周期位置判断

**当前阶段：{phase_name}**

{phase_description}

### 评分明细

| 指标 | 权重 | 子评分 | 贡献 |
|------|------|--------|------|
| 能繁母猪水平 | 25% | {sub1} | {contrib1} |
| 能繁母猪趋势 | 15% | {sub2} | {contrib2} |
| 养殖利润 | 20% | {sub3} | {contrib3} |
| 猪粮比 | 10% | {sub4} | {contrib4} |
| ... | ... | ... | ... |

## 📈 趋势分析

### 能繁母猪存栏趋势（近 6 月）

{6 个月数据表格}

**趋势判断**：{trend_description}

### 猪价走势（近 3 月）

{价格走势描述}

## ⚠️ 告警事项

{本周触发告警列表，无告警则显示"本周无新增告警"}

## 📅 前瞻关注

- {下周关注事项 1}
- {下周关注事项 2}

## 📋 数据更新状态

| 指标 | 最新日期 | 状态 |
|------|---------|------|
| {各指标更新状态} |

---

*本报告由猪周期自动跟踪系统生成。数据来源于 AKShare 等公开数据源，仅供参考，不构成投资建议。*
```

### 5.2 报告存储

```
shared/results/pig-cycle-weekly/
├── 2026/
│   ├── W01-20260106.md       # 按周编号
│   ├── W02-20260113.md
│   ├── ...
│   └── W13-20260330.md
├── 2026-monthly/
│   ├── 2026-01.md
│   ├── 2026-02.md
│   └── ...
└── README.md                  # 索引文件，列出所有报告链接
```

### 5.3 Telegram 告警格式

```
🐷 猪周期告警

🔴 等级：HIGH
📋 指标：生猪价格（外三元）
📉 当前值：9.99 元/kg
🎯 阈值：< 10.0 元/kg
📝 说明：生猪价格跌破 10 元/kg！极端低位，历史性底部区域。

⏰ 触发时间：2026-03-18 10:30 CST

—猪周期自动跟踪系统
```

---

## 6. 运维设计

### 6.1 部署方式

**选择：cron 定时任务（而非 systemd 服务）**

理由：
1. 采集任务是离散的（每天/周/月运行一次），不是持续运行的服务
2. cron 简单可靠，调试方便
3. 每次运行独立进程，互不影响

#### cron 配置

```cron
# 猪周期跟踪系统 - cron 配置
# 用户：openclaw（或当前用户）

# 日度采集：每个工作日 19:30（A股收盘后，数据源更新完毕）
30 19 * * 1-5 cd /path/to/pig-tracker && python jobs/daily_collect.py >> logs/daily.log 2>&1

# 周度分析+报告：每周日 20:00
0 20 * * 0 cd /path/to/pig-tracker && python jobs/weekly_analyze.py >> logs/weekly.log 2>&1

# 月度采集：每月 15 日 19:00（官方月中发布数据）
0 19 15 * * cd /path/to/pig-tracker && python jobs/monthly_collect.py >> logs/monthly.log 2>&1

# 季度数据：每季末月 25 日（国家统计局数据发布）
0 19 25 3,6,9,12 * cd /path/to/pig-tracker && python jobs/quarterly_collect.py >> logs/quarterly.log 2>&1

# 数据库备份：每天 02:00
0 2 * * * cp /path/to/pig-tracker/pig_cycle.duckdb /path/to/pig-tracker/backup/pig_cycle_$(date +\%Y\%m\%d).duckdb

# 清理旧备份（保留 30 天）
0 3 * * 0 find /path/to/pig-tracker/backup/ -name "*.duckdb" -mtime +30 -delete
```

### 6.2 日志策略

```
pig-tracker/logs/
├── daily.log          # 日度采集日志（追加模式）
├── weekly.log         # 周度分析日志
├── monthly.log        # 月度采集日志
├── quarterly.log      # 季度采集日志
└── alert.log          # 告警推送日志
```

- **日志级别**：INFO（正常运行）、WARNING（采集失败但可恢复）、ERROR（需要人工介入）
- **日志轮转**：通过 logrotate 管理，保留 90 天
- **关键信息**：每次运行记录开始时间、结束时间、各指标采集结果、成功/失败数

#### logrotate 配置

```
/path/to/pig-tracker/logs/*.log {
    weekly
    rotate 12
    compress
    missingok
    notifempty
}
```

### 6.3 数据备份

| 策略 | 频率 | 保留 | 方式 |
|------|------|------|------|
| DuckDB 文件备份 | 每日 02:00 | 30 天 | cp 到 backup/ 目录 |
| 远程备份 | 每周日 | 4 份 | rsync 到远程存储（如有的话） |
| 报告文件 | 每次生成 | 永久 | Git 版本管理（shared/results/） |

### 6.4 与 OpenClaw 生态集成

#### md-viewer 集成

- 周报/月报存放在 `shared/results/pig-cycle-weekly/` 目录
- OpenClaw 的 md-viewer 可直接渲染这些 Markdown 文件
- 用户可通过对话请求："给我看最新猪周期周报" → OpenClaw 调用 md-viewer 展示

#### Telegram 通知集成

- 告警推送：通过 OpenClaw 现有的 Telegram 通知能力
- 周报摘要：每周日分析完成后，推送一段 3-5 句摘要到 Telegram
- 实现：`alerter/telegram_alerter.py` 调用 OpenClaw 的通知 API 或直接使用 python-telegram-bot

#### OpenClaw Agent 调用

- 用户可在对话中触发："更新猪周期数据" → OpenClaw 执行 `daily_collect.py`
- 用户可查询："当前猪周期评分是多少？" → OpenClaw 读取 DuckDB 返回最新评分
- 未来可通过 OpenClaw Agent Skill 封装这些操作

---

## 7. 实施路线图

### Phase 1：数据采集 + 存储（MVP）

**目标**：能够自动采集 AKShare 数据并存入 DuckDB

**时间**：2-3 天

**交付物**：
1. `storage/schema.py` — DuckDB 建表脚本
2. `collector/akshare_collector.py` — AKShare 数据采集器
3. `collector/base.py` — 基类（增量逻辑、错误处理）
4. `jobs/daily_collect.py` — 日度采集入口
5. `jobs/monthly_collect.py` — 月度采集入口
6. `pig_cycle.duckdb` — 初始化数据库
7. cron 配置文件

**验收标准**：
- [ ] `daily_collect.py` 能成功采集日度价格（外三元、玉米、豆粕）并存入 DuckDB
- [ ] `monthly_collect.py` 能成功采集能繁母猪存栏、生猪存栏
- [ ] 增量逻辑正确：重复运行不会产生重复数据
- [ ] 采集失败时正确记录日志，不影响其他指标
- [ ] 已有至少 1 个月的历史数据在库中

### Phase 2：分析 + 报告生成

**目标**：自动生成周报 Markdown 文件

**时间**：2-3 天

**交付物**：
1. `analyzer/cycle_scorer.py` — 周期位置评分模型
2. `analyzer/trend_detector.py` — 趋势检测
3. `analyzer/alert_rules.py` — 告警规则引擎
4. `reporter/weekly_report.py` — 周报生成
5. `reporter/templates/weekly.md.j2` — 周报模板
6. `jobs/weekly_analyze.py` — 周度分析入口
7. 首份周报文件

**验收标准**：
- [ ] 评分模型能正确输出综合评分和阶段判定
- [ ] 当前阶段判定与手动分析一致（2026年3月应判定为 P2/P3 附近）
- [ ] 周报 Markdown 格式正确，能在 md-viewer 中正常渲染
- [ ] 告警规则能正确匹配（如猪价<10 应触发 HIGH 告警）
- [ ] 报告中缺失指标正确标注

### Phase 3：告警 + Telegram 通知

**目标**：关键指标突破阈值时自动推送 Telegram 告警

**时间**：1-2 天

**交付物**：
1. `alerter/telegram_alerter.py` — Telegram 推送
2. 告警冷却机制实现
3. 告警记录持久化（alerts 表）
4. cron 告警检查任务（可合并到日度/周度任务中）

**验收标准**：
- [ ] 告警消息能成功推送到 Telegram
- [ ] 冷却机制有效（同一告警不重复推送）
- [ ] 告警记录写入 DuckDB alerts 表
- [ ] 手动触发测试告警成功

### Phase 4：md-viewer 集成 + 完善

**目标**：与 OpenClaw 深度集成，补充第三方数据

**时间**：持续迭代

**交付物**：
1. OpenClaw Agent Skill 定义（用户可通过对话触发数据更新、查看报告）
2. 网页抓取器（华储网收储公告、饲料工业协会数据）
3. 月报模板和生成器
4. 手动录入辅助脚本（CLI 交互式录入第三方数据）
5. 数据质量监控（AKShare vs 计算值对比）

**验收标准**：
- [ ] 用户通过 Telegram 对话可触发数据更新和查看报告
- [ ] 华储网公告自动抓取成功
- [ ] 月报格式完整
- [ ] 手动录入流程可用

---

## 8. 风险和限制

### 8.1 AKShare 数据质量风险

| 风险 | 影响 | 缓解措施 |
|------|------|---------|
| 接口返回数据字段名变化 | 采集失败 | 字段映射配置化（yaml），便于快速调整；增加字段校验 |
| 数据源网站反爬升级 | 采集中断 | 请求间隔 ≥3 秒；备选数据源；手动录入兜底 |
| 历史数据深度不足 | 无法做长期回测 | 初始化时尽可能拉取全部可用历史；必要时手动补录 |
| `futures_pig_spot()` 接口实际字段与文档不一致 | 需要适配 | Phase 1 首先做接口实测，根据实际返回调整字段映射 |

**关键待验证项**（Phase 1 首日需完成）：
- [ ] `ak.futures_pig_rank()` 返回字段名和数据范围
- [ ] `ak.futures_pig_spot()` 返回字段名，确认是否包含能繁母猪、猪粮比、利润等
- [ ] 数据历史深度（最早可追溯到何时）
- [ ] 日度更新延迟（当日数据何时可获取）

### 8.2 农业农村部数据获取难度

| 风险 | 影响 | 缓解措施 |
|------|------|---------|
| 官方数据仅通过新闻发布会发布，无结构化接口 | 无法自动采集月度产能数据 | ① AKShare 可能已封装 ② 新闻稿网页抓取 + 文本提取 ③ 手动录入 |
| 数据发布时间不固定 | cron 采集可能跑空 | 月度任务设为每月 15 日和 20 日各运行一次，取有数据的那次 |
| 季度数据（国家统计局）与月度数据格式不统一 | 存储需适配 | 分表存储（monthly_capacity + quarterly_macro） |

### 8.3 第三方数据源限制

- 涌益咨询、卓创资讯数据为**付费订阅**，免费方案无法获取
- 券商研报引用的数据有**时滞**（研报发布通常滞后 1-2 周）
- 解决方案：Phase 1-2 仅使用 AKShare + 估算值，Phase 4 评估是否值得付费订阅

### 8.4 服务器资源消耗

| 资源 | 估算消耗 | 说明 |
|------|---------|------|
| 磁盘 | <50MB/年 | DuckDB 压缩后极小，报告文件也很小 |
| CPU | 极低 | 每天运行一次，单次 <1 分钟 |
| 内存 | <200MB | Python + AKShare + DuckDB，运行时短暂占用 |
| 网络 | 极低 | 每次采集几个 HTTP 请求 |

**结论**：资源消耗可忽略不计，在当前 OpenClaw 服务器上运行无压力。

### 8.5 模型风险

- 评分模型的权重和阈值基于历史经验设定，**未经回测验证**
- 规模化养殖改变了猪周期的传统规律，历史阈值可能需要动态调整
- **缓解措施**：每季度回溯评分与实际价格走势的匹配度，调整权重

### 8.6 合规与免责

- 本系统仅供个人研究使用，不构成投资建议
- 数据来源于公开渠道，需注意版权
- 报告中应标注"自动生成，仅供参考"

---

## 9. 附录

### 9.1 AKShare 接口速查

```python
import akshare as ak

# 1. 生猪价格排行（日度）
#    含：外三元、内三元、土杂猪、玉米、豆粕
df = ak.futures_pig_rank()

# 2. 生猪供应维度综合数据
#    含：猪肉批发价、储备冻猪肉、饲料原料、白条肉、
#        生猪产能、育肥猪、肉类价格指数、猪粮比价
df = ak.futures_pig_spot()

# 3. 生猪期货主力合约
df = ak.futures_main_sina(symbol="lh0")

# 4. 生猪期货所有合约（用于期限结构分析）
#    待确认具体接口

# 5. 进出口数据
df = ak.china_import_export()
```

### 9.2 能繁母猪绿黄红区间参考

| 区间 | 范围（万头） | 对应正常保有量比例 | 调控措施 |
|------|-------------|------------------|---------|
| 绿色 | 3,588 - 4,095 | 92%-105% | 常规监测 |
| 黄色（下） | 3,315 - 3,588 | 85%-92% | 引导产能调控 |
| 黄色（上） | 4,095 - 4,290 | 105%-110% | 引导产能调控 |
| 红色（下） | < 3,315 | < 85% | 强制性产能调控 |
| 红色（上） | > 4,290 | > 110% | 强制性产能调控 |

正常保有量基准：3,900 万头（2024 年修订版《生猪产能调控实施方案》）

### 9.3 关键阈值汇总

| 指标 | 阈值 | 含义 |
|------|------|------|
| 能繁母猪 | 3,900 万头 | 正常保有量（绿/黄分界线） |
| 猪粮比 | 5.5:1 | 传统盈亏平衡线 |
| 猪粮比 | 5.0:1 | 二级预警线（启动收储） |
| 猪粮比 | 4.5:1 | 一级预警线（加大收储） |
| 猪粮比 | 9.0:1 | 过度上涨预警 |
| 自繁自养利润 | 0 元/头 | 盈亏平衡 |
| 自繁自养利润 | -300 元/头 | 深度亏损 |
| 猪价 | 10 元/kg | 心理关口 / 极端低位 |
| 能繁母猪环比 | -1.0% | 快速去化标志 |

### 9.4 配置文件模板（config.yaml）

```yaml
# pig-tracker/config/config.yaml

database:
  path: "pig_cycle.duckdb"

akshare:
  request_interval: 3        # 请求间隔（秒）
  max_retries: 3
  timeout: 30

report:
  weekly_output_dir: "shared/results/pig-cycle-weekly"
  monthly_output_dir: "shared/results/pig-cycle-monthly"

alert:
  telegram_enabled: true
  cooldown_default_days: 14

collection:
  daily_time: "19:30"
  weekly_time: "20:00"
  monthly_day: 15
  monthly_time: "19:00"

logging:
  level: "INFO"
  dir: "logs"
  retention_days: 90
```

---

*设计文档结束。本文档由 Research Lead 基于猪周期研究报告（R-021）和数据源评估（R-020c）编写，供 dev team 实施参考。*
