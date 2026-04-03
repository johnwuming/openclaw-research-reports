# A股量化回测数据源深度调研

> 调研时间：2026-03-31 | 目标：为 Backtrader 本地回测选择最佳免费数据组合

---

## 一、免费数据源对比

| 维度 | Tushare Pro | AKShare | Baostock |
|------|------------|---------|----------|
| **费用** | 积分制，免费注册约120分；5000分(≈500元)解锁全量日线 | 完全免费开源 | 完全免费开源 |
| **注册** | 需注册获取Token | 无需注册 | 无需注册 |
| **行情数据** | 日线/周线/月线，质量最高（字段规范、复权清晰、日期对齐） | 日线/周线/月线，支持前/后复权 | 日线/周线/月线，支持前/后复权 |
| **财务数据** | 三大报表完整（balancesheet/income/cashflow），含流动资产、总负债 | 有stock_zcfz_em等接口，资产负债表字段较全 | 有季频财务数据，但字段不如Tushare丰富 |
| **退市/ST** | stock_basic含退市日期，但退市股财务数据可能仅近期 | **有专门的"已退市股票"财务报表接口**（东财源） | 未确认是否支持退市股数据 |
| **历史深度** | 日线数据完整度高，覆盖全市场 | 依赖东财/新浪等源，深度各异 | **数据来自证券交易所，适合长期回测（约2005年起）** |
| **数据质量** | ⭐⭐⭐⭐⭐ 最高，"精装修"数据，可直接喂回测引擎 | ⭐⭐⭐ 中等，爬虫数据，偶有字段缺失或延迟 | ⭐⭐⭐⭐ 较好，交易所官方源，但更新滞后 |
| **API稳定性** | 高，但低积分用户频控严格(50-200次/分) | **低风险**：依赖爬虫，数据源改版可导致接口中断 | 较稳定，但项目维护状态存疑 |
| **实时数据** | 支持（需积分） | 支持（延迟不可控，秒到分钟级） | ❌ 不提供 |
| **另类数据** | 有限 | **独一档**：宏观(CPI/PPI)、产业(能繁母猪/光伏装机)、电商舆情等 | 无 |

### 其他数据源

- **JQData**：已无免费版，需付费，数据质量好但性价比不如Tushare
- **东方财富/新浪财经API**：非官方接口，AKShare已封装
- **通达信本地数据**：可导出但格式解析麻烦，不推荐作为主力

---

## 二、NCAV策略关键数据字段评估

NCAV（净流动资产价值）= 流动资产 - 总负债，策略需要：

| 字段 | Tushare Pro | AKShare | Baostock |
|------|------------|---------|----------|
| 流动资产合计 | ✅ balancesheet.total_cur_assets | ✅ stock_zcfz_em | ✅ 有 |
| 总负债 | ✅ balancesheet.total_liab | ✅ stock_zcfz_em | ✅ 有 |
| 总股本 | ✅ basic / daily_basic | ✅ 有 | ✅ 有 |
| 当前股价 | ✅ daily接口 | ✅ 历史行情 | ✅ 日线数据 |
| **退市股财务数据** | ⚠️ 可能仅近期 | ✅ **有专门接口** | ❓ 未确认 |

**结论**：NCAV策略对退市股数据依赖度高（历史上大量NCAV机会出现在ST/退市边缘股），AKShare的退市股接口是关键优势。

---

## 三、数据完整性评估

### 行情数据
- **日线OHLCV**：三个数据源均完整，停牌日处理方式不同（Tushare不输出停牌日，AKShare/Baostock可能输出空行）
- **复权因子**：Tushare最准确（有专门的adj_factor接口），AKShare支持前/后复权参数，Baostock也支持复权
- **新股上市首日**：Tushare和AKShare均能覆盖

### 财务数据
- Tushare的三大报表字段最全、最规范
- AKShare依赖东财源，字段覆盖度接近Tushare
- Baostock财务数据字段相对较少

### 已知问题
- AKShare：部分接口需Node.js环境（PyExecJS），2026年反爬更严，建议请求间隔≥3秒
- Tushare：低积分用户频控严格，恶意超频可导致Token永久封禁
- Baostock：项目维护状态不确定（2025年底仍有相关项目在用）

---

## 四、推荐方案

### 🏆 最佳免费数据组合

```
主力：AKShare（完全免费 + 退市股数据 + 另类数据丰富）
验证：Baostock（免费 + 交易所官方源 + 适合交叉验证）
升级：Tushare Pro 5000积分（≈500元，一次性）→ 全量无限制
```

**推荐理由**：
1. AKShare完全免费，退市股接口是NCAV策略刚需
2. Baostock数据来自交易所，作为交叉验证源提高可靠性
3. 如果预算允许，Tushare Pro 500元升级可大幅提升数据质量和API稳定性
4. 社区主流方案即"Tushare日线主力 + AKShare补充 + Baostock冷备份"

### 存储方案：Parquet + DuckDB

| 方案 | 优点 | 缺点 | 适合场景 |
|------|------|------|----------|
| CSV | 最简单 | 占空间大，读取慢 | 小规模测试 |
| SQLite | 单文件，SQL查询 | 行式存储，分析慢 | 小型项目 |
| **Parquet** | **列式压缩，读写快，适合回测** | 需要额外库 | **⭐ 量化回测首选** |
| Parquet + DuckDB | 高性能分析，无需数据库服务 | 略复杂 | **⭐ 推荐** |

**具体实现建议**：
```
data/
├── daily/          # 日线行情 (Parquet)
│   ├── 000001.parquet
│   └── ...
├── financial/      # 财务数据 (Parquet)
│   ├── balance_sheet.parquet
│   ├── income.parquet
│   └── cashflow.parquet
├── basic/          # 股票基础信息 (Parquet)
│   └── stock_basic.parquet
└── metadata.json   # 上次更新时间等元数据
```

### 自动更新策略

1. **初始化**：一次性拉取全量历史数据（AKShare + Baostock交叉验证）
2. **每日更新**：cron定时任务，交易日19:00后拉取当日数据
   ```bash
   # crontab示例
   0 19 * * 1-5 cd /path/to/project && python update_data.py
   ```
3. **增量逻辑**：记录每只股票最后更新日期，只拉取新数据追加到Parquet
4. **数据校验**：AKShare vs Baostock 交叉验证，OHLC偏差>1%时报警
5. **每周全量校验**：对比两源数据，修复不一致

---

## 五、成本分析

| 方案 | 费用 | 能支撑的回测规模 |
|------|------|-----------------|
| 纯AKShare | **0元** | 全量A股日线+财务+退市股，完全够用 |
| AKShare + Baostock | **0元** | 全量+交叉验证，可靠性更高 |
| + Tushare 120分（免费注册） | **0元** | 有限频次，小规模测试 |
| + Tushare 5000分 | **500元** | 全量A股日线无限制，生产级 |
| Tushare分钟级数据 | **~1000元/月** | 日内策略回测（NCAV策略不需要） |

**NCAV策略建议**：免费组合（AKShare + Baostock）完全够用，因为NCAV是长周期策略，只需日线+季度财务数据。

---

## 六、知识缺口

1. Baostock项目当前维护状态需持续关注（GitHub最后提交时间）
2. 各数据源复权因子准确性缺乏实测对比数据
3. AKShare退市股财务报表接口的字段覆盖范围需实测验证
4. Tushare Pro免费注册初始积分具体数额未确认

---

## 来源

- [掘金-量化数据源选型2026](https://juejin.cn/post/7605537925149261876)
- [AKShare官方文档](https://akshare.akfamily.xyz/data/stock/stock.html)
- [Tushare Pro官方文档](https://tushare.pro/document/2?doc_id=25)
- [投资驿站-Baostock介绍](https://touzihub.com/sites/1236.html)
- [腾讯云-AKShare介绍](https://cloud.tencent.com/developer/article/2454419)
- [GitHub-a-share-mcp项目](https://github.com/24mlight/a-share-mcp-is-just-i-need)
- [知乎-Parquet+DuckDB方案](https://zhuanlan.zhihu.com/p/1906052151362458118)
- [量化投资技术手册-存储格式对比](https://waylandz.com/quant-book/)
- [CSDN-AKShare资产负债表接口](https://blog.csdn.net/Humbunklung/article/details/149671918)
