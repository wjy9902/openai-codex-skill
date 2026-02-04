---
name: openai-codex
description: OpenAI Codex CLI - 编程 agent，可读取、编辑、运行代码。用于：(1) 代码生成和重构，(2) Bug 修复和调试，(3) 代码审查，(4) 理解陌生代码库，(5) 自动化开发任务。触发词：codex、编程 agent、代码生成、代码审查、重构、调试。
metadata:
  {
    "openclaw": { "emoji": "🤖", "requires": { "anyBins": ["codex"] } }
  }
---

# OpenAI Codex CLI

Codex 是 OpenAI 的编程 agent，可在本地终端运行，能读取、修改、执行代码。

## 安装

```bash
npm i -g @openai/codex
```

首次运行会提示登录（ChatGPT 账号或 API Key）。

## 核心命令

### 交互模式

```bash
codex                           # 启动交互式 TUI
codex "解释这个代码库"           # 带初始 prompt 启动
codex resume                    # 恢复之前的会话
codex resume --last             # 恢复最近一次会话
```

### 非交互模式 (exec)

```bash
codex exec "修复 CI 失败"                    # 一次性执行
codex exec --json "分析代码"                 # 输出 JSONL 格式
codex exec -o result.md "生成文档"           # 输出到文件
codex exec --output-schema schema.json "..."  # 指定输出格式
```

### 代码审查

```bash
codex review                    # 交互式审查
codex exec review               # 非交互式审查
```

## 关键参数

| 参数 | 说明 |
|------|------|
| `-m, --model <MODEL>` | 指定模型 (默认 gpt-5-codex) |
| `-s, --sandbox <MODE>` | 沙箱模式: read-only, workspace-write, danger-full-access |
| `-a, --ask-for-approval <POLICY>` | 审批策略: untrusted, on-failure, on-request, never |
| `--full-auto` | 等同于 `-a on-request --sandbox workspace-write` |
| `--yolo` | 跳过所有确认和沙箱 (危险!) |
| `-C, --cd <DIR>` | 指定工作目录 |
| `--add-dir <DIR>` | 添加额外可写目录 |
| `-i, --image <FILE>` | 附加图片 |
| `--search` | 启用实时网络搜索 |
| `-p, --profile <NAME>` | 使用配置文件中的 profile |

## 模型选择

```bash
codex -m gpt-5.2-codex          # 最新最强 (默认)
codex -m gpt-5.1-codex-mini     # 更快更便宜
codex -m gpt-5.2                # 通用 agent 模型
```

交互模式中用 `/model` 切换模型。

## 审批模式

| 模式 | 说明 |
|------|------|
| `untrusted` | 只自动运行可信命令 (ls, cat 等) |
| `on-failure` | 命令失败时才询问 |
| `on-request` | 模型决定何时询问 (推荐) |
| `never` | 从不询问 (配合沙箱使用) |

## 沙箱模式

| 模式 | 说明 |
|------|------|
| `read-only` | 只读，不能修改或执行 |
| `workspace-write` | 可在工作目录内读写执行 (默认) |
| `danger-full-access` | 完全访问，无限制 (危险!) |

## 配置文件

位置: `~/.codex/config.toml`

```toml
# 默认模型
model = "gpt-5.2-codex"

# 审批策略
approval_policy = "on-request"

# 沙箱模式
sandbox_mode = "workspace-write"

# 网络搜索
web_search = "cached"  # cached | live | disabled

# 推理强度
model_reasoning_effort = "high"  # low | medium | high

# MCP 服务器
[mcp_servers.context7]
command = "npx"
args = ["-y", "@upstash/context7-mcp"]

# Profile 预设
[profiles.safe]
approval_policy = "untrusted"
sandbox_mode = "read-only"

[profiles.yolo]
approval_policy = "never"
sandbox_mode = "danger-full-access"
```

## MCP 集成

```bash
# 添加 MCP 服务器
codex mcp add context7 -- npx -y @upstash/context7-mcp

# 查看已配置的 MCP
codex mcp list

# 交互模式中查看
/mcp
```

## SDK 使用 (TypeScript)

```bash
npm install @openai/codex-sdk
```

```typescript
import { Codex } from "@openai/codex-sdk";

const codex = new Codex();
const thread = codex.startThread();

// 执行任务
const result = await thread.run("修复 CI 失败");
console.log(result);

// 继续同一线程
const result2 = await thread.run("实现修复方案");

// 恢复历史线程
const thread2 = codex.resumeThread("<thread-id>");
await thread2.run("继续之前的工作");
```

## 在 OpenClaw 中使用

### 一次性任务

```bash
# 简单任务 - 需要 PTY!
bash pty:true workdir:~/project command:"codex exec --full-auto '添加错误处理'"

# 快速聊天 (Codex 需要 git 仓库)
SCRATCH=$(mktemp -d) && cd $SCRATCH && git init && codex exec "你的问题"
```

### 后台长任务

```bash
# 启动后台任务
bash pty:true workdir:~/project background:true command:"codex exec --full-auto '重构认证模块'"

# 监控进度
process action:log sessionId:XXX

# 检查状态
process action:poll sessionId:XXX

# 发送输入
process action:submit sessionId:XXX data:"yes"

# 终止任务
process action:kill sessionId:XXX
```

### 并行处理多个 Issue

```bash
# 创建 worktree
git worktree add -b fix/issue-78 /tmp/issue-78 main
git worktree add -b fix/issue-99 /tmp/issue-99 main

# 并行启动
bash pty:true workdir:/tmp/issue-78 background:true command:"codex --yolo 'Fix issue #78'"
bash pty:true workdir:/tmp/issue-99 background:true command:"codex --yolo 'Fix issue #99'"

# 监控
process action:list

# 清理
git worktree remove /tmp/issue-78
```

### 完成后自动通知

```bash
bash pty:true workdir:~/project background:true command:"codex --yolo exec '构建 REST API。

完成后运行: openclaw gateway wake --text \"Done: REST API 构建完成\" --mode now'"
```

## 交互模式快捷键

| 操作 | 说明 |
|------|------|
| `@` | 模糊搜索文件并插入路径 |
| `Enter` (运行中) | 注入新指令到当前轮次 |
| `Tab` (运行中) | 排队下一轮 prompt |
| `!command` | 运行本地 shell 命令 |
| `Esc Esc` | 编辑上一条消息 |
| `Ctrl+G` | 打开外部编辑器 |
| `Ctrl+C` / `/exit` | 退出 |

## Slash 命令

| 命令 | 说明 |
|------|------|
| `/model` | 切换模型 |
| `/permissions` | 切换审批模式 |
| `/review` | 代码审查 |
| `/fork` | 从某点分叉会话 |
| `/mcp` | 查看 MCP 服务器 |
| `/status` | 查看当前状态 |
| `/exit` | 退出 |

## 最佳实践

1. **PTY 必须**: 编程 agent 是交互式终端应用，必须用 `pty:true`
2. **Git 仓库**: Codex 只在 git 目录中运行，临时任务用 `mktemp -d && git init`
3. **workdir 隔离**: 指定工作目录，避免 agent 读取无关文件
4. **--full-auto 构建**: 自动批准工作区内的更改
5. **vanilla 审查**: 代码审查不需要特殊参数
6. **并行安全**: 可同时运行多个 Codex 进程
7. **进度更新**: 后台任务要定期汇报进度

## 常见工作流

### 理解代码库

```bash
codex "解释这个代码库的架构和主要模块"
```

### Bug 修复

```bash
codex exec --full-auto "Bug: 点击保存后数据没有持久化。
复现步骤:
1. npm run dev
2. 访问 /settings
3. 修改设置并保存
4. 刷新页面，设置丢失

约束: 不改 API 接口，最小化修改，添加回归测试。"
```

### 代码审查

```bash
# 审查当前工作区
codex review

# 审查特定 PR
git fetch origin pull/123/head:pr-123
git checkout pr-123
codex review --base origin/main
```

### 从设计稿生成代码

```bash
codex -i design.png "根据这个设计稿创建页面。
使用 React + Tailwind，匹配间距和排版。"
```

### 重构

```bash
codex exec --full-auto "重构 auth 模块:
- 分离职责 (token 解析 / session 加载 / 权限检查)
- 减少循环依赖
- 提高可测试性
- 保持公共 API 稳定"
```

## 参考链接

- 官方文档: https://developers.openai.com/codex/
- GitHub: https://github.com/openai/codex
- SDK: https://github.com/openai/codex/tree/main/sdk/typescript
