# Paseo 使用教程

本文面向想实际使用 Paseo 的开发者，覆盖安装、启动 daemon、连接客户端、运行 agent、处理权限、使用 worktree、远程连接、源码开发和常见问题。

Paseo 的核心是一个运行在本机的 `daemon`。它负责管理 Claude Code、Codex、OpenCode 等 coding agent 的生命周期，桌面端、移动端、Web 端和 CLI 都通过 WebSocket 连接这个 daemon。

## 1. 适合谁使用

Paseo 适合以下场景：

- 你经常使用 Claude Code、Codex、OpenCode 等 coding agent。
- 你希望在一个界面里同时管理多个 agent。
- 你希望 agent 在本机运行，使用本机代码、工具链和凭据。
- 你希望从桌面、手机、Web 或命令行查看 agent 进度。
- 你希望让 agent 在 git worktree 中隔离工作，降低影响主分支的风险。

Paseo 本身不是模型服务，也不是云端 IDE。它是本地优先的 agent 编排层。

## 2. 核心概念

### Daemon

`daemon` 是 Paseo 的本地服务进程，默认监听：

```bash
127.0.0.1:6767
```

它负责：

- 创建、运行、停止和恢复 agent。
- 记录 agent timeline 和状态。
- 处理工具调用权限请求。
- 管理终端、workspace、worktree 和远程连接。
- 向桌面端、移动端、Web 端和 CLI 提供统一接口。

默认运行数据保存在：

```bash
~/.paseo
```

### Provider

Provider 是具体的 agent 后端，例如：

- `claude`：Claude Code / Anthropic Agent SDK。
- `codex`：OpenAI Codex。
- `opencode`：OpenCode。

Paseo 不替 provider 登录，也不管理 provider 的 API key。你需要先让对应 CLI 在本机独立可用。

### Agent

Agent 是一次可持续交互的编码任务。它可以运行、等待后续消息、请求权限、被继续发送 prompt，也可以被归档或删除。

### Worktree

Paseo 可以为某个任务自动创建 git worktree，让 agent 在隔离目录里修改代码。这样主 checkout 不会直接被 agent 改动。

## 3. 前置条件

至少安装并配置一个 agent provider。

示例检查：

```bash
claude --version
codex --version
opencode --version
```

任选其一即可，但如果你想在 Paseo 中切换多个 provider，就需要分别安装和登录。

如果要从源码运行 Paseo，还需要 Node.js 和 npm。仓库的 `.tool-versions` 中 Node.js 版本为：

```bash
nodejs 22.20.0
```

## 4. 安装 Paseo

### 方式一：桌面应用

适合大多数用户。

1. 从 `https://paseo.sh/download` 或 GitHub Releases 下载桌面端。
2. 打开桌面应用。
3. 桌面端会自动管理本地 daemon。
4. 在设置页可以查看配对二维码，用手机连接。

### 方式二：CLI / 无头模式

适合服务器、远程机器或习惯终端工作流的用户。

```bash
npm install -g @getpaseo/cli
paseo daemon start
paseo daemon status
```

如果 daemon 正常启动，后续就可以用 `paseo run`、`paseo ls`、`paseo attach` 等命令管理 agent。

## 5. 启动和管理 daemon

启动 daemon：

```bash
paseo daemon start
```

查看状态：

```bash
paseo daemon status
```

重启 daemon：

```bash
paseo daemon restart
```

停止 daemon：

```bash
paseo daemon stop
```

指定端口：

```bash
paseo daemon start --port 7777
```

指定完整监听地址：

```bash
paseo daemon start --listen 127.0.0.1:7777
```

使用独立的运行目录：

```bash
PASEO_HOME=~/.paseo-test paseo daemon start --port 7777
```

这适合测试新配置或同时运行多套 Paseo。

常用日志位置：

```bash
~/.paseo/daemon.log
```

## 6. 连接客户端

### 本机 CLI

CLI 默认连接本机 daemon。

```bash
paseo ls
```

连接指定 daemon：

```bash
paseo --host 127.0.0.1:7777 ls
```

### 桌面端

桌面端通常会自动启动并连接本地 daemon。它适合长期查看 agent 状态、分屏管理多个 agent、处理权限请求。

### 手机端或远程客户端

生成配对二维码和链接：

```bash
paseo daemon pair
```

用手机客户端扫描二维码即可连接。

配对链接包含 daemon 的公钥，是远程连接的信任入口。不要公开分享。

## 7. 查看 provider 和模型

列出 provider：

```bash
paseo provider ls
```

查看某个 provider 的模型：

```bash
paseo provider models claude
paseo provider models codex
paseo provider models opencode
```

查看模型的 thinking 选项：

```bash
paseo provider models codex --thinking
```

如果 provider 不可用，优先检查：

```bash
which claude
which codex
which opencode
```

以及各 provider 是否已经完成登录。

## 8. 创建并运行 agent

最小示例：

```bash
paseo run --provider codex/gpt-5.4 "分析当前项目结构"
```

指定工作目录：

```bash
paseo run --cwd /path/to/project --provider claude "找出这个项目的启动方式"
```

后台运行：

```bash
paseo run --provider codex/gpt-5.4 --detach "修复当前测试失败"
```

设置标题：

```bash
paseo run --provider codex/gpt-5.4 --title "fix failing tests" "运行测试并修复失败"
```

指定 provider、model、mode 和 thinking：

```bash
paseo run \
  --provider codex/gpt-5.4 \
  --mode default \
  --thinking high \
  "重构认证模块并补测试"
```

添加标签：

```bash
paseo run \
  --provider codex/gpt-5.4 \
  --label area=auth \
  --label priority=high \
  "修复登录过期后的跳转问题"
```

附加图片：

```bash
paseo run --provider claude --image screenshot.png "根据截图修复 UI 问题"
```

等待 agent 完成并设置超时：

```bash
paseo run --provider codex/gpt-5.4 --wait-timeout 30m "运行完整测试并修复失败"
```

## 9. 查看和继续 agent

列出当前 agent：

```bash
paseo ls
```

包含已归档 agent：

```bash
paseo ls -a
```

按标签过滤：

```bash
paseo ls --label area=auth
```

实时查看输出：

```bash
paseo attach <agent-id-prefix>
```

查看日志：

```bash
paseo logs <agent-id-prefix>
```

查看详细信息：

```bash
paseo inspect <agent-id-prefix>
```

继续发送任务：

```bash
paseo send <agent-id-prefix> "继续补充边界测试"
```

从文件读取 prompt：

```bash
paseo send <agent-id-prefix> --prompt-file task.md
```

发送图片和说明：

```bash
paseo send <agent-id-prefix> --image screenshot.png "按这张截图调整页面"
```

发送后立即返回：

```bash
paseo send <agent-id-prefix> --no-wait "继续处理剩下的 lint 问题"
```

## 10. 停止、归档和删除 agent

等待 agent 完成：

```bash
paseo wait <agent-id-prefix>
```

停止 agent：

```bash
paseo stop <agent-id-prefix>
```

归档 agent：

```bash
paseo archive <agent-id-prefix>
```

删除 agent：

```bash
paseo delete <agent-id-prefix>
```

通常建议先归档，不要急着删除。归档后仍可通过 `paseo ls -a` 查看历史。

## 11. 处理权限请求

当 agent 要执行敏感操作时，可能会暂停并等待权限确认。

列出待处理权限：

```bash
paseo permit ls
```

允许某个请求：

```bash
paseo permit allow <agent-id-prefix> <request-id-prefix>
```

允许该 agent 的所有待处理请求：

```bash
paseo permit allow <agent-id-prefix> --all
```

拒绝某个请求：

```bash
paseo permit deny <agent-id-prefix> <request-id-prefix> --message "不要修改生产配置"
```

拒绝所有请求并中断 agent：

```bash
paseo permit deny <agent-id-prefix> --all --interrupt
```

权限请求是 Paseo 的重要控制点。对于 shell、文件写入、提交、推送等操作，建议先看清楚描述再允许。

## 12. 使用 git worktree 隔离任务

让 agent 在新 worktree 中运行：

```bash
paseo run \
  --provider codex/gpt-5.4 \
  --worktree feature-auth \
  --base main \
  "实现认证模块重构"
```

查看 Paseo 管理的 worktree：

```bash
paseo worktree ls
```

归档 worktree：

```bash
paseo worktree archive feature-auth
```

默认 worktree 目录：

```bash
~/.paseo/worktrees
```

推荐在较大任务中使用 worktree。这样 agent 的改动不会直接落到主 checkout，便于 review、测试和合并。

## 13. 远程 daemon 使用

CLI 连接远程 daemon：

```bash
paseo --host workstation.local:6767 ls
```

在远程 daemon 上创建 agent：

```bash
paseo --host workstation.local:6767 run --provider codex/gpt-5.4 "运行完整测试"
```

安全建议：

- 默认只监听 `127.0.0.1`。
- 不要随意把 daemon 绑定到 `0.0.0.0`。
- 远程访问优先使用 relay 配对。
- 如果确实要暴露端口，需要自行做好网络隔离、访问控制和反向代理安全配置。

## 14. 自定义 provider

配置文件位置：

```bash
~/.paseo/config.json
```

示例：创建一个继承 Claude 的 provider profile。

```json
{
  "version": 1,
  "agents": {
    "providers": {
      "claude-work": {
        "extends": "claude",
        "label": "Claude Work",
        "description": "Work Anthropic account",
        "env": {
          "ANTHROPIC_API_KEY": "sk-ant-..."
        }
      }
    }
  }
}
```

重启 daemon：

```bash
paseo daemon restart
paseo provider ls
```

也可以配置 Anthropic-compatible endpoint，例如 Z.AI、Qwen 等。详细示例见 `docs/CUSTOM-PROVIDERS.md`。

## 15. 语音和听写

Paseo 包含语音输入、听写、TTS 和本地/云端 speech provider 相关能力。服务端示例环境变量包括：

```bash
OPENAI_API_KEY=
TTS_VOICE=alloy
TTS_MODEL=tts-1
PASEO_DICTATION_DEBUG=1
```

如果只使用文本 agent 编排，不需要配置这些变量。

## 16. 从源码运行

克隆仓库：

```bash
git clone https://github.com/xianyu110/paseo
cd paseo
```

安装依赖：

```bash
npm install
```

启动开发环境：

```bash
npm run dev
```

单独启动服务：

```bash
npm run dev:server
npm run dev:app
npm run dev:desktop
npm run dev:website
```

从源码运行 CLI：

```bash
npm run cli -- daemon status
npm run cli -- ls -a
npm run cli -- run --provider codex/gpt-5.4 "测试本地开发版"
```

构建 daemon：

```bash
npm run build:daemon
```

类型检查：

```bash
npm run typecheck
```

运行测试：

```bash
npm run test
```

## 17. 常见问题

### daemon 连不上

检查状态：

```bash
paseo daemon status
```

查看日志：

```bash
tail -n 100 ~/.paseo/daemon.log
```

确认端口：

```bash
paseo --host 127.0.0.1:6767 ls
```

### provider 不可用

检查二进制是否存在：

```bash
which claude
which codex
which opencode
```

检查 Paseo 识别结果：

```bash
paseo provider ls
```

确认 provider 自身已经登录。

### CLI 连到了错误 daemon

显式指定 host：

```bash
paseo --host 127.0.0.1:6767 ls
```

检查 `PASEO_HOME`：

```bash
echo "$PASEO_HOME"
```

### 开发时 CLI 协议异常

如果修改了 server client、WebSocket 协议或 relay 代码，先重新构建：

```bash
npm run build --workspace=@getpaseo/server
npm run build --workspace=@getpaseo/relay
npm run build:daemon
```

### 想隔离测试环境

使用独立 `PASEO_HOME`：

```bash
PASEO_HOME=~/.paseo-blue PASEO_LISTEN=127.0.0.1:7777 npm run dev
```

## 18. 推荐工作流

日常使用建议：

1. 启动 daemon。

```bash
paseo daemon start
```

2. 检查 provider。

```bash
paseo provider ls
```

3. 为较大任务创建 worktree agent。

```bash
paseo run --provider codex/gpt-5.4 --worktree task-name "完成具体任务"
```

4. 实时查看输出。

```bash
paseo attach <agent-id-prefix>
```

5. 处理权限。

```bash
paseo permit ls
paseo permit allow <agent-id-prefix> <request-id-prefix>
```

6. 检查 worktree 里的改动，运行测试后再合并。

这种流程能发挥 Paseo 的主要价值：多 agent 并行、跨设备监控、本地运行、权限可控和 git worktree 隔离。

## 19. 关键路径速查

| 内容 | 默认位置或命令 |
| --- | --- |
| daemon 地址 | `127.0.0.1:6767` |
| 运行数据 | `~/.paseo` |
| daemon 日志 | `~/.paseo/daemon.log` |
| agent 状态 | `~/.paseo/agents` |
| worktree 目录 | `~/.paseo/worktrees` |
| 配置文件 | `~/.paseo/config.json` |
| 启动 daemon | `paseo daemon start` |
| 查看 daemon | `paseo daemon status` |
| 创建 agent | `paseo run "任务"` |
| 查看 agent | `paseo ls` |
| 实时输出 | `paseo attach <id>` |
| 继续任务 | `paseo send <id> "消息"` |
| 权限列表 | `paseo permit ls` |
| provider 列表 | `paseo provider ls` |

