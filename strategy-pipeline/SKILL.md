---
name: strategy-pipeline
description: 策略生成器蓝图，用于自动化组装加密货币量化策略列表。
tools: ["Read", "Write"]
model: sonnet
effort: medium
---
# 策略生成器蓝图 (Strategy Pipeline)

本蓝图旨在定义 `strategy_gt.py` 脚本的核心结构，通过 SGT 对象自动化生成适配“彩虹框架”的策略列表。该脚本包含 generator（生成）与 tester（测试）两部分功能。

## 1. 核心定位
SGT 是一个“策略厨师”，它接收策略逻辑、参数规模和执行环境的定义，调用 FactorGenerator（因子生成器，简称 FG）产出必要脚本，并最终组装成可运行的策略字典列表。

## 2. 策略生成器对象变量 (Object Variables)

### SGT 与 dict_SGT_ipts

- **SGT（策略生成器）**：单策略组的生成器，接收一份输入字典，产出一份彩虹策略列表。主要计算逻辑在 SGT 内部（三层：策略逻辑、参数逻辑、执行逻辑）。
- **dict_SGT_ipts**：SGT 初始化时的输入字典。`SGT.__init__(dict_SGT_ipts=None)` 接受该字典，key 为变量名（如 `'select_factor_name'`、`'param1'` 等），value 为对应值。未出现的变量在初始化时设为 `None`。

**SGT 使用示例**：
```python
dict_SGT_ipts = {
    'select_factor_name': 'PctChange',
    'select_factor_direction': True,
    'select_param': 'param1',
    'param1': [10, 20],
    'select_coin_num_overPreFilter': 0.1,
    'hold_hour': 12,
    'execution_mode': 'cta_long',
    'backtest_name': 'MyStrategy',  # 必须用户输入
    # strategy_label 不需要用户输入，在 SGTG 中会自动生成
}
sgt = SGT(dict_SGT_ipts)
strategy_list = sgt.generate()
```

### SGTG 与 dict_SGTG_ipts

- **SGTG（SGT 组）**：一个 **list**，元素为多个 **SGT** 实例，即 `SGTG: list[SGT]`。SGTG 套在 SGT 外层，只负责根据输入生成多个 SGT 并放入该列表，**主要计算逻辑仍在各 SGT 内部**。
- **dict_SGTG_ipts**：SGTG 初始化时的输入变量字典。**整体结构与 `dict_SGT_ipts` 相同**（相同的 key 与变量名），但允许其中**一个或多个** item 的 **value 为 `Expand(...)`**。
  - **Expand**：包装类型，表示「该变量在多个 SGT 之间展开」。其值为可迭代对象（如 list），可遍历；若有多个变量取值为 `Expand`，则对这些 `Expand` 的 `.values` 做**笛卡尔积**，得到多组变量组合。
- **SGTG 生成流程**：
  1. **检查 `backtest_name`**：`backtest_name` 必须由用户输入，不能为 `None` 或空值，且不得使用 `Expand`。若违反则报错并终止。
  2. 扫描 `dict_SGTG_ipts`，找出所有 value 类型为 `Expand` 的 key（不包括 `backtest_name`）。
  3. 对这些 key 对应的 `Expand` 实例取 `.values`，做笛卡尔积，得到若干组「变量组合」。
  4. 每一组变量组合与 `dict_SGTG_ipts` 中**未参与 Expand 的** key-value 合并，得到一份完整的 **dict_SGT_ipts**（其中 Expand 已替换为当次组合中的单值）。
  5. **自动生成 `strategy_label`**：`strategy_label` 不需要用户输入，自动生成。具体为：将当前组合中所有参与 Expand 的 key 在该组合下的 value 按 key 顺序用 `"_"` 拼接成**后缀**；最终 `strategy_label` = `backtest_name` + `" -> "` + 后缀（箭头前后各一空格，便于可视化）。若无任何 key 为 `Expand`，则 `strategy_label` = `backtest_name`。并写回该份 `dict_SGT_ipts`。**连接符 ` -> `**：前缀（`backtest_name`）与后缀（参数组合字符串）之间固定使用 `" -> "`，便于后续流程用 `strategy_label.split(' -> ', 1)` 识别前缀与后缀。**意义**：每个 SGT 的 `strategy_label` 带有参数标记（后缀），互不相同，后续按 `strategy_label` 落盘（如保存到**SGT数据目录**或回测结果目录）时不会互相覆盖；而 `backtest_name` 在所有 SGT 中保持一致，满足 §4.2 遍历回测时五元环境变量一致的要求。
  6. 对每一份 `dict_SGT_ipts` 实例化一个 **SGT**，并依次加入 **SGTG**（即一个 list）。
- 若没有任何 key 的 value 为 `Expand`，则笛卡尔积仅有一组，SGTG 中只包含一个 SGT。

**SGTG 使用示例**：
```python
from strategy_gt import SGT, Expand

dict_SGTG_ipts = {
    'backtest_name': '保3_t8',  # 必须用户输入，不能为 None，不能使用 Expand
    'execution_mode': Expand(['cta_long', 'neutral_long']),
    'select_factor_name': 'QvNMean',
    'select_factor_direction': True,
    'select_param': 'param1',
    'param1': Expand([168, 240]),
    'param2': [],
    'select_coin_num_overPreFilter': 0.1,
    'hold_hour': 12,
    'offset_list': 'default',
    'is_use_spot': False,
    # strategy_label 不需要用户输入，会自动生成：保3_t8 -> cta_long_168, 保3_t8 -> cta_long_240, ...
}
sgtg = SGTG(dict_SGTG_ipts)   # 得到 4 个 SGT：2×2 笛卡尔积
for sgt in sgtg:
    strategy_list = sgt.generate()
```

以下列出所有三层输入变量（`self.xxx`）及其含义：

### 2.1 策略生成第一步：策略逻辑 (Strategy Logic) - "选什么 & 怎么滤"
定义策略的核心灵魂，用于生成基础策略模板。本层仅约定选币/前滤/后滤中**参数位**的引用方式（`param1`、`param2`、`param1param2`）；参数的具体取值与展开逻辑见第二层**参数配置**。

- **13 位串联密码位名**（选币用前缀 `select_`，前滤用 `pre_filter_`，后滤用 `post_filter_`）：`s1_meta`, `s2_trans`, `s3_ts_param`, `s4_ts_op`, `s5_ts_param`, `s6_ts_op`, `s7_ts_param`, `s8_ts_op`, `s9_xs_filter_factor`, `s10_xs_filter_range`, `s11_xs_op`, `s12_xs_ts_param`, `s13_xs_ts_op`。

- **选币逻辑**（组装后产出策略的《选币元组》，即 `factor_list` 中的选币因子项）：
  - **选币因子**（二选一）：`self.select_factor_name`（存量因子名，不依赖 FactorGenerator）或 13 位密码（见上，SGT 变量为 `self.select_` + 各位名；按位组装后调 FG 得 factor_name）。
  - **选币方向**：
    - `self.select_factor_direction`: **目标选币方向**（与做多/做空无关）。True 表示目标选「因子值小的」那一侧，False 表示目标选「因子值大的」那一侧。做空且要做空因子值小的，则填 True；第三层 cta_short 时 SGT 会对《选币元组》位 1 取反后再交给框架，以配合框架空头选「复合列大」的倒置逻辑。
  - **选币参数引用**（三选一）：`self.select_param` 取 `'param1'`、`'param2'` 或 `'param1param2'`，表示选币因子参数位**引用** `self.param1`、`self.param2` 或 `(self.param1, self.param2)`，不直接填具体数值。
  - **选币数量**（二选一，按所选模式填入策略的 `select_coin_num`）：选币与过滤的**区间边界统一为左包含**（与 FG 币池前滤及后续过滤一致）。
    - `self.select_coin_num_overPreFilter`：针对**前滤后币池**的目标数量/比例（直接用于选币）。
    - `self.select_coin_num_overall`：针对**完整币池**的目标比例（需根据前滤条件换算为前滤后池的等效比例；仅当独特前滤因子 = 1 个时可自动换算，否则报错）。具体执行逻辑见 §3.1 步骤 5。

- **前滤逻辑**（组装后产出策略的《过滤元组》，即前置 `filter_list` 中的过滤项；因子、方向、参数的约定与选币逻辑一致，仅属性名统一带 `pre_filter` 前缀）：
  - **前滤因子**（二选一）：`self.pre_filter_factor_name` 或 13 位密码（见上，SGT 变量为 `self.pre_filter_` + 各位名）。
  - **前滤方向**：`self.pre_filter_factor_direction`（True: 升序, False: 降序）。
  - **前滤参数引用**（三选一）：`self.pre_filter_param` 取 `'param1'`、`'param2'` 或 `'param1param2'`，表示前滤因子参数位**引用** `self.param1`、`self.param2` 或 `(self.param1, self.param2)`，不直接填具体数值。
  - **前滤类型**：`self.pre_filter_type` 取 `'rank'`、`'pct'` 或 `'val'`，表示按**因子排名**、**截面分位**或**因子值**进行前滤（对应《过滤元组》中允许的所有条件字符串类型）。
  - **前滤区间**：`self.pre_filter_range` 为 2 元组 `(lower, upper)`；`tuple[0]` 为下限（满足 **>=**），`tuple[1]` 为上限（满足 **<**），即左闭右开区间（对应《过滤元组》中的范围语义，框架内需转换为单边条件或两条《过滤元组》）。
  - **额外前滤元组**：`self.additional_pre_filters`，列表 `[(factor_name, param, condition, direction), ...]`（可选）。需要执行的前滤《过滤元组》成品，由用户直接给出，无需 SGT 根据变量生成。

- **后滤逻辑**（与前滤逻辑结构一致，仅属性名统一带 `post_filter` 前缀；组装后产出策略的《过滤元组》放入 **后置** `filter_list_post`，即 `long_filter_list_post` / `short_filter_list_post`）：
  - **后滤因子**（二选一）：`self.post_filter_factor_name` 或 13 位密码（见上，SGT 变量为 `self.post_filter_` + 各位名）。
  - **后滤方向**：`self.post_filter_factor_direction`（True: 升序, False: 降序）。
  - **后滤参数引用**（三选一）：`self.post_filter_param` 取 `'param1'`、`'param2'` 或 `'param1param2'`，表示后滤因子参数位**引用** `self.param1`、`self.param2` 或 `(self.param1, self.param2)`，不直接填具体数值。
  - **后滤类型**：`self.post_filter_type` 取 `'rank'`、`'pct'` 或 `'val'`，表示按**因子排名**、**截面分位**或**因子值**进行后滤（对应《过滤元组》中允许的所有条件字符串类型）。
  - **后滤区间**：`self.post_filter_range` 为 2 元组 `(lower, upper)`；`tuple[0]` 为下限（**>=**），`tuple[1]` 为上限（**<**），即左闭右开区间（对应《过滤元组》中的范围语义，框架内需转换为单边条件或两条《过滤元组》）。
  - **额外后滤元组**：`self.additional_post_filters`，列表 `[(factor_name, param, condition, direction), ...]`（可选）。需要执行的后滤《过滤元组》成品，由用户直接给出，无需 SGT 根据变量生成；同样写入策略的 `filter_list_post`。

### 2.2 策略生成第二步：参数逻辑 (Parameter Logic) - "测多少 & 怎么分"
定义策略的扩展规模和资金分配。

- **参数配置**（SGT 实例级参数；选币/前滤/后滤中的「选币参数」「前滤参数」「后滤参数」会引用此处）：
  - `self.param1`：单参数取值，类型为 `int`、`list[int]`、`None` 或空列表 `[]`。
  - `self.param2`：单参数取值，类型为 `int`、`list[int]`、`None` 或空列表 `[]`。与 `param1` 组合时对应《选币元组》/《过滤元组》中的参数位。
  - **空列表处理**：空列表 `[]` 被视为"不使用该参数"，在标准化时转换为 `[None]`，确保参数展开时至少生成一个策略（参数为 `None`）。

- **取值与展开逻辑**（根据 `param1`、`param2` 的输入数据类型处理；执行时需**遍历第一层生成的半成品《选币元组》/《过滤元组》**，在每一个条件 tuple 上执行对应的参数配置）：
  - **标准化规则**：
    - `int` → `[int]`
    - `None` → `[None]`
    - `list[int]`（非空）→ 照搬
    - `[]`（空列表）→ `[None]`（视为不使用该参数）
  - **全为 int**：若配置为 `param1` 或 `param2` 或 `param1param2`，且对应取值为 int（或双参数时两个均为 int），则在每个半成品条件 tuple 中把**参数位**直接设为该 int 或 `(param1, param2)`。
  - **含 list**：若 `param1` 或 `param2` 任一为 list（或空列表，已标准化为 `[None]`），或 `param1param2` 对应的一侧/两侧为 list，则先将标准化后的列表做**笛卡尔积**展开，得到多组单参或 `(p1, p2)` 组合；将当前半成品选币/过滤条件**复制**为与组合数相同的多份，在每一份中**遍历其中的每个条件 tuple**，把参数位设为该组合中的单个参数或参数对。即：先按参数组合展开，再在每条策略的每个《选币元组》/《过滤元组》tuple 中填入对应参数。

- **资金分配相关**（产出《策略组》时的资金约定，参见《crypto-knowledge-node》中《策略组》）：
  - 《策略组》的**整体 cap_weight 永远为 1**；框架加载时会按组内所有单策略的 `cap_weight` 之和做归一化。
  - 若策略组中存在**带有不同参数的多个单策略**（由上述参数展开逻辑得到），则资金做**等权分配**：每个单策略的 `cap_weight` 均设为 1，由框架自动归一化为 1/N（N 为单策略数量）。
  - 若需自定义权重，可为各单策略设置不同的 `cap_weight` 数值，框架同样会按总和归一化。

### 2.3 策略生成第三步：策略执行 (Strategy Execution) - "何时做 & 如何做"
定义策略的运行环境和对冲模式。

- **持仓逻辑**：
  - `self.hold_hour`: 持仓小时数（int，如 `12`、`24`）。生成单策略 dict 时转换为字符串格式 `f"{hold_hour}H"` 写入 `hold_period` 字段。
  - `self.offset_list`: offset 列表。单策略 dict 中键名为 `offset_list`。
- **币池**：
  - `self.is_use_spot`: 是否使用现货（True/False）。单策略 dict 中键名为 `is_use_spot`。
- **多空模式相关**：
  - `self.execution_mode`: 执行模式（'neutral_long', 'neutral_short', 'cta_long', 'cta_short'）。
  - 杠杆不在策略中设置，另行约定。
- **策略组标识**：
  - `self.backtest_name`: 回测名称（str），**必须由用户输入**，不能为 `None`。在 SGTG 运行过程中保持不变，用于作为回测的主要标识。
  - `self.strategy_label`: 策略组标签字符串，**不需要用户输入**，在 SGTG 中自动生成（`backtest_name` + `" -> "` + 参数组合后缀）；前缀与后缀之间固定用 `" -> "`（箭头前后各一空格，便于可视化），便于解析。

### 2.4 策略组回测环境变量
定义用于 §4 测试流程的回测环境配置。

- `self.start_date`: 回测开始时间（str，如 `'2022-01-01'`），用户输入，默认 `'2022-01-01'`。
- `self.end_date`: 回测结束时间（str，如 `'2026-02-07 23:00:00'`），用户输入，无默认值。
- `self.backtest_name`: 回测名称（str），**必须由用户输入**，不能为 `None`。在 SGTG 运行过程中保持不变，用于作为回测的主要标识。
- `self.strategy_label`: 策略组标签字符串，**不需要用户输入**，在 SGTG 中自动生成（`backtest_name` + `" -> "` + 参数组合后缀）；前缀与后缀之间固定用 `" -> "`（箭头前后各一空格，便于可视化），便于解析。
- `self.strategy_list`: 策略组（`list[dict]`），照搬流程生成的最终策略组（§3.3 的输出）。
- `self.min_kline_num`: 最小 K 线数量（int），用户输入，默认 `168`。
- `self.initial_usdt`: 初始资金（int/float），用户输入，默认 `2_0000`。
- `self.result_folder`: 回测结果文件夹路径（`Path` 对象或 `None`），在 `SGTG.iter_rainbow_backtest()` 执行后自动设置，对应该 SGT 的回测结果路径。

**SGT数据目录（概念）**：指 `SGTs/{self.backtest_name}/{self.strategy_label}` 目录（双层结构），用于存放该 SGT 对应的各种数据文件，包括：
- `strategy_list.py`：策略组配置文件（由 `generate()` 方法自动生成）
- 回测结果数据：在 `iter_rainbow_backtest()` 执行后，会将回测结果数据剪切到此目录
- 其他与该 SGT 相关的数据文件

在蓝图中，当提到"**SGT数据目录**"或"**数据目录**"时，均指代 `SGTs/{self.backtest_name}/{self.strategy_label}` 这个文件夹路径。

## 3. 在 SGT 中生成彩虹策略组

### 3.0 输入与输出契约（总览）
- **输入**：通过 `SGT(dict_SGT_ipts)` 传入输入字典 `dict_SGT_ipts`，其中包含部分或全部 SGT 输入变量（覆盖 §2.1、§2.2、§2.3 的每一个项目）。字典的 key 为变量名，value 为对应的值；未在字典中出现的变量在初始化时设为 `None`。
  - **§2.1 策略逻辑输入**：选币逻辑（因子名或13位密码、方向、参数引用、选币数量）、前滤逻辑（因子名或13位密码、方向、参数引用、前滤类型、前滤区间、额外前滤元组）、后滤逻辑（因子名或13位密码、方向、参数引用、后滤类型、后滤区间、额外后滤元组）。
  - **§2.2 参数逻辑输入**：`param1`、`param2`，及其 int/list 组合与展开规则；资金分配规则（`cap_weight` 约定）。
  - **§2.3 策略执行输入**：`hold_hour`、`offset_list`、`is_use_spot`、`execution_mode`、`strategy_label`。
- **输出**：彩虹可直接加载的**策略组** `strategy_list: list[dict]`。
  - 每个单策略 dict 至少包含：`strategy`、`cap_weight`。
  - 并包含《彩虹》`StrategyConfig.init(**stg_dict)` 所需键：如 `factor_list`（本蓝图统一使用 `factor_list`，不传 `long_factor_list`/`short_factor_list`，框架会自动复制到两侧）、`filter_list`、`filter_list_post`、`long_select_coin_num`、`short_select_coin_num`、`hold_period`（由 `hold_hour` 转换）、`offset_list`、`is_use_spot`、`long_cap_weight`、`short_cap_weight`（以及需要时的 `select_inclusive`）。

### 3.1 第一层：策略逻辑执行流（对应 §2.1）
**目标**：从零开始组装**一个单策略 dict**。执行过程中该单策略 dict 不一定具备完整字段，但最终会基于此**最终单策略 dict** 生成具备完整功能的《策略组》，用于彩虹回测。

- **执行前滤逻辑**（整体可选：若前滤因子名解析为 None 且无额外前滤元组，则 `filter_list=[]`）
  1. **额外前滤元组**：将 **`self.additional_pre_filters`** 赋值到半成品《单策略 dict》的 `filter_list`（若为 `None` 或空列表则 `filter_list` 初始为 `[]`）。
  2. **前滤因子名**：按《次序优先多选一》执行：（1）**`self.pre_filter_factor_name`**（有则取为前滤因子名）；（2）**13 位密码**（有则用 FG 基于 `pre_filter_s1_meta` … `pre_filter_s13_xs_ts_op` 生成因子，取 FG 返回的因子名）。**若均无（均为 None），则跳过下述步骤 3~7，前滤仅包含步骤 1 的额外前滤元组。**
  3. **前滤方向**：将输入变量 `self.pre_filter_factor_direction` 照搬。
  4. **前滤参数引用**：`self.pre_filter_param` 照搬。
  5. **前滤类型**：`self.pre_filter_type` 照搬。
  6. **前滤区间**：取输入 **`self.pre_filter_range`**（如 `(lower, upper)`）。基于该区间**默认生成两个条件**：**≥ 下限**、**< 上限**（左闭右开）；单边或无区间时按约定退化为单条或零条条件。
  7. **初始化《过滤元组》（前置）**：引用**前滤因子名**（位 0）、**前滤参数引用**（位 1，占位）、**前滤类型**与**前滤区间**（共同生成位 2）、**前滤方向**（位 3），将上一步的**两个条件**（≥ 下限、< 上限）分别落成两条半成品《过滤元组》的**位 2**，生成半成品前置《过滤元组》，**追加**到 `filter_list`（与步骤 1 的额外元组合并）。
     **位 2（条件字符串）生成规则**：格式为 `类型:比较符+数值`，无空格；框架仅支持单边条件，区间须拆成两条。**类型**由 **前滤类型** 照搬（`rank` / `pct` / `val`）；**比较符+数值**由 **前滤区间** `(lower, upper)` 得到两条：  
     - 第一条（≥ 下限）：位 2 = `{前滤类型}:>={lower}`  
     - 第二条（< 上限）：位 2 = `{前滤类型}:<{upper}`  
     **案例**：`pre_filter_type='pct'`，`pre_filter_range=(0.2, 0.8)` → 两条《过滤元组》的位 2 分别为 `pct:>=0.2`、`pct:<0.8`；若为 `pre_filter_type='rank'`、`pre_filter_range=(10, 90)` → 位 2 分别为 `rank:>=10`、`rank:<90`。

- **执行选币逻辑**
  1. **选币因子名**：按《次序优先多选一》执行：（1）**`self.select_factor_name`**（有则取为选币因子名）；（2）**13 位密码**（有则用 FG 基于 `select_s1_meta` … `select_s13_xs_ts_op` 生成因子，取 FG 返回的因子名）；均无则报错。
  2. **选币方向**：将输入变量 `self.select_factor_direction` 照搬写入半成品《单策略 dict》（或后续统一写入，依实现顺序而定）。
  3. **选币参数引用**：`self.select_param` 照搬。
  4. **初始化《选币元组》**：引用**选币因子名**、**选币方向**、**选币参数引用**，生成半成品《选币元组》（参数位为占位，权重 1），放入半成品《单策略 dict》的 `factor_list`。
  <!-- 在生成所有选币过滤元组之后，**仅**在 SGT 级别对 `self.param1`/`self.param2` 做笛卡尔积，生成对应个数的子策略 dict 做参数覆盖；**不**在各《选币元组》/《过滤元组》上再做多维度笛卡尔积（否则会按元组维度爆炸）。因此流程为：先在本层完成所有元组的参数位占位 → 待所有元组就绪后，再遍历 param1/param2 的笛卡尔积，让每个子策略 dict 内的**所有元组**统一跟随当前 (p1, p2) 填入参数位。 -->
  5. **选币数量**：按《次序优先多选一》执行：
     - （1）**`select_coin_num_overPreFilter`**（有则直接取该值，语义为前滤后币池中的目标数量/比例）。
     - （2）**`select_coin_num_overall`**（无 overPreFilter 时用此项，语义为完整币池中的目标比例）。使用时需根据前滤条件换算为前滤后比例：
       - **换算逻辑**：扫描 `filter_list` 中所有前滤《过滤元组》，统计其中**独特因子名**个数。若只有 **1 个独特前滤因子**（即该因子对应一对区间条件，如 `>=0.1` 和 `<0.6`），则读取该因子的前滤范围（`upper - lower`，如 `0.6 - 0.1 = 0.5`），然后将 `select_coin_num_overall` 除以该前滤范围，得到前滤后币池中的等效选币比例。例如 `select_coin_num_overall=0.1`、前滤范围 `0.5` → 前滤后选币比例 = `0.1 / 0.5 = 0.2`。
       - 若独特前滤因子 **不为 1 个**（= 0 个或多个），则无法自动换算，应报错。**注意**：此处扫描包含 `additional_pre_filters` 在内的所有 `filter_list` 条目；若额外前滤元组引入了不同因子名，独特因子名将 > 1，此时应使用 `select_coin_num_overPreFilter` 代替。
     - 将得到的单值写入单策略 dict 的 **`long_select_coin_num`**。左包含：设置单策略 dict 的 **`select_inclusive`**（如 `'left'`）。
  <!-- （若为**空头策略**，在第三层再做 `short_select_coin_num` 的转换） -->

- **执行后滤逻辑**（整体可选，结构与前滤一致；变量名统一带 `post_filter` 前缀，生成的后滤元组放入 `filter_list_post`）
  1. **额外后滤元组**：将 **`self.additional_post_filters`** 赋值到半成品《单策略 dict》的 `filter_list_post`（若为 `None` 或空列表则 `filter_list_post` 初始为 `[]`）。
  2. **后滤因子名**：按《次序优先多选一》执行：（1）**`self.post_filter_factor_name`**（有则取为后滤因子名）；（2）**13 位密码**（有则用 FG 基于 `post_filter_s1_meta` … `post_filter_s13_xs_ts_op` 生成因子，取 FG 返回的因子名）。**若均无（均为 None），则跳过下述步骤 3~7，后滤仅包含步骤 1 的额外后滤元组。**
  3. **后滤方向**：`self.post_filter_factor_direction` 照搬。
  4. **后滤参数引用**：`self.post_filter_param` 照搬。
  5. **后滤类型**：`self.post_filter_type` 照搬。
  6. **后滤区间**：取输入 **`self.post_filter_range`**（如 `(lower, upper)`）。基于该区间**默认生成两个条件**：**≥ 下限**、**< 上限**（左闭右开）；单边或无区间时按约定退化为单条或零条条件。
  7. **初始化《过滤元组》（后置）**：引用**后滤因子名**（位 0）、**后滤参数引用**（位 1，占位）、**后滤类型**与**后滤区间**（共同生成位 2）、**后滤方向**（位 3），将上一步的**两个条件**（≥ 下限、< 上限）分别落成两条半成品《过滤元组》的**位 2**，生成半成品后置《过滤元组》，**追加**到 `filter_list_post`（与步骤 1 的额外元组合并）。**位 2 生成规则**：`{后滤类型}:>={lower}`、`{后滤类型}:<{upper}`（格式同前滤）。

### 3.2 第二层：参数逻辑执行流（对应 §2.2）
**目标**：基于输入的 `param1`、`param2`，生成二元参数组合列表 `[(p1, p2), ...]`。遍历该列表，为每个组合生成一个单策略 dict，更新其中所有《选币元组》/《过滤元组》的参数位，并将参数信息作为策略名后缀。

**示例**：`param1=[1,2]`，`param2=None`，某个元组的参数引用为 `'param1'` → 生成二元参数列表 `[(1, None), (2, None)]` → 生成 2 个单策略 dict，该元组的参数位分别替换为 `1`、`2`，策略名后缀分别为 `_1`、`_2`（None 不显示在策略名中）。

1. **生成二元参数组合列表**
   - 读取 `self.param1`、`self.param2`（类型为 `int`、`list[int]`、`None` 或空列表 `[]`）。
   - **标准化为列表**：
     - `int` → `[int]`
     - `None` → `[None]`
     - `list[int]`（非空）→ 照搬
     - `[]`（空列表）→ `[None]`（视为不使用该参数，确保至少生成一个策略）
   - **生成笛卡尔积**：`param_combinations = list(product(param1_list, param2_list))`，得到 `[(p1, p2), ...]`。
   - **示例**：
     - `param1=10, param2=None` → `[(10, None)]`
     - `param1=[1,2], param2=None` → `[(1, None), (2, None)]`
     - `param1=[1,2], param2=[10,20]` → `[(1,10), (1,20), (2,10), (2,20)]`
     - `param1=[168], param2=[]` → `[(168, None)]`（空列表转换为 `[None]`）

2. **遍历参数组合，生成单策略 dict 列表**
   - 对每个 `(p1, p2)` 组合：
     a. **深拷贝基础单策略 dict**（来自第一层输出）。
     b. **遍历所有元组并替换参数位**：
        - 遍历该 dict 中的所有《选币元组》（`factor_list`）和《过滤元组》（`filter_list`、`filter_list_post`）。
        - 检查每个元组的**参数引用**（位 1 或位 2，取决于元组类型）：
          - 若为 `'param1'`：将参数位替换为 `p1`（忽略 `p2`）。
          - 若为 `'param2'`：将参数位替换为 `p2`（忽略 `p1`）。
          - 若为 `'param1param2'`：将参数位替换为 `(p1, p2)`（双参数元组）。
        - **注意**：若参数引用为 `'param1'` 但 `p1` 为 `None`，或引用为 `'param2'` 但 `p2` 为 `None`，或引用为 `'param1param2'` 但 `p1` 或 `p2` 任一为 `None`，均应报错（参数引用存在但对应参数未提供）。
     c. **生成策略名后缀**：
        - **按 p1、p2 固定顺序拼接**：后缀格式为 `f"_{p1}_{p2}"`，其中 `None` 值不显示（跳过）。始终按 param1 在前、param2 在后的顺序，**不做去重、不做排序**，确保不同参数组合不会产生相同后缀。
        - **更新策略名**：读取 `self.strategy_label`，将 `strategy` 字段更新为 `f"{strategy_label}{suffix}"`（若无 `strategy_label`，可用默认前缀如 `"策略"`）。此为最终策略名（前缀 + 参数后缀），后续层不再修改。
        - **示例**：
          - `(p1=10, p2=None)` → 后缀 `"_10"`。
          - `(p1=None, p2=20)` → 后缀 `"_20"`。
          - `(p1=10, p2=20)` → 后缀 `"_10_20"`。
          - `(p1=20, p2=10)` → 后缀 `"_20_10"`（与上条不同，避免碰撞）。
     d. **设置资金权重**：按 §2.2 规则，默认 `cap_weight=1`（等权，由彩虹归一化）。
   - 最终得到 `strategy_dict_list`，长度为参数组合数。

3. **输出半成品单策略 dict 列表**
   - 返回半成品单策略 dict 列表，每个 dict 包含 `factor_list`、`filter_list`、`filter_list_post`、`long_select_coin_num`、`strategy`（已含前缀+参数后缀）、`cap_weight` 等字段。仍需 §3.3 补齐 `hold_period`、`offset_list`、`is_use_spot`、`long_cap_weight`/`short_cap_weight` 等执行字段后方为完整《策略组》。

### 3.3 第三层：策略执行与最终组装（对应 §2.3）
**目标**：补齐所有《单策略 dict》的必须字段，并按 `execution_mode` 执行多空模式与（可选）全市场对冲策略。

1. **补齐持仓逻辑字段**
   - **`hold_period`**：将 `self.hold_hour`（int）转为字符串 `f"{hold_hour}H"` 写入每个《单策略 dict》的 `hold_period`（如 `12` → `'12H'`）。
   - **`offset_list`**：将 `self.offset_list` 照搬写入 `offset_list`；若输入为 `'default'`，则赋值为 `list(range(0, self.hold_hour, 1))`。

2. **补齐币池字段**
   - **`is_use_spot`**：将 `self.is_use_spot` 照搬写入每个《单策略 dict》的 `is_use_spot`。

3. **按 execution_mode 补齐多空模式字段**
   - **`cta_long`**：在所有《单策略 dict》中增加 `'long_cap_weight': 1`、`'short_cap_weight': 0`、`'short_select_coin_num': 0`。
   - **`cta_short`**：在所有《单策略 dict》中增加 `'long_cap_weight': 0`、`'short_cap_weight': 1`；将 `long_select_coin_num` 的值赋给 `short_select_coin_num`，然后将 `long_select_coin_num` 设为 `0`；并将所有《选币元组》的**位 1（是否升序）**取反（`True` ↔ `False`）。**说明**：`select_factor_direction` 表示**目标选币方向**（True=选小的，False=选大的），与多空无关。第一层生成《选币元组》时位 1 按目标方向填入；彩虹框架空头对复合列 `ascending=False`（选**大**），故 cta_short 时须对位 1 取反，使「复合列大 = 目标小」——这样框架空头选大即选到目标方向（小）。例如做空且要做空因子值小的，SGT 填 `select_factor_direction=True`，第三层取反后位 1=False 填入选币元组，框架再对空头选大，即选到小的。
   - **`neutral_long`**：先按 `cta_long` 配置；再**追加**一个「**空**合约全市场」《单策略 dict》到《策略组》作为空头对冲（主策略做多 → 需空头对冲），其 `cap_weight` = 当前子策略个数（= 当前所有子策略 `cap_weight` 之和）。其余字段见下「全市场单策略 dict 模板」。
   - **`neutral_short`**：先按 `cta_short` 配置；再**追加**一个「**多**合约全市场」《单策略 dict》到《策略组》作为多头对冲（主策略做空 → 需多头对冲），其 `cap_weight` = 当前子策略个数。其余字段见下「全市场单策略 dict 模板」。

   **全市场单策略 dict 模板**（供 `neutral_long` / `neutral_short` 追加时使用）：
   - **多合约全市场**：`strategy` 固定名、`offset_list=[0]`、**`hold_period='1H'`**（固定 1H，保证每个时间戳上都有完整币池与收益数据，便于策略分析中的全市场基准计算）、`long_cap_weight=1`、`short_cap_weight=0`、`long_select_coin_num=999`、`short_select_coin_num=0`、`factor_list=[('QvNMean', True, 3, 1)]`（《选币元组》格式：因子名、是否升序、参数、权重；使用已有因子 `QvNMean` 用于全市场选币）、`filter_list`/`filter_list_post=[]`、`is_use_spot=False`、`use_custom_func=False`。
   - **空合约全市场**：同上，但 `long_cap_weight=0`、`short_cap_weight=1`、`long_select_coin_num=0`、`short_select_coin_num=999`，`factor_list=[('QvNMean', True, 3, 1)]`，`strategy` 固定名（如「空合约全市场」）。

4. **输出与校验**
   - 汇总得到最终 `strategy_list: list[dict]`（即彩虹策略组）。
   - **保存到本地文件**：若 `strategy_label` 不为空，将策略组保存到本地文件，路径为**SGT数据目录**下的 `strategy_list.py`。文件内容为可直接 import 的 Python 代码（包含 `strategy_list` 变量）。
   - **保存 SGT 输入逻辑到 MD**：若 `strategy_label` 不为空，将 SGT 输入 dict 的逻辑以文本形式保存到**SGT数据目录**下的 `SGT输入逻辑.md`。工作流：
     - **数据源**：SGT 的输入 dict（即 `dict_SGT_ipts`，可从 SGT 实例属性还原）。
     - **结构**：按以下次序分块：**基础信息** → **选币** → **前滤** → **后滤** → **参数** → **核心逻辑** → **13 位密码** → **拓展**。
     - **可选区块**：若选币/前滤/后滤**无任何因子**（无 `xxx_factor_name`、无 13 位密码、无 `additional_pre_filters`/`additional_post_filters`），则**整块删除**该部分。
     - **13 位密码**：放在**核心逻辑之后**；若存在密码，校验 `xxx_factor_name` 为 `None`，用 FactorGenerator 生成成品因子名并展示；保留原始密码 dict（如 `dict_select_fg13bit = {...}`）。
     - **拓展**：可包含少量拓展说明（如策略组策略数、执行模式等）。
   - 要求该 `strategy_list` 可直接传入 `BacktestConfig.load_strategy_config(strategy_list)`，并满足彩虹对 `strategy`、`cap_weight`、因子与过滤字段的校验。

## 4. 在 SGTG 中测试彩虹策略组

**目标**：SGTG 实例生成多个 SGT 的策略组后，自动运行批量回测，并基于回测结果执行定制化的策略分析，帮助用户快速评估策略表现。

**注意**：回测功能以 SGTG 为主，不再提供单 SGT 的回测方法。

### 4.1. SGTG遍历回测

**目标**：对 SGTG 中所有 SGT 产出的策略组做批量回测，复用现有「参数遍历 + 找最优」流程（`BacktestConfigFactory` + `find_best_params`）。执行前需准备好两类数据：**① 统一回测环境（写入 config）**、**② 各 SGT 的策略列表集合（strategies）**。

**方法签名**：`iter_rainbow_backtest(if_only_calc_factor=False, if_del_select_coin=False)`。`if_only_calc_factor=True` 时仅计算因子、不选币不回测（见 §4.3）；`if_del_select_coin=True` 时剪切到 SGT 数据目录后删除所有非 `config.pkl` 的 pkl 文件以节省磁盘。

**原则**：使用当前遍历需要**唯一的回测环境变量**以及**自由的策略组**。即：回测环境变量（见下，五元：start_date、end_date、min_kline_num、initial_usdt、backtest_name）在 SGTG 内必须一致且不得使用 Expand；策略组则允许各 SGT 不同（由各 SGT 的 generate 自由产出）。注意：`strategy_label` 不参与一致性检查，因为它会被添加参数标记（后缀），各 SGT 的 `strategy_label` 已经不同。
 
**日志规范**：重要信息（开始/结束、关键校验通过/失败、错误堆栈、回测启动/完成等）尽量使用彩虹框架的 `logger`（`from core.utils.log_kit import logger`）输出，避免使用 `print`。

**执行步骤**：

1. **生成 SGTG 并产出各 SGT 的策略组**：
   - 使用 `dict_SGTG_ipts` 实例化 `SGTG`，得到 `sgtg: list[SGT]`。
   - 对每个 SGT 调用 `sgt.generate()`，确保 `sgt.strategy_list` 已生成并可访问。

2. **检查并统一回测环境变量（五元一致）**：
   - **五元回测环境变量**（与 §2.4 及 config 对齐）：`start_date`、`end_date`、`min_kline_num`、`initial_usdt`、`backtest_name`。**五个变量均要求一致**：在 `dict_SGTG_ipts` 中不得对其中任一使用 `Expand`（`backtest_name` 在 SGTG 初始化时已检查），且遍历 SGTG 时每个 SGT 的这五元取值必须完全相同；若任一两两不同则**报错并终止**，并提示哪些变量、哪些 SGT 不一致。
   - **注意**：`strategy_label` 不参与一致性检查，因为在 SGTG 初始化时（见 §2 SGTG 生成流程步骤 5），`strategy_label` 会自动生成（`backtest_name` + `" -> "` + 参数组合后缀），各 SGT 的 `strategy_label` 已经不同；而 `backtest_name` 必须由用户输入且保持不变，所有 SGT 的 `backtest_name` 应该已经一致。因此，遍历回测时依赖 `backtest_name` 即可，无需检查 `strategy_label`。
   - **backtest_name 为 None 时**：`backtest_name` 必须由用户输入，不能为 `None`（在 SGTG 初始化时已检查）。若在遍历回测时发现 `backtest_name` 仍为 `None` 或空值，则直接报错并提示用户提供 `backtest_name`。
   - 若五元一致，取其中任一套（如首个 SGT 的取值）作为**统一回测环境**。

3. **将统一环境写入 config**：
   - 将上一步得到的统一环境变量映射到 `config` 模块：
     - `start_date` → `config.start_date`
     - `end_date` → `config.end_date`
     - `backtest_name` → `config.backtest_name`（必须由用户输入，不能为 `None`）
     - `min_kline_num` → `config.min_kline_num`
     - `initial_usdt` → `config.initial_usdt`
   - 通过写 `config.py` 文件并 **reload(config)**，或直接设置 `config` 模块属性，使后续 `BacktestConfig.init_from_config(load_strategy_list=False)` 读到的均为该套环境。
   - **注意**：`config.strategy_list` 在本流程中**不**在此处写入，策略列表由步骤 5 通过 `generate_configs_by_strategies` 传入。

4. **收集 strategies**：
   - `strategies = [sgt.strategy_list for sgt in sgtg]`。
   - 类型为 `list[list[dict]]`，与现有 `BacktestConfigFactory.generate_configs_by_strategies(strategies)` 的入参一致。

5. **套用现有遍历流程**：
   - 创建 `BacktestConfigFactory(backtest_name=...)`（`backtest_name` 与步骤 3 写入 config 的取值一致或按项目约定）。
   - 调用 `factory.generate_configs_by_strategies(strategies=strategies)`，为每个策略组生成一份 `BacktestConfig`（环境来自 config，策略列表来自 `strategies` 的对应项）。
   - 调用 `find_best_params(factory)`（或项目内等效入口）执行批量回测并得到最优参数/结果。
   - **设置每个 SGT 的回测结果路径并剪切回测数据**：回测执行完成后，`factory.config_list` 与 SGTG 中的每个 SGT 一一对应：`factory.config_list[i]` 对应 `sgtg[i]`（第 i 个 SGT）。遍历 `factory.config_list` 和 `sgtg`，为每个 SGT 执行以下操作：
     - 获取回测结果路径：调用 `config.get_result_folder()` 获取该 SGT 的回测结果文件夹路径。
     - 设置 `result_folder` 属性：将路径赋值给 `sgt.result_folder`，便于后续访问。
     - **剪切回测数据到 SGT数据目录**：将回测结果文件夹下的所有内容（文件和子文件夹）**剪切**（移动）到该 SGT 的**SGT数据目录**下（`SGTs/{backtest_name}/{strategy_label}`）。这样回测结果数据会统一存放在每个 SGT 对应的数据目录中，便于管理和查看。如果目标目录中已存在同名文件/文件夹，则先删除再移动。
     - **可选：删除选币 pkl 文件以节省磁盘**：`iter_rainbow_backtest(if_del_select_coin=True)` 时，剪切完成后，在 SGT 数据目录内递归删除所有**非 `config.pkl`** 的 `.pkl` 文件（即选币结果等大文件），仅保留 `config.pkl`。默认 `if_del_select_coin=False`，不删除。`strategy_gt_batch` 中建议配置为 `True`，因遍历因子生成时选币文件占用磁盘较多。
     ```python
     for sgt, config in zip(sgtg, factory.config_list):
         result_folder_path = config.get_result_folder()
         sgt.result_folder = result_folder_path
         # 剪切回测结果数据到 SGT数据目录（SGTs/{backtest_name}/{strategy_label}）
         # 将 result_folder_path 下的所有内容移动到 SGT数据目录
         # 若 if_del_select_coin=True，则删除 sgt_dir 下所有非 config.pkl 的 pkl 文件
     ```
     这样每个 SGT 对象都可以通过 `sgt.result_folder` 直接访问其回测结果路径，且回测数据已统一存放在**SGT数据目录**下。

6. **顺序约束**：
   - 必须先对每个 SGT 执行 `generate()`，再收集 `strategy_list`，再执行环境检查与写 config；否则 `sgt.strategy_list` 可能为空或未定义。

**输出**：
- 回测结果文件夹路径（或 `find_best_params` 所返回的结果路径）。
- 回测是否成功完成（成功/失败 + 错误信息）。
- 若步骤 2 五元不一致则报错，不进入步骤 3～5。
- **注意**：回测完成后，每个 SGT 对象的 `result_folder` 属性会被自动设置，可通过 `sgt.result_folder` 直接访问该 SGT 的回测结果路径。

### 4.2. SGTG策略表现多维可视化

**目标**：在 §4.1 彩虹回测完成之后，基于各 SGT 的**SGT数据目录**下的回测结果，计算资金曲线表现指标，并生成以 Expand 变量组合为维度的多维热力图，便于对比不同参数组合在不同时间段、不同指标上的表现。

**前置条件**：已执行 `iter_rainbow_backtest()`，且回测结果已剪切到各 SGT 的**SGT数据目录**（`SGTs/{backtest_name}/{strategy_label}`）。

**执行步骤**：

0. **构建参键实例 dict**  
   **缩略称呼**：Expand 组合（combo）简称**参键**，SGT 实例简称**实例**；「参键 → 实例」的映射简称**参键实例 dict**。后续基于**参键**读取**实例**的信息（如 `strategy_label`、数据目录、`equity_metrics` 等）。  
   建立参键实例 dict。分三种情况：

   - **默认（有 Expand）**：约定 SGTG **必须**提供 **expand_keys**（参与 Expand 的变量名列表）以及基于参键查询对应实例的接口（例如「由参键得到实例」的映射方法）。若当前生成器脚本未提供该接口，则需在生成器中实现并暴露。
   - **需要次序时**：若某流程**显式依赖**「参键的遍历顺序」或「实例在 SGTG 中的下标」，则可基于**默认的笛卡尔积顺序**（Expand 变量名顺序固定 + 对各自取值做笛卡尔积的遍历顺序与 §2 SGTG 生成时一致）得到**有序的参键列表**，与 **SGTG 本体 list**（有序的实例序列）按位一一对应（第 i 个参键 ↔ SGTG[i]）。此方式仅在「必须用次序」时使用，**默认不依赖次序**。
   - **无 expand_keys**：表示未使用任何 Expand，SGTG 中仅有一个实例，直接使用该唯一实例即可，无需建立参键实例 dict。

1. **读取资金曲线并计算指标**  
   遍历 SGTG 中的每个 SGT，执行以下流程：
   
   a. **数据源**：在生成器执行完 `iter_rainbow_backtest()` 之后，针对每个 SGT，读取其**SGT数据目录**下的 `资金曲线.csv` 文件（路径形如 `SGTs/{backtest_name}/{strategy_label}/资金曲线.csv`）。加载为 DataFrame，确保包含《资金曲线》约定的核心列（`candle_begin_time`、`净值`、`净值max2here`、`净值dd2here` 等，见《crypto-knowledge-node》）。
   
   b. **计算指标**：针对**全量**和**每个单独年度**，计算以下五项指标：
      - 净值、最大回撤、回撤面积、净值回撤比、周度收益几何平均。
      - **计算方法**：严格参考《equity-curve-compute》Skill（`~/.cursor/skills/equity-curve-compute/SKILL.md`）中 §3.1（整体指标）与 §3.2（分年度指标）的计算逻辑。
      - **输出形态**：按《equity-curve-compute》§2 约定，将计算结果组织为**一维 dict**，key 为指标名（整体无后缀，年度带 `_YYYY` 后缀，如 `净值`、`净值_2022`、`最大回撤_2023`），value 为数值。
   
   c. **数据保存**：
      - **文件保存**：将上述 dict 转换为**单列 DataFrame**（指标名为索引，一列为数值），以 **CSV 格式**保存到该 SGT 的**SGT数据目录**下，文件名固定为 **`资金曲线指标.csv`**。
      - **实例属性**：将生成的 dict **嵌入到 SGT 实例中**（例如赋值给 `sgt.equity_metrics` 或类似属性名，由实现约定），方便后续步骤（如步骤 2 构建 Expand 组合 × 指标表格、步骤 3 绘制热力图）直接从 SGT 实例读取指标，无需重复读取 CSV。

2. **构建参键 × 实例表现 df**  
   **目的**：便于检查不同参数下的策略表现差异，并为后续多维热力图提供数据。
   
   - **数据源**：当前 SGTG 实例中的**参键实例 dict**，以及其中所有**实例**的 `equity_metrics`（需先执行步骤 1 `compute_equity`）。
   
   - **工作流**：遍历参键实例 dict 中的每一个实例，每个实例对应 df 的一行。
     1. 在 **expand_keys 对应列**下填入参键：每个 `expand_key` 一列（列名为该变量名），填入参键（combo）在该维度的取值，**顺序与 expand_keys 列表一致**。
     2. 在 **strategy_label** 列下填入该实例的 `strategy_label`。
     3. 遍历该实例的 `equity_metrics` 中的每一项，**key 作为列名，value 作为单元格取值**。若某实例缺少某指标 key，则填 `nan`。
   
   - **输出**：将所有实例的参键、strategy_label 及表现指标合并为一个 df，保存到 **`SGTs/{backtest_name}/参键xSGT实例表现df.csv`**。`backtest_name` 取用户输入的 `dict_SGTG_ipts` 中的配置值（即 SGTG 的 `_backtest_name`）。

3. **绘制多维热力图**  
   - **数据源**：步骤 2 产出的**参键 × 实例表现 df**。
   
   - **工具**：使用 **plotly** 的 `make_subplots` 绘制多维热力图。
   
   - **画布（Canvas）**：整体 subplot 网格，由 **x 轴 = 指标**、**y 轴 = 时间段** 划分。
     - **画布 x 轴**：五个表现指标类型，固定顺序为「净值、最大回撤、回撤面积、净值回撤比、周度收益gmean」。画布列数 = 5。
     - **画布 y 轴**：时间段（全量、各年度，如 全量、2022、2023、2024）。画布行数 = 时间段个数。时间段从 df 的指标列名推断：无后缀（如 `净值`）→ 全量；带 `_YYYY`（如 `净值_2022`）→ 该年；去重并排序得到画布 y 轴的时间段列表。
     - **画布格子数**：例如 6 个时间段 × 5 个指标 = **30 个子图**。每个子图对应一个 (时间段, 指标) 组合。
   
   - **子图（每个画布格子内）**：在某个 (时间段, 指标) 下，展示**不同 SGT（参键）**的取值。
     - **子图 xy 轴**：由 SGTG 的 **expand_keys** 与 **subplot_xy** 参数共同决定。
     - **subplot_xy**（`plot_heatmap(subplot_xy=...)` 输入）：
       - **expand_keys 个数 = 2**：`'xy'` = x 轴第 1 位、y 轴第 2 位；`'yx'` = x 轴第 2 位、y 轴第 1 位；其他输入**报错**。
       - **expand_keys 个数 = 1**：`'x'` = 仅 x 轴展示参键，y 轴占位；`'y'` = 仅 y 轴展示参键，x 轴占位；其他输入**报错**。
    - **参键排序**（用于子图轴上的参键遍历顺序）：优先按数值排序，无法数值化则按字符串排序。**数值**（int、float）直接参与比较；**2 位 tuple**（如 `(0.1, 0.3)`）取其两元素的**算术平均**作为排序键；其他情况（字符串、1 位/3+ 位 tuple、bool、非数值 tuple 等）**转为字符串后按字符串排序**。
    - **参键空值（None）处理**：Expand 取值可为 `None`（表示该维度「无该参数」），参键 × 实例表现 df 中对应列存为 NA。热力图中需展示该类参键：取轴参键取值时对 df 的 expand 列做 **fillna('None')** 再 unique()，使空值在轴上显示为字符串 `'None'`；筛行取该参键对应实例时，若当前遍历到的参键值为字符串 `'None'`，用 **pd.isna(df[expand_key])** 匹配，否则用 **df[expand_key] == v**。子图尺寸计算（各 expand 选项个数）同样基于 fillna('None').unique() 的个数。
    - **expand_keys 个数 = 2**：子图内每个格子 = 一个参键，颜色 = 该参键对应 SGT 在该 (指标, 时间段) 的取值。
    - **expand_keys 个数 = 1**：每个轴位置 = 一个参键（由 subplot_xy 决定 x 或 y 展示）。
    - **expand_keys 个数 = 0**：仅有一个 SGT，子图内退化为单值展示（如文本或单色块）。
    - **expand_keys 个数 > 2**：不支持，**直接报错**。
   
   - **取值**：画布 (时段_i, 指标_j) 对应子图 (ri, cj)。子图内参键 (v0, v1) 对应 df 中该参键行的列 `指标_j` 在 `时段_i` 下的取值（如 `净值`、`回撤面积_2023`）。
   
   - **可视化偏好**：
     - **子图尺寸**：动态计算。取 **expand_values 中最大个数**（各 expand_key 对应选项个数的最大值）× 60 作为每个子图的边长（像素）。整体 figure 的 height ≈ 时段数 × 子图边长，width ≈ 指标数 × 子图边长（含边距与标题）。
     - **子图格子标注**：每个热力图格子内显示对应数值，保留 2 位小数。若画布列为**最大回撤**、**净值回撤比**或**周度收益gmean**，则数值以百分比格式显示（依旧保留 2 位小数）。
     - **子图轴标题**：子图 x 轴、y 轴分别加标题，标题为对应的 **expand_key** 变量名；子图 x 轴标题的字体大小略小；子图 y 轴标题仅在画布第一列（最左侧）展示。
     - **子图 y 轴刻度标签**：子图 y 轴的刻度标签（ticktext，参键值）仅在画布第一列（最左侧）展示，其他列隐藏。
     - **颜色**：采用**蓝灰红三段式配色法**（参见 plotly-settings skill）：最小值→蓝、基准值→灰、最大值→红，基准由 `smart_zMinMidMax` 计算。
   
   - **输出**：将 `fig` 以 **HTML** 形式保存到 `SGTs/{backtest_name}/参键xSGT实例表现热力图.html`，以 **PNG** 形式保存到 `SGTs/{backtest_name}/参键xSGT实例表现热力图.png`（同名仅后缀不同；PNG 需 kaleido，失败时仅记录 warning），并调用 `fig.show()` 展示。主标题为 `{backtest_name} | 参键×实例表现（画布: 行=时段, 列=指标 | 子图: 参键）`。

**说明**：无 Expand 时 SGTG 仅含一个实例，热力图退化为单个子图，展示该实例的「指标×时间段」矩阵即可。

### 4.3. SGTG额外因子计算

**目标**：在 §4.1 回测与 §4.2 可视化完成之后，基于用户指定的额外因子列表执行一次单 SGT 回测（仅计算因子），将因子数据写入 `data/cache`，供后续 strategy_analysis 等流程按需读取。置于主流程之后执行，避免常规回测覆盖已计算的额外因子。

**输入**：`l_custom_factors`（list[str]），需要计算的额外因子名列表。例如 `['LowPrice', 'PctChange']`。若为空则跳过本流程。

**执行步骤**：

1. **若 `l_custom_factors` 为空**：直接跳过。

2. **构造额外因子 SGTG**：
   - 创建仅包含一个 SGT 的 SGTG。
   - `dict_backtest_env` 与主流程用户输入保持一致。
   - `backtest_name` 固定为 `"额外因子计算"`。
   - 其余项（选币、前滤、后滤、参数等）与主流程 `dict_SGTG_ipts` 保持一致。
   - **唯一差异**：`additional_pre_filters` 由 `l_custom_factors` 生成。

3. **生成 `additional_pre_filters`**：
   - 默认因子均为**单参数**（配置为 `param1` 或 `param2`，不出现 `param1param2`）。
   - 位 0：因子名，来自 `l_custom_factors`。
   - 位 1：param_ref 可选范围，由 `dict_SGTG_ipts` 中的 `param1`、`param2` 判断：若 `param1` 不为 `None` 且不为 `[]`，则录入 `param1`；若 `param2` 不为 `None` 且不为 `[]`，则录入 `param2`。得到 list（如 `['param1']`、`['param2']` 或 `['param1', 'param2']`）。
   - 位 2：过滤条件字符串，使用 `pct:>=0`（或等价「全通过」条件），确保不实际过滤任何币。
   - 位 0 与位 1 做**笛卡尔积**，得到若干元组 `(factor_name, param_ref, condition)`，作为 `additional_pre_filters`。

4. **执行回测**：
   - 对上述 SGTG 调用 `sgt.generate()` 与 `sgtg.iter_rainbow_backtest(if_only_calc_factor=True)`（仅计算因子，不选币、不回测）。
   - 因子数据落盘至 `data/cache`（`factor_{factor_col_name}.pkl`、`factor_full_{factor_col_name}.pkl`、`all_factors_kline.pkl` 等）。

5. **pkl → parquet 转换**（步骤 4 之后执行）：
   - 将 `data/cache` 下的**币池索引表**（`all_factors_kline.pkl`、`all_factors_kline_full.pkl`）与**因子文件**（`factor_*.pkl`、`factor_full_*.pkl`）转为 parquet 格式，供 strategy_analysis 等优先读取。
   - 转换后保留原 pkl 文件（Core 仍使用 pkl）；parquet 为增量副本。
   - **数据整合约定**（参考 core_2）：币池索引表与因子使用**相同排序** `['candle_begin_time', 'symbol', 'is_spot']`，按行位置 horizontal concat，不依赖 index。转换时：币池索引表排序后 `to_parquet(index=False)`；因子按对应 kline 排序后的 index 重排后 `to_parquet(index=False)`。strategy_analysis 读取 parquet 时按行位置合并（`.values` 赋值）。

6. **结束**：本流程不参与统计与可视化，仅依赖 cache 中留存的数据。

**输出**：无显式输出；`data/cache` 中新增或更新对应因子的 pkl 文件，及 parquet 副本。

## 5. 命名规范
- **Strategy Name**（《单策略 dict》中的 `strategy` 字段）：普通策略由 §3.2 一次性生成完整名称 `{strategy_label}{param_suffix}`（`param_suffix` 由第二层根据参数组合生成，如 `_10`、`_10_20`，None 不显示），后续层不再修改。多合约全市场、空合约全市场使用固定策略名，不使用 `strategy_label` 前缀。
