# CLI Agent Orchestrator 开发指南（中文版）

本文档帮助你快速搭建本地开发环境，进行 CAO 项目的开发和测试。

## 环境要求

| 依赖 | 版本要求 | 说明 |
|------|---------|------|
| Python | ≥ 3.10 | 核心运行时 |
| uv | 最新版 | Python 包管理器（替代 pip） |
| tmux | ≥ 3.2 | 终端会话管理 |
| Node.js | ≥ 18 | Web UI 开发（可选） |
| Git | 最新版 | 版本控制 |

## 快速开始

### 1. 克隆仓库

```bash
git clone https://github.com/awslabs/cli-agent-orchestrator.git
cd cli-agent-orchestrator/
```

### 2. 安装依赖

```bash
# 使用 uv 自动创建虚拟环境并安装所有依赖
uv sync
```

### 3. 验证安装

```bash
# 检查 CLI 是否可用
uv run cao --help

# 快速测试
uv run pytest test/providers/test_q_cli_unit.py -v -k "test_initialization"
```

### 4. 安装 Agent Profiles

```bash
# 安装 supervisor agent
uv run cao install code_supervisor

# 安装其他 worker agents
uv run cao install developer
uv run cao install reviewer
```

### 5. 启动服务

```bash
# 启动后端服务
uv run cao-server

# 另开终端，启动 supervisor agent
uv run cao launch --agents code_supervisor
```

## Web UI 开发

Web UI 是一个 React + Vite + Tailwind 应用，位于 `web/` 目录。

### 开发模式（热重载）

```bash
# 终端 1：启动后端
uv run cao-server

# 终端 2：启动前端开发服务器
cd web/
npm install          # 首次安装依赖
npm run dev          # 启动在 http://localhost:5173
```

### 生产模式

```bash
# 构建前端
cd web/
npm install && npm run build   # 输出到 web/dist/

# 后端自动服务静态文件
cd ..
uv run cao-server              # 访问 http://localhost:9889
```

## 运行测试

### 单元测试（推荐日常使用）

```bash
# 运行所有单元测试（排除 E2E 和集成测试）
uv run pytest test/ --ignore=test/e2e --ignore=test/providers/test_q_cli_integration.py -v

# 带覆盖率报告
uv run pytest test/ --ignore=test/e2e --cov=src --cov-report=term-missing -v

# 运行单个测试文件
uv run pytest test/providers/test_claude_code_unit.py -v

# 运行特定测试类
uv run pytest test/providers/test_codex_provider_unit.py::TestCodexBuildCommand -v
```

### 集成测试

需要先安装并配置好对应的 CLI 工具：

```bash
# Q CLI 集成测试（需要先配置 Q CLI）
uv run pytest test/providers/test_q_cli_integration.py -v

# 跳过集成测试
uv run pytest test/providers/ -m "not integration" -v
```

### E2E 测试

需要运行中的 CAO 服务器和已认证的 CLI 工具：

```bash
# 运行所有 E2E 测试
uv run pytest -m e2e test/e2e/ -v

# 运行特定 provider 的 E2E 测试
uv run pytest -m e2e test/e2e/ -v -k codex
```

### 测试标记（Markers）

```bash
# 只运行集成测试
uv run pytest -m integration -v

# 跳过慢速测试
uv run pytest -m "not slow" -v

# 并行运行测试（更快）
uv run pytest -n auto -v
```

## 代码质量检查

### 格式化

```bash
# 格式化代码
uv run black src/ test/

# 只检查不修改
uv run black --check src/ test/
```

### 导入排序

```bash
# 排序导入
uv run isort src/ test/

# 只检查不修改
uv run isort --check-only src/ test/
```

### 类型检查

```bash
uv run mypy src/
```

### 一键检查

```bash
# 格式化 + 导入排序 + 类型检查 + 测试
uv run black src/ test/ && \
uv run isort src/ test/ && \
uv run mypy src/ && \
uv run pytest test/ --ignore=test/e2e -v
```

## 开发工作流

### 1. 创建功能分支

```bash
git checkout -b feature/your-feature-name
```

### 2. 编写代码

源码位于 `src/cli_agent_orchestrator/`

### 3. 编写测试

测试位于 `test/`

### 4. 本地验证

```bash
# 运行单元测试
uv run pytest test/ --ignore=test/e2e --ignore=test/providers/test_q_cli_integration.py -v

# 代码质量检查
uv run black src/ test/
uv run isort src/ test/
uv run mypy src/
```

### 5. 提交代码

```bash
git add .
git commit -m "feat: 添加新功能描述"
git push origin feature/your-feature-name
```

### 6. 创建 PR

在 GitHub 上创建 Pull Request，CI 会自动运行测试和代码检查。

## 项目结构

```
cli-agent-orchestrator/
├── src/cli_agent_orchestrator/     # 主要源码
│   ├── api/                        # FastAPI 服务端
│   ├── cli/                        # CLI 命令
│   ├── clients/                    # 数据库和 tmux 客户端
│   ├── mcp_server/                 # MCP 服务器实现
│   ├── models/                     # 数据模型
│   ├── providers/                  # Agent 提供者（Q CLI, Claude Code 等）
│   ├── services/                   # 业务逻辑服务
│   └── utils/                      # 工具函数
├── test/                           # 测试套件
│   ├── api/                        # API 测试
│   ├── cli/                        # CLI 测试
│   ├── e2e/                        # 端到端测试
│   ├── providers/                  # Provider 测试
│   └── ...
├── web/                            # Web UI（React + Vite）
├── docs/                           # 文档
├── examples/                       # 示例工作流
└── pyproject.toml                  # 项目配置
```

## 常用命令速查

```bash
# 安装依赖
uv sync

# 启动服务器
uv run cao-server

# 启动 agent
uv run cao launch --agents code_supervisor

# 安装 agent profile
uv run cao install developer

# 查看帮助
uv run cao --help

# 运行单元测试
uv run pytest test/ --ignore=test/e2e -v

# 代码格式化
uv run black src/ test/

# 类型检查
uv run mypy src/

# 关闭所有 sessions
uv run cao shutdown --all
```

## 故障排除

### 导入错误

```bash
# 重新同步依赖
uv sync

# 如果还不行，删除虚拟环境重建
rm -rf .venv
uv sync
```

### 测试失败

```bash
# 详细输出
uv run pytest -vv

# 运行单个失败的测试
uv run pytest test/path/to/test.py::test_name -vv

# 显示 print 语句
uv run pytest -s
```

### tmux 问题

```bash
# 查看所有 sessions
tmux list-sessions

# 附加到 session
tmux attach -t <session-name>

# 在 tmux 中分离：Ctrl+b, 然后 d
```

## 相关资源

- [项目 README](README.md)
- [贡献指南](CONTRIBUTING.md)
- [API 文档](docs/api.md)
- [Agent Profile 配置](docs/agent-profile.md)
- [Tool Restrictions 工具限制](docs/tool-restrictions.md)
- [uv 官方文档](https://docs.astral.sh/uv/)
- [pytest 官方文档](https://docs.pytest.org/)
