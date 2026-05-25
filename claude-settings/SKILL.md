---
name: claude-settings
description: Claude Code 权限体系与 settings.json 的优先级规则。涵盖 allow/ask/deny 合并逻辑、路径模式作用域、内置写保护、受保护路径等。用于排查权限问题、配置免确认编辑等场景。
tools: ["Read"]
model: sonnet
effort: low
---

# 0. 生成规范

- **双向锁定**：本文件与 `~/.claude/skills/workflow-text-format/SKILL.md` 建立双向锁定（一对多关系），是其锁定脚本之一。
- **格式遵循**：本文件正文与格式整体逻辑以 `workflow-text-format` 为准；`# 2. 层级标识`、`# 3. 对象标识`、`# 4. 功能模块` 等可执行条文以该 Skill 为唯一准据。
- **树形次级增改**：在本文件中新增、调整或加深嵌套树形结构（`├─`、`└─`、`│`）时，须先查阅该 Skill `# 1` 下「树形次级增改」全文与「树形次级案例」，并按其自检逐项通过。

# 1. Settings 层级优先级

- **层级（由高到低）**：
  ├─ **Managed settings**：最高，不可被其他层级覆盖
  ├─ **Command line arguments**：会话级临时覆盖
  ├─ **Local project settings**：`.claude/settings.local.json`
  ├─ **Shared project settings**：`.claude/settings.json`
  └─ **User settings**：`~/.claude/settings.json`（最低）

# 2. 数组合并而非覆盖

- **规则**：数组型 settings（如 `permissions.allow`、`permissions.ask`、`permissions.deny`）跨 scope 合并（concat + dedup），不是高层级替换低层级。
- **结果**：最终生效的规则是所有 scope 的汇总。

# 3. 路径模式作用域

- **路径前缀规则**：`Read`、`Write`、`Edit` 的路径参数遵循 gitignore 风格，前缀决定作用域。
  ├─ `//path`：文件系统根目录的绝对路径，如 `Edit(//etc/hosts)` 匹配 `/etc/hosts`
  ├─ `~/path`：Home 目录路径，如 `Edit(~/.bashrc)` 匹配 `/Users/xxx/.bashrc`
  ├─ `/path`：相对于项目根目录，如 `Edit(/src/**.ts)` 匹配 `<project>/src/**.ts`
  └─ `path` 或 `./path`：相对于当前工作目录，如 `Edit(**.py)` 匹配 `<cwd>/**.py`
- **`Edit(**)` 不等于全文件系统可写**：`Edit(**)` 不带任何前缀，作用域限定在当前工作目录及子目录，不能编辑项目外的文件。这是最常见的误解。
- **无限制访问写法**：要允许无限制文件访问，必须写 `Edit`（不带括号），不是 `Edit(**)`。
- **`Read` 不加括号可跨目录**：`Read`（不带括号 / 不带路径参数）可以读取当前工作目录之外的任意文件，与其他工具的权限模型不同。

# 4. 内置写保护

- **多层安全机制**：
  ├─ **默认写限制**：只能写入启动目录及子目录，不能修改父目录文件。即使权限规则写了绝对路径也会拦截。
  ├─ **受保护路径（Protected Paths）**：以下路径在任何 mode（除 `bypassPermissions` 外）下写入都强制弹窗确认或直接拒绝。
  │   ├─ **受保护目录**：`.git`、`.vscode`、`.idea`、`.husky`、`.claude`（特定子目录除外）
  │   └─ **受保护文件**：`.bashrc`、`.bash_profile`、`.zshrc`、`.zprofile`、`.profile`、`.gitconfig`、`.gitmodules`、`.ripgreprc`、`.mcp.json`、`.claude.json`
  └─ **Sandbox（可选）**：启用后通过 macOS Seatbelt / Linux bubblewrap 在 OS 级别强制文件系统和网络隔离，仅允许写入当前工作目录。

# 5. allow / ask / deny 评估顺序

- **评估顺序**：deny → ask → allow，先匹配先赢。
- **流程图**：
  ```
  deny 命中 → 直接被阻止 ✋
     ↓ 未命中
  ask 命中 → 弹窗询问用户 ✋
     ↓ 未命中
  allow 命中 → 自动允许 ✅
     ↓ 未命中
  fallback 到 permission mode 的默认行为
  ```
- **全局 ask + 项目 allow = ask 胜出**：这是最常见的误解场景。
  ├─ **场景**：全局 `ask: ["Edit", "Write"]` + 项目 `allow: ["Edit(**)", "Write(**)"]
  ├─ **合并后**：同时存在 ask 和 allow，评估时 ask 先匹配
  └─ **结果**：弹窗确认，allow 永远不被评估。即使项目级 allow 了 Edit/Write，只要全局 ask 还在，就会弹窗。
- **deny 绝对优先**：任一 scope 的 deny 规则在所有层级都无法被 allow 覆盖。

# 6. defaultMode 与 permission rules 的关系

- **关系**：`defaultMode: "acceptEdits"` 设定的是回退基线（fallback 行为）。
- **评估流程**：
  ├─ 先走 permission rules（deny → ask → allow）
  └─ 规则都没匹配到，才 fallback 到 mode 的默认行为
- **结论**：permission rules 在 mode 之上。acceptEdits 模式下 Edit 通常自动允许，但若全局有 `ask: ["Edit"]`，ask 规则会先拦截，仍然弹窗。

# 7. 免确认编辑配置

- **前提**：必须确保所有 scope 的 deny 和 ask 都不匹配 Edit/Write。
- **配置示例**：
  ```json
  // 全局 ~/.claude/settings.json
  "ask": [],  // 删除 "Edit"、"Write"
  ```
- **效果**：fallback 到项目级的 `allow: ["Edit(**)", "Write(**)"]` 或 `defaultMode: "acceptEdits"` 才能自动放行。
- **项目外编辑**：`Edit(**)` 不覆盖项目外部路径；修改如 `~/.claude/skills/` 下的文件需额外配置 `Edit(~/.claude/skills/**)` 才能免确认。
