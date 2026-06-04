# CodexBridge 接入个人微信实战笔记

这是一篇把公开视频教程落地到 Windows 本机环境的实战记录：使用开源项目 CodexBridge，把 Codex 接入个人微信，实现从微信里和 Codex 交互。

本文只记录可公开复用的配置思路、命令和踩坑修复。所有本机用户名、绝对目录、账号状态文件、二维码、日志路径和登录标识都已脱敏。

## 参考项目

本次使用的核心开源仓库：

- GitHub: [Gan-Xing/CodexBridge](https://github.com/Gan-Xing/CodexBridge)

相关导航仓库：

- GitHub: [0x78rensai-pixel/github-index](https://github.com/0x78rensai-pixel/github-index)

## 目标

把 CodexBridge 跑起来，让微信消息可以进入 CodexBridge，再由 CodexBridge 调用本机 Codex CLI。

最终目标：

- 克隆 CodexBridge。
- 安装 Node.js/npm。
- 安装并验证 Codex CLI。
- 完成微信扫码登录。
- 启动 `weixin:serve` 后台服务。
- 用 `/helps` 或 `/status` 在微信里验证链路。

## 环境约定

示例中使用占位路径，按自己的机器替换：

```text
<workspace-root>
<codexbridge-repo>
<codex-cli-path>
```

例如：

```text
<workspace-root>\CodexBridge
```

不要把以下内容写入公开仓库：

- 本机用户名
- 真实绝对目录
- 微信账号状态文件
- 二维码图片
- 登录日志
- token、cookie、auth 文件

## 安装与配置过程

### 1. 克隆 CodexBridge

```powershell
git clone https://github.com/Gan-Xing/CodexBridge.git "<workspace-root>\CodexBridge"
```

进入项目：

```powershell
cd "<workspace-root>\CodexBridge"
```

项目 README 中的核心流程是：

```powershell
npm install
codex --version
npm run weixin:login
npm run weixin:serve -- --cwd "<workspace-root>"
```

### 2. 安装 Node.js/npm

如果本机没有 `npm`，可以安装 Node.js：

```powershell
winget install OpenJS.NodeJS --accept-package-agreements --accept-source-agreements
```

安装后确认：

```powershell
node --version
npm --version
```

如果 Windows 上 `node` 命中了受限路径并出现：

```text
Access is denied.
```

可以在启动脚本里把真实 Node.js 安装目录放到 PATH 最前：

```bat
set "PATH=<node-install-dir>;%PATH%"
```

### 3. 安装 Codex CLI

```powershell
npm install -g @openai/codex@latest --include=optional
```

验证：

```powershell
codex --version
```

如果 `codex.cmd` 内部调用的 `node` 仍然命中受限路径，可以同样通过调整 PATH 解决。

### 4. 安装 CodexBridge 依赖

优先按项目说明执行：

```powershell
npm install
```

如果 `ffmpeg-static` 在安装脚本中下载 GitHub 二进制失败，例如出现网络超时：

```text
ETIMEDOUT
```

而当前只需要文字桥接，可以先用：

```powershell
npm install --ignore-scripts
```

后续如果要处理语音、视频或媒体文件，再单独配置：

```text
CODEXBRIDGE_FFMPEG_PATH
CODEXBRIDGE_FFPROBE_PATH
```

### 5. 修改 Windows 启动脚本

CodexBridge 自带的 Windows 脚本可能包含作者本机路径。需要改成自己的环境，但不要把真实路径提交到公开笔记里。

建议配置项：

```bat
set "PATH=<node-install-dir>;%PATH%"
set "CODEX_REAL_BIN=<codex-cli-path>"
set "CODEX_APP_SERVER_TRANSPORT=stdio"
set "CODEXBRIDGE_DEFAULT_CWD=<workspace-root>"
set "CODEXBRIDGE_LOCALE=zh-CN"
```

涉及脚本：

```text
start-weixin-login.cmd
start-weixin-serve.cmd
install-weixin-service-after-login.cmd
```

### 6. 微信登录轮询超时处理

扫码登录时，二维码可能已经生成，但状态轮询因为网络抖动超时：

```text
HTTPS request timed out
ETIMEDOUT
```

实战中可把临时网络错误视为可重试错误，让登录轮询继续等待，而不是直接退出。

修复思路：

- `AbortError` 继续重试。
- `ETIMEDOUT`、`ECONNRESET`、`EHOSTUNREACH`、`ENETUNREACH` 等临时网络错误继续重试。
- 不改变二维码登录协议，只增强轮询容错。

修改后运行：

```powershell
npm run typecheck
```

确认 TypeScript 检查通过。

### 7. 微信扫码登录

```powershell
npm run weixin:login -- --timeout-sec 480
```

终端会生成二维码路径和扫码提示。用微信扫码并确认。

登录成功后会在本机状态目录保存账号信息。这个目录包含登录状态，不要提交到公开仓库。

### 8. 启动微信桥接服务

```powershell
npm run weixin:serve -- --cwd "<workspace-root>"
```

如果服务启动成功，会出现类似信息：

```text
WeChat bridge started
state_dir: <state-dir>
serve_lock: <serve-lock>
default_cwd: <workspace-root>
```

这些运行状态路径可以用于本机排查，但不应写入公开博客。

## 验证方法

在微信里发送：

```text
/helps
```

或：

```text
/status
```

如果能收到回复，说明微信到 CodexBridge 的链路已经通了。

## 是否影响 VS Code SSH

一般不会。

CodexBridge 微信接入主要是本机后台 Node 进程轮询微信接口，并在需要时调用 Codex CLI。它不会改 VS Code Remote SSH 的配置，也不会占用 SSH 端口。

可能的间接影响：

- 高频触发 Codex 任务时，会占用 CPU、内存和网络。
- 如果在微信里明确要求 Codex 修改某个仓库，它会按指令改对应文件。

## 踩坑总结

### PATH 优先级

Windows 上可能存在多个 `node.exe` 或 `codex.exe`。如果命中受限路径，可能出现：

```text
Access is denied.
```

解决方式是把真实 Node.js 安装目录放到 PATH 最前。

### ffmpeg-static 下载失败

`ffmpeg-static` 可能需要从 GitHub 下载二进制文件，网络不稳定时会失败。只做文字桥接时可以先跳过安装脚本。

### 二维码过期

二维码过期后重新运行登录命令即可。扫码后要尽快在微信里确认。

### 登录状态不能公开

不要提交以下内容：

```text
<state-dir>
<login-qrcode-files>
<account-state-files>
<runtime-logs>
<auth-files>
```

## 后续计划

- 把 CodexBridge 的登录轮询容错修复整理成补丁或 PR。
- 给 `ffmpeg-static` 下载失败提供更稳定的配置方案。
- 整理一份微信入口常用命令清单，例如 `/helps`、`/status`、`/threads`、`/open`、`/model`。
- 将更多实战记录沉淀到导航仓库中。
