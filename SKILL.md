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

## ⚠️ 常见问题与解决方案

### 401 Unauthorized 错误

**原因：**
- ChatGPT 登录的 token 有效期约 1 小时
- refresh token 只能用一次，多个 session 会冲突
- 用 Google 登录更容易遇到这个问题（已知 bug）

**解决方案：**
1. 每次使用前检查登录状态：`codex login status`
2. 如果失效，重新登录：`codex auth login`
3. 设置自动刷新脚本（每 30 分钟运行 `codex login status`）
4. 或者使用 API Key（不会过期）：`export OPENAI_API_KEY="sk-xxx"`

### CODEX_HOME 环境变量陷阱

**问题：** 设置 `CODEX_HOME=$(pwd)/.codex` 会导致 Codex 读取项目目录的 auth.json，而不是 `~/.codex/auth.json`，造成 401 错误。

**正确用法：**
```bash
# ❌ 错误 - 会导致 401（除非项目 .codex 里也有 auth.json）
export CODEX_HOME=$(pwd)/.codex && codex exec ...

# ✅ 正确 - 使用默认 ~/.codex/ 目录
cd /path/to/project && codex exec --full-auto "任务描述"
```

**何时需要 CODEX_HOME：**
- 只有使用 Spec-Kit 的 slash 命令（/speckit.xxx）时才需要
- 此时要确保 auth.json 也复制到项目的 .codex 目录

### Token 自动刷新脚本

```bash
#!/bin/bash
# 保存为 codex-token-refresh.sh，每 30 分钟运行

export PATH="/opt/homebrew/bin:$PATH"
LOG_FILE="$HOME/.codex/token-refresh.log"

echo "[$(date '+%Y-%m-%d %H:%M:%S')] 检查 Codex 登录状态..." >> "$LOG_FILE"

STATUS=$(codex login status 2>&1)

if echo "$STATUS" | grep -q "Logged in"; then
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] ✅ Token 有效" >> "$LOG_FILE"
else
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] ❌ Token 失效，需要重新登录" >> "$LOG_FILE"
fi
```

macOS launchd 配置：
```xml
<!-- ~/Library/LaunchAgents/com.codex.token-refresh.plist -->
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>com.codex.token-refresh</string>
    <key>ProgramArguments</key>
    <array>
        <string>/path/to/codex-token-refresh.sh</string>
    </array>
    <key>StartInterval</key>
    <integer>1800</integer>
    <key>RunAtLoad</key>
    <true/>
</dict>
</plist>
```

### npm install 超时问题

**问题：** Codex exec 运行 `npm install` 时可能超时（默认 10 秒 yieldMs）

**解决方案：**
1. 在调用 Codex 前手动安装依赖：`npm install package-name`
2. 或者增加 exec 的 yieldMs 超时时间
3. 或者在 Codex 任务描述中说明"假设依赖已安装"

### 多文件修改的最佳实践

**经验：** Codex 擅长一次性修改多个相关文件，但要在 prompt 中明确说明：
- 需要修改哪些文件
- 每个文件的具体改动
- 改动之间的依赖关系

**示例：**
```bash
codex exec --full-auto "实现照片点击放大功能：
1. 在 record-detail.tsx 添加 ImageViewing 组件
2. 用 TouchableOpacity 包裹每张照片
3. 添加 imageViewerVisible 和 imageIndex 状态
4. 同时修改 worker 和 admin 两个版本的 record-detail.tsx"
```
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

## Spec-Kit 工作流 (规范驱动开发)

Spec-Kit 是 GitHub 的规范驱动开发工具包，让 Codex 按结构化流程开发，而非 vibe coding。

### 安装 Spec-Kit

```bash
uv tool install specify-cli --from git+https://github.com/github/spec-kit.git
```

### 初始化项目

```bash
# 新项目
specify init my-project --ai codex

# 现有项目
cd existing-project
specify init . --ai codex --force
```

初始化后会创建:
- `.codex/prompts/` - Slash 命令定义
- `.specify/` - 规范、模板、脚本

### Spec-Kit Slash 命令

在 Codex 交互模式中使用:

| 命令 | 说明 |
|------|------|
| `/speckit.constitution` | 建立项目原则和开发规范 |
| `/speckit.specify` | 定义要构建什么 (需求和用户故事) |
| `/speckit.clarify` | 澄清模糊需求 (可选，在 plan 前) |
| `/speckit.plan` | 创建技术实现计划 (指定技术栈) |
| `/speckit.tasks` | 生成可执行的任务列表 |
| `/speckit.analyze` | 交叉检查一致性 (可选，在 implement 前) |
| `/speckit.implement` | 执行所有任务，构建功能 |

### 完整工作流示例

```bash
# 1. 初始化项目
specify init photo-app --ai codex
cd photo-app

# 2. 启动 Codex (设置 CODEX_HOME)
export CODEX_HOME=$(pwd)/.codex
codex

# 3. 在 Codex 中依次执行:

# 建立项目原则
/speckit.constitution Create principles focused on code quality, testing, UX consistency, and performance.

# 定义需求 (关注 what 和 why，不关注技术栈)
/speckit.specify Build a photo album app. Users can organize photos into albums grouped by date. Albums can be reordered via drag-and-drop. Photos display in a tile grid within each album.

# 澄清需求 (可选)
/speckit.clarify

# 创建技术计划 (指定技术栈)
/speckit.plan Use Vite + React + TypeScript. Store metadata in local SQLite. No image upload, local files only.

# 生成任务列表
/speckit.tasks

# 检查一致性 (可选)
/speckit.analyze

# 执行实现
/speckit.implement
```

### 目录结构

初始化后的项目结构:

```
project/
├── .codex/
│   └── prompts/           # Slash 命令定义
│       ├── speckit.constitution.md
│       ├── speckit.specify.md
│       ├── speckit.plan.md
│       ├── speckit.tasks.md
│       └── speckit.implement.md
├── .specify/
│   ├── memory/
│   │   └── constitution.md  # 项目原则
│   ├── scripts/             # 辅助脚本
│   ├── specs/               # 功能规范
│   │   └── 001-feature/
│   │       ├── spec.md      # 需求规范
│   │       ├── plan.md      # 实现计划
│   │       └── tasks.md     # 任务列表
│   └── templates/           # 模板文件
└── ...
```

### 在 OpenClaw 中使用 Spec-Kit

```bash
# 后台运行完整工作流
bash pty:true workdir:~/my-project background:true command:"
export CODEX_HOME=\$(pwd)/.codex
codex exec --full-auto '
/speckit.constitution Create principles for a production-ready web app.
/speckit.specify Build a task management app with Kanban boards.
/speckit.plan Use Next.js 14, Prisma, PostgreSQL.
/speckit.tasks
/speckit.implement

When finished, run: openclaw gateway wake --text \"Done: Spec-Kit workflow complete\" --mode now
'"
```

### 注意事项

1. **CODEX_HOME**: 必须设置为项目的 `.codex` 目录，否则 slash 命令不可用
2. **英文 Prompt**: 和 Codex 交互时使用英文
3. **先 specify 后 plan**: 需求定义不要涉及技术栈，技术栈在 plan 阶段指定
4. **迭代优化**: 每个阶段都可以和 Codex 对话优化，不要把第一次输出当最终版

## OpenSpec 工作流 (推荐)

OpenSpec 是比 Spec-Kit 更轻量、更灵活的规范驱动开发框架。支持 20+ AI 编程工具，包括 Codex。

### 为什么选 OpenSpec 而非 Spec-Kit

| 特性 | OpenSpec | Spec-Kit |
|------|----------|----------|
| 安装 | npm 一键安装 | Python + uv |
| 工作流 | 灵活迭代，随时更新 | 严格阶段门控 |
| 配置 | 自动检测工具 | 手动配置 |
| 维护 | 活跃更新 | GitHub 官方但较重 |

### 安装 OpenSpec

```bash
npm install -g @fission-ai/openspec@latest
```

### 初始化项目

```bash
cd your-project
openspec init --tools codex
```

这会创建：
- `.codex/skills/` - Codex 技能文件
- `~/.codex/prompts/` - 全局 slash 命令
- `openspec/` - 规范和变更目录

### OpenSpec Slash 命令

| 命令 | 说明 |
|------|------|
| `/opsx:new` | 创建新的变更 |
| `/opsx:explore` | 探索想法，调研问题 |
| `/opsx:continue` | 创建下一个 artifact |
| `/opsx:ff` | 快速生成所有规划文档 |
| `/opsx:apply` | 执行任务，实现代码 |
| `/opsx:verify` | 验证实现是否符合规范 |
| `/opsx:archive` | 归档完成的变更 |

### 完整工作流示例

```bash
# 1. 初始化项目
cd my-project
git init
openspec init --tools codex

# 2. 创建新变更
codex exec --full-auto "/opsx:new add-user-auth"

# 3. 快速生成所有规划文档 (proposal → design → specs → tasks)
codex exec --full-auto "/opsx:ff add-user-auth

实现用户认证功能：
- JWT token 认证
- 登录/注册 API
- 密码加密存储"

# 4. 执行实现
codex exec --full-auto "/opsx:apply add-user-auth"

# 5. 验证并归档
codex exec --full-auto "/opsx:verify add-user-auth"
codex exec --full-auto "/opsx:archive add-user-auth"
```

### 生成的目录结构

```
project/
├── .codex/
│   └── skills/                    # Codex 技能文件
│       ├── openspec-new-change/
│       ├── openspec-ff-change/
│       ├── openspec-apply-change/
│       └── ...
├── openspec/
│   ├── changes/                   # 进行中的变更
│   │   └── add-user-auth/
│   │       ├── .openspec.yaml     # 变更元数据
│   │       ├── proposal.md        # 为什么做
│   │       ├── design.md          # 怎么做
│   │       ├── specs/             # 需求规范
│   │       │   └── user-auth/
│   │       │       └── spec.md
│   │       └── tasks.md           # 任务清单
│   ├── specs/                     # 主规范 (归档后合并)
│   └── config.yaml                # 项目配置
└── ...
```

### 在 OpenClaw 中使用 OpenSpec

```bash
# 后台运行完整工作流
cd ~/my-project && codex exec --full-auto "
/opsx:new add-dark-mode

/opsx:ff add-dark-mode
实现暗色模式：
- 主题切换按钮
- localStorage 持久化
- 跟随系统偏好

/opsx:apply add-dark-mode

完成后运行: openclaw gateway wake --text 'Done: 暗色模式实现完成' --mode now
"
```

### OpenSpec vs Spec-Kit 选择建议

- **选 OpenSpec**: 快速迭代、灵活工作流、不想装 Python
- **选 Spec-Kit**: 需要严格阶段控制、已有 Python 环境、偏好 GitHub 官方工具

### 注意事项

1. **不需要 CODEX_HOME**: OpenSpec 的 prompts 安装到全局 `~/.codex/prompts/`，不需要设置环境变量
2. **先 init 后使用**: 每个项目需要运行 `openspec init --tools codex`
3. **中文描述**: OpenSpec 支持中文描述需求，Codex 会正确理解
4. **灵活迭代**: 可以随时更新 proposal/design/specs，不像 Spec-Kit 有严格阶段门控

## 参考链接

- 官方文档: https://developers.openai.com/codex/
- GitHub: https://github.com/openai/codex
- SDK: https://github.com/openai/codex/tree/main/sdk/typescript
- Spec-Kit: https://github.com/github/spec-kit
- OpenSpec: https://github.com/Fission-AI/OpenSpec
