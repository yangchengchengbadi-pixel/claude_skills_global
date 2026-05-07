---
name: plotly-settings
description: Plotly 热力图配色规范与实现。采用蓝灰红三段式配色法（最小值→蓝、基准值→灰、最大值→红），适用于有基准语义的指标。Use when creating Plotly heatmaps, choosing colorscales, or when the user mentions plotly heatmap colors or 热力图配色.
tools: ["Read"]
model: sonnet
effort: low
---
# Plotly 热力图配色规范

## 配色方案：蓝灰红三段式配色法

**语义**：最小值→蓝色、基准值→浅灰、最大值→红色。中间灰色突出「优于/劣于基准」的对比。

## 实现步骤

### 1. 计算配色基准点 `smart_zMinMidMax`

```python
def smart_zMinMidMax(z_values):
    """
    智能计算配色基准点。
    参数: z_values 为 2D 数组/列表（如热力图 z 矩阵）
    返回: (zmin2, zmid, zmax2) 调整后的最小值、中值和最大值
    """
    import numpy as np

    z_list = [z for z in np.array(z_values).flatten() if not np.isnan(z)]
    if not z_list:
        return (0, 0.5, 1)

    z_median = np.median(z_list)
    dist_to_0 = abs(z_median - 0)
    dist_to_05 = abs(z_median - 0.5)
    dist_to_1 = abs(z_median - 1)
    min_dist = min(dist_to_0, dist_to_05, dist_to_1)
    if min_dist == dist_to_0:
        zmid = 0
    elif min_dist == dist_to_05:
        zmid = 0.5
    else:
        zmid = 1

    zmax = max(z_list)
    zmin = min(z_list)
    delta_upper = abs(zmax - zmid)
    delta_lower = abs(zmid - zmin)
    zdelta = max(delta_upper, delta_lower)
    zmax2 = zmid + zdelta
    zmin2 = zmid - zdelta
    return zmin2, zmid, zmax2
```

### 2. 计算基准在 [0,1] 中的位置

```python
zmin2, zmid, zmax2 = smart_zMinMidMax(zvalue)
baseline_pos = 0.5 if zmin2 == zmax2 else (zmid - zmin2) / (zmax2 - zmin2)
```

### 3. 自定义 colorscale

```python
colorscale=[
    [0.0, 'blue'],           # 最小值 → 蓝
    [baseline_pos, 'lightgray'],  # 基准值 → 灰
    [1.0, 'red']              # 最大值 → 红
]
```

### 4. 热力图调用示例

```python
import plotly.graph_objects as go

fig = go.Figure(data=go.Heatmap(
    z=zvalue,
    x=x_labels,
    y=y_labels,
    colorscale=colorscale,
    zmin=zmin2,
    zmax=zmax2,
    text=text_matrix,
    texttemplate="%{text}",
    textfont={"size": 12},
))
```

## 与线性 RdBu 的对比

| 维度 | 蓝灰红三段式配色法 | 线性 RdBu |
|------|--------------------|-----------|
| 基准点 | 中位数相关，映射为灰 | 无或固定 zmid=0 |
| 适用场景 | 有「好/坏」或「偏离基准」语义的指标 | 简单 min-max 展示 |
| 视觉 | 灰=基准，蓝<基准，红>基准 | 蓝=小，红=大 |
