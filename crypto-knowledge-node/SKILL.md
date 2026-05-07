---
name: crypto-knowledge-node
description: 存储 B 圈策略开发系统中的核心知识节点、对象定义及环境规范。作为其他 Skill 的底层知识库，定义如 K 线 DataFrame 结构、目录规范、核心对象属性等。
tools: ["Read"]
model: sonnet
effort: low
---
# B 圈知识节点 (Crypto Knowledge Node)

本 Skill 用于定义系统中反复出现的关键对象 and 规范。在其他文档中引用时，建议使用 `《 》` 标识。

## 1. 逻辑对象

### 《彩虹》
- **定义**：由“邢不行”组织的量化交易团队，是当前 B 圈量化交易框架的开发者 and 维护者。

### 《选币元组》（选币因子项）
- **定义**：策略配置中用于**计算复合选币因子并参与排序选币**的一条配置。对应配置键 `factor_list`（或 `long_factor_list` / `short_factor_list`）中的**一个 tuple**。
- **Tuple 结构**（4 元组）：`(因子名, 是否升序, 参数, 权重)`  
  - **位 0**：`因子名`（str），与 `factors/` 下脚本名一致，由 `FactorHub.get_by_name` 解析。  
  - **位 1**：`是否升序`（bool），True 表示升序排名、选因子值小的在前，False 表示降序。  
  - **位 2**：`参数`（int 或 tuple），传入该因子 `signal` 的 `args[1]`，如 `168`、`(100, 124)`。  
  - **位 3**：`权重`（float），多条因子时按权重加权合成复合因子。  
- **与框架交互**：配置加载时经 `FactorConfig.parse_config_list` 转为 `FactorConfig`，列名为 `{因子名}_{参数}`；框架用 `calc_factor_common` / `calc_factor_expr_polars` 计算各因子并加权得到复合因子列，再按该列排序并用 `select_coin_num` 选币。

### 《过滤元组》（过滤项）
- **定义**：策略配置中用于**前置或后置过滤**的一条配置。对应配置键 `filter_list`（前置）或 `filter_list_post`（后置），以及多空分离时的 `long_filter_list` / `short_filter_list` 等中的**一个 tuple**。
- **Tuple 结构**（3 或 4 元组）：`(因子名, 参数, 条件字符串 [, 是否升序])`  
  - **位 0**：`因子名`（str），与 `factors/` 下脚本名一致。  
  - **位 1**：`参数`（int 或 tuple），该因子计算所用参数，列名拼接为 `{因子名}_{参数}`。  
  - **位 2**：`条件字符串`（str），格式为 `类型:范围`，无空格。  
    - **类型**：`rank`（按截面排名）、`pct`（按截面分位 0~1）、`val`（按因子值）。  
    - **范围**：单边比较 + 数值，支持 `>=`、`<=`、`==`、`!=`、`>`、`<`，如 `rank:>1`、`pct:<0.2`、`val:>=100`。框架**不支持**区间写法（如 `rank:(1,20)`），区间需拆成两条过滤。  
  - **位 3**（可选）：`是否升序`（bool），缺省为 True，用于 rank/pct 的排序方向。  
- **与框架交互**：配置加载时经 `FilterFactorConfig.init` 转为 `FilterFactorConfig`，列名同上；前滤在 `filter_before_select` 中、后滤在 `filter_after_select` 中，通过 `filter_common` / `filter_common_polars` 按 `method.how`（rank/pct/val）对列做截面排名或取值，再按 `method.range` 与条件比较，保留满足条件的行。

### 《策略组》
- **定义**：由多个《单策略 dict》组成的**列表**，对应配置中的 `strategy_list`（如 `config.strategy_list`）。用于回测时一次性加载一组策略。
- **组合方式**：
  - **物理层**：Python `list`，元素为《单策略 dict》，在 config 中通常写作 `strategy_list = [ {...}, {...}, ... ]`。
  - **组内资金归一化**：框架加载时对组内**所有**《单策略 dict》的 `cap_weight` 求和，再将每个 dict 的 `cap_weight` 赋为 `原值 / 总和`，从而**策略组整体资金权重恒为 1**。若希望等权，则每个 dict 的 `cap_weight` 均设为 1，由框架自动归一化为 1/N。
- **与框架交互**：通过 `BacktestConfig.load_strategy_config(strategy_list)` 传入《策略组》；框架先做上述 cap_weight 归一化，再对列表中每个《单策略 dict》调用 `StrategyConfig.init(..., **stg_dict)` 实例化为策略配置。除此之外的组级逻辑（如遍历、汇总因子列等）均在框架内部完成。

### 《单策略 dict》
- **定义**：策略组中的一个元素，即一个**单策略**的配置字典。与《彩虹》`StrategyConfig` 及 `BacktestConfig.load_strategy_config(stg_dict)` 的入参一致，加载时以 `**stg_dict` 传入 `StrategyConfig.init`。
- **核心字段**：
  - **身份与周期**：`strategy`（str，策略名，与 strategy 目录下文件名对应）、`hold_period`（如 `'24H'`、`'1D'`）、`offset_list`（list[int]，调仓偏移）。
  - **资金权重**：`cap_weight`（float，该策略在组内相对占比；由《策略组》加载时按组内总和归一化）。
  - **多空与选币**：`long_cap_weight`、`short_cap_weight`（多空资金比例）、`long_select_coin_num`、`short_select_coin_num`（选币数量/比例）、`is_use_spot`（是否使用现货）。
  - **选币与过滤**：`factor_list`（《选币元组》列表）、`filter_list`（《过滤元组》前置）、`filter_list_post`（《过滤元组》后置）；多空分离时可为 `long_factor_list`/`short_factor_list`、`long_filter_list`/`short_filter_list` 等。
  - **其它**：`use_custom_func`（是否使用自定义因子/过滤函数）等。
- **与框架交互**：作为《策略组》`list` 的一个元素传入 `load_strategy_config`；框架用其键值构造 `StrategyConfig` 实例，用于选币、过滤、资金分配等后续逻辑。

### 《中性做多/做空》
- **定义**：中性策略的持仓结构，由「指定币种方向」与「合约全市场对冲」两部分组成，实现对冲市场 Beta、暴露选币 Alpha。
- **中性做空**：做空**指定币种**（由因子选出的标的）的同时，做多**合约全市场**（或等权/市值加权的一篮子合约）。净敞口偏向空头选币、多头市场，赚取选币相对市场的负超额收益。
- **中性做多**：做多**指定币种**（由因子选出的标的）的同时，做空**合约全市场**（或等权/市值加权的一篮子合约）。净敞口偏向多头选币、空头市场，赚取选币相对市场的正超额收益。
- **与 §1.1 方向生成**：定性方向（如「中性做空跌得多的」）即描述**指定币种**的选币逻辑；对冲端（合约全市场）为固定结构，不参与方向描述。

## 2. 数据对象

### 《彩虹预处理数据》
- **数据路径**：项目根目录, `coin-binance-spot-swap-preprocess-pkl-1h-YYYY-MM-DD`, 示例: `coin-binance-spot-swap-preprocess-pkl-1h-2026-02-08/`
- **数据结构**：
    - **[类型1] 原始字典 (dict_pkl)**：
        - **物理层**：`spot_dict.pkl`, `swap_dict.pkl`
        - **逻辑层**：`dict` { `symbol`: `pd.DataFrame` }
        - **字段层**：行代表 1h K 线。核心列: `['candle_begin_time', 'symbol', 'open', 'high', 'low', 'close', 'volume', 'quote_volume', 'trade_num', 'taker_buy_base_asset_volume', 'taker_buy_quote_asset_volume', 'funding_fee', 'avg_price_1m', 'avg_price_5m', '是否交易']`
    - **[类型2] 透视表 (pivot_pkl)**：
        - **物理层**：`market_pivot_spot.pkl`, `market_pivot_swap.pkl`
        - **逻辑层**：`pd.DataFrame` (MultiIndex 或 Pivot 结构)
        - **字段层**：行通常为时间戳，列为币种，用于快速横截面计算。

### 《彩虹市值数据》
- **数据路径**：项目根目录, `coin-cap-YYYY-MM-DD`, 示例: `coin-cap-2026-02-08/`
- **数据结构**：
    - **物理层**：目录下各币种的 `.csv` 文件（如 `ENS-USDT.csv`）
    - **逻辑层**：`pd.DataFrame`
    - **字段层**：行代表每日市值。核心列: `['candle_begin_time', 'symbol', 'id', 'name', 'max_supply', 'circulating_supply', 'total_supply', 'usd_price', 'max_mcap', 'circulating_mcap', 'total_mcap']`。注意: 需跳过首行说明文字。

### 《时序因子》
- **数据路径**：项目根目录, `factors/`, 示例: `factors/PctChange.py`
- **数据结构与组成规范**：
    - **物理层**：`.py` 脚本文件，文件名即为因子名称（如 `PctChange.py`）。
    - **逻辑层**：包含标准化的函数接口与配置声明。
    - **核心组成部分**：
        1. **Python 包引用 (Imports)**：默认包含 `numpy as np`, `pandas as pd`, `talib as ta` 以及 `from numba import jit`。
        2. **额外数据源声明 (Extra Data)**：通过 `extra_data_dict` 变量定义，根据元因子的需求自动导入（如需要市值数据时声明 `{'coin-cap': ['circulating_supply']}`）。
        3. **定制计算函数 (Custom Functions)**：根据计算流的需求，内置特定的高性能或自定义算子（如 `ewm_fixed`, `weighted_mean_numba` 等）。
        4. **signal 函数 (Required)**：因子计算的核心入口。
            - 4.1 **读取输入**：`df = args[0]`, `factor_name = args[2]`；若 `args[1]` 为 `int` 则 `n = args[1]`，若为 `tuple` 则解包为 `n1, n2`，其余情况报错。
            - 4.2 **初始列备份**：`source_cls = df.columns.tolist()`。
            - 4.3 **因子计算与清理**：执行具体的因子加工逻辑，并在 `return` 前清理所有中间过程变量，仅保留原始列与最终因子列。
        5. **signal_multi_params 函数 (Optional)**：支持同因子多参数的批量聚合计算。
            - **核心优势**：通过“一次预处理，多次核心计算”，在批量寻参场景下可实现 3 倍以上的性能提升。
            - 5.1 **函数初始化**：定义函数签名并初始化结果字典 `ret`。
            - 5.2 **常量逻辑执行**：在循环外执行与参数无关的基础数据加工。
            - 5.3 **参数遍历循环**：进入 `param_list` 循环，并将当前参数解析为因子脚本变量（`n` 或 `n1, n2`），其余情况报错。
            - 5.4 **变量逻辑计算**：在循环内执行与参数相关的核心计算流。
            - 5.5 **结果聚合与返回**：将计算出的因子序列存入 `ret` 并最终返回。

### 《横截面因子》
- **数据路径**：项目根目录, `sections/`, 示例: `sections/pct.py`
- **数据结构与组成规范**：
    - **物理层**：`.py` 脚本文件。
    - **逻辑层**：包含 `signal` 函数和 `get_factor_list` 函数。
    - **核心组成部分**：
        1. **Python 包引用 (Imports)**：默认包含 `numpy as np`, `pandas as pd`。
        2. **signal 函数 (Required)**：因子计算的核心入口。
            - 2.1 **读取输入**：`df = args[0]`, `factor_name = args[2]`；若 `args[1]` 为 `int` 则 `n = args[1]`，若为 `tuple` 则 `n = args[1]` 且解包为 `n1, n2`（`n` 变量始终保留用于列名拼接），其余情况报错。
            - 2.2 **初始列备份**：`source_cls = df.columns.tolist()`。
            - 2.3 **因子计算与清理**：执行具体的横截面加工逻辑（通常包含 `groupby('candle_begin_time')`），并在 `return` 前清理所有中间过程变量，仅保留原始列与最终因子列。
        3. **get_factor_list 函数 (Required)**：返回该横截面因子所依赖的基础时序因子列表，供框架预先计算。

### 《资金曲线》
- **定义**：回测产出的一条**按时间戳逐笔**的权益与净值序列，用于计算收益、回撤等表现指标。通常由《彩虹》回测引擎在策略回测结束后写入每个策略（或 SGT）对应的数据目录。
- **数据路径**：依回测/策略落盘约定，例如 SGT 数据目录下的 `资金曲线.csv`，路径形如 `SGTs/{backtest_name}/{strategy_label}/资金曲线.csv`。
- **数据结构**：
  - **物理层**：单文件 `资金曲线.csv`，UTF-8 编码，首行为列名。
  - **逻辑层**：`pd.DataFrame`，每行对应一个**时间切片**（一根 K 线或一个调仓时刻），按 `candle_begin_time` 升序。
  - **字段层**（列名与含义，参考项目内 `资金曲线.csv` 案例）：
    | 列名 | 类型 | 含义 |
    |------|------|------|
    | （首列/索引） | int | 行号，从 0 起，可选保留为 CSV 首列 |
    | candle_begin_time | str/datetime | 蜡烛开始时间，如 `2022-01-01 00:00:00` |
    | equity | float | 当前权益（总资产） |
    | turnover | float | 成交额 |
    | fee | float | 手续费 |
    | funding_fee | float | 资金费率相关费用 |
    | marginRatio | float | 保证金比例等 |
    | long_pos_value | float | 多头持仓市值 |
    | short_pos_value | float | 空头持仓市值 |
    | 净值 | float | 累计净值（相对初始资金），用于收益与回撤计算 |
    | 涨跌幅 | float | 相对前一时段的涨跌幅，首行可为空 |
    | 是否爆仓 | float | 是否爆仓标记（如 0/1） |
    | long_short_ratio | float | 多空比 |
    | leverage_ratio | float | 杠杆比例 |
    | symbol_long_num | int | 多头持仓标的数 |
    | symbol_short_num | int | 空头持仓标的数 |
    | long_max_ratio | float | 多头最大单标的占比 |
    | long_max_ratio_symbol | str | 多头占比最大标的，如 `SPOT_ACM-USDT` |
    | short_max_ratio | float | 空头最大单标的占比 |
    | short_max_ratio_symbol | str | 空头占比最大标的，可为 `nan` |
    | short_max_ratio_abs | float | 空头最大占比绝对值 |
    | top3_long | str | 多头前三大持仓描述，如 `SPOT_ACM-USDT(0.07%); ...` |
    | top3_short | str | 空头前三大持仓描述，无空头时为 `NO_SHORT` |
    | 净值max2here | float | 截至当前行的净值历史最大值（用于回撤计算） |
    | 净值dd2here | float | 截至当前行的回撤（净值相对 净值max2here 的跌幅，≤0） |
- **与指标计算**：基于《资金曲线》的 `净值`、`净值max2here`、`净值dd2here` 及时间维度可计算整体或分时段的净值、最大回撤、回撤面积、净值回撤比、周度收益 gmean 等，供 §4.2 策略表现多维可视化使用。

### 《选币结果》
- **定义**：回测产出的、按**时间 × 标的**展开的选币明细表，记录每个调仓时刻、每个被选入标的的行情、策略标签、多空方向及目标分配比例等。通常由《彩虹》回测引擎在策略回测结束后写入每个策略（或 SGT）对应的数据目录。
- **数据路径**：依回测/策略落盘约定，例如 SGT 数据目录下的 `选币结果.pkl`，路径形如 `SGTs/{backtest_name}/{strategy_label}/选币结果.pkl`。
- **数据结构**：
  - **物理层**：单文件 `选币结果.pkl`，由 `pd.read_pickle` 读取。
  - **逻辑层**：`pd.DataFrame`，每行对应**一个时间切片下的一个标的**（即 candle_begin_time + symbol 粒度），按时间升序；行数约为「调仓次数 × 当次选中的标的数」量级。
  - **字段层**（列名与含义，参考项目内 `选币结果.pkl` 案例）：
    | 列名 | 类型 | 含义 |
    |------|------|------|
    | candle_begin_time | datetime64[ns] | 蜡烛开始时间（调仓时刻） |
    | symbol | object | 标的代码，如 IOTA-USDT、XRP-USDT |
    | is_spot | int8 | 是否现货，0=合约、1=现货 |
    | close | float64 | 当前收盘价 |
    | next_close | float64 | 下一周期收盘价 |
    | symbol_spot | object | 现货代码 |
    | symbol_swap | object | 合约代码 |
    | 是否交易 | int8 | 是否参与交易（如 1=交易） |
    | strategy | object | 策略标签，如 #0.黄果树t3 -> meanbd0_168_168 |
    | cap_weight | float64 | 资金权重（如 0.5） |
    | 方向 | int8 | 多空方向，如 -1=空、1=多 |
    | offset | int8 | 调仓偏移（如 0,1,…,11） |
    | target_alloc_ratio | float64 | 目标分配比例 |
    | order_first | category | 下单顺序或类型，如 swap |
- **与框架/分析**：供策略分析、选币归因、亏损时段内亏损贡献最大的币种分析等使用；可与《资金曲线》按 `candle_begin_time` 对齐，分析某时段持仓标的与收益/回撤贡献。

### 《币池索引表》
- **定义**：全市场在**时间 × 标的**维度上的有效索引与标记表。每一行表示「某一时刻、某一标的」在币池中有效（有 K 线、可参与因子计算），并带有少量标记字段（是否交易、现货/合约映射、当前/下一根收盘价等）。**不包含完整 K 线**（无 open/high/low/volume 等），**不包含因子值**；主要用于确定因子与 K 线计算的**有效域**及快速查找标的与价格引用。
- **数据路径**：项目缓存目录下的同一逻辑数据、两种物理格式，路径形如 `data/cache/all_factors_kline.pkl` 或 `data/cache/all_factors_kline.parquet`。
- **数据格式**：
  - **`.pkl`**：由 `pd.read_pickle` 读取，兼容性好，读写速度与体积依环境而定。
  - **`.parquet`**：由 `pd.read_parquet` 读取，列式存储，便于按列读取与压缩，通常体积更小、大表下 IO 更友好；依赖 `pyarrow` 或 `fastparquet`。
  - 两种格式的**数据结构（列名、类型、含义）完全一致**，仅物理存储不同；可按需择一使用或同时保留。
- **数据结构**：
  - **物理层**：单文件 `all_factors_kline.pkl` 或 `all_factors_kline.parquet`，对应上述两种格式。
  - **逻辑层**：`pd.DataFrame`，每行对应**一个时间切片下的一个标的**（candle_begin_time + symbol 粒度），按时间升序；行数约为「时间点数 × 该时刻有效标的数」量级。
  - **字段层**（列名与含义，两种格式一致）：
    | 列名 | 类型 | 含义 |
    |------|------|------|
    | candle_begin_time | datetime64[ns] | 蜡烛开始时间 |
    | symbol | category/object | 标的代码，如 1000SHIB-USDT、ADA-USDT |
    | is_spot | int8 | 是否现货，0=合约、1=现货 |
    | close | float64 | 当前收盘价 |
    | next_close | float64 | 下一周期收盘价 |
    | symbol_spot | object | 现货代码 |
    | symbol_swap | object | 合约代码 |
    | 是否交易 | int8 | 是否参与交易（如 1=交易） |
- **与框架/用途**：作为因子计算与 K 线有效范围的索引，供下游按「时间×标的」筛选、对齐或做轻量价格引用；不替代《彩虹预处理数据》中的完整 K 线或因子脚本产出的因子表。

### 《全量K线list》
- **定义**：由《彩虹预处理数据》中的 `spot_dict.pkl`、`swap_dict.pkl` 经过滤与对齐后，按**单币种**拆成的 K 线 DataFrame 列表。每个元素对应一个标的的完整 K 线序列，供因子计算时按币种遍历。
- **数据路径**：`data/cache/all_candle_df_list.pkl`。由 `load_spot_and_swap_data` 在回测启动时生成并落盘，生命周期内覆盖 cache 目录。
- **数据格式**：仅 **`.pkl`**。由 `pd.read_pickle` 读取，得到 `list[pd.DataFrame]`。
- **数据结构**：
  - **物理层**：单文件 `all_candle_df_list.pkl`。
  - **逻辑层**：`list`，长度为参与交易的标的数量（合约 + 现货，经黑白名单、`min_kline_num` 过滤后）；元素顺序为先合约后现货（或反之，取决于 `select_scope_set`、`order_first_set`）。
  - **元素结构**：每个元素为 `pd.DataFrame`，行代表该标的的 1h K 线，列与《彩虹预处理数据》dict_pkl 单币种 DataFrame 一致；经 `align_spot_swap_mapping` 后包含 `symbol_spot`、`symbol_swap` 等映射列。核心列：`candle_begin_time`、`symbol`、`open`、`high`、`low`、`close`、`volume`、`quote_volume`、`funding_fee`、`avg_price_1m`、`avg_price_5m`、`是否交易` 等（参见《彩虹预处理数据》字段层）。
- **与框架/用途**：由 `core.utils.functions.load_spot_and_swap_data` 在回测前生成；`core.select_coin` 中因子计算时通过 `pd.read_pickle` 读取，按 `candle_df_list[i]` 逐币种计算因子，再合并为《币池索引表》。

### 《回测config》
- **定义**：单次回测的**完整配置对象**，对应《彩虹》中的 `core.model.backtest_config.BacktestConfig`。包含回测时间、手续费、杠杆、保证金率、黑白名单、Rebalance 模式、再择时，以及《策略组》解析后的策略列表（《单策略 dict》→ `StrategyConfig`）；回测结束后会随结果一起序列化落盘，便于复现与策略分析时精确还原当次回测参数。
- **数据路径**：回测（或 SGT）结果目录下的 `config.pkl`，路径形如 `backtest_path/{fullname}/config.pkl` 或 SGT 目录 `SGTs/{backtest_name}/{strategy_label}/config.pkl`（回测结果剪切到 SGT 后）。
- **数据格式**：仅 **`.pkl`**。由 `pd.read_pickle` 读取，反序列化后得到 `BacktestConfig` 的**实例**（非 DataFrame）；`BacktestConfig.save()` 会写入 `get_result_folder() / 'config.pkl'`。
- **逻辑层**：`core.model.backtest_config.BacktestConfig` 的实例。读取后可直接访问属性（如 `conf.start_date`、`conf.strategy_list`）或调用方法（如 `conf.get_fullname()`、`conf.get_result_folder()`）。
- **属性层**（从 `BacktestConfig.__init__` 与类定义抓取，供查阅与脚本引用）：
  - **身份与时间**：`name`（str，账户/回测名）、`start_date` / `end_date`（str）、`iter_round`（int|str，遍历轮次，0 表示非遍历）。
  - **账户与交易**：`account_type`、`rebalance_mode`、`initial_usdt`、`leverage`、`margin_rate`、`avg_price_col`；`swap_c_rate` / `spot_c_rate`（合约/现货手续费）、`swap_min_order_limit` / `spot_min_order_limit`。
  - **名单与过滤**：`black_list`、`white_list`（List[str]）、`min_kline_num`（int）。
  - **再择时**：`timing`（Optional[TimingSignal]，None 表示不再择时）。
  - **策略与因子**：`strategy_list`（List[StrategyConfig]）、`strategy_name_list`、`strategy_list_raw`（原始《策略组》）；`factor_params_dict`、`factor_col_name_list`（回测config内所有因子参数组合）、`hold_period_list`、`max_hold_period`、`max_offset_len`；`select_scope_set`、`order_first_set`；`is_day_period` / `is_hour_period`（bool）。
  - **结果**：`report`（Optional[pd.DataFrame]，策略评价结果）；类属性 `data_file_fingerprint`（str）。
- **方法摘要**（常用）：`get_fullname(as_folder_name=False)` 返回完整名称（用于目录名或展示）；`get_result_folder()` 返回本次回测结果目录 `Path`；`load_strategy_config(strategy_list, re_timing_config=None)` 从《策略组》加载并填充 `strategy_list` 等；`save()` 将当前实例写入 `get_result_folder()/config.pkl`；`get_strategy_config_sheet(...)` 导出策略与因子配置表；`set_report(report)` 设置 `report` 并带 fullname。
- **与框架**：由 `BacktestConfig.init_from_config()` 从项目 `config` 模块构建，或由 `BacktestConfigFactory.generate_configs_by_strategies` 等批量生成；回测引擎使用其决定时间范围、手续费、策略列表等；落盘后可供策略分析、复盘、参数表导出等使用。

### 《因子数据》
- **定义**：将仅含因子列的 `factor_*` 与《币池索引表》水平合并后得到的**完整因子表**，包含时间、标的及因子值，供因子分析、可视化等使用。
- **生成方式**：因子文件与《币池索引表》在 `calc_factors` 时使用**相同排序** `['candle_begin_time', 'symbol', 'is_spot']`，行顺序一一对应；通过 `pl.concat(..., how='horizontal')` 或 `pd.concat(..., axis=1)` 水平合并即可得到《因子数据》。
- **数据路径与格式**：因子原始数据与《币池索引表》可能为 **`.pkl`** 或 **`.parquet`** 格式（取决于《彩虹》框架版本）。读取时需**适配两种格式**：优先尝试 `.parquet`，若不存在则回退到 `.pkl`；两种格式的列结构与行顺序约定一致。
  - 裁切版：`all_factors_kline.parquet` / `.pkl` + `factor_{因子列名}.parquet` / `.pkl`
  - 全量版：`all_factors_kline_full.parquet` / `.pkl` + `factor_full_{因子列名}.parquet` / `.pkl`（用于截面因子等需全量时序的场景）
- **数据结构**：
  - **物理层**：由《币池索引表》选取的列（如 `candle_begin_time`、`symbol`、`is_spot`）与因子文件的因子列合并而成（因子文件与《币池索引表》可为 pkl 或 parquet）。
  - **字段层**：至少含 `candle_begin_time`、`symbol`、`is_spot` 及一列因子值（如 `TpNMeanbd0_168`）；可按需从《币池索引表》选取更多列（如 `close`、`next_close`、`是否交易` 等）。
- **行对齐约定**：合并前需校验 `len(币池索引表) == len(因子文件)`，否则说明二者来自不同 `calc_factors` 运行，应报错或跳过合并。
- **与框架/用途**：选币时 `preload_all_factor_data` 将 K 线与多因子列水平合并为全局 LazyFrame；单因子分析、可视化时可按需合并单因子与《币池索引表》得到《因子数据》。

## 3. 框架核心机制

### 《数据喂养机制》
- **K线数据**：框架自动将《彩虹预处理数据》加载为 `df` 传给因子脚本的 `signal` 函数。
- **市值数据**：通过 `core/data_bridge.py` 中的 `load_coin_cap` 加载。因子脚本需通过 `extra_data_dict` 声明需求，框架会自动处理 `candle_begin_time` 对齐及 `ffill` 填充。
- **因子加载**：`core/utils/factor_hub.py` 会根据因子名自动从 `factors/`（时序）或 `sections/`（横截面）目录加载对应的 `.py` 模块。

## 4. 目录规范

### 《项目数据目录规范》
- **因子存储**：`factors/` (存放时序因子脚本)
- **横截面脚本**：`sections/` (存放横截面因子脚本)
- **回测脚本**：根目录下的 `backtest.py`, `backtest_multi_equity.py` 等。

## 5. 蓝图常用逻辑

- **定义**：在多个选项中按**固定顺序**选一个有效值。依次尝试选项（1）（2）…；若当前选项**有数据**则取该值并结束；若**无数据**则尝试下一项；若**全部无数据**则报错。
- **用途**：蓝图中的「二选一（按优先级）」「多选一（按优先级）」等，均按此逻辑执行，便于统一表述与引用。

---
## 维护说明
当系统环境或对象结构发生变化时，应优先更新本节点，以确保所有引用该知识的蓝图保持同步。
