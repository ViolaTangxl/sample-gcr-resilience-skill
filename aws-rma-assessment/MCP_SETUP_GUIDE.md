# AWS MCP 服务器设置指南

本指南介绍如何为 AWS 韧性评估 Skill 配置 MCP 服务器，支持 **Kiro** 和 **Claude Code** 两种工具。

## 推荐方案：AWS 官方 Core MCP Server

[awslabs.core-mcp-server](https://github.com/awslabs/mcp/tree/main/src/core-mcp-server) 是 AWS 官方提供的统一 MCP 入口，通过基于角色的环境变量动态代理多个子服务器，一个配置即可覆盖韧性评估所需的全部能力。

### 前置条件

- Python 3.12+
- [uv](https://github.com/astral-sh/uv)（Python 包管理器）
- AWS 凭证已配置
- Node.js

```bash
# 安装 uv（如未安装）
# macOS
brew install uv
# 或
pip install uv
```

### 韧性评估推荐角色

以下角色组合覆盖了韧性分析所需的 AWS 服务能力：

| 角色环境变量 | 说明 | 包含的 MCP 服务器 | 韧性评估用途 |
|-------------|------|-------------------|-------------|
| `aws-foundation` | AWS 知识和 API | aws-knowledge-server, aws-api-server | 查询 AWS 文档和 API |
| `monitoring-observability` | 监控与可观测性 | cloudwatch-server, cloudwatch-appsignals-server, prometheus-server, cloudtrail-server | 分析告警、指标、日志 |
| `solutions-architect` | 解决方案架构 | diagram-server, pricing-server, cost-explorer-server, aws-knowledge-server | 架构图、成本分析 |
| `security-identity` | 安全与身份 | iam-server, support-server, well-architected-security-server | IAM 审计、Well-Architected 评估 |

> **最小推荐组合**：`aws-foundation` + `monitoring-observability` + `solutions-architect`
>
> 按需添加：如使用 EKS/ECS 加 `container-orchestration`，使用 Lambda 加 `serverless-architecture`，使用 RDS/Aurora 加 `sql-db-specialist`，使用 DynamoDB 加 `nosql-db-specialist`。

### 只读安全说明

韧性评估只需要**只读访问**。以下是各角色中子服务器的只读特性：

| 子服务器 | 只读行为 |
|---------|---------|
| cloudwatch-server | 默认只读（仅 Describe/Get/List 操作） |
| cloudtrail-server | 默认只读（仅查询事件） |
| iam-server | 默认只读（仅 List/Get 操作） |
| well-architected-security-server | 默认只读（仅评估） |
| pricing-server / cost-explorer-server | 默认只读（仅查询） |
| diagram-server / aws-knowledge-server | 不涉及 AWS 资源变更 |

> **注意**：`core-mcp-server` 代理的子服务器中，`monitoring-observability`、`solutions-architect`、`security-identity` 角色下的服务器本身就是只读性质的，不会对你的 AWS 资源做任何变更。如果你额外启用了 `container-orchestration` 或 `serverless-architecture` 等角色，请**不要**在 args 中传入 `--allow-write` 参数。

---

## 配置方式

### Kiro 配置

配置文件位置：
- 工作区级别（推荐）：`.kiro/settings/mcp.json`
- 用户级别（全局）：`~/.kiro/settings/mcp.json`

```bash
mkdir -p .kiro/settings
```

编辑 `.kiro/settings/mcp.json`：

```json
{
  "mcpServers": {
    "awslabs-core-mcp-server": {
      "command": "uvx",
      "args": ["awslabs.core-mcp-server@latest"],
      "env": {
        "FASTMCP_LOG_LEVEL": "ERROR",
        "AWS_PROFILE": "default",
        "AWS_REGION": "us-east-1",
        "aws-foundation": "true",
        "monitoring-observability": "true",
        "solutions-architect": "true",
        "security-identity": "true"
      },
      "disabled": false,
      "autoApprove": []
    }
  }
}
```

生效方式：Kiro 自动检测配置变更并重连，也可在命令面板搜索 "MCP" 手动重连。

---

### Claude Code 配置

**方法 1：命令行（推荐）**

```bash
claude mcp add awslabs-core-mcp-server \
  -e FASTMCP_LOG_LEVEL=ERROR \
  -e AWS_PROFILE=default \
  -e AWS_REGION=us-east-1 \
  -e aws-foundation=true \
  -e monitoring-observability=true \
  -e solutions-architect=true \
  -e security-identity=true \
  -- uvx awslabs.core-mcp-server@latest
```

**方法 2：手动编辑** `.claude/settings.local.json`

```json
{
  "mcpServers": {
    "awslabs-core-mcp-server": {
      "command": "uvx",
      "args": ["awslabs.core-mcp-server@latest"],
      "env": {
        "FASTMCP_LOG_LEVEL": "ERROR",
        "AWS_PROFILE": "default",
        "AWS_REGION": "us-east-1",
        "aws-foundation": "true",
        "monitoring-observability": "true",
        "solutions-architect": "true",
        "security-identity": "true"
      }
    }
  }
}
```

生效方式：使用 `/mcp` 命令查看状态，或重启 Claude Code。

---

## 配置 AWS 凭证

```bash
# 方式 1：使用 AWS CLI 配置
aws configure

# 方式 2：使用 AWS SSO
aws configure sso

# 验证凭证
aws sts get-caller-identity
```

### 最小 IAM 权限策略（只读访问）

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ec2:Describe*",
        "rds:Describe*",
        "s3:List*",
        "s3:GetBucket*",
        "lambda:List*",
        "lambda:Get*",
        "dynamodb:List*",
        "dynamodb:Describe*",
        "cloudwatch:Describe*",
        "cloudwatch:Get*",
        "cloudwatch:List*",
        "logs:Describe*",
        "logs:Get*",
        "logs:StartQuery",
        "logs:GetQueryResults",
        "logs:StopQuery",
        "logs:ListLogAnomalyDetectors",
        "logs:ListAnomalies",
        "eks:List*",
        "eks:Describe*",
        "ecs:List*",
        "ecs:Describe*",
        "elbv2:Describe*",
        "apigateway:GET",
        "iam:List*",
        "iam:Get*",
        "cloudtrail:LookupEvents",
        "cloudtrail:GetTrailStatus",
        "ce:GetCostAndUsage",
        "ce:GetCostForecast",
        "pricing:GetProducts",
        "pricing:DescribeServices",
        "bedrock:InvokeModel"
      ],
      "Resource": "*"
    }
  ]
}
```

> `bedrock:InvokeModel` 是 `core-mcp-server` 的 `prompt_understanding` 工具所需的权限。如果不需要此功能，可以移除。

---

## 验证配置

| 工具 | 验证方式 |
|------|---------|
| Kiro | Kiro 功能面板 → MCP Server 视图，确认状态为 "running" |
| Claude Code | 运行 `claude mcp list` 或在对话中输入 `/mcp` |

---

## 故障排查

### MCP 服务器未连接

```bash
# 验证 uv 已安装
uv --version

# 手动测试 MCP 服务器启动（15 秒超时）
timeout 15s uvx awslabs.core-mcp-server@latest 2>&1 || echo "Command completed or timed out"

# 验证 AWS 凭证
aws sts get-caller-identity
```

### 环境变量格式问题

> 角色环境变量支持两种格式：小写带连字符（`aws-foundation`）或大写带下划线（`AWS_FOUNDATION`）。某些系统可能不支持连字符格式，请根据环境选择。

---

## 高级配置

### 多 AWS 账户

为不同账户创建独立的 MCP 服务器实例：

```json
{
  "mcpServers": {
    "aws-production": {
      "command": "uvx",
      "args": ["awslabs.core-mcp-server@latest"],
      "env": {
        "FASTMCP_LOG_LEVEL": "ERROR",
        "AWS_PROFILE": "production",
        "AWS_REGION": "us-east-1",
        "aws-foundation": "true",
        "monitoring-observability": "true"
      },
      "disabled": false
    },
    "aws-staging": {
      "command": "uvx",
      "args": ["awslabs.core-mcp-server@latest"],
      "env": {
        "FASTMCP_LOG_LEVEL": "ERROR",
        "AWS_PROFILE": "staging",
        "AWS_REGION": "us-west-2",
        "aws-foundation": "true",
        "monitoring-observability": "true"
      },
      "disabled": false
    }
  }
}
```

### 按需扩展角色

根据你的 AWS 架构，按需启用额外角色：

```json
{
  "env": {
    "container-orchestration": "true",
    "serverless-architecture": "true",
    "sql-db-specialist": "true",
    "nosql-db-specialist": "true",
    "caching-performance": "true"
  }
}
```

---

## 配置文件速查表

| 项目 | Kiro | Claude Code |
|------|------|-------------|
| 工作区配置路径 | `.kiro/settings/mcp.json` | `.claude/settings.local.json` |
| 用户级配置路径 | `~/.kiro/settings/mcp.json` | `~/.config/claude/settings.json` |
| 查看 MCP 状态 | Kiro 功能面板 → MCP Server 视图 | `claude mcp list` 或 `/mcp` |
| 重连 MCP | 自动检测 / 命令面板搜索 "MCP" | 重启 Claude Code |

---

## 参考资源

- [AWS MCP Servers（官方仓库）](https://github.com/awslabs/mcp)
- [Core MCP Server 文档](https://github.com/awslabs/mcp/tree/main/src/core-mcp-server)
- [CloudWatch MCP Server 文档](https://github.com/awslabs/mcp/tree/main/src/cloudwatch-mcp-server)
- [Model Context Protocol 文档](https://modelcontextprotocol.io/)
- [AWS CLI 配置文档](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-files.html)
- [uv 安装指南](https://docs.astral.sh/uv/getting-started/installation/)

---

**更新日期：** 2026-03-14
