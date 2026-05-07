---
name: factor-pipeline
description: 加密货币因子参数化范式，采用“13位串联密码”逻辑，通过线性流水线（Pipeline）生成因子。适用于自动化因子挖掘、参数寻参及遗传算法迭代。当用户设计因子生成逻辑、定义因子编码或优化因子挖掘流程时使用。
tools: ["Read", "Write"]
model: sonnet
effort: medium
---
# 因子生成自动化：13位串联密码流水线范式

## 0. 任务 Checklist (完成后删除)
- [x] **1. 数据 Schema 确认** (已在《知识节点》定义)
- [x] **2. 算子库实现方案** (Numpy, Pandas, TALib, Numba.jit)
- [x] **3. 因子脚本模板规范** (signal(*args) 签名, 变量清理等)
- [ ] **4. 密码位选项库构建** (正在进行中...)
- [ ] **5. 环境与路径约束确认**
- [ ] **6. 异常处理与数据对齐逻辑确认**

## 1. 核心逻辑：13位线性流水线
因子通过一个定长的 **13位密码向量** 表示。若某一步骤不需要，则填入 `None`。

| 密码位 | 模块名称 | 逻辑功能说明 |
| :--- | :--- | :--- |
| **1** | **元因子 (Meta)** | 基础原材料（如 Close, Volume, FundingRate, Turnover 等） |
| **2** | **定性转换 (Trans)** | 方向性过滤（如：只在涨/跌时统计，Only_Up, Only_Down） |
| **3** | **时序流1参数** | 第一阶段时序计算的窗口 or 参数设置 |
| **4** | **时序计算流1** | 基础提取/平滑（如 MA, STD, EMA, Min, Max） |
| **5** | **时序流2参数** | 第二阶段时序计算的窗口 or 参数设置 |
| **6** | **时序计算流2** | 特征增强/变换（如 Diff, ROC, Log, Abs） |
| **7** | **时序流3参数** | 第三阶段时序计算的窗口 or 参数设置 |
| **8** | **时序计算流3** | 时序归一化/相对位置（如 TS_Rank, ZScore, MinMax） |
| **9** | **XS币池前滤因子** | 定义“战场”：用于筛选计算范围的参考因子（如 Volume） |
| **10** | **XS币池前滤范围** | 定义筛选比例（如 Top 50%, Bottom 20%） |
| **11** | **XS当时计算流** | 横截面核心计算（如 CS_Rank, CS_ZScore, Vs_BTC） |
| **12** | **XS时序参数设置** | 横截面结果再处理的窗口设置 |
| **13** | **XS时序计算流** | 排名/特征的稳定性处理（如 XS_MA, XS_STD） |

## 2. 密码位选项库 (Factor Code Library)

### 2.1 [位1] 元因子 (Meta)
- `close`: `df['close']`
- `high`: `df['high']`
- `low`: `df['low']`
- `tp`: `(df['high'] + df['low'] + df['close']) / 3` (典型价格)
- `c1`: `df['high'] - df['low']` (绝对振幅)
- `zf`: `(df['high'] - df['low']) / df['open']` (相对振幅)
- `co`: `df['close'] / df['open']`
- `hl`: `df['high'] / df['low']`
- `ho`: `df['high'] / df['open']`
- `closepos`: `(df['close'] - df['low']) / (df['high'] - df['low'])`
- `return1h`: `df['close'].pct_change(1)`
- `returnzfr`: `(df['close'] - df['open']) / (df['high'] - df['low'] + 1e-8)`
- `qv`: `df['quote_volume']` (成交额)
- `tbqv`: `df['taker_buy_quote_asset_volume']` (主动买入额)
- `tbqvr`: `df['taker_buy_quote_asset_volume'] / df['quote_volume']` (主动买入比)
- `tn`: `df['trade_num']` (成交笔数)
- `ff8hmean`: `df['funding_fee'].rolling(window=8, min_periods=1).mean()` (资金费平滑)
- `cmcap`: `df['circulating_supply'] * df['close']` (流通市值)
- `turnover`: `df['quote_volume'] / (df['circulating_supply'] * df['close'])` (换手率)

### 2.2 [位2] 定性转换 (Trans)
- `onlyz`: `np.where(df['close'] > df['close'].shift(1), var, 0)` (仅保留上涨样本，其余归零)
- `onlyd`: `np.where(df['close'] < df['close'].shift(1), var, 0)` (仅保留下跌样本，其余归零)
- `None`: 不执行转换，直接传递 `var`

### 2.3 [位3, 5, 7, 12] 时序参数 (TS Params)
定义当前计算流使用的窗口参数 `n_cur`。
- `n`: `n_cur = n` (由 `signal` 输入的单参数或全局参数)
- `n1`: `n_cur = n1` (当输入为 `tuple` 时，取第一位)
- `n2`: `n_cur = n2` (当输入为 `tuple` 时，取第二位)
- `nbi`: `n_cur = [int(n/2), n]` (生成二等分窗口列表)
- `ntri`: `n_cur = [int(n/3), int(n*2/3), n]` (生成三等分窗口列表)
- `n1bi`: `n_cur = [int(n1/2), n1]`
- `n1tri`: `n_cur = [int(n1/3), int(n1*2/3), n1]`
- `n2bi`: `n_cur = [int(n2/2), n2]`
- `n2tri`: `n_cur = [int(n2/3), int(n2*2/3), n2]`
- `[fixed_int]`: `n_cur = int(fixed_int)` (如 24, 168 等固定数值)

### 2.4 [位4, 6, 8] 时序计算流 (TS Operators)
基于当前变量 `{var}` 和窗口 `n_cur` 执行计算。若 `n_cur` 为列表，则对列表内每个参数执行计算并取均值。

#### 2.4.FUC. 常用计算流 (Frequently Used Compute)
定义通用计算模板，其中 `a` 代表输入序列（Series）。
- `mean(a)`: `a.rolling(window=n_cur, min_periods=1).mean()`
- `ewm3k(a)`: `ewm_fixed(a, window=3000, span=n_cur, min_periods=3)`
- `ewm3k2n(a)`: `ewm_fixed(a, window=3000, span=2*n_cur, min_periods=3)`
- `max(a)`: `a.rolling(window=n_cur, min_periods=1).max()`
- `min(a)`: `a.rolling(window=n_cur, min_periods=1).min()`
- `median(a)`: `a.rolling(window=n_cur, min_periods=1).median()`
- `std(a)`: `a.rolling(window=n_cur, min_periods=2).std()`
- `rank(a)`: `a.rolling(window=n_cur, min_periods=1).rank(ascending=True, pct=True)`
- `lr(a)`: `ta.LINEARREG(a, timeperiod=n_cur)`
- `md(a, b)`: `(a - b).abs().rolling(window=n_cur, min_periods=1).mean()`
- `chg(a)`: `a - a.shift(1)`
- `abs(a)`: `a.abs()`
- `sum(a)`: `a.rolling(window=n_cur, min_periods=1).sum()`
- `top2avg(a)`: `(a.rolling(window=n_cur, min_periods=1).max() + a.rolling(window=n_cur, min_periods=1).quantile((n_cur-1)/n_cur)) / 2`
- `q25(a)`: `a.rolling(window=n_cur, min_periods=1).quantile(0.25)`
- `q75(a)`: `a.rolling(window=n_cur, min_periods=1).quantile(0.75)`
- `diff1(a)`: `a / a.shift(1)`

#### 2.4.A. 基础区间统计 (Basic Stats)
- `mean`: `df['{var}-mean'] = FUC.mean(df['{var}'])`
- `ewm3k`: `df['{var}-ewm3k'] = FUC.ewm3k(df['{var}'])`
- `ewmsqrtn3k`: `df['{var}-ewmsqrtn3k'] = ewm_fixed(df['{var}'], window=3000, span=int(np.sqrt(n_cur)), min_periods=3)`
- `max`: `df['{var}-max'] = FUC.max(df['{var}'])`
- `min`: `df['{var}-min'] = FUC.min(df['{var}'])`
- `range`: `df['{var}-range'] = FUC.max(df['{var}']) - FUC.min(df['{var}'])`
- `std`: `df['{var}-std'] = FUC.std(df['{var}'])`
- `mediand`: `df['{var}-median'] = FUC.median(df['{var}'])` \n `df['{var}-mediand'] = FUC.md(df['{var}'], df['{var}-median'])`
- `cumchg`: `df['{var}-chg'] = FUC.chg(df['{var}'])` \n `df['{var}-cumchg'] = FUC.sum(FUC.abs(df['{var}-chg']))`
- `skew`: `df['{var}-skew'] = df['{var}'].rolling(window=n_cur, min_periods=2).skew()`
- `kurtp3`: `df['{var}-kurtp3'] = df['{var}'].rolling(window=n_cur, min_periods=2).kurt() + 3`
- `lr`: `df['{var}-lr'] = FUC.lr(df['{var}'])`

#### 2.4.B. 绕1归一化 (Normalization Around 1)
- `mtm1`: `df['{var}-mtm1'] = df['{var}'] / df['{var}'].shift(n_cur)`
- `meanbias1`: `df['{var}-meanbias1'] = df['{var}'] / FUC.mean(df['{var}'])`
- `meanbias05`: `df['{var}-meanbias05'] = 0.5 * df['{var}'] / FUC.mean(df['{var}'])`
- `top2bias1`: `df['{var}-top2avg'] = FUC.top2avg(df['{var}'])` \n `df['{var}-top2bias1'] = df['{var}'] / df['{var}-top2avg']`
- `diff1mean`: `df['{var}-diff1'] = FUC.diff1(df['{var}'])` \n `df['{var}-diff1mean'] = FUC.mean(df['{var}-diff1'])`
- `diff1mean05`: `df['{var}-diff1'] = FUC.diff1(df['{var}'])` \n `df['{var}-diff1mean05'] = 0.5 * FUC.mean(df['{var}-diff1'])`
- `diff1`: `df['{var}-diff1'] = FUC.diff1(df['{var}'])`
- `lrmtm1`: `df['{var}-lrmtm1'] = FUC.lr(df['{var}']) / df['{var}'].shift(n_cur)`
- `lrmeanbias1`: `df['{var}-lrmeanbias1'] = FUC.lr(df['{var}']) / FUC.mean(df['{var}'])`
- `lrmeanbias05`: `df['{var}-lrmeanbias05'] = 0.5 * FUC.lr(df['{var}']) / FUC.mean(df['{var}'])`
- `ewm3kbias1v1`: `df['{var}-ma'] = FUC.ewm3k(df['{var}'])` \n `df['{var}-ma2'] = FUC.ewm3k2n(df['{var}'])` \n `df['{var}-ewm3kbias1v1'] = df['{var}-ma'] / (df['{var}-ma2'] + 1e-10)`
- `ewm3kbias1v2`: `df['{var}-ma'] = FUC.ewm3k(df['{var}'])` \n `df['{var}-ma2'] = FUC.ewm3k(df['{var}-ma'])` \n `df['{var}-ewm3kbias1v2'] = df['{var}-ma'] / (df['{var}-ma2'] + 1e-10)`
- `ewm3kbias05v1`: `执行 ewm3kbias1v1 逻辑` \n `df['{var}-ewm3kbias05v1'] = 0.5 * df['{var}-ewm3kbias1v1']`
- `ewm3kbias05v2`: `执行 ewm3kbias1v2 逻辑` \n `df['{var}-ewm3kbias05v2'] = 0.5 * df['{var}-ewm3kbias1v2']`

#### 2.4.C. 绕0归一化 (Normalization Around 0)
- `mtm0`: `df['{var}-mtm0'] = df['{var}'] / df['{var}'].shift(n_cur) - 1`
- `meanbias0`: `df['{var}-meanbias0'] = df['{var}'] / FUC.mean(df['{var}']) - 1`
- `top2bias0`: `执行 top2bias1 逻辑` \n `df['{var}-top2bias0'] = df['{var}-top2bias1'] - 1`
- `diff0mean`: `df['{var}-diff0'] = FUC.diff1(df['{var}']) - 1` \n `df['{var}-diff0mean'] = FUC.mean(df['{var}-diff0'])`
- `ewm3kbias0v1`: `执行 ewm3kbias1v1 逻辑` \n `df['{var}-ewm3kbias0v1'] = df['{var}-ewm3kbias1v1'] - 1`
- `ewm3kbias0v2`: `执行 ewm3kbias1v2 逻辑` \n `df['{var}-ewm3kbias0v2'] = df['{var}-ewm3kbias1v2'] - 1`
- `lrmtm0`: `df['{var}-lrmtm0'] = FUC.lr(df['{var}']) / df['{var}'].shift(n_cur) - 1`
- `lrmeanbias0`: `df['{var}-lrmeanbias0'] = FUC.lr(df['{var}']) / FUC.mean(df['{var}']) - 1`

#### 2.4.D. 线性映射与排名 (Linear Mapping & Rank)
- `meanbd0`: `df['{var}-mean'] = FUC.mean(df['{var}'])` \n `df['{var}-meanbd0'] = (df['{var}'] - df['{var}-mean']) / FUC.md(df['{var}'], df['{var}-mean'])`
- `minbd0`: `df['{var}-min'] = FUC.min(df['{var}'])` \n `df['{var}-mean'] = FUC.mean(df['{var}'])` \n `df['{var}-minbd0'] = (df['{var}'] - df['{var}-min']) / FUC.md(df['{var}'], df['{var}-mean'])`
- `meanbs0`: `df['{var}-mean'] = FUC.mean(df['{var}'])` \n `df['{var}-meanbs0'] = (df['{var}'] - df['{var}-mean']) / FUC.std(df['{var}'])`
- `top2bd0`: `df['{var}-top2avg'] = FUC.top2avg(df['{var}'])` \n `df['{var}-top2bd0'] = (df['{var}'] - df['{var}-top2avg']) / FUC.md(df['{var}'], df['{var}-top2avg'])`
- `top2bs0`: `df['{var}-top2avg'] = FUC.top2avg(df['{var}'])` \n `df['{var}-top2bs0'] = (df['{var}'] - df['{var}-top2avg']) / FUC.std(df['{var}'])`
- `magiccci3k`: `df['{var}-ma'] = FUC.ewm3k(df['{var}'])` \n `df['{var}-ma2'] = FUC.ewm3k(df['{var}-ma'])` \n `df['{var}-magiccci3k'] = (df['{var}-ma'] - df['{var}-ma2']) / FUC.ewm3k((df['{var}-ma'] - df['{var}-ma2']).abs())`
- `minmax`: `df['{var}-minmax'] = (df['{var}'] - FUC.min(df['{var}'])) / (FUC.max(df['{var}']) - FUC.min(df['{var}'])) + (df['{var}']/FUC.min(df['{var}']))*0.001 + (df['{var}']/FUC.max(df['{var}']))*0.001`
- `minmaxo`: `df['{var}-minmaxo'] = (df['{var}'] - FUC.min(df['{var}'])) / (FUC.max(df['{var}']) - FUC.min(df['{var}']))`
- `maxmino`: `df['{var}-maxmino'] = (FUC.max(df['{var}']) - df['{var}']) / (FUC.max(df['{var}']) - FUC.min(df['{var}']))`
- `zscore01`: `df['{var}-zscore'] = (df['{var}'] - FUC.mean(df['{var}'])) / FUC.std(df['{var}'])` \n `df['{var}-zscore01'] = np.where(df['{var}-zscore'] < -3, 0, np.where(df['{var}-zscore'] > 3, 1, (df['{var}-zscore'] + 3) / 6)) + (df['{var}-zscore'] + 100) / 10000`
- `minmean`: `df['{var}-minmean'] = (df['{var}'] - FUC.min(df['{var}'])) / (FUC.mean(df['{var}']) - FUC.min(df['{var}'])) + (df['{var}']/FUC.min(df['{var}']))*0.001 + (df['{var}']/FUC.mean(df['{var}']))*0.001`
- `meanmax`: `df['{var}-meanmax'] = (df['{var}'] - FUC.mean(df['{var}'])) / (FUC.max(df['{var}']) - FUC.mean(df['{var}'])) + (df['{var}']/FUC.mean(df['{var}']))*0.001 + (df['{var}']/FUC.max(df['{var}']))*0.001`
- `q25q75`: `df['{var}-q25q75'] = (df['{var}'] - FUC.q25(df['{var}'])) / (FUC.q75(df['{var}']) - FUC.q25(df['{var}'])) + (df['{var}']/FUC.q25(df['{var}']))*0.001 + (df['{var}']/FUC.q75(df['{var}']))*0.001`
- `q`: `df['{var}-q'] = FUC.rank(df['{var}']) + ((df['{var}'] - FUC.mean(df['{var}'])) / FUC.std(df['{var}']) + 100) * 0.00001`
- `rank`: `df['{var}-rank'] = FUC.rank(df['{var}'])`
- `rankd`: `df['{var}-rankd'] = df['{var}'].rolling(n_cur, min_periods=1).rank(ascending=False, pct=True)`

#### 2.4.E. 复合定性指标 (Qualitative Indicators)
- `rsi`: 
    1. `df['{var}-diff'] = FUC.chg(df['{var}'])`
    2. `df['{var}-up'] = np.where(df['{var}-diff'] > 0, df['{var}-diff'], 0)`
    3. `df['{var}-down'] = np.where(df['{var}-diff'] < 0, -df['{var}-diff'], 0)`
    4. `df['{var}-rsu'] = FUC.mean(df['{var}-up'])`
    5. `df['{var}-rsd'] = FUC.mean(df['{var}-down'])`
    6. `df['{var}-rsi'] = df['{var}-rsu'] / (df['{var}-rsu'] + df['{var}-rsd'] + 1e-8)`
- `zdrsi`: 
    1. `df['close_diff'] = FUC.chg(df['close'])`
    2. `df['{var}-up'] = np.where(df['close_diff'] > 0, df['{var}'], 0)`
    3. `df['{var}-down'] = np.where(df['close_diff'] < 0, df['{var}'], 0)`
    4. `df['{var}-rsu'] = FUC.mean(df['{var}-up'])`
    5. `df['{var}-rsd'] = FUC.mean(df['{var}-down'])`
    6. `df['{var}-zdrsi'] = df['{var}-rsu'] / (df['{var}-rsu'] + df['{var}-rsd'] + 1e-8)`
- `zdratio`: 
    1. `[执行 zdrsi 步骤 1-5]`
    2. `df['{var}-zdratio'] = df['{var}-rsu'] / (df['{var}-rsd'] + 1e-8)`
- `bc0`: 
    1. `df['{var}-mean'] = FUC.mean(df['{var}'])`
    2. `df['{var}-std'] = FUC.std(df['{var}'])`
    3. `df['{var}-upper'] = df['{var}-mean'] + 2 * df['{var}-std']`
    4. `df['{var}-lower'] = df['{var}-mean'] - 2 * df['{var}-std']`
    5. `df['{var}-count'] = 0`, `df.loc[df['{var}'] > df['{var}-upper'], '{var}-count'] = 1`, `df.loc[df['{var}'] < df['{var}-lower'], '{var}-count'] = -1`
    6. `df['{var}-bc0'] = FUC.sum(df['{var}-count'])`
- `er`: 
    1. `df['{var}-pctchange'] = FUC.abs(FUC.chg(df['{var}']))`
    2. `df['{var}-er'] = (df['{var}'] - df['{var}'].shift(n_cur)) / (FUC.sum(df['{var}-pctchange']) + 1e-08)`

### 2.5 [位11, 13] 横截面计算流 (XS Operators)
基于全币种在同一时间戳的横截面状态进行计算（位11），或对横截面结果进行时序平滑（位13）。

#### 2.5.A. [位11] XS 当时计算流 (Instantaneous XS)
- `xrank`: `df['{var}-xrank'] = df.groupby('candle_begin_time')['{var}'].rank(ascending=True, pct=True)`
- `xmean`: `df['{var}-xmean'] = df.groupby('candle_begin_time', group_keys=False)['{var}'].transform('mean')`
- `xbtc`: 
    ```python
    btc_symbol = 'BTC-USDT' if 'BTC-USDT' in df['symbol'].values else 'BTCUSDT'
    btc_values = df[df['symbol'] == btc_symbol].set_index('candle_begin_time')['{var}']
    df['{var}-xbtc'] = df['candle_begin_time'].map(btc_values)
    ```
- `xeth`: 
    ```python
    eth_symbol = 'ETH-USDT' if 'ETH-USDT' in df['symbol'].values else 'ETHUSDT'
    eth_values = df[df['symbol'] == eth_symbol].set_index('candle_begin_time')['{var}']
    df['{var}-xeth'] = df['candle_begin_time'].map(eth_values)
    ```

#### 2.5.B. [位13] XS 时序计算流 (XS Time-Series)
- `xmax`: `df['{var}-xmax'] = df.groupby('symbol')['{var}'].transform(lambda x: x.rolling(window=n_cur, min_periods=1).max())`

## 3. 因子生成器脚本：将串联密码转化为因子脚本

本模块定义如何通过 **《因子生成器脚本》(Generator)** 将 15 位密码逻辑自动化组装，并生成最终的 **《因子脚本》(Factor Script)**。

### 3.1 核心驱动：因子生成器脚本 (Generator)
- **定义**：负责解析 13 位密码并拼接代码的“主控程序”。它在内存中维护生成状态，通过流水线密码位的配置，在因子脚本中分步增加计算流信息，最终将完成的因子脚本保存在本地。
- **变量流转指针 (active_var)**：生成器内部维护的动态指针，用于追踪流水线中最新的有效列名，实现各密码位间的逻辑串联（状态仅存在于生成器内存中）。
- **文件位置**：该生成器脚本位于项目根目录下，文件名为 `factor_generator.py`。

### 3.2 准备阶段：计算流模板库平铺化 (Flattening Operators)
- **定义**：在生成因子脚本之前，生成器需预先将 **2.4 选项库** 中的多层 FUC 嵌套逻辑“展开”为单层、可即时替换的**计算流模板库**。
- **平铺化要求**：每一个计算流选项必须对应一段**完整且可独立执行**的代码片段。
    - **示例**：对于 `rsi`，平铺后的模板应包含其完整的 6 步计算逻辑（如下所示），其中所有的 FUC 引用都已被替换为真实的 Pandas/Numpy 代码，且变量名已统一为 `{var}` 相关的宏占位符。
    ```python
    # 平铺后的 rsi 模板示例（生成器内部存储格式）
    df['{var}-diff'] = df['{var}'] - df['{var}'].shift(1)
    df['{var}-up'] = np.where(df['{var}-diff'] > 0, df['{var}-diff'], 0)
    df['{var}-down'] = np.where(df['{var}-diff'] < 0, -df['{var}-diff'], 0)
    df['{var}-rsu'] = df['{var}-up'].rolling(window=n_cur, min_periods=2).mean()
    df['{var}-rsd'] = df['{var}-down'].rolling(window=n_cur, min_periods=2).mean()
    df['{var}-rsi'] = df['{var}-rsu'] / (df['{var}-rsu'] + df['{var}-rsd'] + 1e-8)
    ```
    - **禁止引用**：平铺后的库中不得再包含 `FUC.xxx` 或其他函数调用，必须是纯粹的底层实现代码。
- **宏替换原则与安全性检查**：
    - **精确匹配**：生成器在执行替换时，应确保 `{var}` 占位符被精确识别。
    - **字符合法性**：平铺后的代码片段应为纯净的 Python 运算逻辑。生成器应检查模板中除 `{var}` 和 `n_cur` 外不含多余的花括号，避免转义冲突。
    - 生成器应准备好一套“平铺版”计算流代码库。
    - 组装时，仅执行简单的字符串替换（如 `{var}` -> `active_var`），直接将完整的计算步骤文本写入《因子脚本》。
    - 目的：确保最终生成的因子脚本逻辑直观、无深层函数依赖、且易于调试。

### 3.3 时序因子脚本：预处理与声明逻辑
生成器基于逻辑判断，将以下必要信息填充至《时序因子》的 Header 区域：
- **额外数据注入**：若 [位1] 涉及市值类元因子，注入 `extra_data_dict = {'coin-cap': ['circulating_supply']}`。
- **定制函数注入**：若计算流中包含字符串 **“3k”**，注入 `ewm_fixed` 函数的完整源代码：
    ```python
    def ewm_fixed(arr, window=1000, span=None, alpha=None, min_periods=None):
        if span is not None:
            alpha = 2/(span+1)
        if span is None and alpha is None:
            raise ValueError('You must pass either span or alpha')
        y = arr.ewm(alpha=alpha, adjust=False).mean()
        w = window
        residual = (1-alpha)**(w) * y + (alpha-1)*(1-alpha)**(w-1) * arr.shift(-1)
        residual = residual.shift(w).fillna(0)
        ewm_fixed = y - residual
        if min_periods is not None:
            ewm_fixed[:min_periods-1] = np.nan
        return ewm_fixed
    ```

### 3.4 时序因子脚本：signal 函数构建与组装逻辑 (位1 - 位8)
生成器按照以下顺序，将主要计算流程文本写入《时序因子》的 **signal 函数** 内部：
- **[位1] 执行**：在 **4.3 因子计算** 起始处，生成以元因子命名的列（如 `df['close'] = df['close']`），随后将 `active_var` 更新为该列名。
- **[位2] 执行**：基于 `active_var` 执行定性转换，生成新列 `{active_var}-{trans_name}`，随后将 `active_var` 更新为该新列名。
- **阶段性存证 (ts_pre_code)**：在位 2 执行结束后，生成器应将从位 1 到位 2 生成的所有代码文本保存至内部变量 `ts_pre_code`，供后续加速函数使用。
- **[位3, 5, 7] 执行**：配置当前的 `n_cur` 变量（int 或 list），为后续算子提供窗口参数。
- **[位4, 6, 8] 执行**：
    - **模式判定**：若 `n_cur` 为 `int` 执行 **单窗口模式**；若为 `list` 执行 **多窗口模式**。
    - **单窗口模式**：
        1. 根据当前位名称，从“平铺版计算流模板库”中读取完整的计算代码文本。
        2. 执行宏替换：将计算流模板中的 `{var}` 替换为 `active_var`。注意：计算流模板中的 `n_cur` 保持不变，它将直接引用因子脚本中已定义的变量。
        3. 将生成的代码文本写入《时序因子》。
        4. **更新指针**：将 `active_var` 更新为该步骤生成的最终列名。
    - **多窗口模式**：
        1. 生成初始化累加代码：`df['temp_sum'] = 0`。
        2. 遍历 `n_cur` 列表中的每个参数 `p`，对每个 `p` 重复执行上述“单窗口模式”的计算逻辑。注意：此时需将计算流模板中的 `n_cur` 文本替换为循环变量名 `p`，生成临时列 `df[f'{active_var}-{op}-{p}']`。
        3. 生成累加代码：`df['temp_sum'] += df[f'{active_var}-{op}-{p}']`。
        4. 生成均值代码：`df['{active_var}-{op}'] = df['temp_sum'] / len(n_cur)`。
        5. **更新指针**：仅在计算完算术平均后，将 `active_var` 更新为最终的均值列名。
- **阶段性存证 (ts_core_code)**：在位 8 计算逻辑执行结束（但在“执行结束收尾”之前），生成器应将从位 3 到位 8 生成的所有代码文本保存至内部变量 `ts_core_code`。
- **[位8] 执行结束收尾**：在时序计算流程全部结束后，生成最终因子赋值代码：`df[factor_name] = df[active_var]`。
- **signal 函数清理 df 过程变量**：在 `signal` 函数 `return` 前，生成器必须写入固定的清理代码：
    ```python
    # 清理 df 过程变量
    df.drop(columns=list(set(df.columns.values).difference(set(source_cls + [factor_name]))), inplace=True)
    return df
    ```

### 3.5 时序因子脚本：signal_multi_params 自动化生成
生成器在构建 `signal` 函数的同时，应基于生成过程中保存的本地变量（`ts_pre_code`, `ts_core_code`）同步构建 `signal_multi_params` 函数以实现计算加速。
- **组装逻辑**（与《知识节点》5.1-5.5 吻合）：
    1. **函数初始化**：
        ```python
        def signal_multi_params(df, param_list) -> dict:
            ret = dict()
        ```
    2. **常量逻辑执行**：将 `ts_pre_code`（由 3.4 阶段性存证生成）写入循环外。
    3. **参数遍历循环**：
        ```python
            for param in param_list:
                # 参数转换逻辑
                if isinstance(param, int): n = param
                elif isinstance(param, tuple): n1, n2 = param
                else: raise ValueError('Invalid parameter type')
        ```
    4. **变量逻辑计算**：
        - 将 `ts_core_code`（由 3.4 阶段性存证生成）写入循环内。
        - **输出转换**：将原代码末尾的 `df[factor_name] = df[active_var]` 替换为 `ret[str(param)] = df[active_var]`。
    5. **结果聚合与返回**：
        ```python
            return ret
        ```

### 3.6 时序因子脚本：命名与保存
- **时序因子名称 (ts_factor_name)**：
    - **逻辑**：将位 1 到位 8 的密码中每一位的字符串（若非 None）首字母大写，然后依次拼接，不使用任何连接符号。生成的字符串保存在生成器内部变量 `ts_factor_name` 中。
    - **参数位特殊处理**：针对参数位 3, 5, 7，若其对应的后续计算流（位 4, 6, 8）为 `None`，则该参数位在命名时也视为 `None`（不录入文件名）。
    - **示例**：`close` + `onlyz` + `n` (位3) + `mean` (位4) + `n1` (位5) + `None` (位6) -> `ts_factor_name = "CloseOnlyzNMean"`。
- **脚本保存**：
    - **路径**：将生成的脚本保存至《时序因子》指定的目录下（`factors/`）。
    - **文件名**：`{ts_factor_name}.py`。


### 3.7 横截面因子脚本：signal 函数构建与组装逻辑 (位9 - 位13)
生成器按照以下顺序，将主要计算流程文本写入《横截面因子》的 **signal 函数** 内部：
- **[初始化] 执行**：
    - **2.1 读取输入**：生成参数解析代码（`n=args[1]`；若为 tuple 则同时解包 `n1, n2=args[1]`）。
    - **2.2 初始列备份**：生成 `source_cls = df.columns.tolist()`。
    - **初始化 XS 指针 (active_var)**：生成器将 `active_var` 更新为字符串 `f"{ts_factor_name}_{{n}}"`（注：`ts_factor_name` 为生成器变量，`{n}` 为因子脚本运行时变量）。
    - **构建 get_factor_list**：同步生成该横截面因子所依赖的基础因子列表函数（详见 3.8）。
- **[位9, 10] 执行**：若位 9 不为 `None`，执行币池筛选逻辑：
    1. **备份完整数据**：`df_full = df.copy()`。
    2. **计算筛选排名**：基于位 9 指定的时序因子列（列名为 `f"{bit9_factor_name}_{{n}}"`），计算每个时间戳的横截面**升序**百分比排名。注意：`groupby` 需设置 `group_keys=False` 以保留原始索引。
    3. **截取子集**：根据位 10 提供的区间（如 `(0.5, 1.0)`），筛选出符合条件的行并覆盖当前 `df`（筛选条件：`rank_pct >= 下限` 且 `rank_pct < 上限`，其中排名为升序排名）。
    - **注意**：后续计算流将仅在筛选后的子集上运行，显著降低计算量并聚焦特定币域。
- **[位11] 执行**：
    1. 根据当前位名称，从“平铺版计算流模板库”中读取 XS 计算代码。
    2. 执行宏替换：将模板中的 `{var}` 替换为 `active_var`。
    3. **f-string 强制要求**：生成器在写入代码时，必须确保列名访问使用 f-string 格式（如 `df[f'{active_var}-{op_name}']`），以解析 `active_var` 中的运行时变量 `{n}`。
    4. 将生成的代码文本写入《横截面因子》。
    5. **更新指针**：将 `active_var` 更新为该步骤生成的最终列名。
- **[位12] 执行**：配置当前的 `n_cur` 变量（int 或 list），为后续 XS 时序算子提供窗口参数。
- **[位13] 执行**：
    - **模式判定**：若 `n_cur` 为 `int` 执行 **单窗口模式**；若为 `list` 执行 **多窗口模式**。
    - **单窗口模式**：
        1. 读取 XS 时序计算模板。
        2. 执行宏替换：将 `{var}` 替换为 `active_var`，保持 `n_cur` 不变。
        3. 确保生成的代码使用 f-string 访问列名。
        4. **更新指针**：将 `active_var` 更新为生成的最终列名。
    - **多窗口模式**：
        1. 生成初始化累加代码：`df['temp_sum'] = 0`。
        2. 遍历 `n_cur` 列表中的每个参数 `p`，对每个 `p` 重复执行上述“单窗口模式”的计算逻辑。此时需将计算流模板中的 `n_cur` 文本替换为循环变量名 `p`。
        3. 生成累加代码与均值代码，并将 `active_var` 更新为最终的均值列名。
- **[收尾] 执行**：
    1. **小币池截面映射到完整币池**：若执行过位 9 筛选，执行以下逻辑：
        - **校验位 11**：若位 11 为 `xmean`，则执行时间戳唯一映射：
            ```python
            xsec_values = df.groupby('candle_begin_time')['{active_var}'].first()
            df_full['{active_var}'] = df_full['candle_begin_time'].map(xsec_values)
            df = df_full
            ```
        - **兜底报错**：若位 11 不为 `xmean`，则直接报错退出，提示：“当前小币池截面状态仅支持时间戳唯一的截面算术平均相关数据，其他功能未开发”。
    2. **最终赋值与清理**：
        ```python
        # 最终因子赋值
        df[factor_name] = df['{active_var}']
        # 清理 df 过程变量
        df.drop(columns=list(set(df.columns.values).difference(set(source_cls + [factor_name]))), inplace=True)
        return df
        ```

### 3.8 横截面因子脚本：get_factor_list 函数构建逻辑
生成器需根据当前流水线状态，自动化生成该横截面因子所依赖的基础因子声明：
- **依赖项识别**：
    1. **核心时序因子**：引用生成器内部变量 `ts_factor_name`（由 3.6 定义，即位 1 到位 8 拼接而成的名称）。
    2. **筛选参考因子**：若位 9 不为 `None`，则将位 9 指定的因子名（`bit9_factor_name`）也加入依赖列表。
- **组装格式**：将识别出的因子名与参数 `n` 组成 tuple，并放入 list 中返回。
    ```python
    def get_factor_list(n):
        return [
            ('{ts_factor_name}', n), 
            ('{bit9_factor_name}', n)  # 仅当位 9 非 None 时存在，列名为 f"{bit9_factor_name}_{n}"
        ]
    ```

### 3.9 横截面因子脚本：命名与保存
- **横截面因子名称 (xs_factor_name)**：
    - **逻辑**：将位 1 到位 13 的密码中每一位的字符串（若非 None）首字母大写，然后依次拼接，不使用任何连接符号。生成的字符串保存在生成器内部变量 `xs_factor_name` 中。
    - **参数位特殊处理**：针对参数位 3, 5, 7, 12，若其对应的后续计算流（位 4, 6, 8, 13）为 `None`，则该参数位在命名时也视为 `None`（不录入文件名）。
    - **示例**：`close` + `onlyz` + `n` (位3) + `None` (位4) + `xrank` (位11) -> `CloseOnlyzXrank`。
- **脚本保存**：
    - **路径**：将生成的脚本保存至《横截面因子》指定的目录下（`sections/`）。
    - **文件名**：`{xs_factor_name}.py`。



## 4. 设计优势
- **高度解耦**：将“怎么算（时序）”与“在哪算（横截面）”完全分离。
- **兼容性强**：简单因子只需填充 1, 3, 4 位，其余为 `None` 即可。
- **搜索空间优化**：定长编码非常适合遗传算法（GA）进行交叉和变异。

---
## 参考案例
- 逻辑原型：`legacy_work/fa101_factor_factory_9_xsec.py`
- 存储路径：`~/.cursor/skills/factor-pipeline/SKILL.md`
