---
name: ccr-install
description: Claude Code Router（CCR）安装与配置流程，实现 DeepSeek 主力模型 + Gemini WebSearch 的混合路由方案。在用户需要安装 CCR、配置模型路由、或解决 WebSearch 在 DeepSeek 下不可用问题时使用。
tools: ["Read", "Bash", "Write", "Edit"]
model: sonnet
effort: low
---

# 0. 生成规范

- **双向锁定**：本文件 `/Users/jiachenyang/.claude/skills/ccr-install/SKILL.md` 与 `/Users/jiachenyang/.claude/skills/workflow-text-format/SKILL.md` 建立 **双向锁定**：对本文件正文的增删改须符合后者 **`# 2`**～**`# 4`**；在本文件中引入后者尚未记载的文本格式时，须 **回写** `workflow-text-format/SKILL.md`；与存量 **冲突** 时 **停手** 并向用户说明，请用户选定方案后再改。

- **格式依据**：工作流正文与树形层级的 **整体逻辑** 以 **workflow-text-format** 为准。

- **树形次级**：新增、调整或加深嵌套 `├─`、`└─`、`│` 时，须先查阅 **workflow-text-format** **`# 1`** 下 **树形次级增改** 与 **`# 2` · 树形次级**，定稿前完成该 Skill 所列自检。

# 1. 概述

- **CCR 是什么**：Claude Code Router 是一个本地代理层，运行在 `localhost`，拦截 Claude Code 发出的所有 API 请求，根据请求类型（普通对话、WebSearch、长上下文等）自动分发到不同的模型后端。

- **为什么需要**：DeepSeek V4 Pro 在 thinking 模式下不支持 `tool_choice` 参数，导致 WebSearch 不可用（报错 `deepseek-reasoner does not support this tool_choice`）。CCR 将 WebSearch 请求路由到 Gemini（免费），其余请求仍走 DeepSeek，**不改变现有主力模型**。

- **架构**：
    ├─ Claude Code → `localhost:19456`（CCR 代理）
    ├─ 检测到 `web_search` 工具调用 → Gemini API（仅搜索）
    └─ 未检测到 → DeepSeek API（代码、对话、分析等主力任务）

# 2. 前置条件

- **Node.js**：CCR 通过 npm 安装，需要 Node.js ≥ 18。检查命令：`node --version`。

- **DeepSeek API Key**：现有配置不变，沿用当前 key。

- **Gemini API Key**：WebSearch 路由需要。免费申请地址：[Google AI Studio](https://aistudio.google.com/apikey)。免费额度每分钟 15 次请求，够日常使用。AIzaSyD-u9kxPwP1qcKuZDhhJgiC45m2XdKxeE4

# 3. 安装

- **npm 全局安装**：
    ```bash
    npm install -g @musistudio/claude-code-router
    ```

- **验证安装**：
    ```bash
    ccr -v
    ```

- **卸载（如需）**：
    ```bash
    npm uninstall -g @musistudio/claude-code-router
    ```

# 4. 配置

- **配置文件位置**：`~/.claude-code-router/config.json`

- **配置模板**（替换 `GEMINI_API_KEY` 为实际 key）：
    ```json
    {
      "Providers": [
        {
          "name": "deepseek",
          "api_base_url": "https://api.deepseek.com/chat/completions",
          "api_key": "sk-cc7d73ef973e460fb046adea25a639ad",
          "models": ["deepseek-v4-pro", "deepseek-v4-flash"],
          "transformer": {
            "use": ["deepseek"],
            "deepseek-v4-flash": { "use": ["tooluse"] }
          }
        },
        {
          "name": "gemini",
          "api_base_url": "https://generativelanguage.googleapis.com/v1beta/models/",
          "api_key": "你的-Gemini-API-Key",
          "models": ["gemini-2.5-flash", "gemini-2.5-pro"],
          "transformer": { "use": ["gemini"] }
        }
      ],
      "Router": {
        "default": "deepseek,deepseek-v4-pro",
        "think": "deepseek,deepseek-v4-pro",
        "background": "deepseek,deepseek-v4-flash",
        "webSearch": "gemini,gemini-2.5-flash"
      }
    }
    ```

- **安全建议**：API Key 建议使用环境变量插值，避免硬编码在配置文件中：
    ```json
    "api_key": "${DEEPSEEK_API_KEY}"
    ```
    然后在 shell 配置（`~/.zshrc`）中设置：
    ```bash
    export DEEPSEEK_API_KEY="sk-xxx"
    export GEMINI_API_KEY="xxx"
    ```

- **注意**：通过 CCR Web UI 保存配置时，环境变量引用可能被覆盖为字面值（已知 issue `#1373`）。建议直接手写 `config.json`。

# 5. 启动与使用

- **启动 CCR 代理**：
    ```bash
    ccr code
    ```
    该命令启动代理服务并进入 Claude Code 会话。

- **重启服务**（修改配置后）：
    ```bash
    ccr restart
    ```

- **管理界面**：
    ```bash
    ccr ui
    ```
    打开 Web 界面在线管理配置和模型。

- **动态切换模型**：在 Claude Code 会话中直接使用 `/model` 命令切换。

- **CLI 模型管理**：
    ```bash
    ccr model
    ```
    交互式终端界面，查看和修改当前模型配置。

# 6. 路由逻辑

- **工作原理**：CCR 检查每轮 API 请求的 `req.body.tools` 数组。若其中包含 `type` 以 `web_search` 开头的工具 → 路由到 `webSearch` 配置的模型；否则走 `default`。

- **多轮任务示例**（用户问「搜索 Python 3.13 新特性并分析对回测框架的影响」）：
    ├─ 第 1 轮：模型判断需要搜索 → 调用 WebSearch → CCR 检测到 `web_search` → **Gemini** 执行搜索
    ├─ 第 2 轮：搜索结果返回 → 模型分析 → 无 `web_search` → **DeepSeek** 执行分析
    └─ 第 3 轮（如有）：模型需要读网页 → 调用 WebFetch → 无 `web_search` → **DeepSeek** 执行抓取

- **用户无需干预**：全程自动路由，同一会话内模型切换透明。

# 7. 常见问题

- **CCR 是否影响 DeepSeek 调用质量**：不影响。CCR 只做路由 + 格式适配，DeepSeek 接收到的请求内容与直接连接一致，输出质量不变。

- **延迟增加多少**：本地代理转发，额外延迟 < 5ms，可忽略。

- **Gemini 免费额度够用吗**：WebSearch 使用频率通常不高（非每次对话都搜索），免费额度每分钟 15 次日常够用。如需更高频率，可升级 Gemini 付费方案。

- **能否只用 DeepSeek 不做混合路由**：当前 DeepSeek V4 Pro 不支持 WebSearch（`tool_choice` 报错），不配 Gemini 则 WebSearch 仍不可用。这是 DeepSeek API 的限制，CCR 无法修复，只能将 WebSearch 路由到其他支持 `tool_choice` 的模型。

- **DeepSeek V4 thinking + 工具调用报 400**：这是 DeepSeek API 在 thinking 模式下要求回传 `reasoning_content` 的已知问题（CCR issue `#1378`），社区正在修复中。该问题与是否使用 CCR 无关——直接连 DeepSeek 也会遇到。

    - **故障表现**：WebSearch 成功后，DeepSeek 基于搜索结果继续推理时，CCR 未回传上一轮的 `reasoning_content`，DeepSeek API 直接返回 400。报错信息：`The reasoning_content in the thinking mode must be passed back to the API`。

    - **触发规律**：并非每次 WebSearch 后都炸。纯总结/归纳类指令通常不触发 thinking 模式，可顺利执行；复杂推理任务更容易触发。本会话实验：两次「搜索 + 总结」类指令均无报错，一次「搜索 Claude Code 最新特性」触发了报错。

    - **妥协可行方案**：将「搜索 + 分析」拆分为两步，中间插入一条简短过渡指令：
        ```
        第 1 步：请搜索 XXX，先不要分析
        第 2 步：请继续总结（新会话或同一会话中发出）
        ```
        实际验证：将搜索和分析放在同一条指令中（如「请搜索 XXX，并总结最重要的几点」），搜索完成后手动输入「请继续总结」，DeepSeek 在总结阶段不走 thinking，可成功执行。节奏会稍慢，但功能完整。

    - **根治等待**：社区方案中，`yxlao/deepseek-cursor-proxy` 代理可自动注入 `reasoning_content`；CCR 上游修复后升级即可根本解决。在此之前，上述拆步法可作为日常使用策略。

- **CCR 更新**：
    ```bash
    npm update -g @musistudio/claude-code-router
    ```

- **项目地址**：[github.com/musistudio/claude-code-router](https://github.com/musistudio/claude-code-router)（33k+ stars，MIT 许可）
