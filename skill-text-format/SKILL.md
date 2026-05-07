---
name: skill-text-format
description: Skill 文档的文本格式规范。当编写或修改 Skill、分解功能模块、需要统一文档中的函数名/文件名/工作流/输出展示格式时使用。
tools: ["Read", "Write"]
model: sonnet
effort: low
---
# Skill 文本格式规范

本 Skill 定义功能模块描述的文本格式。**每次分解一个功能模块，必须按此规范执行**。

## 0. 核心原则（分解功能模块时必遵）

1. **不同信息类型用对应符号包裹**，产生不同颜色，便于可视化区分
2. **函数、目录的层级关系用树形符号展示**
3. **每个 item 必须输出简要说明**
4. **输入、工作流、输出三部分结构完整**
5. **某一话题的核心概念**：只用无序列表，每条以 `**概念名**` 加粗前缀 + 冒号 + 正文；**不用 `###` 在该话题下再分级**；话题总起为顶层 bullet，其下次级概念**缩进两格**为子 bullet（见 §1 示例）

## 1. 信息类型与符号

| 信息类型 | 格式 | 示例 | 用途 |
|----------|------|------|------|
| 函数/方法名 | 单星号 `*` | *init_history* | 斜体，与正文区分 |
| 文件/目录/路径 | 单反引号 `` ` `` | `realtime_data.py`、`data/exginfo` | 行内代码样式 |
| 配置/变量/类名 | 单反引号 `` ` `` | `BmacHandle`、`config.py` | 代码标识符 |
| **话题内核心概念** | 无序列表 + `**概念名**` + 冒号 + 正文；话题下的次级条目前加**两格缩进** | 见下方示例 | 同一话题下不用 `###`；总起一条 + 子条缩进为次级 |

**约定**：`*` 仅用于函数/方法名；若需普通斜体强调，可改用 `_text_`。话题核心概念用 `**概念名**` 作条目前缀加粗，与函数名斜体区分。

**示例（同一话题，不用 `###`）**：首条为话题总起；其下次级概念**缩进两格**再写子 bullet。

```
- **BMAC**：…（话题总起）
  - **全称**：…
  - **定义**：…
  - **项目**：…
```

## 2. 树形符号（层级关系）

函数调用链、目录结构、输出列表等**层级关系**必须用树形符号：

| 符号 | 含义 |
|------|------|
| `├` | 有后续分支 |
| `└` | 最后一个分支 |
| `│` | 延续上一级 |

**示例（工作流）**：
```
  - *init_history*（`realtime_data.py`）：阶段2入口，协调初始化全流程
    ├─ *init_dirs*（`core/utils/file_system.py`）：清空 exginfo/resample/preprocess
    ├─ *run_download_all_history*（`core/history_data/history.py`）：下载 base 5m 历史
    │   ├─ *get_type_symbols*（`core/history_data/utils.py`）：从 Binance exginfo 获取 symbol
    │   └─ *init_history_candles*（`core/history_data/candle.py`）：历史 K 线初始化三阶段
    └─ *resample_history*（`core/resample_candle.py`）：base 5m → 1h
```

**示例（输出）**：
```
- **输出**：落盘至 `data/`
  ├─ `exginfo`：交易所信息、现货/合约映射
  │   ├─ `exginfo_spot.pkl`、`exginfo_swap.pkl`：pd.DataFrame，列 symbol、status、baseAsset、quoteAsset 等（Binance 原始）
  │   └─ `spot_swap_matches.pkl`：pd.DataFrame，列 spot、swap
  ├─ `binance_spot_5m`、`binance_swap_5m`：5m K 线，按 symbol 分目录
  ├─ `funding`：永续资金费率，按 symbol 分文件
  └─ `preprocess_1h_resample`：spot_dict、swap_dict、market_pivot
```

## 3. 功能模块三部分结构

每个功能模块必须包含：

### 3.1 输入

- 列出输入来源（如 `BmacHandle`、配置文件）
- 若有前置条件，一并说明

### 3.2 工作流

- 顶层入口函数 + 文件路径 + 简要说明
- 用树形符号展开子函数/子步骤
- **每个节点**：*函数名*（`文件路径`）：简要说明

### 3.3 输出

- 用树形符号列出输出项
- **每个输出**：`路径`：简要说明（数据内容、结构、用途）
- **按需展开次级**：若目录下含多种文件，用树形符号列出次级文件（`├` `└` `│`）
- **叶节点写格式**：在树形结构的最终端（具体文件级），补充格式与简要描述，如 pd.DataFrame 的核心列名

## 4. 每项必有简介

- 工作流中的函数：冒号后接该步骤做什么
- 输出中的目录/文件：冒号后接数据内容或用途
- 避免仅有名称、无说明的裸项

## 5. 与检索的关系

- **文本搜索**：`*` 和 `` ` `` 为 Markdown 标记，不影响搜索
- **语义检索**：通常忽略格式，仍可索引内容
