---
name: frequently-used-tools
description: coin_backtest 仓库内置常用小工具的索引：Markdown 表格转 CSV（md_table_to_csv.py）等。 在用户需要将 md 任务表导出 Excel、批量转换表格或提及「md 转 csv / 项目脚本」时使用。
tools: ["Read", "Bash"]
model: sonnet
effort: low
---
# 常用小工具（frequently-used-tools）

本 Skill 记录本机/仓库中**可重复执行**的固定脚本；每新增一个工具，在此增加**一段**说明（路径、做什么、怎么跑）。当前收录如下。

## *md_table_to_csv*

在 `coin_backtest` 根目录下的 `md_table_to_csv.py` 将 Markdown 的 `|` 表导出为 **UTF-8 BOM** CSV。脚本**顶部**有两处可改配置：`DEFAULT_INPUT_MD`（默认要读的 md 文件，相对本脚本目录或写绝对路径）、`DEFAULT_OUTPUT_DIR`（无参数运行时的 CSV 输出目录）；另有 `DEFAULT_STRIP_MARKDOWN` 控制无参时是否默认去掉 `**` 与反引号。直接执行 `python3 md_table_to_csv.py`（IDE 里「运行」、不带任何参数）即按上述默认把表写到输出目录、主文件名与 md 相同。若**在命令行里**指定了 md 路径，则与旧版行为一致：未加 `-o` 时 CSV 与源 md 同目录。仍支持 `--table N`、`--all`、`-o` 指定文件或目录、以及 `--strip-markdown` / `--no-strip-markdown` 覆盖无参时的去格式行为。成功时标准输出会打印生成 CSV 的绝对路径。
