---
name: plotly-kaleido-setup
description: Plotly 静态图导出（fig.write_image）所需的 Kaleido 环境配置与排错。当用户遇到 "No module named 'kaleido'"、Plotly 与 Kaleido 版本不兼容、或 PNG 导出失败时使用。
tools: ["Read", "Bash"]
model: sonnet
effort: low
---
# Plotly + Kaleido 环境配置

当使用 Plotly 的 `fig.write_image()` 导出 PNG 等静态图时，需要 Kaleido 依赖。本 skill 指导检查环境、切换正确 Python 环境并安装兼容版本。

## 1. 检查当前环境

**问题**：`pip install kaleido` 显示已安装，但脚本仍报 `No module named 'kaleido'`。

**原因**：kaleido 可能安装在**其他 Python 环境**，而脚本运行在**不同环境**（如 conda env）。

**检查步骤**：

```bash
# 1. 确认脚本实际使用的 Python 路径
which python
# 或查看 Cursor/IDE 运行配置中的解释器路径，如：
# /Users/xxx/anaconda3/envs/Alpha/bin/python

# 2. 用该 Python 直接检查 kaleido
/Users/xxx/anaconda3/envs/Alpha/bin/python -c "import kaleido; print(kaleido.__version__)"

# 3. 检查该环境的 pip 安装列表
/Users/xxx/anaconda3/envs/Alpha/bin/pip list | grep -i kaleido
```

若 `import kaleido` 报错或 `pip list` 无 kaleido，说明**当前运行环境未安装**。

## 2. 切换到正确环境并安装

**原则**：必须在**脚本实际运行的 Python 环境**中安装 kaleido。

```bash
# 激活目标 conda 环境
conda activate Alpha   # 或脚本使用的环境名

# 使用该环境的 pip 显式安装（避免 PATH 混用系统 pip）
/path/to/conda/envs/Alpha/bin/pip install kaleido
```

或直接指定完整路径：

```bash
/Users/xxx/anaconda3/envs/Alpha/bin/pip install kaleido
```

## 3. 版本兼容性

| Plotly 版本 | 兼容 Kaleido 版本 |
|-------------|-------------------|
| 5.x（如 5.24.1） | **0.2.1** |
| 6.1.1+ | **1.x**（如 1.2.0） |

**常见报错**：

```
Plotly version 5.24.1 is not compatible with Kaleido 1.2.0.
Please upgrade Plotly to 6.1.1+, or downgrade Kaleido to 0.2.1.
```

**处理**：若项目有依赖（如 `xbx-py11` 要求 `plotly==5.24.1`），应**降级 Kaleido** 而非升级 Plotly：

```bash
pip install kaleido==0.2.1
```

若可升级 Plotly：

```bash
pip install -U "plotly>=6.1.1"
```

## 4. 代码侧注意

Plotly 需**先 import kaleido** 才能正确检测，否则可能误报「需安装 kaleido」：

```python
try:
    import kaleido  # 必须在 write_image 前导入
    fig.write_image(str(png_path))
except Exception as e:
    logger.warning(f'PNG 保存失败: {e}')
```

## 5. 快速排错流程

1. **No module named 'kaleido'** → 在脚本运行的 Python 环境中安装 kaleido
2. **版本不兼容警告** → 按上表选择 plotly 或 kaleido 版本并调整
3. **项目依赖冲突**（如 xbx-py11 要求 plotly 5.24.1）→ 保持 plotly 不变，安装 `kaleido==0.2.1`
