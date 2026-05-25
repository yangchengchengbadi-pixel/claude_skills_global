---
name: websearch-fallback
description: 当 WebSearch 工具不可用（如 DeepSeek 模型不支持 tool_choice 参数）时，使用 WebFetch + DuckDuckGo HTML 作为搜索引擎替代方案。
tools: ["WebFetch"]
model: sonnet
effort: low
---

# WebSearch 不可用时的搜索替代方案

当 `WebSearch` 因模型不支持 `tool_choice` 参数而报错（如 DeepSeek 系列模型）时，使用 `WebFetch` 工具 + DuckDuckGo HTML 端点进行搜索。

## 用法

```
WebFetch("https://html.duckduckgo.com/html/?q=关键词", "提取搜索结果中的重要信息")
```

## 示例

```python
# 搜索"今天的日期"
WebFetch("https://html.duckduckgo.com/html/?q=今天的日期", "提取搜索结果中的重要信息")

# 搜索技术文档
WebFetch("https://html.duckduckgo.com/html/?q=python subprocess timeout", "提取关键代码示例和使用方法")
```

## 注意事项

- **Google 不可用**：Google Search 会被反爬机制拦截（`SG_REL` 错误），不要使用
- **仅限 HTML 端点**：DuckDuckGo 的 `html.duckduckgo.com` 对程序化访问较友好，正常 API 可能有限制
- **中文搜索**：查询参数中的中文关键词会被自动处理，无需特殊编码
- **结果精度**：相比原生搜索 API，HTML 页面可能缺少部分元数据（如日期、来源详情），优先使用第一页结果中的链接和摘要
