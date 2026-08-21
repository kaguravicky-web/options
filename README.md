# 期权流分析框架（Weekly Options Flow Framework）

**数据来源**：moomoo API（主力，本地OpenD网关）+ Barchart.com（仅Max Pain / Gamma Flip / Call-Put Wall两页）
**范围**：短线账户持仓 + 当日重点关注标的
**时间窗口**：当周到期（本周五）

**每日报告**：[`reports/`](./reports) 目录下存放每次生成的带日期分析报告（HTML），命名格式 `{日期}-options-flow.html`。

## 数据获取方式（已验证可行，2026-08-21起改为混合数据源）

### 主力数据源：moomoo API（本地，无每日次数限制）

前提：`moomoo_OpenD.exe` 网关在本机运行，Python装有`moomoo_api`包，直连`127.0.0.1:11111`（无需浏览器/额度）。

```python
from moomoo import OpenQuoteContext, RET_OK, SubType
ctx = OpenQuoteContext(host='127.0.0.1', port=11111)
```

| 字段 | API调用 | 返回内容 |
|------|---------|----------|
| 标的总览（P/C Volume/OI、IV、IV Rank/Percentile、HV 30/60/90/120/365d及百分位） | `ctx.get_option_underlying_overview(code_list=['US.{TICKER}'])` | 一行DataFrame，字段名见下 |
| 到期日列表 | `ctx.get_option_expiration_date(code='US.{TICKER}')` | 含`expiration_cycle`（WEEK/MONTH）区分周期权/月期权 |
| 完整期权链（合约代码列表） | `ctx.get_option_chain(code='US.{TICKER}', start='{YYYY-MM-DD}', end='{YYYY-MM-DD}')` | 148档左右，仅含code/strike，无Greeks |
| 逐行权价Greeks + OI + 成交量（**Barchart完全拿不到的数据**） | 先`ctx.subscribe(codes, [SubType.QUOTE])`，再`ctx.get_stock_quote(codes)` | delta/gamma/vega/theta/open_interest/implied_volatility/volume/premium，逐个行权价真实值 |

标的总览字段名：`call_volume` `put_volume` `call_open_interest` `put_open_interest` `iv` `iv_rank` `iv_percentile` `pre_iv` `hv_30d`/`hv_60d`/`hv_90d`/`hv_120d`/`hv_365d`（各带`_percentile`）。

**⚠️ 用完记得`ctx.unsubscribe(codes, [SubType.QUOTE])`和`ctx.close()`**，避免占用订阅额度。

### 仍从Barchart拿的两项：Max Pain、Gamma Flip Point / Call Wall / Put Wall

这两个是**加工过的衍生指标**（不是原始数据），moomoo没有现成字段，需要用逐行权价Gamma×OI按标准公式自己汇总计算——但公式的符号约定（谁被视为净空/净多）没有跟Barchart的黑箱结果交叉验证过，贸然自己算容易在判断方向上出错，风险比继续用Barchart的现成结果更大。**因此这两项继续从Barchart抓，其余全部改用moomoo**：

| 字段 | 页面URL模式 | 示例返回 |
|------|------------|----------|
| Max Pain | `/stocks/quotes/{TICKER}/max-pain-chart?expiration={FRIDAY}-w` | "Max Pain: 745.00" |
| Gamma Flip Point / Call Wall / Put Wall | `/stocks/quotes/{TICKER}/gamma-exposure?expiration={FRIDAY}-w` | "gamma flip point is 744.85"，"put wall is 750.00" |

**⚠️⚠️ 必须锁定到期日**：两个页面默认都不是"本周五"单独的数据（Max Pain默认最近到期日/可能是0DTE，Gamma Exposure默认聚合未来4个到期日）。必须在URL后拼接 `?expiration=YYYY-MM-DD-w`（如周五是2026-08-21，就是`?expiration=2026-08-21-w`）。

**⚠️ 路径区别**：个股用 `/stocks/quotes/{TICKER}/...`，ETF用 `/etfs-funds/quotes/{TICKER}/...`。

**⚠️ 免费账号限制**：Barchart免费账号每天限20次页面浏览。改用混合数据源后每个标的只需2次（Max Pain + Gamma Exposure，P/C Ratio页不再需要，已被moomoo标的总览取代），额度压力比之前的3次/标的降低约三分之一。

**❌ Maxpain.com已失效**：域名停放待售，已移除（不影响，Max Pain现走Barchart）。

**未来可选优化**：用moomoo原始Gamma×OI数据自己复刻Max Pain/Gamma Flip计算，跟Barchart当天数字交叉验证若干次、确认公式方向正确后，再考虑完全脱离Barchart。这一步尚未开始。

**Options Prices / Volatility & Greeks页面**（逐行权价明细表）：纯文本提取拿不到网格数据，如需明细仍需截图。

---

## 一、标的总览表

| Ticker | 当前价 | Max Pain(周五) | 偏离方向 | Call Wall | Put Wall | Gamma Flip | P/C Ratio | Gamma环境 | IV Rank | 本周偏向 |
|--------|--------|----------|----------|-----------|----------|-----------|-----------|-----------|---------|----------|
|        |        |          |          |           |          |           |           |           |         |          |

**字段说明**：
- **偏离方向**：当前价相对Max Pain的位置（高于/低于 + 偏离百分比）
- **Gamma Flip**：见下方判断规则，这是判断Gamma环境的**主判据**
- **本周偏向**：多/空/中性 + 简要理由，理由必须引用Gamma Flip而非单纯OI比值

---

## 二、Gamma环境判断规则（已修正：以Gamma Flip Point为主判据）

### 为什么要改
旧版规则用"Call OI是否远大于Put OI"来推断正负Gamma，这是一个粗糙的代理指标——它只数了合约张数，没算每张合约实际贡献多少Gamma（跟行权价距离现价的远近、Greeks大小都有关）。Barchart的Gamma Exposure页面**直接算好了Flip Point**，这是基于真实Gamma大小的计算结果，比数OI张数准确得多。2026-07-06实测NVDA/TSLA时，两种判据方向就对不上。

### 主判据：价格 vs Gamma Flip Point

| 条件 | 做市商仓位 | 标记 | 含义 |
|------|-----------|------|------|
| 当前价 > Gamma Flip | 净多Gamma | **正Gamma 🔴** | 做市商"追涨杀跌反着来"（涨了卖、跌了买）→ 压制波动，走势变平滑，上涨不容易冲太远 |
| 当前价 < Gamma Flip | 净空Gamma | **负Gamma 🟢** | 做市商"追涨杀跌顺着来"（涨了买、跌了卖）→ 放大波动，趋势容易加速（助涨也助跌，不是只助涨） |
| 当前价 ≈ Gamma Flip（±1%内） | 临界 | **中性/过渡区** | 环境不稳定，容易来回切换，慎入 |

> ⚠️ 负Gamma不是"只助涨"，是双向放大——助涨也助跌，取决于当时价格往哪边走。同理正Gamma也不是"只压涨"，是双向压制。

### 辅助判据：OI比值（仅作情绪参考，不再作为主判据）
Put/Call OI Ratio 依然有用，但只用来判断**筹码情绪结构**（谁在囤保险 vs 谁在囤看涨仓位），不能单独用来判断Gamma方向。

---

## 三、进攻信号检查表（重新定位为"排除法过滤器"，不是"开仓触发器"）

> ⚠️ 重新定位（2026-07-06修正）：期权流数据的可靠用法是"排除坏交易"，不是"寻找好交易"。满5颗星≠该进场，只代表"没有明显的期权结构性理由阻止你进场"。真正的入场信号仍然应该来自你的技术面/催化剂体系，这份清单只是最后一道过滤网。

- [ ] 当前价格与Gamma Flip Point的相对位置，跟你计划的方向一致（做多时最好在负Gamma区或刚站上Flip；正Gamma区追涨要格外谨慎）
- [ ] 股价逼近或突破Call Wall（注意Wall本身会构成阻力，突破需要放量确认，不能只看价格触达）
- [ ] 有技术setup（EMA9/21信号 / HMA支撑确认）
- [ ] 有基本面/叙事催化剂
- [ ] CTA尚未提前入场（无假突破风险）

不满足其中任意一条 → 视为一个警示信号，不代表禁止交易，但应降低仓位或收紧止损。全部满足也不代表高胜率，只代表期权结构没有跟你唱反调。

---

## 四、防守警报规则

| 警报类型 | 触发条件 | 等级 |
|----------|----------|------|
| 价格穿越Gamma Flip Point（反向） | Gamma环境发生结构性反转 | 🔴 |
| 跌破Put Wall | 做市商助跌引擎启动 | 🔴 |
| Put OI异常放大 | 机构大量买保险 | 🟡 |
| 股价远高于Max Pain（周五档） | 到期前向下拉回引力强，DTE越小引力越强 | 🟡 |
| P/C Ratio从低位突然跳升 | 情绪逆转信号 | 🟡 |

### Put Wall 双重含义
- **股价在Put Wall上方**：短期支撑，可守仓
- **股价跌破Put Wall**：助跌引擎启动，需评估止损 → 自动标记 ⚠️ 防守状态

### 自动防守标记规则
```
IF 短线持仓当前股价距Put Wall的距离 ≤ 5%
THEN 输出「⚠️ [Ticker] 逼近Put Wall，建议确认止损位」

IF 短线持仓当前股价与Gamma Flip Point发生反向穿越
THEN 输出「⚠️ [Ticker] Gamma环境反转，结构性止损信号」
```

---

## 五、IV Rank与仓位/策略建议（新增）

IV Rank反映当前隐含波动率在过去一年区间里的位置，直接决定"现在买期权贵不贵"，跟方向判断（Max Pain/Gamma）是独立维度，两者要一起看：

| IV Rank | 含义 | 策略倾向 |
|---------|------|----------|
| 高（>50%） | 期权定价包含较大预期波动，权利金贵 | 慎买裸多头期权（赔率差），可考虑价差策略降低权利金成本 |
| 低（<30%） | 权利金相对便宜 | 更适合直接买方向性期权，赔率更友好 |
| 中等（30-50%） | 中性 | 正常评估，结合Gamma环境判断 |

> 例：2026-07-06实测NVDA IV Rank 32.71%、TSLA IV Rank 24.10%，均偏低，如果方向判断成立，直接买裸期权的赔率环境尚可。

---

## 六、Pin Risk 提示（0-1 DTE专用）

| DTE | Pin风险等级 | 备注 |
|-----|------------|------|
| 0 DTE | 高 | Max Pain吸附力最强，OI越厚吸附越强 |
| 1 DTE | 中高 | 需结合OI厚度判断可靠性 |
| 2 DTE+ | 中 | 周三收盘/周四早晨为最佳读图窗口 |

> Max Pain吸附力与OI厚度正相关：大盘流动性好的标的（如GOOG/MSFT）吸附力强；OI稀薄的标的吸附力弱，需谨慎参考。
> ⚠️ 注意：很多热门标的（SPY/QQQ/NVDA/TSLA等）现在有**每日到期**的期权，如果只看"最近到期日"很容易把0DTE的Pin Risk错当成本周五的数据——务必对照第二节的`?expiration=`参数锁死到期日。

---

## 七、Notes 字段建议格式

每个标的的Notes应包含：
- **Entry**：入场条件（价位 + 触发信号），并注明当前Gamma环境（正/负）
- **Stop**：止损位，双止损——价格止损（锚定Put Wall或关键Gamma墙） + 结构止损（Gamma Flip被反向穿越）
- **Target**：目标位（通常锚定Call Wall或次级阻力，正Gamma环境下目标应更保守）

示例：
```
Entry: 突破Call Wall主档 + EMA9上穿EMA21，且当前处于负Gamma区（利于趋势延续）
Stop: 价格跌破Put Wall次档，或价格反向穿越Gamma Flip Point（两者先触发者为准）
Target: Call Wall第三档 / Max Pain上方N%（正Gamma环境下目标下调至Call Wall第一档）
```

---

*本模板用于结构化输出，配合代码工具做批量期权分析与回测。当前处于1个月实盘测试期（2026-07起），交易记录使用用户自有系统，本框架仅负责生成结构化期权流快照。*
