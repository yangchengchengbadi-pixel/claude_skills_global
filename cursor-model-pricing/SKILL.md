---
name: cursor-model-pricing
description: Cursor Pro 可用模型及 token 价格的参考信息。此 skill 仅用于存储信息，不用于被调用执行任何操作。
tools: ["Read"]
model: sonnet
effort: low
---
# Cursor Pro 模型与价格参考

> **说明**：此 skill 仅作为信息存储用途，不包含任何可执行操作。
> 如需查阅模型价格，可在对话中输入 `/cursor-model-pricing` 调出此文档。
> 信息来源：[Cursor 官方模型文档](https://cursor.com/docs/models)，更新时间：2026-02。

---

## 一、可用模型列表

| 模型 | 提供商 | 默认上下文 | Max Mode | 能力 |
|------|--------|-----------|----------|------|
| Claude 4.5 Sonnet | Anthropic | 200k | 1M | Agent、Thinking |
| Claude 4.6 Opus | Anthropic | 200k | 1M | Agent、Thinking |
| Composer 1.5 | Cursor | 200k | - | Agent、Thinking |
| Gemini 3 Flash | Google | 200k | 1M | Agent、Thinking |
| Gemini 3 Pro | Google | 200k | 1M | Agent、Thinking |
| GPT-5.2 | OpenAI | 272k | - | Agent、Thinking |
| GPT-5.3 Codex | OpenAI | 272k | - | Agent、Thinking |
| Grok Code | xAI | 256k | - | Agent、Thinking |

---

## 二、Token 价格（美元 / 百万 token）

| 模型 | 输入 (Input) | 缓存写入 (Cache Write) | 缓存读取 (Cache Read) | 输出 (Output) |
|------|-------------|----------------------|---------------------|--------------|
| Claude 4.5 Sonnet | $3.00 | $3.75 | $0.30 | $15.00 |
| Claude 4.6 Opus | $5.00 | $6.25 | $0.50 | $25.00 |
| Composer 1.5 | $3.50 | - | $0.35 | $17.50 |
| Gemini 3 Flash | $0.50 | - | $0.05 | $3.00 |
| Gemini 3 Pro | $2.00 | - | $0.20 | $12.00 |
| GPT-5.2 | $1.75 | - | $0.175 | $14.00 |
| GPT-5.3 Codex | $1.75 | - | $0.175 | $14.00 |
| Grok Code | $0.20 | - | $0.02 | $1.50 |

### Auto 模式价格

| 类别 | 价格（/ 百万 token） |
|------|---------------------|
| 输入 + 缓存写入 | $1.25 |
| 输出 | $6.00 |
| 缓存读取 | $0.25 |

---

## 三、$20 额度预估可用次数

以下估算基于 Cursor 官方给出的中位数 token 用量，并结合典型的编码对话场景（每次请求约 3k~5k 输入 token + 1k~2k 输出 token，含缓存读取）。实际用量因对话长度、上下文大小、工具调用次数等因素波动较大，仅供参考。

| 模型 | 预估单次成本 | $20 约可用次数 | 说明 |
|------|------------|---------------|------|
| Claude 4.5 Sonnet | ~$0.08–0.10 | **~200–250 次** | 官方中位数约 225 次 |
| Claude 4.6 Opus | ~$0.12–0.15 | **~130–170 次** | 最强但最贵，输出 token 价格为 Sonnet 的 1.67 倍 |
| Composer 1.5 | ~$0.09–0.11 | **~180–220 次** | Cursor 自研模型，价格与 Sonnet 接近 |
| Gemini 3 Flash | ~$0.02–0.03 | **~650–1000 次** | 性价比极高 |
| Gemini 3 Pro | ~$0.06–0.08 | **~250–330 次** | 比 Flash 贵但能力更强 |
| GPT-5.2 | ~$0.04–0.06 | **~330–500 次** | 官方中位数约 650 次（含较轻量场景） |
| GPT-5.3 Codex | ~$0.04–0.06 | **~330–500 次** | 与 GPT-5.2 同价 |
| Grok Code | ~$0.01–0.02 | **~1000–2000 次** | 最便宜，适合高频轻量任务 |
| Auto 模式 | ~$0.04–0.06 | **~330–500 次** | 自动选模型，性价比与效果的平衡点 |

### 影响实际用量的关键因素

- **上下文长度**：对话越长、引用文件越多，输入 token 越多，单次成本越高
- **输出长度**：模型生成的代码/文字越多，输出 token 消耗越大
- **工具调用**：Agent 模式下每次文件读取、搜索、命令执行都增加 token
- **缓存命中率**：缓存读取价格远低于输入价格，长对话中后续请求可利用缓存降低成本
- **Max Mode**：开启后上下文更大，但 token 消耗也更高

---

## 四、省额度建议

1. 日常使用 **Auto 模式**（Cursor 自动选择，性价比平衡）
2. 高频轻量任务用 **Grok Code** 或 **Gemini 3 Flash**
3. 及时开新对话，避免上下文膨胀
4. 只 `@` 必要的文件，控制输入 token
5. 复杂任务才选 Claude 4.6 Opus
