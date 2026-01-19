# 缠论买卖点信号工程规范（Spec v0.1）

本文档用于把“缠论（分型/笔/中枢/背驰/区间套）”落地为**可复现、可解释、可回测**的工程规则，供本项目后续实现与迭代使用。

> 说明：缠论原文强调“结构与级别”，并不提供固定天数或唯一实现。本规范选择一套**确定性规则版本**；同一份数据与同一份参数配置下，输出必须一致。

---

## 适用范围

- **默认级别**：日线 `D`（后续可扩展到 `60m/30m/5m/1m`）
- **输入数据**：按时间升序的 OHLCV（交易日序列允许缺口）
- **输出目标**：
  - 结构：分型 → 笔 → 中枢 → 离开/回抽 → 背驰证据
  - 信号：`1/2/3 买点`、`1/2/3 卖点`（以及置信度/证据）

---

## 1. 数据要求

### 历史长度

- **建议**：`>= 260` 交易日（日线一买/二买/三买在结构上更有意义）
- **最低**：`>= 120` 交易日（可输出候选信号，但误报率更高）

### 指标参数（统一口径）

- **MACD**：`(fast=12, slow=26, signal=9)`
- **定义**：
  - `DIF = EMA12(close) - EMA26(close)`
  - `DEA = EMA9(DIF)`
  - `HIST = DIF - DEA`（**不乘2**，统一口径）

---

## 2. 分型（Fractal）定义（确定性）

采用 **5K 分型**（两边各 2 根 K 线确认），索引为 `i`：

- **顶分型**：`high[i] > high[i-1], high[i-2], high[i+1], high[i+2]`
- **底分型**：`low[i] < low[i-1], low[i-2], low[i+1], low[i+2]`

### 冲突与去噪规则

- **最小间隔**：相邻分型的索引差 `>= FRACTAL_MIN_GAP`
- **同类相邻冲突**：若连续出现同类分型，保留更“极端”的那个
  - 顶分型：保留 `high` 更高者
  - 底分型：保留 `low` 更低者

### 配置项（默认值）

- `FRACTAL_WINDOW = 2`
- `FRACTAL_MIN_GAP = 3`

---

## 3. 笔（Bi）定义（v0.1 作为主输入单位）

### 构造规则

- 只连接“**相邻异类分型**”
  - 底 → 顶：上笔
  - 顶 → 底：下笔
- 一笔必须满足：
  - 分型间隔 `>= BI_MIN_GAP`
  - 幅度过滤（可选）：`abs(price_end - price_start) / price_start >= BI_MIN_PCT`

### 笔区间定义

- 上笔区间：`[low(start_bottom), high(end_top)]`
- 下笔区间：`[low(end_bottom), high(start_top)]`

### 配置项（默认值）

- `BI_MIN_GAP = 3`
- `BI_MIN_PCT = 0.0`（v0.1 默认不启用幅度阈值，后续可根据回测启用）

---

## 4. 中枢（Zhongshu）识别（严格三笔重叠）

对连续三笔 `B1, B2, B3`，每笔区间为 `[Lk, Hk]`：

- `ZS_LOW = max(L1, L2, L3)`
- `ZS_HIGH = min(H1, H2, H3)`
- 若 `ZS_LOW < ZS_HIGH`，则形成一个中枢 `ZS = [ZS_LOW, ZS_HIGH]`

### 中枢状态机（必须落地）

- `forming`：刚由三笔形成
- `active`：后续笔与中枢发生交互（进入/穿越/震荡）
- `left_up / left_down`：已离开中枢
- `broken`：离开后又重新进入中枢区间（触发“扩张/新中枢”逻辑）

### 离开判定（默认保守）

- `leave_up`：某一笔满足 `L > ZS_HIGH`
- `leave_down`：某一笔满足 `H < ZS_LOW`

### 中枢区间更新策略（v0.1 选型）

- `ZS_UPDATE_MODE = fixed`：中枢区间以首次形成的 `[ZS_LOW, ZS_HIGH]` 为准，不滚动更新
  - 中枢“扩张/升级”通过生成“新中枢”处理（避免区间在震荡中漂移导致不可复核）

### 配置项（默认值）

- `ZS_UPDATE_MODE = fixed`
- `ZS_LEAVE_REQUIREMENT = 1`（一笔离开即可视为离开；后续可调为 2 以提高稳健性）

---

## 5. 背驰（Divergence）力度度量（统一口径）

背驰本质：**同级别两段同向走势**对比，后段力度不如前段。

### 5.1 走势段（Seg）划分（v0.1）

- 优先使用“中枢离开后的同向笔序列”作为一个段
- 若没有可用中枢：退化为“相邻两段同向的笔序列”

### 5.2 力度特征（建议同时保留，便于复核）

对每个段 `Seg`：

- **PriceEfficiency**：`abs(delta_price) / bars`
- **MACD_Area**：段内 `sum(abs(HIST))`
- **MACD_Extreme**：
  - 下跌段：`min(HIST)`（越负越强）
  - 上涨段：`max(HIST)`（越正越强）

### 5.3 背驰判定（下跌末端的一买场景）

满足：

- 价格创新低（后段段末价格更低）
- 且动能衰竭（满足其一）：
  - `MACD_Area_new < MACD_Area_prev`
  - 或 `MACD_Extreme_new > MACD_Extreme_prev`（更接近 0，力度更弱）

### 5.4 背驰打分（推荐）

为了减少“非黑即白”，输出：

- `div_score ∈ [0, 100]`
- 并保留 `area_ratio/extreme_ratio/efficiency_ratio` 作为证据

配置项（默认值）：

- `DIV_SCORE_THRESHOLD = 60`

---

## 6. 1/2/3 买点确认规则（v0.1）

### 6.1 一买（Buy1）

**含义**：下跌末端背驰 + 出现向上启动的确认。

确认条件：

- 处于下跌末端（可用笔方向连续性或 MA 过滤作为辅助）
- 出现下跌段背驰（`div_score >= DIV_SCORE_THRESHOLD`）
- 触发确认（二选一）：
  - **结构触发**：出现向上一笔，且突破前一反弹关键点
  - **指标近似触发**：MACD 金叉 / HIST 明显持续收敛后转强

输出：`buy_point=1`，并记录 `divergence_bar/confirm_bar/evidence`

### 6.2 二买（Buy2）

**含义**：一买后离开向上 → 第一次回抽确认（更稳）。

确认条件（v0.1 日线近似）：

- 一买后已出现离开向上
- 第一次回抽不破坏关键结构（例如不破一买低点/不深回到关键中枢）
- 回抽末端出现“次级别一买”的近似：回抽内部背驰 + 再次向上启动

> v0.2：使用区间套严格确认：`日线二买 = 60/30m 一买`

### 6.3 三买（Buy3）

**含义**：向上离开中枢后回抽，但回抽不回中枢（趋势中继确认）。

确认条件：

- 存在中枢 `ZS` 且已向上离开
- 回抽低点不进入中枢区间（常用：回抽笔 `L > ZS_HIGH`）
- 回抽结束后再次向上启动（至少一上笔确认）

输出：`buy_point=3`，并给出 `ZS=[low,high]` 与回抽关键点

---

## 7. 1/2/3 卖点确认规则（对称）

卖点与买点对称：

- **一卖（Sell1）**：上涨末端背驰 + 向下启动确认
- **二卖（Sell2）**：一卖后向下离开 → 第一次反弹不过/背驰 → 再次下行启动
- **三卖（Sell3）**：向下离开中枢后反抽不回中枢（`high < ZS_LOW`）+ 再次下行启动

---

## 8. 多级别联动（区间套）规范（预留接口）

定义 `LowerTFConfirm(level, window)`：

- 输入：高一级别的某个“回抽窗口 window=[t_start,t_end]”
- 在更低级别上仅分析该窗口，寻找严格的 `Buy1/Sell1`

规则示例：

- `Buy2(D) := Buy1(30m or 60m) within pullback_window`
- 确认映射：低级别确认点映射到最近的日线确认日

v0.1：可不实现，但必须预留字段与接口。

---

## 9. 输出数据结构（建议）

每只股票输出（结构化，便于 LLM 与报告层消费）：

- `chanlun.level = "D"`
- `chanlun.buy = {type: 1|2|3|none, score, confirm_date, key_price, evidence[]}`
- `chanlun.sell = {type: 1|2|3|none, score, confirm_date, key_price, evidence[]}`
- `chanlun.zs = [{id, low, high, state, start_date, end_date}]`
- `chanlun.debug = {fractals_count, bis_count, params}`

---

## 10. v0.1 默认配置清单（便于团队统一）

- `FRACTAL_WINDOW=2`
- `FRACTAL_MIN_GAP=3`
- `BI_MIN_GAP=3`
- `BI_MIN_PCT=0.0`
- `ZS_UPDATE_MODE=fixed`
- `ZS_LEAVE_REQUIREMENT=1`
- `DIV_SCORE_THRESHOLD=60`

---

## 11. 后续迭代（v0.2+ 方向）

- 引入线段（Segment）并固化切换规则（可配置）
- 区间套（多级别）落地：日线二买/三买用 60/30m 严格确认
- 中枢扩张/升级的阈值策略（宽度/段数/波幅）可配置并可回测

