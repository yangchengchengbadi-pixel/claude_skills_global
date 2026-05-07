---
name: install-homebrew
description: 在 macOS 上安装 Homebrew、安装后把 brew 加入 PATH、排查「command not found: brew」、 首次克隆卡顿是否正常现象，以及用 brew 安装 Claude Code（claude-code@latest）。 在用户安装 brew、配置 zsh PATH、或 curl 官方脚本遇 403 需改走 Homebrew 路线时使用。
tools: ["Read", "Bash"]
model: sonnet
effort: low
---
# install-homebrew（macOS）

## 1. 官方安装 Homebrew

在终端执行（整行复制）：

```bash
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
```

按提示回车；若要求输入密码，为**本机登录密码**（输入时不显示字符属正常）。

国内访问 GitHub 可能较慢；若长时间停在 **Downloading and installing Homebrew** / **Enumerating objects**，多为正在克隆仓库，**等待数十分钟**可接受。若完全无进度且超时，可换网络、热点，或查阅当前可用的 **Homebrew 国内镜像** 文档（以镜像站最新说明为准）。

## 2. 安装成功后的 PATH（必读）

安装结束若出现 **Next steps**，须把 `brew` 加入 PATH。**Apple 芯片**常见安装前缀为 `/opt/homebrew`。在终端执行：

```bash
echo >> ~/.zprofile
echo 'eval "$(/opt/homebrew/bin/brew shellenv zsh)"' >> ~/.zprofile
eval "$(/opt/homebrew/bin/brew shellenv zsh)"
```

然后验证：

```bash
brew --version
```

若仍提示 **`command not found: brew`**：

- 先确认当前会话已执行第三行的 **`eval "$(/opt/homebrew/bin/brew shellenv zsh)"`**；或执行 `source ~/.zprofile`。
- 或用绝对路径验证安装是否存在：`/opt/homebrew/bin/brew --version`（Intel Mac 可能是 `/usr/local/bin/brew`，以安装程序提示为准）。

## 3. 用 Homebrew 安装 Claude Code

PATH 正常后：

```bash
brew install --cask claude-code@latest
```

若 **`curl https://claude.ai/install.sh`** 返回 **403** 等地域或策略限制，可优先采用本 Skill 完成 **Homebrew + cask** 路线，而不依赖一键 pipe 脚本。

## 4. `curl -fsSL … | bash` 中与 curl 相关的简短释义（便于对照文档）

- **`curl`**：按 URL 拉取内容。
- **`-f`**：HTTP 出错则命令失败；**`-s`**：少打进度；**`-S`**：与 `-s` 搭配仍显示错误；**`-L`**：跟随重定向。
- **`| bash`**：把下载内容交给 shell 执行；**仅对可信官方脚本**使用。
