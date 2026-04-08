# CLI Agent Orchestrator 中文使用指南

> CAO（发音 "kay-oh"）是一个轻量级的 AI Agent 编排系统，支持多 Agent 协作、跨提供者编排和定时工作流。

## 目录

- [核心概念](#核心概念)
- [快速开始](#快速开始)
- [CLI 命令详解](#cli-命令详解)
- [Agent Profile 配置](#agent-profile-配置)
- [三种编排模式](#三种编排模式)
- [工具限制与访问控制](#工具限制与访问控制)
- [Provider 支持列表](#provider-支持列表)
- [Web UI 使用](#web-ui-使用)
- [定时工作流 Flows](#定时工作流-flows)
- [API 接口](#api-接口)
- [实战示例](#实战示例)
- [故障排除](#故障排除)

---

## 核心概念

### 架构概览

```
┌─────────────────────────────────────────────────────────────┐
│                        Supervisor Agent                      │
│                    (协调者 - 分配任务)                         │
└─────────────────┬───────────────────┬───────────────────────┘
                  │                   │
        ┌─────────▼─────────┐ ┌───────▼─────────┐
        │   Developer Agent │ │  Reviewer Agent │
        │   (开发者 - 写代码) │ │  (审查者 - 审代码) │
        └───────────────────┘ └─────────────────┘
```

### 关键术语

| 术语 | 说明 |
|------|------|
| **Session** | 一个工作空间，可包含多个 Agent 终端 |
| **Terminal** | 运行在 tmux 中的单个 Agent 实例 |
| **Agent Profile** | Agent 的配置文件（Markdown + YAML frontmatter） |
| **Provider** | AI CLI 工具（如 Claude Code、Kiro CLI 等） |
| **Handoff** | 同步委托 - 等待任务完成并返回结果 |
| **Assign** | 异步委托 - 立即返回，Agent 后台工作 |
| **Flow** | 定时工作流，按 cron 表达式自动执行 |

---

## 快速开始

### 环境要求

```bash
# 检查 Python 版本（需要 3.10+）
python3 --version

# 检查 tmux 版本（需要 3.2+）
tmux -V

# 检查 uv 是否安装
uv --version
```

### 安装步骤

```bash
# 1. 克隆项目
git clone https://github.com/awslabs/cli-agent-orchestrator.git
cd cli-agent-orchestrator

# 2. 安装依赖
uv sync

# 3. 安装 Agent Profiles
uv run cao install code_supervisor
uv run cao install developer
uv run cao install reviewer

# 4. 启动服务器
uv run cao-server

# 5. 另开终端，启动 Supervisor Agent
uv run cao launch --agents code_supervisor
```

### tmux 常用操作

```bash
# 列出所有会话
tmux list-sessions

# 附加到会话
tmux attach -t cao-my-session

# 在 tmux 内分离
Ctrl+b, 然后 d

# 切换窗口
Ctrl+b, 然后 n          # 下一个窗口
Ctrl+b, 然后 p          # 上一个窗口
Ctrl+b, 然后 <number>   # 切换到指定窗口
Ctrl+b, 然后 w          # 列出所有窗口
```

---

## CLI 命令详解

### cao launch - 启动 Agent

```bash
# 基本用法
uv run cao launch --agents <profile_name>

# 指定 Provider
uv run cao launch --agents developer --provider claude_code

# 指定工作目录
uv run cao launch --agents developer --working-directory /path/to/project

# 指定会话名称
uv run cao launch --agents developer --session-name my-project

# 后台模式（不自动附加到 tmux）
uv run cao launch --agents developer --headless

# 跳过确认提示（限制仍然生效）
uv run cao launch --agents developer --auto-approve

# 自定义允许的工具
uv run cao launch --agents developer --allowed-tools execute_bash --allowed-tools fs_read

# 危险模式：取消所有限制
uv run cao launch --agents developer --yolo
```

**参数说明：**

| 参数 | 必需 | 说明 |
|------|------|------|
| `--agents` | ✅ | Agent 配置文件名称 |
| `--provider` | | Provider 类型，默认 kiro_cli |
| `--session-name` | | 自定义会话名称 |
| `--working-directory` | | 工作目录，默认当前目录 |
| `--headless` | | 后台模式，不附加 tmux |
| `--allowed-tools` | | 覆盖允许的工具（可重复） |
| `--auto-approve` | | 跳过确认提示 |
| `--yolo` | | 取消所有限制（危险） |

### cao install - 安装 Agent Profile

```bash
# 安装内置 Agent
uv run cao install developer
uv run cao install reviewer
uv run cao install code_supervisor

# 从本地文件安装
uv run cao install ./my-custom-agent.md

# 从 URL 安装
uv run cao install https://example.com/agents/custom-agent.md

# 安装并设置环境变量
uv run cao install ./agent.md --env API_KEY=xxx --env DEBUG=true
```

### cao flow - 管理定时工作流

```bash
# 添加 Flow
uv run cao flow add examples/flow/morning-trivia.md

# 列出所有 Flow
uv run cao flow list

# 启用/禁用 Flow
uv run cao flow enable morning-trivia
uv run cao flow disable morning-trivia

# 手动运行 Flow
uv run cao flow run morning-trivia

# 删除 Flow
uv run cao flow remove morning-trivia
```

### cao env - 管理环境变量

```bash
# 设置环境变量
uv run cao env set ANTHROPIC_API_KEY sk-xxx

# 获取环境变量
uv run cao env get ANTHROPIC_API_KEY

# 列出所有环境变量
uv run cao env list

# 删除环境变量
uv run cao env unset ANTHROPIC_API_KEY
```

### cao shutdown - 关闭会话

```bash
# 关闭所有 CAO 会话
uv run cao shutdown --all

# 关闭指定会话
uv run cao shutdown --session cao-my-session
```

### cao info - 显示信息

```bash
uv run cao info
```

---

## Agent Profile 配置

### 配置文件结构

Agent Profile 是带有 YAML frontmatter 的 Markdown 文件：

```markdown
---
name: my_agent                    # 必需：唯一标识符
description: "我的自定义 Agent"     # 必需：描述
role: developer                   # 可选：角色（supervisor/developer/reviewer）
provider: claude_code             # 可选：指定 Provider
allowedTools:                     # 可选：工具白名单
  - "@builtin"
  - "fs_*"
  - "execute_bash"
  - "@cao-mcp-server"
mcpServers:                       # 可选：MCP 服务器配置
  cao-mcp-server:
    type: stdio
    command: uvx
    args:
      - "--from"
      - "git+https://github.com/awslabs/cli-agent-orchestrator.git@main"
      - "cao-mcp-server"
model: "claude-sonnet-4"          # 可选：指定模型
---

# System Prompt

你是一个专业的软件开发助手。你的任务是：

1. 编写高质量的代码
2. 遵循项目的编码规范
3. 编写必要的单元测试

## 工作方式

- 收到任务后，先分析需求
- 编写代码实现
- 运行测试验证
- 提交结果报告
```

### 内置角色

| 角色 | 默认工具 | 适用场景 |
|------|---------|---------|
| `supervisor` | `@cao-mcp-server`, `fs_read`, `fs_list` | 协调多个 Agent，分配任务 |
| `developer` | `@builtin`, `fs_*`, `execute_bash`, `@cao-mcp-server` | 编写代码，执行命令 |
| `reviewer` | `@builtin`, `fs_read`, `fs_list`, `@cao-mcp-server` | 代码审查，只读访问 |

### 工具命名空间

| 命名空间 | 说明 |
|---------|------|
| `@builtin` | 内置工具 |
| `fs_*` | 文件系统操作（read/write/list） |
| `execute_bash` | Bash 命令执行 |
| `@cao-mcp-server` | CAO MCP 服务器工具（handoff/assign/send_message） |

### 配置文件存放位置

```
~/.aws/cli-agent-orchestrator/
├── agent-store/           # 本地安装的 Agent
│   ├── developer.md
│   └── reviewer.md
├── agent-context/         # cao install 安装的 Agent
│   └── my-custom.md
└── settings.json          # 全局设置
```

### 跨 Provider 配置

在 Agent Profile 中指定 `provider` 字段，可以让不同 Agent 使用不同的 AI 提供者：

```markdown
---
name: supervisor
provider: kiro_cli
---

我是运行在 Kiro CLI 上的主管。
```

```markdown
---
name: developer
provider: claude_code
---

我是运行在 Claude Code 上的开发者。
```

---

## 三种编排模式

### Handoff（同步委托）

**特点：** 阻塞等待，适合需要立即获得结果的顺序任务。

```python
# 主管调用
result = await handoff(
    agent_profile="reviewer",
    message="请审查 src/auth.py 文件的代码质量",
    timeout=300  # 可选，超时时间（秒）
)
# 等待 reviewer 完成后才会继续执行
print(result)  # reviewer 的输出结果
```

**工作流程：**

```
Supervisor                Reviewer Agent
    │                          │
    │──── handoff ────────────>│
    │                          │ (执行任务)
    │                          │ (完成后自动退出)
    │<─── 返回结果 ────────────│
    │                          │
    ▼ (继续执行)
```

### Assign（异步委托）

**特点：** 立即返回，Agent 在后台工作，适合并行执行独立任务。

```python
# 主管调用
terminal_id = await assign(
    agent_profile="data_analyst",
    message="""
    分析 /data/sales.csv 文件。
    完成后将结果发送给 supervisor（terminal_id: {supervisor_id}）
    """
)
# 立即返回，不等待完成
print(f"任务已分配给: {terminal_id}")

# Agent 完成后会通过 send_message 发送结果
```

**工作流程：**

```
Supervisor                Analyst 1            Analyst 2
    │                        │                     │
    │── assign ─────────────>│                     │
    │── assign ──────────────────────────────────>│
    │                        │                     │
    ▼ (继续做其他事)          │ (后台工作)           │ (后台工作)
    │                        │                     │
    │<── send_message ───────│                     │
    │<── send_message ───────────────────────────│
    │                        │                     │
    ▼ (聚合所有结果)
```

### Send Message（消息传递）

**特点：** Agent 之间的异步消息传递，消息在接收方空闲时自动送达。

```python
# 发送消息给指定终端
await send_message(
    receiver_id="terminal_abc123",
    message="分析结果：平均值=3.0，中位数=3.0"
)
```

**消息递送机制：**

1. 消息保存到接收方的收件箱
2. 系统监控接收方终端状态
3. 当接收方变为 IDLE 状态时，消息自动递送
4. 按 FIFO 顺序处理队列中的消息

**重要提示：** 不要使用 `sleep` 或 `echo` 等待结果，这会阻塞消息递送。

---

## 工具限制与访问控制

### 三层控制机制

```
┌─────────────────────────────────────────────────┐
│                  role（高层）                     │
│    supervisor / developer / reviewer            │
│              ↓ 映射到预设工具列表                  │
├─────────────────────────────────────────────────┤
│              allowedTools（低层）                 │
│    execute_bash, fs_read, @cao-mcp-server       │
│              ↓ 细粒度覆盖                         │
├─────────────────────────────────────────────────┤
│              --yolo（逃生舱）                     │
│         绕过所有限制 + 跳过确认                    │
└─────────────────────────────────────────────────┘
```

### 启动选项对比

| 选项 | 确认提示 | 工具限制 | 适用场景 |
|------|---------|---------|---------|
| 默认 | ✅ 显示 | ✅ 生效 | 生产环境 |
| `--auto-approve` | ❌ 跳过 | ✅ 生效 | 自动化脚本 |
| `--yolo` | ❌ 跳过 | ❌ 无限制 | 本地开发（危险） |

### 自定义角色

在 `~/.aws/cli-agent-orchestrator/settings.json` 中定义：

```json
{
  "roles": {
    "data_analyst": ["fs_read", "execute_bash", "@cao-mcp-server"],
    "secure_dev": ["fs_read", "fs_write", "@cao-mcp-server"],
    "readonly": ["fs_read", "fs_list"]
  }
}
```

使用自定义角色：

```markdown
---
name: analyst
role: data_analyst
---

数据分析 Agent
```

---

## Provider 支持列表

| Provider | 二进制文件 | 认证方式 | 文档 |
|----------|-----------|---------|------|
| **Kiro CLI** | `kiro-cli` | AWS 凭证 | [docs/kiro-cli.md](kiro-cli.md) |
| **Claude Code** | `claude` | Anthropic API Key | [docs/claude-code.md](claude-code.md) |
| **Codex CLI** | `codex` | OpenAI API Key | [docs/codex-cli.md](codex-cli.md) |
| **Gemini CLI** | `gemini` | Google AI API Key | [docs/gemini-cli.md](gemini-cli.md) |
| **Kimi CLI** | `kimi` | Moonshot API Key | [docs/kimi-cli.md](kimi-cli.md) |
| **Copilot CLI** | `copilot` | GitHub 认证 | [docs/copilot-cli.md](copilot-cli.md) |
| **Q CLI** | `q` | AWS 凭证 | AWS 官方文档 |

### 安装 Provider

```bash
# Claude Code
npm install -g @anthropic-ai/claude-code

# Gemini CLI
npm install -g @anthropic-ai/gemini-cli

# Codex CLI
npm install -g @openai/codex

# Kimi CLI
pip install kimi-cli

# Kiro CLI - 参考 https://kiro.dev/docs/kiro-cli

# Copilot CLI
gh extension install github/gh-copilot
```

### 查看已安装的 Provider

```bash
# 通过 API 查看
curl http://localhost:9889/agents/providers

# 或通过 Web UI 查看
```

---

## Web UI 使用

### 启动方式

**开发模式（热重载）：**

```bash
# 终端 1：启动后端
uv run cao-server

# 终端 2：启动前端
cd web
npm install
npm run dev  # 访问 http://localhost:5173
```

**生产模式：**

```bash
# 构建前端
cd web
npm install && npm run build

# 启动后端（自动服务前端）
cd ..
uv run cao-server  # 访问 http://localhost:9889
```

### 功能模块

#### 1. Sessions 面板

- 查看所有活跃会话
- 创建新会话（选择 Provider + Agent Profile）
- 删除会话
- 查看会话中的所有终端

#### 2. Terminals 管理

- 打开实时终端（xterm.js）
- 查看终端输出
- 发送输入消息
- 优雅退出（发送 `/exit` 命令）
- 强制关闭（终止 tmux 窗口）

#### 3. Inbox 消息

- 查看发送给当前终端的消息
- 查看消息状态（pending/delivered/failed）

#### 4. Flows 管理

- 添加定时工作流
- 启用/禁用 Flow
- 手动触发 Flow
- 查看下次运行时间

#### 5. Settings 配置

- 配置各 Provider 的 Agent 目录
- 添加额外的 Agent 搜索目录

---

## 定时工作流 Flows

### Flow 配置文件

```markdown
---
name: daily-standup
schedule: "0 9 * * 1-5"    # 工作日早上 9 点
agent_profile: developer
provider: kiro_cli         # 可选
script: ./check-tasks.sh   # 可选：条件执行脚本
---

请生成今天的站会报告：
1. 查看昨天的 git 提交
2. 总结完成的工作
3. 列出今天计划的任务
```

### Cron 表达式说明

```
┌───────────── 分钟 (0-59)
│ ┌───────────── 小时 (0-23)
│ │ ┌───────────── 日期 (1-31)
│ │ │ ┌───────────── 月份 (1-12)
│ │ │ │ ┌───────────── 星期 (0-6, 0=周日)
│ │ │ │ │
* * * * *
```

**常用示例：**

| 表达式 | 说明 |
|--------|------|
| `0 9 * * *` | 每天早上 9:00 |
| `0 9 * * 1-5` | 工作日早上 9:00 |
| `*/30 * * * *` | 每 30 分钟 |
| `0 0 * * 0` | 每周日午夜 |
| `0 9 1 * *` | 每月 1 号早上 9:00 |

### 条件执行脚本

脚本输出 JSON 决定是否执行：

```bash
#!/bin/bash
# health-check.sh

STATUS=$(curl -s -o /dev/null -w "%{http_code}" "https://api.example.com/health")

if [ "$STATUS" != "200" ]; then
  # 服务异常，执行 Flow
  echo '{"execute": true, "output": {"status_code": "'$STATUS'"}}'
else
  # 服务正常，跳过
  echo '{"execute": false, "output": {}}'
fi
```

在 Flow 中使用脚本输出：

```markdown
---
name: monitor-service
schedule: "*/5 * * * *"
agent_profile: developer
script: ./health-check.sh
---

服务状态异常（HTTP [[status_code]]）。
请调查并修复问题。
```

---

## API 接口

### 基础信息

- **Base URL**: `http://localhost:9889`
- **响应格式**: JSON

### 健康检查

```bash
GET /health

# 响应
{"status": "ok", "service": "cli-agent-orchestrator"}
```

### 会话管理

```bash
# 创建会话
POST /sessions?provider=kiro_cli&agent_profile=developer&working_directory=/path

# 响应
{
  "id": "abcd1234",
  "session_name": "cao-session-1",
  "provider": "kiro_cli",
  "agent_profile": "developer"
}

# 列出会话
GET /sessions

# 获取会话详情
GET /sessions/{session_name}

# 删除会话
DELETE /sessions/{session_name}
```

### 终端管理

```bash
# 获取终端信息
GET /terminals/{terminal_id}

# 发送输入
POST /terminals/{terminal_id}/input?message=hello

# 获取输出
GET /terminals/{terminal_id}/output?mode=full

# 优雅退出
POST /terminals/{terminal_id}/exit

# 强制删除
DELETE /terminals/{terminal_id}

# 获取工作目录
GET /terminals/{terminal_id}/working-directory
```

### 消息传递

```bash
# 发送消息到收件箱
POST /terminals/{receiver_id}/inbox/messages?sender_id=xxx&message=hello

# 获取收件箱消息
GET /terminals/{terminal_id}/inbox/messages?limit=10&status=pending
```

### WebSocket 实时终端

```javascript
const ws = new WebSocket('ws://localhost:9889/terminals/{id}/ws');

// 发送输入
ws.send(JSON.stringify({ type: 'input', data: 'hello\n' }));

// 发送窗口大小变化
ws.send(JSON.stringify({ type: 'resize', rows: 24, cols: 80 }));

// 接收输出
ws.onmessage = (event) => {
  // event.data 是 ArrayBuffer
};
```

### Provider 信息

```bash
GET /agents/providers

# 响应
[
  {"name": "kiro_cli", "binary": "kiro-cli", "installed": true},
  {"name": "claude_code", "binary": "claude", "installed": false}
]
```

---

## 实战示例

### 示例 1：并行数据分析

**场景：** 主管分配 3 个数据分析任务，并行执行后汇总结果。

**Agent 配置：**

```markdown
<!-- analysis_supervisor.md -->
---
name: analysis_supervisor
role: supervisor
mcpServers:
  cao-mcp-server:
    type: stdio
    command: uvx
    args:
      - "--from"
      - "git+https://github.com/awslabs/cli-agent-orchestrator.git@main"
      - "cao-mcp-server"
---

你是数据分析主管。你的任务是：

1. 将数据分析任务分配给 3 个分析师（使用 assign）
2. 使用 handoff 让报告生成器生成报告
3. 汇总所有分析师的结果
```

```markdown
<!-- data_analyst.md -->
---
name: data_analyst
role: developer
---

你是数据分析师。分析指定的数据集后，将结果发送给主管。
```

**执行：**

```bash
uv run cao launch --agents analysis_supervisor
```

### 示例 2：代码审查工作流

**场景：** 开发者写完代码后，自动触发审查。

**工作流程：**

```
Developer → Handoff → Reviewer → 返回审查结果
```

**Agent 配置：**

```markdown
---
name: code_reviewer
role: reviewer
---

你是代码审查员。审查指定的代码变更，提供：

1. 代码质量评分
2. 潜在问题列表
3. 改进建议
```

**调用方式：**

```python
# 在 Developer Agent 中调用
result = await handoff(
    agent_profile="code_reviewer",
    message="请审查 src/auth.py 的最近变更",
    timeout=300
)
```

### 示例 3：定时监控

**场景：** 每 5 分钟检查服务健康状态，异常时自动通知。

**Flow 配置：**

```markdown
---
name: monitor-api
schedule: "*/5 * * * *"
agent_profile: developer
script: ./check-api.sh
---

API 服务状态异常！
HTTP 状态码：[[status_code]]
请立即检查并修复。
```

**脚本：**

```bash
#!/bin/bash
# check-api.sh

STATUS=$(curl -s -o /dev/null -w "%{http_code}" "https://api.example.com/health")

if [ "$STATUS" == "200" ]; then
  echo '{"execute": false, "output": {}}'
else
  echo '{"execute": true, "output": {"status_code": "'$STATUS'"}}'
fi
```

---

## 故障排除

### 常见问题

#### 1. 终端状态一直显示 PROCESSING

**原因：** Agent 可能卡在等待用户输入或命令执行中。

**解决：**
```bash
# 附加到 tmux 查看
tmux attach -t cao-session-name

# 或通过 Web UI 打开终端查看
```

#### 2. 消息没有送达

**原因：** 接收方终端处于非 IDLE 状态。

**解决：** 等待接收方完成当前任务，消息会在空闲时自动送达。

#### 3. Agent 无法执行某些命令

**原因：** 工具限制生效。

**解决：**
```bash
# 检查 Agent 的 role 和 allowedTools
# 或使用 --yolo 取消限制（仅限本地开发）
uv run cao launch --agents developer --yolo
```

#### 4. Provider 未安装

**症状：** API 返回 `installed: false`

**解决：**
```bash
# 安装对应的 CLI 工具
npm install -g @anthropic-ai/claude-code  # Claude Code
pip install kimi-cli                       # Kimi CLI
```

#### 5. 环境变量未生效

**解决：**
```bash
# 检查环境变量
uv run cao env list

# 设置环境变量
uv run cao env set ANTHROPIC_API_KEY sk-xxx
```

### 日志位置

```bash
# 终端输出日志
~/.cao/terminal-logs/

# 服务器日志（终端输出）
# cao-server 的标准输出
```

### 重置环境

```bash
# 关闭所有会话
uv run cao shutdown --all

# 清理数据库
rm -rf ~/.aws/cli-agent-orchestrator/terminal.db

# 重新初始化
uv run cao init
```

---

## 附录

### 相关文档

- [开发指南（中文）](../DEVELOPMENT_CN.md)
- [Agent Profile 详细配置](agent-profile.md)
- [工具限制参考](tool-restrictions.md)
- [API 完整文档](api.md)
- [设置指南](settings.md)

### 示例代码

- [并行数据分析](../examples/assign/)
- [跨 Provider 编排](../examples/cross-provider/)
- [定时工作流](../examples/flow/)

### 获取帮助

- GitHub Issues: https://github.com/awslabs/cli-agent-orchestrator/issues
- 项目文档: https://github.com/awslabs/cli-agent-orchestrator#readme
