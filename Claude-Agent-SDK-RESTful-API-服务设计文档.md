# Claude Agent SDK RESTful API 服务设计文档

## 文档元信息
| 项目 | 说明 |
|------|------|
| **文档名称** | Claude Agent SDK RESTful API 服务设计文档 |
| **版本号** | v1.0.0 |
| **创建日期** | 2026-03-30 |
| **文档状态** | 设计阶段 |
| **作者** | Claude |

## 目录
1. [项目概述](#1-项目概述)
2. [技术架构](#2-技术架构)
3. [API 接口规范](#3-api-接口规范)
4. [数据模型设计](#4-数据模型设计)
5. [服务组件设计](#5-服务组件设计)
6. [安全设计](#6-安全设计)
7. [错误处理](#7-错误处理)
8. [部署指南](#8-部署指南)
9. [开发指南](#9-开发指南)
10. [监控与运维](#10-监控与运维)
11. [测试策略](#11-测试策略)
12. [附录](#12-附录)

---

## 1. 项目概述

### 1.1 项目背景
Claude Code CLI 提供了强大的 AI 编程助手功能，但其使用局限于命令行界面。本项目旨在将 Claude Agent SDK 封装为一个 RESTful API 服务，使 Web 应用和后端服务能够通过标准的 HTTP/SSE 协议调用 Claude 的所有能力。

### 1.2 项目目标

#### 核心功能目标
| 功能 | 描述 | 优先级 |
|------|------|--------|
| 会话管理 | 创建、查询、更新、删除会话 | P0 |
| 流式查询 | 支持 SSE 实时流式响应 | P0 |
| 工具调用 | 支持所有内置工具（Read, Write, Edit, Bash, etc.） | P0 |
| MCP 集成 | 支持注册和管理 MCP 服务器 | P1 |
| 自定义 Agent | 支持程序化定义子 Agent | P1 |
| 会话持久化 | 会话自动保存和恢复 | P0 |
| 多租户 | API Key 隔离，支持多用户 | P1 |

#### 非功能性目标
| 指标 | 目标值 |
|------|--------|
| API 响应延迟 | < 100ms (P95) |
| SSE 首字节延迟 | < 500ms |
| 并发会话数 | > 1000 |
| 可用性 | 99.5% |
| 错误率 | < 0.1% |

### 1.3 技术选型

#### 核心技术栈
```yaml
运行时环境:
  - Node.js: 18.x LTS (支持 ES2022+)
  - TypeScript: 5.0+

Web 框架:
  - Fastify: 4.x (性能优异，原生 SSE 支持)
  - 理由: 相比 Express 性能提升 ~30%，内置类型验证

Claude SDK:
  - @anthropic-ai/claude-agent-sdk: 最新版
  - 功能: query(), listSessions(), tool(), etc.

数据库层:
  - PostgreSQL: 15+ (推荐，完整 JSON 支持)
  - MySQL: 8.0+ (备选)
  - Drizzle ORM: 1.0+ (类型安全，高性能)
  - Knex.js: 3.0+ (查询构建器，备选)

缓存层:
  - Redis: 7.x (会话状态缓存，可选)
  - 理由: 减少数据库查询压力

文件系统:
  - 可选: 用于文件检查点和 SDK 兼容存储

认证:
  - JWT: jsonwebtoken 9.x
  - API Key: 数据库存储

日志:
  - Pino: 8.x (结构化日志，高性能)

文档:
  - OpenAPI: 3.0 (自动生成 API 文档)
```

---

## 2. 技术架构

### 2.1 整体架构图

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              客户端层 (Client Layer)                         │
│                                                                              │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐              │
│  │   Web 浏览器     │  │  后端服务        │  │  移动应用        │              │
│  │  (Fetch/SSE)    │  │  (Axios/HTTP)   │  │  (HTTP Client)  │              │
│  │                 │  │                 │  │                 │              │
│  │  - React SPA    │  │  - Node.js API  │  │  - React Native │              │
│  │  - Vue SPA      │  │  - Python API   │  │  - Flutter      │              │
│  └─────────────────┘  └─────────────────┘  └─────────────────┘              │
└─────────────────────────────────────────────────────────────────────────────┘
                                        │ HTTPS
                                        ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                           API 网关层 (API Gateway)                           │
│  ┌────────────────────────────────────────────────────────────────────┐     │
│  │  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐   │     │
│  │  │ SSL 终止 │ │ 限流控制 │ │ 认证中间件│ │ CORS     │ │ 日志记录  │   │     │
│  │  └──────────┘ └──────────┘ └──────────┘ └──────────┘ └──────────┘   │     │
│  └────────────────────────────────────────────────────────────────────┘     │
│  ┌────────────────────────────────────────────────────────────────────┐     │
│  │                        路由分发 (Router)                             │     │
│  │  ┌────────────┐ ┌────────────┐ ┌────────────┐ ┌────────────┐        │     │
│  │  │ /sessions  │ │ /queries   │ │ /mcp       │ │ /capabilities│       │     │
│  │  └────────────┘ └────────────┘ └────────────┘ └────────────┘        │     │
│  └────────────────────────────────────────────────────────────────────┘     │
└─────────────────────────────────────────────────────────────────────────────┘
                                        │
                                        ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                          业务服务层 (Service Layer)                          │
│  ┌────────────────────────────────────────────────────────────────────┐     │
│  │                      SessionManager                                 │     │
│  │  • createSession()      - 创建新会话                                │     │
│  │  • getSession()         - 获取会话状态                              │     │
│  │  • deleteSession()      - 删除会话                                  │     │
│  │  • executeQuery()       - 执行查询，返回 AsyncIterable               │     │
│  │  • listSessions()       - 列出所有会话                              │     │
│  │  • resumeSession()      - 恢复历史会话                              │     │
│  └────────────────────────────────────────────────────────────────────┘     │
│  ┌────────────────────────────────────────────────────────────────────┐     │
│  │                      SSEStreamHandler                               │     │
│  │  • streamToResponse()   - 将 SDK 消息流转换为 SSE                   │     │
│  │  • sendHeartbeat()      - 发送心跳保持连接                          │     │
│  │  • handleDisconnect()   - 处理客户端断开                            │     │
│  └────────────────────────────────────────────────────────────────────┘     │
│  ┌────────────────────────────────────────────────────────────────────┐     │
│  │                      StorageService                                 │     │
│  │  • saveSession()        - 持久化会话                                │     │
│  │  • loadSession()        - 加载会话                                  │     │
│  │  • listSessions()       - 扫描会话文件                              │     │
│  └────────────────────────────────────────────────────────────────────┘     │
└─────────────────────────────────────────────────────────────────────────────┘
                                        │
                                        ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                         Claude Agent SDK 层 (SDK Layer)                      │
│  ┌────────────────────────────────────────────────────────────────────┐     │
│  │  query({ prompt, options }) → Query extends AsyncGenerator           │     │
│  │  listSessions({ dir, limit }) → SDKSessionInfo[]                    │     │
│  │  getSessionMessages(sessionId) → SessionMessage[]                   │     │
│  │  tool(name, description, schema, handler) → SdkMcpToolDefinition     │     │
│  │  createSdkMcpServer({ name, tools }) → McpServer                    │     │
│  └────────────────────────────────────────────────────────────────────┘     │
└─────────────────────────────────────────────────────────────────────────────┘
                                        │
                                        ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                          存储层 (Storage Layer)                              │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐              │
│  │  PostgreSQL     │  │   Redis 缓存    │  │  MCP 服务器     │              │
│  │  (主数据库)     │  │  (可选)         │  │  (外部进程)     │              │
│  │                 │  │                 │  │                 │              │
│  │  - sessions     │  │  - 会话状态     │  │  - stdio        │              │
│  │  - messages     │  │  - 限流计数     │  │  - SSE          │              │
│  │  - api_keys     │  │  - 热数据       │  │  - HTTP         │              │
│  │  - users        │  │                 │  │                 │              │
│  └─────────────────┘  └─────────────────┘  └─────────────────┘              │
│                                                                              │
│  ┌────────────────────────────────────────────────────────────────────┐     │
│  │  文件系统 (可选 - 文件检查点/SDK 兼容)                              │     │
│  │  /data/checkpoints/ - 文件检查点                                    │     │
│  │  /data/sessions/ - JSONL 会话文件 (可选)                            │     │
│  └────────────────────────────────────────────────────────────────────┘     │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 2.2 数据流图

#### 查询请求完整流程
```
┌──────────┐     POST /api/v1/sessions      ┌──────────┐
│  客户端   │ ──────────────────────────────▶│ API 服务  │
│          │                                    │          │
│          │     201 Created + sessionId       │          │
│          │ ◀─────────────────────────────────│          │
│          │                                    │          │
│          │     POST /api/v1/sessions/:id/query│          │
│          │ ──────────────────────────────▶   │          │
│          │                                    │          │
│          │     Accept: text/event-stream      │          │
│          │ ◀══════════════════════════════════│          │
│          │═════════════ SSE Stream ═══════════│          │
│          │                                    │          │
│          │     event: message                 │          │
│          │     data: {"type":"system",...}    │          │
│          │                                    │          │
│          │     event: message                 │          │
│          │     data: {"type":"assistant",...} │          │
│          │                                    │          │
│          │     event: message                 │          │
│          │     data: {"type":"tool_progress", │          │
│          │            "tool_name":"Bash"}     │          │
│          │                                    │          │
│          │     event: message                 │          │
│          │     data: {"type":"result",        │          │
│          │            "subtype":"success"}    │          │
│          │                                    │          │
│          │     event: done                    │          │
│          │     data: {"sessionId":"...",      │          │
│          │            "durationMs":12345}     │          │
│          │════════════════════════════════════│          │
└──────────┘                                    └──────────┘
```

### 2.3 部署架构

#### 开发环境
```
┌─────────────────────────────────────────────────────────┐
│                    开发机                              │
│                                                         │
│  ┌─────────────────────────────────────────────────┐   │
│  │          Claude API Service                     │   │
│  │  (Fastify + Agent SDK + 文件存储)               │   │
│  │                                                  │   │
│  │  PORT=3000                                       │   │
│  │  ANTHROPIC_API_KEY=sk-ant-xxx                   │   │
│  │  SESSIONS_DIR=./data/sessions                   │   │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
│  http://localhost:3000/api/v1                          │
└─────────────────────────────────────────────────────────┘
```

#### 生产环境
```
┌─────────────────────────────────────────────────────────────────────────┐
│                              生产环境                                    │
│                                                                          │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │                       负载均衡器 (Nginx)                        │    │
│  │  • SSL 终止 (Let's Encrypt)                                     │    │
│  │  • 静态资源服务                                                 │    │
│  │  • 请求路由 /api/* → API 服务                                    │    │
│  │  • 健康检查 /health                                             │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                                  │                                       │
│                ┌─────────────────┼─────────────────┐                    │
│                ▼                 ▼                 ▼                    │
│  ┌─────────────────┐ ┌─────────────────┐ ┌─────────────────┐            │
│  │  API 实例 1     │ │  API 实例 2     │ │  API 实例 N     │            │
│  │  (Node.js)      │ │  (Node.js)      │ │  (Node.js)      │            │
│  │  • 端口 3001    │ │  • 端口 3002    │ │  • 端口 300N    │            │
│  │  • PM2 管理     │ │  • PM2 管理     │ │  • PM2 管理     │            │
│  └─────────────────┘ └─────────────────┘ └─────────────────┘            │
│         │                   │                   │                        │
│         └───────────────────┼───────────────────┘                        │
│                             ▼                                            │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │                      Redis 集群                                │    │
│  │  • 会话状态缓存                                                │    │
│  │  • 限流计数器                                                  │    │
│  │  • 热数据缓存                                                  │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                             │                                            │
│                             ▼                                            │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │                    PostgreSQL 集群                             │    │
│  │  • 主数据库: 会话、消息、用户、API Keys                          │    │
│  │  • JSONB 类型存储消息和选项                                      │    │
│  │  • 索引优化: session_id, created_at, owner_id                   │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                             │                                            │
│                             ▼                                            │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │               文件系统 (可选 - 检查点)                            │    │
│  │  /data/checkpoints/ - 文件检查点                                 │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                                                                          │
└──────────────────────────────────────────────────────────────────────────┘
```

---

## 3. API 接口规范

### 3.1 通用规范

#### 基础 URL
```
生产环境: https://api.example.com/api/v1
开发环境: http://localhost:3000/api/v1
```

#### 通用请求头
```
Content-Type: application/json
X-API-Key: <your-api-key>
User-Agent: <your-app-name>/<version>
```

#### 通用响应头
```
Content-Type: application/json
X-Request-ID: <uuid>
X-RateLimit-Remaining: <number>
X-RateLimit-Reset: <unix-timestamp>
```

#### 响应状态码
| 状态码 | 含义 | 使用场景 |
|--------|------|----------|
| 200 | OK | GET 请求成功 |
| 201 | Created | POST 创建资源成功 |
| 204 | No Content | DELETE 成功 |
| 400 | Bad Request | 请求参数错误 |
| 401 | Unauthorized | 未认证/认证失败 |
| 403 | Forbidden | 权限不足 |
| 404 | Not Found | 资源不存在 |
| 429 | Too Many Requests | 超出限流 |
| 500 | Internal Server Error | 服务器内部错误 |

### 3.2 健康检查

#### GET /health
健康检查端点，用于负载均衡器探测。

**请求:**
```http
GET /health HTTP/1.1
```

**响应 (200):**
```json
{
  "status": "healthy",
  "version": "1.0.0",
  "timestamp": 1711804800000,
  "checks": {
    "sdk": "ok",
    "storage": "ok",
    "redis": "ok"
  }
}
```

### 3.3 会话管理 API

#### POST /api/v1/sessions - 创建会话

创建一个新的 Claude 会话。

**请求体:**
```json
{
  "initialPrompt": "可选的初始提示词",
  "options": {
    "model": "claude-sonnet-4-20250514",
    "agent": "general-purpose",
    "permissionMode": "bypassPermissions",
    "allowedTools": ["Read", "Write", "Edit", "Bash", "Grep", "Glob"],
    "systemPrompt": {
      "type": "preset",
      "preset": "claude_code",
      "append": "你是一个专业的代码助手。"
    },
    "cwd": "/path/to/project",
    "enableFileCheckpointing": true,
    "agents": {
      "explorer": {
        "description": "探索代码库结构",
        "prompt": "你是代码库探索专家...",
        "tools": ["Read", "Glob", "Grep"]
      },
      "reviewer": {
        "description": "代码审查专家",
        "prompt": "你是代码审查专家...",
        "tools": ["Read", "Grep"],
        "model": "claude-opus-4-20250514"
      }
    },
    "mcpServers": {
      "filesystem": {
        "type": "sdk",
        "name": "filesystem",
        "instance": <McpServer instance>
      }
    }
  }
}
```

**响应 (201 Created):**
```json
{
  "sessionId": "550e8400-e29b-41d4-a716-446655440000",
  "createdAt": 1711804800000,
  "status": "idle",
  "capabilities": {
    "models": [
      {
        "value": "claude-sonnet-4-20250514",
        "displayName": "Claude Sonnet 4",
        "description": "平衡性能和智能",
        "supportsAdaptiveThinking": true
      }
    ],
    "tools": ["Read", "Write", "Edit", "Bash", "Grep", "Glob", "WebSearch", "WebFetch"],
    "agents": [
      {
        "name": "explorer",
        "description": "探索代码库结构"
      },
      {
        "name": "reviewer",
        "description": "代码审查专家"
      }
    ]
  }
}
```

**错误响应 (400):**
```json
{
  "error": {
    "code": "INVALID_REQUEST",
    "message": "Invalid model specified",
    "details": {
      "field": "options.model",
      "value": "invalid-model"
    }
  }
}
```

#### GET /api/v1/sessions - 列出会话

获取会话列表。

**查询参数:**
| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| dir | string | 否 | 按目录过滤 |
| limit | number | 否 | 最大返回数 (默认 50) |
| includeWorktrees | boolean | 否 | 包含 git worktree 会话 (默认 true) |

**请求:**
```http
GET /api/v1/sessions?dir=/path/to/project&limit=10 HTTP/1.1
```

**响应 (200 OK):**
```json
{
  "sessions": [
    {
      "sessionId": "550e8400-e29b-41d4-a716-446655440000",
      "summary": "实现用户认证功能",
      "lastModified": 1711804800000,
      "fileSize": 12345,
      "customTitle": "用户认证模块",
      "firstPrompt": "帮我实现一个 JWT 认证系统",
      "gitBranch": "feature/auth",
      "cwd": "/path/to/project",
      "tag": "in-progress",
      "createdAt": 1711700000000
    }
  ],
  "total": 1
}
```

#### GET /api/v1/sessions/:id - 获取会话详情

获取单个会话的详细信息。

**请求:**
```http
GET /api/v1/sessions/550e8400-e29b-41d4-a716-446655440000 HTTP/1.1
```

**响应 (200 OK):**
```json
{
  "sessionId": "550e8400-e29b-41d4-a716-446655440000",
  "summary": "实现用户认证功能",
  "lastModified": 1711804800000,
  "status": "idle",
  "currentQuery": null,
  "fileSize": 12345,
  "customTitle": "用户认证模块",
  "firstPrompt": "帮我实现一个 JWT 认证系统",
  "gitBranch": "feature/auth",
  "cwd": "/path/to/project",
  "tag": "in-progress",
  "createdAt": 1711700000000,
  "messageCount": 42
}
```

#### PATCH /api/v1/sessions/:id - 更新会话

更新会话元数据（标题、标签等）。

**请求体:**
```json
{
  "title": "新的会话标题",
  "tag": "completed"
}
```

**响应 (200 OK):**
```json
{
  "sessionId": "550e8400-e29b-41d4-a716-446655440000",
  "summary": "新的会话标题",
  "lastModified": 1711804900000,
  "customTitle": "新的会话标题",
  "tag": "completed"
}
```

#### DELETE /api/v1/sessions/:id - 删除会话

删除指定会话。

**请求:**
```http
DELETE /api/v1/sessions/550e8400-e29b-41d4-a716-446655440000 HTTP/1.1
```

**响应 (204 No Content):**
```
(无响应体)
```

### 3.4 查询 API (SSE 流式)

#### POST /api/v1/sessions/:id/query - 发送查询

向指定会话发送查询，通过 SSE 流式返回响应。

**请求体:**
```json
{
  "prompt": "请帮我分析这个项目的架构",
  "options": {
    "maxTurns": 10,
    "maxBudgetUsd": 1.0,
    "effort": "high",
    "thinking": {
      "type": "adaptive"
    },
    "outputFormat": {
      "type": "json_schema",
      "schema": {
        "type": "object",
        "properties": {
          "summary": { "type": "string" },
          "recommendations": { "type": "array" }
        }
      }
    }
  }
}
```

**响应头:**
```http
Content-Type: text/event-stream
Cache-Control: no-cache
Connection: keep-alive
```

**SSE 流格式:**
```
event: message
data: {"type":"system","subtype":"init","uuid":"...","session_id":"...","model":"claude-sonnet-4-20250514",...}

event: message
data: {"type":"assistant","uuid":"...","message":{"id":"msg_...","role":"assistant","content":[...]}}

event: message
data: {"type":"stream_event","event":{"type":"content_block_delta","delta":{"type":"text_delta","text":"你好"}}}

event: message
data: {"type":"stream_event","event":{"type":"content_block_delta","delta":{"type":"text_delta","text":"！我是"}}}

event: message
data: {"type":"stream_event","event":{"type":"content_block_delta","delta":{"type":"text_delta","text":"Claude"}}}

event: message
data: {"type":"tool_progress","tool_use_id":"toolu_...","tool_name":"Bash","elapsed_time_seconds":5}

event: message
data: {"type":"assistant","uuid":"...","message":{"id":"msg_...","role":"assistant","content":[...],"stop_reason":"tool_use"}}

event: message
data: {"type":"result","subtype":"success","result":"查询完成","duration_ms":15000,"total_cost_usd":0.002}

event: done
data: {"sessionId":"550e8400-e29b-41d4-a716-446655440000","durationMs":15000,"messageCount":8,"totalCostUsd":0.002}
```

**SSE 事件类型说明:**

| 事件类型 | 说明 |
|----------|------|
| `message` | SDK 消息，包含所有 SDKMessage 类型 |
| `error` | 执行过程中的错误 |
| `done` | 流结束，包含统计信息 |

**SDKMessage 类型:**

| type | subtype | 说明 |
|------|---------|------|
| system | init | 系统初始化消息 |
| system | status | 状态更新 |
| assistant | - | AI 助手响应 |
| user | - | 用户消息 |
| stream_event | - | 流式增量内容 |
| result | success | 成功完成 |
| result | error_max_turns | 超过最大轮次 |
| tool_progress | - | 工具执行进度 |
| task_started | - | 后台任务启动 |
| task_progress | - | 后台任务进度 |

#### GET /api/v1/sessions/:id/messages - 获取会话消息

获取会话中的所有消息（非流式）。

**查询参数:**
| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| limit | number | 否 | 最大消息数 |
| offset | number | 否 | 跳过的消息数 |

**请求:**
```http
GET /api/v1/sessions/550e8400-e29b-41d4-a716-446655440000/messages?limit=20 HTTP/1.1
```

**响应 (200 OK):**
```json
{
  "messages": [
    {
      "type": "user",
      "uuid": "msg-001",
      "session_id": "550e8400-e29b-41d4-a716-446655440000",
      "message": {
        "role": "user",
        "content": "请帮我分析这个项目"
      }
    },
    {
      "type": "assistant",
      "uuid": "msg-002",
      "session_id": "550e8400-e29b-41d4-a716-446655440000",
      "message": {
        "id": "msg_...",
        "role": "assistant",
        "content": [
          {
            "type": "text",
            "text": "我来帮你分析..."
          }
        ],
        "model": "claude-sonnet-4-20250514",
        "stop_reason": "end_turn"
      }
    }
  ],
  "total": 42,
  "limit": 20,
  "offset": 0
}
```

### 3.5 能力发现 API

#### GET /api/v1/capabilities - 获取服务器能力

获取服务器支持的所有能力。

**请求:**
```http
GET /api/v1/capabilities HTTP/1.1
```

**响应 (200 OK):**
```json
{
  "version": "1.0.0",
  "sdkVersion": "0.0.1",
  "models": [
    {
      "value": "claude-opus-4-20250514",
      "displayName": "Claude Opus 4",
      "description": "最强大的模型，复杂任务首选",
      "supportsEffort": true,
      "supportedEffortLevels": ["low", "medium", "high", "max"],
      "supportsAdaptiveThinking": true
    },
    {
      "value": "claude-sonnet-4-20250514",
      "displayName": "Claude Sonnet 4",
      "description": "平衡性能和智能",
      "supportsEffort": true,
      "supportedEffortLevels": ["low", "medium", "high"],
      "supportsAdaptiveThinking": true
    },
    {
      "value": "claude-haiku-4-20250514",
      "displayName": "Claude Haiku 4",
      "description": "快速响应，简单任务",
      "supportsEffort": false,
      "supportsAdaptiveThinking": false
    }
  ],
  "supportedCommands": [
    { "name": "help", "description": "显示帮助", "argumentHint": "[command]" },
    { "name": "commit", "description": "创建 git commit", "argumentHint": "[message]" }
  ],
  "availableTools": [
    "Read", "Write", "Edit", "Glob", "Grep", "Bash",
    "NotebookEdit", "WebFetch", "WebSearch", "AskUserQuestion",
    "Agent", "TaskOutput", "TodoWrite", "ExitPlanMode",
    "ListMcpResources", "ReadMcpResource", "Config", "EnterWorktree"
  ],
  "permissionModes": ["default", "acceptEdits", "bypassPermissions", "plan", "dontAsk"],
  "features": {
    "streaming": true,
    "fileCheckpointing": true,
    "mcpServers": true,
    "customAgents": true,
    "thinking": ["adaptive", "enabled", "disabled"]
  }
}
```

### 3.6 MCP 服务器管理 API

#### POST /api/v1/mcp/servers - 注册 MCP 服务器

注册一个新的 MCP 服务器。

**请求体:**
```json
{
  "name": "my-mcp-server",
  "config": {
    "type": "stdio",
    "command": "node",
    "args": ["./my-mcp-server/index.js"],
    "env": {
      "API_KEY": "xxx"
    }
  }
}
```

**响应 (201 Created):**
```json
{
  "name": "my-mcp-server",
  "status": "connected",
  "serverInfo": {
    "name": "my-mcp-server",
    "version": "1.0.0"
  },
  "tools": [
    {
      "name": "fetch_data",
      "description": "从外部 API 获取数据",
      "annotations": {
        "readOnly": true,
        "openWorld": true
      }
    }
  ]
}
```

#### GET /api/v1/mcp/servers - 列出 MCP 服务器

获取所有已注册的 MCP 服务器状态。

**请求:**
```http
GET /api/v1/mcp/servers HTTP/1.1
```

**响应 (200 OK):**
```json
{
  "servers": [
    {
      "name": "filesystem",
      "status": "connected",
      "serverInfo": {
        "name": "filesystem",
        "version": "1.0.0"
      },
      "tools": [
        {
          "name": "read_file",
          "description": "读取文件",
          "annotations": {
            "readOnly": true
          }
        }
      ]
    },
    {
      "name": "database",
      "status": "failed",
      "error": "Connection refused"
    }
  ]
}
```

#### DELETE /api/v1/mcp/servers/:name - 删除 MCP 服务器

删除指定 MCP 服务器。

**请求:**
```http
DELETE /api/v1/mcp/servers/my-mcp-server HTTP/1.1
```

**响应 (204 No Content):**
```
(无响应体)
```

---

## 4. 数据模型设计

### 4.1 会话模型

#### Session (会话)
```typescript
interface Session {
  // 基础信息
  sessionId: string;           // UUID
  status: SessionStatus;       // 会话状态
  createdAt: number;           // 创建时间戳
  lastModified: number;        // 最后修改时间戳

  // 内容
  summary: string;             // 会话摘要
  customTitle?: string;        // 用户自定义标题
  firstPrompt?: string;        // 首条提示词
  tag?: string;                // 用户标签

  // 上下文
  cwd?: string;                // 工作目录
  gitBranch?: string;          // Git 分支

  // 统计
  fileSize?: number;           // 会话文件大小
  messageCount?: number;       // 消息数量

  // 扩展
  metadata?: Record<string, unknown>;
}

enum SessionStatus {
  IDLE = 'idle',           // 空闲
  ACTIVE = 'active',       // 执行中
  ERROR = 'error',         // 错误
  CLOSED = 'closed'        // 已关闭
}
```

#### ActiveSession (活跃会话 - 内存中)
```typescript
interface ActiveSession extends Session {
  // SDK Query 实例
  query: Query;

  // 运行时状态
  currentQuery?: string;     // 当前查询
  currentMessageId?: string; // 当前消息 ID

  // SSE 连接
  connections: Set<SSEConnection>;

  // 选项
  options: SessionOptions;
}

interface SSEConnection {
  id: string;
  response: FastifyReply;
  createdAt: number;
  lastHeartbeat: number;
}
```

### 4.2 消息模型

#### SDKMessage (SDK 消息基类)
```typescript
// 所有 SDKMessage 的基础字段
interface BaseMessage {
  type: string;
  uuid: string;
  session_id: string;
  parent_tool_use_id: string | null;
}

// 主要消息类型
type SDKMessage =
  | SDKSystemMessage          // 系统消息
  | SDKAssistantMessage       // AI 响应
  | SDKUserMessage            // 用户消息
  | SDKPartialAssistantMessage// 流式增量
  | SDKResultMessage          // 最终结果
  | SDKStatusMessage          // 状态更新
  | SDKToolProgressMessage    // 工具进度
  | SDKTaskStartedMessage     // 任务启动
  | SDKTaskProgressMessage    // 任务进度
  | SDKError;                 // 错误消息
```

#### SDKResultMessage (结果消息)
```typescript
interface SDKResultMessage {
  type: 'result';
  subtype: 'success' | 'error_max_turns' | 'error_during_execution';
  uuid: string;
  session_id: string;

  // 时间统计
  duration_ms: number;
  duration_api_ms: number;

  // 执行统计
  is_error: boolean;
  num_turns: number;

  // 结果
  result?: string;             // 成功时的文本结果
  stop_reason?: string;        // 停止原因
  errors?: string[];           // 错误列表

  // 使用统计
  total_cost_usd: number;
  usage: NonNullableUsage;
  modelUsage: Record<string, ModelUsage>;

  // 权限
  permission_denials: SDKPermissionDenial[];

  // 结构化输出
  structured_output?: unknown;
}

interface NonNullableUsage {
  input_tokens: number;
  output_tokens: number;
  cache_creation_input_tokens: number | null;
  cache_read_input_tokens: number | null;
}

interface ModelUsage {
  inputTokens: number;
  outputTokens: number;
  cacheReadInputTokens: number;
  cacheCreationInputTokens: number;
  webSearchRequests: number;
  costUSD: number;
  contextWindow: number;
  maxOutputTokens: number;
}
```

### 4.2a 数据库 Schema 设计

#### PostgreSQL 数据库结构

```sql
-- ========== 会话表 ==========
CREATE TABLE sessions (
  -- 主键
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),

  -- 所有者
  owner_id UUID REFERENCES users(id) ON DELETE SET NULL,
  api_key_id UUID REFERENCES api_keys(id) ON DELETE SET NULL,

  -- 基础信息
  session_id VARCHAR(255) UNIQUE NOT NULL,  -- SDK 使用的会话 ID
  status VARCHAR(50) NOT NULL DEFAULT 'idle', -- idle|active|error|closed

  -- 内容
  summary TEXT NOT NULL,
  custom_title TEXT,
  first_prompt TEXT,
  tag VARCHAR(255),

  -- 上下文
  cwd TEXT,
  git_branch VARCHAR(255),

  -- 配置 (JSONB 存储完整的 SessionOptions)
  options JSONB,

  -- 统计
  message_count INTEGER DEFAULT 0,
  total_cost_usd DECIMAL(10, 6) DEFAULT 0,

  -- 时间戳
  created_at BIGINT NOT NULL,
  last_modified BIGINT NOT NULL,
  closed_at BIGINT,

  -- 元数据
  metadata JSONB,

  -- 索引
  INDEX idx_sessions_owner (owner_id),
  INDEX idx_sessions_api_key (api_key_id),
  INDEX idx_sessions_status (status),
  INDEX idx_sessions_created (created_at DESC),
  INDEX idx_sessions_last_modified (last_modified DESC),
  INDEX idx_sessions_tag (tag),
  INDEX idx_sessions_cwd (cwd)
);

-- ========== 消息表 ==========
CREATE TABLE session_messages (
  -- 主键
  id BIGSERIAL PRIMARY KEY,

  -- 关联会话
  session_id VARCHAR(255) NOT NULL REFERENCES sessions(session_id) ON DELETE CASCADE,

  -- 消息类型 (SDKMessage 的 type 字段)
  message_type VARCHAR(100) NOT NULL,
  subtype VARCHAR(100),

  -- UUID (SDK 消息的 uuid)
  uuid UUID NOT NULL,

  -- 父级 tool_use_id
  parent_tool_use_id UUID,

  -- 完整消息内容 (JSONB 存储 SDKMessage)
  content JSONB NOT NULL,

  -- 时间戳
  created_at BIGINT NOT NULL,

  -- 索引
  INDEX idx_messages_session (session_id),
  INDEX idx_messages_created (created_at DESC),
  INDEX idx_messages_type (message_type),
  UNIQUE (session_id, uuid)
);

-- ========== API Keys 表 ==========
CREATE TABLE api_keys (
  -- 主键
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),

  -- 所有者
  owner_id UUID REFERENCES users(id) ON DELETE CASCADE,

  -- API Key (哈希存储)
  key_hash VARCHAR(255) UNIQUE NOT NULL,
  key_prefix VARCHAR(20) NOT NULL,  -- 用于显示 (前 8 位)

  -- 元数据
  name TEXT,
  description TEXT,

  -- 权限
  scopes TEXT[],

  -- 配额
  rate_limit INTEGER DEFAULT 100,
  max_sessions INTEGER DEFAULT 100,

  -- 时间戳
  created_at BIGINT NOT NULL,
  last_used_at BIGINT,
  expires_at BIGINT,
  revoked_at BIGINT,

  -- 索引
  INDEX idx_api_keys_owner (owner_id),
  INDEX idx_api_keys_hash (key_hash),
  INDEX idx_api_keys_prefix (key_prefix)
);

-- ========== 用户表 ==========
CREATE TABLE users (
  -- 主键
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),

  -- 认证信息
  email VARCHAR(255) UNIQUE NOT NULL,
  password_hash VARCHAR(255),

  -- 个人信息
  name TEXT,
  avatar_url TEXT,

  -- 状态
  is_active BOOLEAN DEFAULT true,
  is_admin BOOLEAN DEFAULT FALSE,

  -- 时间戳
  created_at BIGINT NOT NULL,
  updated_at BIGINT NOT NULL,
  last_login_at BIGINT,

  -- 索引
  INDEX idx_users_email (email),
  INDEX idx_users_active (is_active)
);

-- ========== 查询统计表 ==========
CREATE TABLE query_stats (
  -- 主键
  id BIGSERIAL PRIMARY KEY,

  -- 关联
  session_id VARCHAR(255) REFERENCES sessions(session_id) ON DELETE SET NULL,
  api_key_id UUID REFERENCES api_keys(id) ON DELETE SET NULL,

  -- 统计信息
  num_turns INTEGER NOT NULL,
  duration_ms BIGINT NOT NULL,
  duration_api_ms BIGINT NOT NULL,

  -- Token 使用
  input_tokens INTEGER NOT NULL,
  output_tokens INTEGER NOT NULL,
  cache_creation_input_tokens INTEGER,
  cache_read_input_tokens INTEGER,

  -- 成本
  total_cost_usd DECIMAL(10, 6) NOT NULL,

  -- 模型使用 (JSONB)
  model_usage JSONB,

  -- 结果
  result_type VARCHAR(50), -- success|error_max_turns|error_during_execution
  stop_reason VARCHAR(100),

  -- 时间戳
  created_at BIGINT NOT NULL,

  -- 索引
  INDEX idx_query_stats_session (session_id),
  INDEX idx_query_stats_api_key (api_key_id),
  INDEX idx_query_stats_created (created_at DESC)
);

-- ========== MCP 服务器配置表 ==========
CREATE TABLE mcp_servers (
  -- 主键
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),

  -- 所有者
  owner_id UUID REFERENCES users(id) ON DELETE CASCADE,

  -- 配置
  name VARCHAR(255) NOT NULL,
  config JSONB NOT NULL,  -- McpServerConfig

  -- 状态
  is_enabled BOOLEAN DEFAULT true,

  -- 时间戳
  created_at BIGINT NOT NULL,
  updated_at BIGINT NOT NULL,

  -- 唯一约束
  UNIQUE (owner_id, name),

  -- 索引
  INDEX idx_mcp_servers_owner (owner_id),
  INDEX idx_mcp_servers_enabled (is_enabled)
);
```

#### Drizzle ORM Schema

```typescript
// src/db/schema.ts
import { pgTable, uuid, varchar, text, boolean, integer, decimal, bigint, index, jsonb } from 'drizzle-orm/pg-core';
import { sql } from 'drizzle-orm';

export const users = pgTable('users', {
  id: uuid('id').primaryKey().defaultRandom(),
  email: varchar('email', { length: 255 }).notNull().unique(),
  passwordHash: varchar('password_hash', { length: 255 }),
  name: text('name'),
  avatarUrl: text('avatar_url'),
  isActive: boolean('is_active').default(true),
  isAdmin: boolean('is_admin').default(false),
  createdAt: bigint('created_at', { mode: 'number' }).notNull().$defaultFn(() => Date.now()),
  updatedAt: bigint('updated_at', { mode: 'number' }).notNull().$defaultFn(() => Date.now()),
  lastLoginAt: bigint('last_login_at', { mode: 'number' }),
}, (table) => ({
  emailIdx: index('idx_users_email').on(table.email),
  activeIdx: index('idx_users_active').on(table.isActive),
}));

export const apiKeys = pgTable('api_keys', {
  id: uuid('id').primaryKey().defaultRandom(),
  ownerId: uuid('owner_id').references(() => users.id, { onDelete: 'set null' }),
  keyHash: varchar('key_hash', { length: 255 }).notNull().unique(),
  keyPrefix: varchar('key_prefix', { length: 20 }).notNull(),
  name: text('name'),
  description: text('description'),
  scopes: text('scopes').array(),
  rateLimit: integer('rate_limit').default(100),
  maxSessions: integer('max_sessions').default(100),
  createdAt: bigint('created_at', { mode: 'number' }).notNull().$defaultFn(() => Date.now()),
  lastUsedAt: bigint('last_used_at', { mode: 'number' }),
  expiresAt: bigint('expires_at', { mode: 'number' }),
  revokedAt: bigint('revoked_at', { mode: 'number' }),
}, (table) => ({
  ownerIdx: index('idx_api_keys_owner').on(table.ownerId),
  hashIdx: index('idx_api_keys_hash').on(table.keyHash),
  prefixIdx: index('idx_api_keys_prefix').on(table.keyPrefix),
}));

export const sessions = pgTable('sessions', {
  id: uuid('id').primaryKey().defaultRandom(),
  ownerId: uuid('owner_id').references(() => users.id, { onDelete: 'set null' }),
  apiKeyId: uuid('api_key_id').references(() => apiKeys.id, { onDelete: 'set null' }),
  sessionId: varchar('session_id', { length: 255 }).notNull().unique(),
  status: varchar('status', { length: 50 }).notNull().default('idle'),
  summary: text('summary').notNull(),
  customTitle: text('custom_title'),
  firstPrompt: text('first_prompt'),
  tag: varchar('tag', { length: 255 }),
  cwd: text('cwd'),
  gitBranch: varchar('git_branch', { length: 255 }),
  options: jsonb('options'),
  messageCount: integer('message_count').default(0),
  totalCostUsd: decimal('total_cost_usd', { precision: 10, scale: 6 }).default('0'),
  createdAt: bigint('created_at', { mode: 'number' }).notNull().$defaultFn(() => Date.now()),
  lastModified: bigint('last_modified', { mode: 'number' }).notNull().$defaultFn(() => Date.now()),
  closedAt: bigint('closed_at', { mode: 'number' }),
  metadata: jsonb('metadata'),
}, (table) => ({
  ownerIdx: index('idx_sessions_owner').on(table.ownerId),
  apiKeyIdx: index('idx_sessions_api_key').on(table.apiKeyId),
  statusIdx: index('idx_sessions_status').on(table.status),
  createdIdx: index('idx_sessions_created').on(table.createdAt),
  lastModifiedIdx: index('idx_sessions_last_modified').on(table.lastModified),
  tagIdx: index('idx_sessions_tag').on(table.tag),
  cwdIdx: index('idx_sessions_cwd').on(table.cwd),
}));

export const sessionMessages = pgTable('session_messages', {
  id: bigint('id', { mode: 'number' }).primaryKey().generatedAlwaysAsIdentity(),
  sessionId: varchar('session_id', { length: 255 }).notNull().references(() => sessions.sessionId, { onDelete: 'cascade' }),
  messageType: varchar('message_type', { length: 100 }).notNull(),
  subtype: varchar('subtype', { length: 100 }),
  uuid: uuid('uuid').notNull(),
  parentToolUseId: uuid('parent_tool_use_id'),
  content: jsonb('content').notNull(),
  createdAt: bigint('created_at', { mode: 'number' }).notNull().$defaultFn(() => Date.now()),
}, (table) => ({
  sessionIdx: index('idx_messages_session').on(table.sessionId),
  createdIdx: index('idx_messages_created').on(table.createdAt),
  typeIdx: index('idx_messages_type').on(table.messageType),
  uniqueConstraint: sql`UNIQUE (session_id, uuid)`,
}));

export const queryStats = pgTable('query_stats', {
  id: bigint('id', { mode: 'number' }).primaryKey().generatedAlwaysAsIdentity(),
  sessionId: varchar('session_id', { length: 255 }).references(() => sessions.sessionId, { onDelete: 'set null' }),
  apiKeyId: uuid('api_key_id').references(() => apiKeys.id, { onDelete: 'set null' }),
  numTurns: integer('num_turns').notNull(),
  durationMs: bigint('duration_ms', { mode: 'number' }).notNull(),
  durationApiMs: bigint('duration_api_ms', { mode: 'number' }).notNull(),
  inputTokens: integer('input_tokens').notNull(),
  outputTokens: integer('output_tokens').notNull(),
  cacheCreationInputTokens: integer('cache_creation_input_tokens'),
  cacheReadInputTokens: integer('cache_read_input_tokens'),
  totalCostUsd: decimal('total_cost_usd', { precision: 10, scale: 6 }).notNull(),
  modelUsage: jsonb('model_usage'),
  resultType: varchar('result_type', { length: 50 }),
  stopReason: varchar('stop_reason', { length: 100 }),
  createdAt: bigint('created_at', { mode: 'number' }).notNull().$defaultFn(() => Date.now()),
}, (table) => ({
  sessionIdx: index('idx_query_stats_session').on(table.sessionId),
  apiKeyIdx: index('idx_query_stats_api_key').on(table.apiKeyId),
  createdIdx: index('idx_query_stats_created').on(table.createdAt),
}));

export const mcpServers = pgTable('mcp_servers', {
  id: uuid('id').primaryKey().defaultRandom(),
  ownerId: uuid('owner_id').references(() => users.id, { onDelete: 'cascade' }),
  name: varchar('name', { length: 255 }).notNull(),
  config: jsonb('config').notNull(),
  isEnabled: boolean('is_enabled').default(true),
  createdAt: bigint('created_at', { mode: 'number' }).notNull().$defaultFn(() => Date.now()),
  updatedAt: bigint('updated_at', { mode: 'number' }).notNull().$defaultFn(() => Date.now()),
}, (table) => ({
  ownerIdx: index('idx_mcp_servers_owner').on(table.ownerId),
  enabledIdx: index('idx_mcp_servers_enabled').on(table.isEnabled),
  uniqueConstraint: sql`UNIQUE (owner_id, name)`,
}));

// TypeScript 类型
export type User = typeof users.$inferSelect;
export type NewUser = typeof users.$inferInsert;

export type ApiKey = typeof apiKeys.$inferSelect;
export type NewApiKey = typeof apiKeys.$inferInsert;

export type Session = typeof sessions.$inferSelect;
export type NewSession = typeof sessions.$inferInsert;

export type SessionMessage = typeof sessionMessages.$inferSelect;
export type NewSessionMessage = typeof sessionMessages.$inferInsert;

export type QueryStat = typeof queryStats.$inferSelect;
export type NewQueryStat = typeof queryStats.$inferInsert;

export type McpServer = typeof mcpServers.$inferSelect;
export type NewMcpServer = typeof mcpServers.$inferInsert;
```

### 4.3 请求/响应模型

#### CreateSessionRequest
```typescript
interface CreateSessionRequest {
  initialPrompt?: string;
  options?: SessionOptions;
}

/**
 * 完整的 SessionOptions - 来自 Claude Agent SDK Options 类型
 * 包含所有 SDK query() 函数支持的参数
 */
interface SessionOptions {
  // ========== 基础配置 ==========

  /** Claude 模型名称 (默认从 CLI 配置) */
  model?: string;

  /** 主线程 Agent 名称 (必须在 agents 选项中定义) */
  agent?: string;

  /** 权限模式 */
  permissionMode?: PermissionMode;

  /**
   * 自动批准的工具列表
   * 注意: 这不会限制 Claude 只使用这些工具;
   * 未列出的工具会回退到 permissionMode 和 canUseTool
   */
  allowedTools?: string[];

  /** 始终拒绝的工具列表 (优先级高于 allowedTools) */
  disallowedTools?: string[];

  /**
   * 自定义权限检查函数
   * 用于细粒度控制工具使用
   */
  canUseTool?: CanUseTool;

  /**
   * 启用跳过权限
   * 使用 permissionMode: 'bypassPermissions' 时必须设置
   */
  allowDangerouslySkipPermissions?: boolean;

  // ========== 系统提示 ==========

  /**
   * 系统提示配置
   * - 字符串: 自定义提示
   * - 对象: 使用预设
   */
  systemPrompt?: string | SystemPromptPreset;

  // ========== 工作目录 ==========

  /** 当前工作目录 (默认 process.cwd()) */
  cwd?: string;

  /** Claude 可访问的额外目录 */
  additionalDirectories?: string[];

  // ========== 高级配置 ==========

  /** 启用文件检查点 (用于回滚) */
  enableFileCheckpointing?: boolean;

  /** 最大预算 (美元) */
  maxBudgetUsd?: number;

  /** 最大 agentic 轮次 */
  maxTurns?: number;

  /**
   * 控制响应的努力程度
   * 与 adaptive thinking 配合控制思考深度
   */
  effort?: 'low' | 'medium' | 'high' | 'max';

  /** 思考/推理行为配置 */
  thinking?: ThinkingConfig;

  /**
   * (已弃用) 最大思考 tokens
   * 使用 thinking 代替
   */
  maxThinkingTokens?: number;

  /**
   * 自定义进程生成函数
   * 用于在 VM、容器或远程环境中运行 Claude Code
   */
  spawnClaudeCodeProcess?: (options: SpawnOptions) => SpawnedProcess;

  /**
   * JavaScript 运行时
   * 默认自动检测
   */
  executable?: 'bun' | 'deno' | 'node';

  /** 传递给可执行文件的参数 */
  executableArgs?: string[];

  /** Claude Code 可执行文件路径 */
  pathToClaudeCodeExecutable?: string;

  // ========== 扩展功能 ==========

  /**
   * 程序化定义子 Agents
   */
  agents?: Record<string, AgentDefinition>;

  /**
   * MCP 服务器配置
   */
  mcpServers?: Record<string, McpServerConfig>;

  /**
   * Hook 回调配置
   */
  hooks?: Partial<Record<HookEvent, HookCallbackMatcher[]>>;

  /**
   * 内置工具行为配置
   */
  toolConfig?: ToolConfig;

  /**
   * 工具配置
   * - 数组: 工具名称列表
   * - 对象: 使用预设
   */
  tools?: string[] | { type: 'preset'; preset: 'claude_code' };

  // ========== 输出格式 ==========

  /**
   * 定义输出格式
   */
  outputFormat?: { type: 'json_schema'; schema: JSONSchema };

  // ========== 会话管理 ==========

  /**
   * 特定会话 ID
   * 默认自动生成 UUID
   */
  sessionId?: string;

  /**
   * 继续最近的对话
   */
  continue?: boolean;

  /**
   * 要恢复的会话 ID
   */
  resume?: string;

  /**
   * 在特定消息 UUID 处恢复会话
   */
  resumeSessionAt?: string;

  /**
   * 恢复时创建新会话 ID
   */
  forkSession?: boolean;

  /**
   * 禁用会话持久化到磁盘
   * 禁用后无法恢复会话
   */
  persistSession?: boolean;

  // ========== 设置源 ==========

  /**
   * 控制加载哪些文件系统设置
   * - 'user': ~/.claude/settings.json
   * - 'project': .claude/settings.json (版本控制)
   * - 'local': .claude/settings.local.json (git忽略)
   *
   * 注意: 必须包含 'project' 才能加载 CLAUDE.md 文件
   * 省略时不加载任何文件系统设置 (SDK 默认隔离行为)
   */
  settingSources?: SettingSource[];

  // ========== Beta 功能 ==========

  /**
   * 启用 Beta 功能
   * 例如: ['context-1m-2025-08-07']
   */
  betas?: SdkBeta[];

  // ========== 插件 ==========

  /**
   * 从本地路径加载自定义插件
   */
  plugins?: SdkPluginConfig[];

  // ========== 提示建议 ==========

  /**
   * 启用提示建议
   * 每轮后发出 prompt_suggestion 消息
   */
  promptSuggestions?: boolean;

  // ========== 沙箱配置 ==========

  /**
   * 配置沙箱行为
   */
  sandbox?: SandboxSettings;

  // ========== 权限提示工具 ==========

  /**
   * 权限提示的 MCP 工具名称
   */
  permissionPromptToolName?: string;

  // ========== 其他 ==========

  /** 额外的参数 */
  extraArgs?: Record<string, string | null>;

  /** 环境变量 */
  env?: Record<string, string | undefined>;

  /** stderr 输出回调 */
  stderr?: (data: string) => void;

  /**
   * 强制 MCP 配置验证
   */
  strictMcpConfig?: boolean;

  /**
   * 是否包含部分消息事件
   */
  includePartialMessages?: boolean;

  /**
   * 取消控制器
   */
  abortController?: AbortController;

  /**
   * 调试模式
   */
  debug?: boolean;

  /**
   * 调试日志文件路径
   * 隐式启用调试模式
   */
  debugFile?: string;
}

// ========== 相关类型定义 ==========

type PermissionMode =
  | 'default'           // 标准权限行为
  | 'acceptEdits'       // 自动接受文件编辑
  | 'bypassPermissions' // 跳过所有权限检查
  | 'plan'             // 规划模式 - 不执行
  | 'dontAsk';          // 不提示权限，未预批准则拒绝

interface SystemPromptPreset {
  type: 'preset';
  preset: 'claude_code';
  append?: string;
}

type ThinkingConfig =
  | { type: 'adaptive' }         // 模型决定何时以及多少推理 (Opus 4.6+)
  | { type: 'enabled'; budgetTokens?: number }  // 固定思考 token 预算
  | { type: 'disabled' };       // 无扩展思考

type SettingSource = 'user' | 'project' | 'local';

type SdkBeta = string;

interface AgentDefinition {
  description: string;
  tools?: string[];
  disallowedTools?: string[];
  prompt: string;
  model?: 'sonnet' | 'opus' | 'haiku' | 'inherit';
  mcpServers?: AgentMcpServerSpec[];
  skills?: string[];
  maxTurns?: number;
  criticalSystemReminder_EXPERIMENTAL?: string;
}

type AgentMcpServerSpec =
  | string
  | Record<string, McpServerConfigForProcessTransport>;

type ToolConfig {
  askUserQuestion?: {
    previewFormat?: 'markdown' | 'html';
  };
}

type SandboxSettings = {
  enabled?: boolean;
  autoAllowBashIfSandboxed?: boolean;
  excludedCommands?: string[];
  allowUnsandboxedCommands?: boolean;
  network?: SandboxNetworkConfig;
  filesystem?: SandboxFilesystemConfig;
  ignoreViolations?: Record<string, string[]>;
  enableWeakerNestedSandbox?: boolean;
  ripgrep?: { command: string; args?: string[] };
};

type CanUseTool = (
  toolName: string,
  input: unknown
) => Promise<{ allowed: boolean } | { allowed: boolean; feedback?: string }>;

// ========== MCP 服务器配置类型 ==========

/**
 * MCP 服务器配置 (联合类型)
 * 支持 stdio, SSE, HTTP, SDK, claudeai-proxy 五种类型
 */
type McpServerConfig =
  | McpStdioServerConfig      // stdio 进程通信
  | McpSSEServerConfig        // SSE 服务器
  | McpHttpServerConfig       // HTTP 服务器
  | McpSdkServerConfig        // SDK 内联服务器
  | McpClaudeAIProxyConfig;   // Claude AI 代理

interface McpStdioServerConfig {
  type: 'stdio';
  command: string;
  args?: string[];
  env?: Record<string, string>;
}

interface McpSSEServerConfig {
  type: 'sse';
  url: string;
  headers?: Record<string, string>;
}

interface McpHttpServerConfig {
  type: 'http';
  url: string;
  headers?: Record<string, string>;
}

interface McpSdkServerConfig {
  type: 'sdk';
  name: string;
  instance: McpSdkServerConfigWithInstance;
}

interface McpClaudeAIProxyConfig {
  type: 'claudeai-proxy';
  cloudName?: string;
}

type McpServerConfigForProcessTransport =
  | McpStdioServerConfig
  | McpSSEServerConfig
  | McpHttpServerConfig
  | McpSdkServerConfig;
```

#### QueryRequest
```typescript
interface QueryRequest {
  prompt: string;
  options?: {
    // 查询时可以覆盖的部分选项
    maxTurns?: number;
    maxBudgetUsd?: number;
    effort?: 'low' | 'medium' | 'high' | 'max';
    thinking?: ThinkingConfig;
    outputFormat?: { type: 'json_schema'; schema: JSONSchema };
  };
}
```

### 4.4 错误模型

#### ErrorResponse
```typescript
interface ErrorResponse {
  error: {
    code: ErrorCode;
    message: string;
    details?: unknown;
    requestId?: string;
    timestamp: number;
  };
}

enum ErrorCode {
  // 通用错误
  UNKNOWN_ERROR = 'UNKNOWN_ERROR',
  INVALID_REQUEST = 'INVALID_REQUEST',
  UNAUTHORIZED = 'UNAUTHORIZED',
  FORBIDDEN = 'FORBIDDEN',
  NOT_FOUND = 'NOT_FOUND',
  RATE_LIMITED = 'RATE_LIMITED',
  INTERNAL_ERROR = 'INTERNAL_ERROR',

  // 会话错误
  SESSION_NOT_FOUND = 'SESSION_NOT_FOUND',
  SESSION_CLOSED = 'SESSION_CLOSED',
  SESSION_ACTIVE = 'SESSION_ACTIVE',

  // 查询错误
  QUERY_FAILED = 'QUERY_FAILED',
  QUERY_CANCELLED = 'QUERY_CANCELLED',
  QUERY_TIMEOUT = 'QUERY_TIMEOUT',

  // SDK 错误
  SDK_ERROR = 'SDK_ERROR',
  SDK_AUTHENTICATION_FAILED = 'SDK_AUTHENTICATION_FAILED',
  SDK_RATE_LIMIT = 'SDK_RATE_LIMIT',
  SDK_QUOTA_EXCEEDED = 'SDK_QUOTA_EXCEEDED',

  // MCP 错误
  MCP_SERVER_ERROR = 'MCP_SERVER_ERROR',
  MCP_SERVER_NOT_FOUND = 'MCP_SERVER_NOT_FOUND',
}
```

---

## 5. 服务组件设计

### 5.1 SessionManager (会话管理器)

核心服务，管理所有会话的生命周期。

```typescript
class SessionManager {
  // ========== 状态 ==========
  private sessions: Map<string, ActiveSession> = new Map();
  private storage: StorageService;
  private config: SessionManagerConfig;

  // ========== 配置 ==========
  interface SessionManagerConfig {
    maxSessions?: number;           // 最大会话数 (默认 1000)
    sessionTimeout?: number;        // 会话超时 (默认 30分钟)
    cleanupInterval?: number;       // 清理间隔 (默认 5分钟)
  }

  // ========== 公共方法 ==========

  /**
   * 创建新会话
   */
  async createSession(
    request: CreateSessionRequest
  ): Promise<Session> {
    // 1. 检查会话数限制
    if (this.sessions.size >= this.config.maxSessions) {
      throw new Error('MAX_SESSIONS_EXCEEDED');
    }

    // 2. 生成会话 ID
    const sessionId = randomUUID();

    // 3. 创建 SDK Query 实例
    const query = this.createQuery(request);

    // 4. 初始化会话
    const session: ActiveSession = {
      sessionId,
      status: 'idle',
      createdAt: Date.now(),
      lastModified: Date.now(),
      summary: request.initialPrompt || 'New Session',
      query,
      connections: new Set(),
      options: request.options || {},
    };

    // 5. 存储会话
    this.sessions.set(sessionId, session);

    // 6. 异步持久化
    this.storage.saveSession(session).catch(err => {
      this.logger.error('Failed to save session', { sessionId, error: err });
    });

    // 7. 如果有初始提示，执行查询
    if (request.initialPrompt) {
      this.executeQuery(sessionId, request.initialPrompt)
        .catch(err => this.logger.error('Initial query failed', { error: err }));
    }

    return session;
  }

  /**
   * 获取会话
   */
  async getSession(sessionId: string): Promise<Session | null> {
    // 先从内存获取
    const active = this.sessions.get(sessionId);
    if (active) {
      return active;
    }

    // 从存储加载
    const stored = await this.storage.loadSession(sessionId);
    if (stored) {
      // 重新创建 Query 实例
      const query = this.createQuery({ options: stored.options });
      const activeSession: ActiveSession = {
        ...stored,
        query,
        connections: new Set(),
      };
      this.sessions.set(sessionId, activeSession);
      return activeSession;
    }

    return null;
  }

  /**
   * 删除会话
   */
  async deleteSession(sessionId: string): Promise<void> {
    const session = this.sessions.get(sessionId);
    if (session) {
      // 关闭所有 SSE 连接
      for (const conn of session.connections) {
        conn.response.raw.end();
      }
      session.connections.clear();

      // 关闭 Query
      session.query?.close();
    }

    // 从内存移除
    this.sessions.delete(sessionId);

    // 从存储删除
    await this.storage.deleteSession(sessionId);
  }

  /**
   * 执行查询
   */
  async executeQuery(
    sessionId: string,
    prompt: string,
    options?: QueryOptions
  ): Promise<AsyncIterable<SDKMessage>> {
    const session = await this.getSession(sessionId);
    if (!session) {
      throw new Error('SESSION_NOT_FOUND');
    }
    if (session.status === 'closed') {
      throw new Error('SESSION_CLOSED');
    }

    // 更新状态
    session.status = 'active';
    session.currentQuery = prompt;
    session.lastModified = Date.now();

    // 使用 SDK streamInput
    await session.query.streamInput(
      (async function* () {
        yield { type: 'user', message: { role: 'user', content: prompt } };
      })()
    );

    // 返回消息流
    return session.query;
  }

  /**
   * 列出会话
   */
  async listSessions(options: ListSessionsOptions): Promise<Session[]> {
    const { dir, limit } = options;

    // 从存储扫描
    const stored = await this.storage.listSessions(options);

    // 合并内存中的会话
    const allSessions = new Map<string, Session>();

    for (const session of stored) {
      allSessions.set(session.sessionId, session);
    }

    for (const [id, session] of this.sessions) {
      allSessions.set(id, session);
    }

    // 过滤和排序
    let result = Array.from(allSessions.values());

    if (dir) {
      result = result.filter(s => s.cwd === dir);
    }

    result.sort((a, b) => b.lastModified - a.lastModified);

    if (limit) {
      result = result.slice(0, limit);
    }

    return result;
  }

  // ========== 私有方法 ==========

  private createQuery(request: CreateSessionRequest): Query {
    const sdk = require('@anthropic-ai/claude-agent-sdk');

    // 合并默认配置
    const options = {
      ...this.defaultOptions,
      ...request.options,
      sessionId: undefined, // 由 SDK 生成
    };

    return sdk.query({
      prompt: request.initialPrompt || '',
      options,
    });
  }

  /**
   * 清理过期会话
   */
  private cleanupExpiredSessions(): void {
    const now = Date.now();
    const timeout = this.config.sessionTimeout;

    for (const [id, session] of this.sessions) {
      if (now - session.lastModified > timeout && session.status === 'idle') {
        this.deleteSession(id).catch(err => {
          this.logger.error('Cleanup failed', { sessionId: id, error: err });
        });
      }
    }
  }

  /**
   * 启动后台清理
   */
  start(): void {
    this.cleanupTimer = setInterval(() => {
      this.cleanupExpiredSessions();
    }, this.config.cleanupInterval);
  }

  /**
   * 停止服务
   */
  async stop(): Promise<void> {
    clearInterval(this.cleanupTimer);

    // 关闭所有会话
    const promises = Array.from(this.sessions.keys()).map(id =>
      this.deleteSession(id)
    );
    await Promise.all(promises);
  }
}
```

### 5.2 SSEStreamHandler (SSE 流处理器)

处理 SSE 流式响应。

```typescript
class SSEStreamHandler {
  private logger = pino({ name: 'SSEStreamHandler' });

  /**
   * 将 SDK 消息流转换为 SSE
   */
  async streamToResponse(
    sessionId: string,
    messages: AsyncIterable<SDKMessage>,
    response: FastifyReply
  ): Promise<void> {
    // 设置 SSE 响应头
    response.raw.writeHead(200, {
      'Content-Type': 'text/event-stream',
      'Cache-Control': 'no-cache',
      'Connection': 'keep-alive',
      'X-Accel-Buffering': 'no', // 禁用 Nginx 缓冲
    });

    const connId = randomUUID();
    let messageCount = 0;
    let startTime = Date.now();

    try {
      // 心跳保活
      const heartbeat = setInterval(() => {
        response.raw.write(': heartbeat\n\n');
      }, 30000);

      // 发送消息
      for await (const message of messages) {
        messageCount++;

        // 发送 SSE 事件
        this.sendSSEEvent(response.raw, 'message', message);

        // 检查连接是否断开
        if (response.raw.destroyed) {
          this.logger.info('Client disconnected', { sessionId, connId });
          break;
        }
      }

      clearInterval(heartbeat);

      // 发送完成事件
      const duration = Date.now() - startTime;
      this.sendSSEEvent(response.raw, 'done', {
        sessionId,
        durationMs: duration,
        messageCount,
      });

    } catch (error) {
      this.logger.error('Stream error', { sessionId, connId, error });

      // 发送错误事件
      this.sendSSEEvent(response.raw, 'error', {
        code: 'STREAM_ERROR',
        message: error.message,
      });
    } finally {
      response.raw.end();
    }
  }

  /**
   * 发送 SSE 事件
   */
  private sendSSEEvent(
    response: http.ServerResponse,
    event: string,
    data: unknown
  ): void {
    try {
      response.write(`event: ${event}\n`);
      response.write(`data: ${JSON.stringify(data)}\n\n`);
    } catch (err) {
      this.logger.error('Failed to send SSE event', { error: err });
    }
  }
}
```

### 5.3 StorageService (存储服务)

数据库会话持久化服务。

```typescript
import { eq, and, desc, sql } from 'drizzle-orm';
import { drizzle } from 'drizzle-orm/node-postgres';
import pg from 'pg';
import * as schema from './db/schema';

class StorageService {
  private db: ReturnType<typeof drizzle>;
  private logger = pino({ name: 'StorageService' });
  private redis?: Redis;

  constructor(config: {
    databaseUrl: string;
    redisUrl?: string;
  }) {
    // PostgreSQL 连接池
    const pool = new pg.Pool({
      connectionString: config.databaseUrl,
      max: 20,
      idleTimeoutMillis: 30000,
      connectionTimeoutMillis: 2000,
    });

    this.db = drizzle(pool, { schema });

    // 可选 Redis 缓存
    if (config.redisUrl) {
      this.redis = new Redis(config.redisUrl);
    }
  }

  // ========== 会话操作 ==========

  /**
   * 保存会话
   */
  async saveSession(session: Session, apiKeyId?: string): Promise<void> {
    const now = Date.now();

    await this.db.insert(schema.sessions).values({
      sessionId: session.sessionId,
      ownerId: session.ownerId,
      apiKeyId,
      status: session.status,
      summary: session.summary,
      customTitle: session.customTitle,
      firstPrompt: session.firstPrompt,
      tag: session.tag,
      cwd: session.cwd,
      gitBranch: session.gitBranch,
      options: session.options as any,
      messageCount: session.messageCount || 0,
      totalCostUsd: session.totalCostUsd?.toString() || '0',
      createdAt: session.createdAt,
      lastModified: now,
      closedAt: session.closedAt,
      metadata: session.metadata as any,
    }).onConflictDoUpdate({
      target: schema.sessions.sessionId,
      set: {
        status: session.status,
        summary: session.summary,
        customTitle: session.customTitle,
        tag: session.tag,
        messageCount: session.messageCount || 0,
        totalCostUsd: session.totalCostUsd?.toString() || '0',
        lastModified: now,
        closedAt: session.closedAt,
        metadata: session.metadata as any,
      },
    });

    // 清除缓存
    if (this.redis) {
      await this.redis.del(`session:${session.sessionId}`);
      await this.redis.del(`sessions:list:*`);
    }
  }

  /**
   * 加载会话
   */
  async loadSession(sessionId: string): Promise<Session | null> {
    // 尝试从缓存获取
    if (this.redis) {
      const cached = await this.redis.get(`session:${sessionId}`);
      if (cached) {
        return JSON.parse(cached);
      }
    }

    // 从数据库获取
    const result = await this.db
      .select()
      .from(schema.sessions)
      .where(eq(schema.sessions.sessionId, sessionId))
      .limit(1);

    if (!result[0]) {
      return null;
    }

    const session = this.mapDbSessionToSession(result[0]);

    // 缓存结果
    if (this.redis) {
      await this.redis.setex(`session:${sessionId}`, 300, JSON.stringify(session));
    }

    return session;
  }

  /**
   * 删除会话
   */
  async deleteSession(sessionId: string): Promise<void> {
    await this.db
      .delete(schema.sessions)
      .where(eq(schema.sessions.sessionId, sessionId));

    // 清除缓存
    if (this.redis) {
      await this.redis.del(`session:${sessionId}`);
      await this.redis.del(`sessions:list:*`);
    }
  }

  /**
   * 列出会话
   */
  async listSessions(options: {
    ownerId?: string;
    apiKeyId?: string;
    dir?: string;
    tag?: string;
    limit?: number;
    offset?: number;
  }): Promise<{ sessions: Session[]; total: number }> {
    const { ownerId, apiKeyId, dir, tag, limit = 50, offset = 0 } = options;

    // 构建查询条件
    const conditions = [];
    if (ownerId) conditions.push(eq(schema.sessions.ownerId, ownerId));
    if (apiKeyId) conditions.push(eq(schema.sessions.apiKeyId, apiKeyId));
    if (dir) conditions.push(eq(schema.sessions.cwd, dir));
    if (tag) conditions.push(eq(schema.sessions.tag, tag));

    const whereClause = conditions.length > 0
      ? and(...conditions)
      : undefined;

    // 获取总数
    const [{ count }] = await this.db
      .select({ count: sql<number>`count(*)::int` })
      .from(schema.sessions)
      .where(whereClause);

    // 获取分页数据
    const results = await this.db
      .select()
      .from(schema.sessions)
      .where(whereClause)
      .orderBy(desc(schema.sessions.lastModified))
      .limit(limit)
      .offset(offset);

    const sessions = results.map(r => this.mapDbSessionToSession(r));

    return { sessions, total: count };
  }

  // ========== 消息操作 ==========

  /**
   * 保存消息
   */
  async saveMessage(
    sessionId: string,
    message: SDKMessage
  ): Promise<void> {
    await this.db.insert(schema.sessionMessages).values({
      sessionId,
      messageType: message.type,
      subtype: (message as any).subtype,
      uuid: message.uuid,
      parentToolUseId: message.parent_tool_use_id,
      content: message as any,
      createdAt: Date.now(),
    });

    // 更新会话消息计数
    await this.db
      .update(schema.sessions)
      .set({
        messageCount: sql<number>`message_count + 1`,
        lastModified: Date.now(),
      })
      .where(eq(schema.sessions.sessionId, sessionId));
  }

  /**
   * 批量保存消息 (用于恢复会话)
   */
  async saveMessages(sessionId: string, messages: SDKMessage[]): Promise<void> {
    if (messages.length === 0) return;

    const values = messages.map(m => ({
      sessionId,
      messageType: m.type,
      subtype: (m as any).subtype,
      uuid: m.uuid,
      parentToolUseId: m.parent_tool_use_id,
      content: m as any,
      createdAt: Date.now(),
    }));

    await this.db.insert(schema.sessionMessages).values(values);
  }

  /**
   * 获取会话消息
   */
  async getMessages(
    sessionId: string,
    options: { limit?: number; offset?: number } = {}
  ): Promise<{ messages: SDKMessage[]; total: number }> {
    const { limit = 100, offset = 0 } = options;

    // 获取总数
    const [{ count }] = await this.db
      .select({ count: sql<number>`count(*)::int` })
      .from(schema.sessionMessages)
      .where(eq(schema.sessionMessages.sessionId, sessionId));

    // 获取消息
    const results = await this.db
      .select()
      .from(schema.sessionMessages)
      .where(eq(schema.sessionMessages.sessionId, sessionId))
      .orderBy(desc(schema.sessionMessages.createdAt))
      .limit(limit)
      .offset(offset);

    const messages = results.map(r => r.content as SDKMessage);

    return { messages, total: count };
  }

  // ========== 查询统计 ==========

  /**
   * 保存查询统计
   */
  async saveQueryStats(
    sessionId: string,
    apiKeyId: string | null,
    result: SDKResultMessage
  ): Promise<void> {
    await this.db.insert(schema.queryStats).values({
      sessionId,
      apiKeyId,
      numTurns: result.num_turns,
      durationMs: result.duration_ms,
      durationApiMs: result.duration_api_ms,
      inputTokens: result.usage.input_tokens,
      outputTokens: result.usage.output_tokens,
      cacheCreationInputTokens: result.usage.cache_creation_input_tokens,
      cacheReadInputTokens: result.usage.cache_read_input_tokens,
      totalCostUsd: result.total_cost_usd.toString(),
      modelUsage: result.model_usage as any,
      resultType: result.subtype,
      stopReason: result.stop_reason,
      createdAt: Date.now(),
    });

    // 更新会话总成本
    await this.db
      .update(schema.sessions)
      .set({
        totalCostUsd: sql<number>`total_cost_usd + ${result.total_cost_usd}`,
      })
      .where(eq(schema.sessions.sessionId, sessionId));
  }

  /**
   * 获取 API Key 统计
   */
  async getApiKeyStats(apiKeyId: string, days: number = 30): Promise<{
    totalQueries: number;
    totalCost: number;
    totalTokens: number;
  }> {
    const since = Date.now() - days * 24 * 60 * 60 * 1000;

    const result = await this.db
      .select({
        totalQueries: sql<number>`count(*)::int`,
        totalCost: sql<number>`sum(total_cost_usd)::decimal`,
        totalTokens: sql<number>`sum(input_tokens + output_tokens)::int`,
      })
      .from(schema.queryStats)
      .where(
        and(
          eq(schema.queryStats.apiKeyId, apiKeyId),
          sql<number>`created_at >= ${since}`
        )
      );

    return result[0] || { totalQueries: 0, totalCost: 0, totalTokens: 0 };
  }

  // ========== MCP 服务器 ==========

  /**
   * 保存 MCP 服务器配置
   */
  async saveMcpServer(
    ownerId: string,
    name: string,
    config: McpServerConfig
  ): Promise<void> {
    await this.db.insert(schema.mcpServers).values({
      ownerId,
      name,
      config: config as any,
      createdAt: Date.now(),
      updatedAt: Date.now(),
    }).onConflictDoUpdate({
      target: [schema.mcpServers.ownerId, schema.mcpServers.name],
      set: {
        config: config as any,
        updatedAt: Date.now(),
      },
    });
  }

  /**
   * 获取用户的 MCP 服务器
   */
  async getMcpServers(ownerId: string): Promise<McpServer[]> {
    const results = await this.db
      .select()
      .from(schema.mcpServers)
      .where(eq(schema.mcpServers.ownerId, ownerId));

    return results.map(r => ({
      ...r,
      config: r.config as McpServerConfig,
    }));
  }

  // ========== 辅助方法 ==========

  private mapDbSessionToSession(db: any): Session {
    return {
      sessionId: db.sessionId,
      ownerId: db.ownerId,
      status: db.status,
      createdAt: Number(db.createdAt),
      lastModified: Number(db.lastModified),
      summary: db.summary,
      customTitle: db.customTitle,
      firstPrompt: db.firstPrompt,
      tag: db.tag,
      cwd: db.cwd,
      gitBranch: db.gitBranch,
      options: db.options as SessionOptions,
      messageCount: db.messageCount,
      totalCostUsd: parseFloat(db.totalCostUsd),
      closedAt: db.closedAt ? Number(db.closedAt) : undefined,
      metadata: db.metadata,
    };
  }

  /**
   * 健康检查
   */
  async healthCheck(): Promise<void> {
    await this.db.execute(sql`SELECT 1`);
    if (this.redis) {
      await this.redis.ping();
    }
  }

  /**
   * 关闭连接
   */
  async close(): Promise<void> {
    if (this.redis) {
      await this.redis.quit();
    }
  }
}
```

### 5.4 AuthService (认证服务)

基于数据库的 API Key 认证服务。

```typescript
import { eq } from 'drizzle-orm';
import * as schema from './db/schema';

class AuthService {
  private db: ReturnType<typeof drizzle>;
  private redis?: Redis;

  constructor(db: ReturnType<typeof drizzle>, redis?: Redis) {
    this.db = db;
    this.redis = redis;
  }

  /**
   * 验证 API Key 并返回关联信息
   */
  async validateApiKey(apiKey: string): Promise<{
    valid: boolean;
    apiKeyId?: string;
    ownerId?: string;
    rateLimit?: number;
  }> {
    // 尝试从缓存获取
    const cacheKey = `apikey:hash:${this.hashApiKey(apiKey)}`;
    if (this.redis) {
      const cached = await this.redis.get(cacheKey);
      if (cached) {
        return JSON.parse(cached);
      }
    }

    // 从数据库查询
    const keyHash = this.hashApiKey(apiKey);
    const result = await this.db
      .select()
      .from(schema.apiKeys)
      .where(eq(schema.apiKeys.keyHash, keyHash))
      .limit(1);

    if (!result[0]) {
      return { valid: false };
    }

    const key = result[0];

    // 检查是否过期或已撤销
    const now = Date.now();
    if (key.revokedAt && key.revokedAt < now) {
      return { valid: false };
    }
    if (key.expiresAt && key.expiresAt < now) {
      return { valid: false };
    }

    const response = {
      valid: true,
      apiKeyId: key.id,
      ownerId: key.ownerId,
      rateLimit: key.rateLimit,
    };

    // 缓存结果
    if (this.redis) {
      await this.redis.setex(cacheKey, 300, JSON.stringify(response));
    }

    // 更新最后使用时间
    await this.db
      .update(schema.apiKeys)
      .set({ lastUsedAt: now })
      .where(eq(schema.apiKeys.id, key.id));

    return response;
  }

  /**
   * 创建新的 API Key
   */
  async createApiKey(
    ownerId: string,
    options: {
      name?: string;
      description?: string;
      scopes?: string[];
      rateLimit?: number;
      maxSessions?: number;
      expiresAt?: number;
    } = {}
  ): Promise<{ apiKey: string; prefix: string }> {
    // 生成 API Key
    const apiKey = this.generateApiKey();
    const keyHash = this.hashApiKey(apiKey);
    const keyPrefix = apiKey.slice(0, 8);

    await this.db.insert(schema.apiKeys).values({
      id: randomUUID(),
      ownerId,
      keyHash,
      keyPrefix,
      name: options.name,
      description: options.description,
      scopes: options.scopes || [],
      rateLimit: options.rateLimit || 100,
      maxSessions: options.maxSessions || 100,
      expiresAt: options.expiresAt,
      createdAt: Date.now(),
    });

    // 清除缓存
    if (this.redis) {
      await this.redis.del(`apikey:hash:${keyHash}`);
      await this.redis.del(`user:${ownerId}:apikeys`);
    }

    return { apiKey, prefix: keyPrefix };
  }

  /**
   * 列出用户的 API Keys
   */
  async listApiKeys(ownerId: string): Promise<Array<{
    id: string;
    prefix: string;
    name: string;
    createdAt: number;
    lastUsedAt?: number;
    expiresAt?: number;
  }>> {
    const cacheKey = `user:${ownerId}:apikeys`;
    if (this.redis) {
      const cached = await this.redis.get(cacheKey);
      if (cached) {
        return JSON.parse(cached);
      }
    }

    const results = await this.db
      .select({
        id: schema.apiKeys.id,
        keyPrefix: schema.apiKeys.keyPrefix,
        name: schema.apiKeys.name,
        createdAt: schema.apiKeys.createdAt,
        lastUsedAt: schema.apiKeys.lastUsedAt,
        expiresAt: schema.apiKeys.expiresAt,
      })
      .from(schema.apiKeys)
      .where(eq(schema.apiKeys.ownerId, ownerId))
      .orderBy(desc(schema.apiKeys.createdAt));

    const keys = results.map(r => ({
      id: r.id,
      prefix: r.keyPrefix,
      name: r.name || 'Unnamed',
      createdAt: Number(r.createdAt),
      lastUsedAt: r.lastUsedAt ? Number(r.lastUsedAt) : undefined,
      expiresAt: r.expiresAt ? Number(r.expiresAt) : undefined,
    }));

    if (this.redis) {
      await this.redis.setex(cacheKey, 60, JSON.stringify(keys));
    }

    return keys;
  }

  /**
   * 撤销 API Key
   */
  async revokeApiKey(apiKeyId: string, ownerId: string): Promise<boolean> {
    const result = await this.db
      .update(schema.apiKeys)
      .set({ revokedAt: Date.now() })
      .where(
        and(
          eq(schema.apiKeys.id, apiKeyId),
          eq(schema.apiKeys.ownerId, ownerId)
        )
      );

    if (this.redis) {
      await this.redis.del(`apikey:hash:*`);
      await this.redis.del(`user:${ownerId}:apikeys`);
    }

    return result.rowCount > 0;
  }

  // ========== 辅助方法 ==========

  private generateApiKey(): string {
    // 生成 64 字符的十六进制字符串
    return Array.from({ length: 64 }, () =>
      Math.floor(Math.random() * 16).toString(16)
    ).join('');
  }

  private hashApiKey(apiKey: string): string {
    // 使用 SHA-256 哈希 API Key
    return createHash('sha256').update(apiKey).digest('hex');
  }
}
```

---

## 6. 安全设计

### 6.1 认证机制

#### API Key 认证
```typescript
// 认证中间件
async function authMiddleware(
  request: FastifyRequest,
  reply: FastifyReply
): Promise<void> {
  const apiKey = request.headers['x-api-key'] as string;

  if (!apiKey || !authService.validateApiKey(apiKey)) {
    reply.status(401).send({
      error: {
        code: 'UNAUTHORIZED',
        message: 'Invalid or missing API key',
      },
    });
    return;
  }

  // 附加用户信息到请求
  request.user = { apiKey };
}
```

#### API Key 生成
```bash
# 使用 OpenSSL 生成安全的 API Key
openssl rand -hex 32
# 输出: a1b2c3d4e5f6... (64 字符)
```

### 6.2 授权控制

#### 会话隔离
每个 API Key 只能访问自己创建的会话。

```typescript
class SessionManager {
  async createSession(request: CreateSessionRequest, apiKey: string): Promise<Session> {
    const session = { ... };
    session.ownerApiKey = apiKey; // 标记所有者
    // ...
  }

  async getSession(sessionId: string, apiKey: string): Promise<Session | null> {
    const session = await this.getSessionInternal(sessionId);

    // 检查所有权
    if (session && session.ownerApiKey !== apiKey) {
      throw new Error('FORBIDDEN');
    }

    return session;
  }
}
```

#### 工具权限控制
```typescript
interface PermissionConfig {
  // 工具级权限
  allowedTools?: string[];
  disallowedTools?: string[];

  // 自定义权限检查
  canUseTool?: CanUseTool;
}

// 示例：限制 Bash 工具使用
const options: SessionOptions = {
  allowedTools: ['Read', 'Write', 'Edit', 'Grep', 'Glob'],
  // Bash 不在允许列表中，因此被禁用
};
```

### 6.3 限流策略

#### 基于令牌桶的限流
```typescript
import { TokenBucket } from 'token-bucket';

class RateLimiter {
  private buckets: Map<string, TokenBucket> = new Map();

  constructor(
    private config: {
      rate: number;      // 令牌/秒
      burst: number;     // 桶容量
    }
  ) {}

  /**
   * 检查是否允许请求
   */
  async checkLimit(apiKey: string): Promise<boolean> {
    let bucket = this.buckets.get(apiKey);

    if (!bucket) {
      bucket = new TokenBucket({
        rate: this.config.rate,
        burst: this.config.burst,
      });
      this.buckets.set(apiKey, bucket);
    }

    return bucket.consume(1);
  }

  /**
   * 获取剩余配额
   */
  getRemaining(apiKey: string): number {
    const bucket = this.buckets.get(apiKey);
    return bucket ? bucket.tokens : this.config.burst;
  }
}
```

#### 限流中间件
```typescript
async function rateLimitMiddleware(
  request: FastifyRequest,
  reply: FastifyReply
): Promise<void> {
  const apiKey = request.headers['x-api-key'] as string;
  const allowed = await rateLimiter.checkLimit(apiKey);

  if (!allowed) {
    const remaining = rateLimiter.getRemaining(apiKey);
    const resetAt = Date.now() + 60000; // 1 分钟后重置

    reply.status(429).setHeader('X-RateLimit-Remaining', '0');
    reply.setHeader('X-RateLimit-Reset', resetAt.toString());

    reply.send({
      error: {
        code: 'RATE_LIMITED',
        message: 'Too many requests',
        retryAfter: 60,
      },
    });
    return;
  }

  const remaining = rateLimiter.getRemaining(apiKey);
  reply.setHeader('X-RateLimit-Remaining', remaining.toString());
}
```

### 6.4 CORS 配置

```typescript
import fastifyCors from '@fastify/cors';

await fastify.register(fastifyCors, {
  // 生产环境：指定允许的来源
  origin: (origin, cb) => {
    const allowedOrigins = process.env.ALLOWED_ORIGINS?.split(',') || [];

    if (!origin || allowedOrigins.includes(origin)) {
      cb(null, true);
    } else {
      cb(new Error('Not allowed by CORS'), false);
    }
  },

  // 允许的凭证
  credentials: true,

  // 允许的请求头
  allowedHeaders: ['Content-Type', 'X-API-Key'],

  // 允许的方法
  methods: ['GET', 'POST', 'PATCH', 'DELETE'],

  // 预检缓存时间
  maxAge: 86400,
});
```

### 6.5 安全最佳实践

| 实践 | 说明 |
|------|------|
| HTTPS 强制 | 生产环境必须使用 HTTPS |
| API Key 轮换 | 定期轮换 API Key，设置过期时间 |
| 敏感数据脱敏 | 日志中不记录 API Key、敏感内容 |
| 输入验证 | 严格验证所有输入参数 |
| 输出编码 | JSON 编码防止注入 |
| 依赖安全 | 定期更新依赖，使用 npm audit |
| 秘钥管理 | 使用环境变量或密钥管理服务 |

---

## 7. 错误处理

### 7.1 错误分类

```typescript
enum ErrorCategory {
  // 客户端错误 (4xx)
  CLIENT_ERROR = 'CLIENT_ERROR',

  // 服务器错误 (5xx)
  SERVER_ERROR = 'SERVER_ERROR',

  // SDK 错误
  SDK_ERROR = 'SDK_ERROR',

  // 网络错误
  NETWORK_ERROR = 'NETWORK_ERROR',
}

class AppError extends Error {
  constructor(
    public code: ErrorCode,
    public category: ErrorCategory,
    message: string,
    public statusCode: number = 500,
    public details?: unknown
  ) {
    super(message);
    this.name = 'AppError';
  }
}

// 便捷构造函数
class Errors {
  static unauthorized(message = 'Unauthorized') {
    return new AppError(ErrorCode.UNAUTHORIZED, ErrorCategory.CLIENT_ERROR, message, 401);
  }

  static forbidden(message = 'Forbidden') {
    return new AppError(ErrorCode.FORBIDDEN, ErrorCategory.CLIENT_ERROR, message, 403);
  }

  static notFound(message = 'Not found') {
    return new AppError(ErrorCode.NOT_FOUND, ErrorCategory.CLIENT_ERROR, message, 404);
  }

  static rateLimited(message = 'Rate limit exceeded') {
    return new AppError(ErrorCode.RATE_LIMITED, ErrorCategory.CLIENT_ERROR, message, 429);
  }

  static internal(message = 'Internal server error') {
    return new AppError(ErrorCode.INTERNAL_ERROR, ErrorCategory.SERVER_ERROR, message, 500);
  }
}
```

### 7.2 错误处理中间件

```typescript
async function errorHandler(
  error: Error,
  request: FastifyRequest,
  reply: FastifyReply
): Promise<void> {
  const requestId = request.id;

  // 记录错误
  logger.error({
    requestId,
    error: error.message,
    stack: error.stack,
    url: request.url,
    method: request.method,
  });

  // 应用错误
  if (error instanceof AppError) {
    reply.status(error.statusCode).send({
      error: {
        code: error.code,
        message: error.message,
        details: error.details,
        requestId,
        timestamp: Date.now(),
      },
    });
    return;
  }

  // 未知错误
  reply.status(500).send({
    error: {
      code: ErrorCode.INTERNAL_ERROR,
      message: 'An unexpected error occurred',
      requestId,
      timestamp: Date.now(),
    },
  });
}
```

### 7.3 错误响应示例

```json
// 400 Bad Request
{
  "error": {
    "code": "INVALID_REQUEST",
    "message": "Invalid request body",
    "details": {
      "issues": [
        {
          "path": ["options", "model"],
          "message": "Invalid model name"
        }
      ]
    },
    "requestId": "req_123456",
    "timestamp": 1711804800000
  }
}

// 404 Not Found
{
  "error": {
    "code": "SESSION_NOT_FOUND",
    "message": "Session not found",
    "details": {
      "sessionId": "550e8400-e29b-41d4-a716-446655440000"
    },
    "requestId": "req_123456",
    "timestamp": 1711804800000
  }
}

// 429 Rate Limited
{
  "error": {
    "code": "RATE_LIMITED",
    "message": "Rate limit exceeded",
    "details": {
      "retryAfter": 60,
      "limit": 100,
      "window": "1m"
    },
    "requestId": "req_123456",
    "timestamp": 1711804800000
  }
}
```

---

## 8. 部署指南

### 8.1 环境要求

```yaml
系统要求:
  操作系统: Linux (Ubuntu 20.04+, CentOS 8+) / macOS 12+ / Windows 10+
  Node.js: 18.x LTS 或 20.x LTS
  内存: 最低 512MB，推荐 2GB+
  磁盘: 最低 1GB 可用空间
  网络: 需要访问 Anthropic API

运行时依赖:
  - Node.js 18+
  - npm 9+ 或 pnpm 8+
  - Redis 7+ (可选，用于缓存)
  - NFS/GlusterFS (多实例部署时需要)
```

### 8.2 安装步骤

#### 1. 克隆代码
```bash
git clone https://github.com/your-org/claude-api-service.git
cd claude-api-service
```

#### 2. 安装依赖
```bash
npm install
# 或
pnpm install
```

#### 3. 配置环境变量
```bash
cp .env.example .env
vi .env
```

#### 4. 构建项目
```bash
npm run build
```

#### 5. 启动服务
```bash
# 开发模式
npm run dev

# 生产模式
npm start
```

### 8.3 环境变量配置

```bash
# .env 文件示例

# ========== 服务配置 ==========
PORT=3000
NODE_ENV=production

# ========== Claude SDK ==========
ANTHROPIC_API_KEY=sk-ant-xxxxx
CLAUDE_DEFAULT_MODEL=claude-sonnet-4-20250514

# ========== 数据库配置 ==========
DATABASE_URL=postgresql://user:password@localhost:5432/claude_api?schema=public
# 或者使用连接池格式:
# DATABASE_URL=postgres://user:password@host:port/database?schema=public&pgbouncer=true

# ========== Redis 缓存 (可选) ==========
REDIS_URL=redis://localhost:6379
REDIS_PASSWORD=
# 留空则禁用 Redis 缓存

# ========== 认证 ==========
# 注意: API Keys 现在存储在数据库中
# 可以通过管理员 API 或命令行工具创建

# ========== 安全 ==========
ALLOWED_ORIGINS=http://localhost:5173,https://yourdomain.com
ENABLE_CORS=true

# ========== 限流 ==========
RATE_LIMIT_ENABLED=true
RATE_LIMIT_MAX=100
RATE_LIMIT_WINDOW_MS=60000

# ========== 会话配置 ==========
MAX_SESSIONS=1000
SESSION_TIMEOUT_MS=1800000
CLEANUP_INTERVAL_MS=300000

# ========== 文件检查点 (可选) ==========
ENABLE_FILE_CHECKPOINTING=false
CHECKPOINTS_DIR=./data/checkpoints

# ========== 日志 ==========
LOG_LEVEL=info
LOG_FORMAT=json

# ========== 监控 ==========
ENABLE_METRICS=true
METRICS_PORT=9090
```

### 8.4 数据库初始化

#### PostgreSQL 安装

**Ubuntu/Debian:**
```bash
sudo apt update
sudo apt install postgresql-15 postgresql-contrib-15
sudo systemctl start postgresql
sudo systemctl enable postgresql
```

**macOS:**
```bash
brew install postgresql@15
brew services start postgresql@15
```

**Docker:**
```bash
docker run -d \
  --name postgres \
  -e POSTGRES_USER=claude \
  -e POSTGRES_PASSWORD=your_password \
  -e POSTGRES_DB=claude_api \
  -p 5432:5432 \
  -v postgres_data:/var/lib/postgresql/data \
  postgres:15-alpine
```

#### 创建数据库和用户

```bash
# 进入 PostgreSQL
sudo -u postgres psql

# 创建数据库和用户
CREATE DATABASE claude_api;
CREATE USER claude_api_user WITH PASSWORD 'your_secure_password';
GRANT ALL PRIVILEGES ON DATABASE claude_api TO claude_api_user;
\q
```

#### 运行数据库迁移

```bash
# 使用 Drizzle Kit 生成迁移
npm run db:generate

# 执行迁移
npm run db:migrate

# 或使用 drizzle-kit
npx drizzle-kit push:pg
```

#### 创建初始用户和管理员 API Key

```bash
# 使用内置命令行工具
npm run cli:create-user --email admin@example.com --password admin_password

# 创建管理员 API Key
npm run cli:create-api-key --user-id <user_id> --name "Admin Key"

# 输出示例:
# API Key created successfully!
# Key: a1b2c3d4e5f6... (请妥善保管)
# Prefix: a1b2c3d4
```

### 8.5 Docker 部署

#### Dockerfile
```dockerfile
# 多阶段构建
FROM node:20-alpine AS builder

WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

# 生产镜像
FROM node:20-alpine

RUN addgroup -g 1001 -S nodejs && \
    adduser -S nodejs -u 1001

WORKDIR /app

COPY --from=builder --chown=nodejs:nodejs /app/dist ./dist
COPY --from=builder --chown=nodejs:nodejs /app/node_modules ./node_modules
COPY --from=builder --chown=nodejs:nodejs /app/package.json ./

USER nodejs

EXPOSE 3000

CMD ["node", "dist/index.js"]
```

#### docker-compose.yml
```yaml
version: '3.8'

services:
  api:
    build: .
    ports:
      - "3000:3000"
    environment:
      - NODE_ENV=production
      - ANTHROPIC_API_KEY=${ANTHROPIC_API_KEY}
      - DATABASE_URL=postgresql://claude:${POSTGRES_PASSWORD}@postgres:5432/claude_api?schema=public
      - REDIS_URL=redis://redis:6379
    depends_on:
      - postgres
      - redis
    restart: unless-stopped

  postgres:
    image: postgres:15-alpine
    environment:
      - POSTGRES_USER=claude
      - POSTGRES_PASSWORD=${POSTGRES_PASSWORD}
      - POSTGRES_DB=claude_api
    volumes:
      - postgres_data:/var/lib/postgresql/data
    restart: unless-stopped

  redis:
    image: redis:7-alpine
    volumes:
      - redis_data:/data
    restart: unless-stopped

volumes:
  postgres_data:
  redis_data:
```

#### 首次启动

```bash
# 复制环境变量模板
cp .env.example .env

# 编辑 .env 文件，设置必要的环境变量
vi .env

# 启动服务
docker-compose up -d

# 等待数据库启动
sleep 5

# 运行数据库迁移
docker-compose exec api npm run db:migrate

# 创建初始管理员用户和 API Key
docker-compose exec api npm run cli:create-user --email admin@example.com --password admin_password
docker-compose exec api npm run cli:create-api-key --email admin@example.com --name "Admin Key"
```

### 8.6 Nginx 配置

```nginx
# /etc/nginx/conf.d/claude-api.conf

upstream claude_api {
    least_conn;
    server 127.0.0.1:3001;
    server 127.0.0.1:3002;
    server 127.0.0.1:3003;
}

server {
    listen 443 ssl http2;
    server_name api.example.com;

    # SSL 证书
    ssl_certificate /etc/letsencrypt/live/api.example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/api.example.com/privkey.pem;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;

    # 日志
    access_log /var/log/nginx/claude-api-access.log;
    error_log /var/log/nginx/claude-api-error.log;

    # API 路由
    location /api/ {
        proxy_pass http://claude_api;

        # SSE 特殊处理
        proxy_http_version 1.1;
        proxy_set_header Connection '';
        proxy_buffering off;
        proxy_cache off;
        chunked_transfer_encoding off;

        # 通用代理头
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # 超时
        proxy_connect_timeout 60s;
        proxy_send_timeout 300s;
        proxy_read_timeout 300s;
    }

    # 健康检查
    location /health {
        proxy_pass http://claude_api/health;
        access_log off;
    }
}

# HTTP 重定向到 HTTPS
server {
    listen 80;
    server_name api.example.com;
    return 301 https://$server_name$request_uri;
}
```

### 8.7 PM2 配置

```javascript
// ecosystem.config.js
module.exports = {
  apps: [
    {
      name: 'claude-api',
      script: './dist/index.js',
      instances: 'max',
      exec_mode: 'cluster',
      env: {
        NODE_ENV: 'production',
        PORT: 3000,
        DATABASE_URL: 'postgresql://user:password@localhost:5432/claude_api',
      },
      error_file: './logs/error.log',
      out_file: './logs/out.log',
      log_date_format: 'YYYY-MM-DD HH:mm:ss Z',
      merge_logs: true,
      max_memory_restart: '1G',
      autorestart: true,
      watch: false,
    },
  ],
};
```

```bash
# 启动
pm2 start ecosystem.config.js

# 查看状态
pm2 status

# 查看日志
pm2 logs

# 重启
pm2 restart claude-api

# 停止
pm2 stop claude-api
```

---

## 9. 开发指南

### 9.1 项目结构

```
claude-api-service/
├── src/
│   ├── routes/                    # 路由定义
│   │   ├── sessions.ts            # 会话管理路由
│   │   ├── queries.ts             # 查询路由
│   │   ├── mcp.ts                 # MCP 管理路由
│   │   ├── capabilities.ts        # 能力发现路由
│   │   └── index.ts               # 路由聚合
│   │
│   ├── services/                  # 业务服务
│   │   ├── SessionManager.ts      # 会话管理器
│   │   ├── SSEStreamHandler.ts    # SSE 处理器
│   │   ├── StorageService.ts      # 存储服务
│   │   ├── AuthService.ts         # 认证服务
│   │   └── RateLimiter.ts         # 限流服务
│   │
│   ├── middleware/                # 中间件
│   │   ├── auth.ts                # 认证中间件
│   │   ├── rateLimit.ts           # 限流中间件
│   │   ├── errorHandler.ts        # 错误处理
│   │   └── logger.ts              # 日志中间件
│   │
│   ├── types/                     # 类型定义
│   │   ├── api.ts                 # API 类型
│   │   ├── sdk.ts                 # SDK 类型
│   │   └── models.ts              # 数据模型
│   │
│   ├── db/                        # 数据库
│   │   ├── schema.ts              # Drizzle ORM Schema
│   │   ├── migrations/            # 数据库迁移文件
│   │   └── seed.ts                # 种子数据
│   │
│   ├── config/                    # 配置
│   │   ├── defaults.ts            # 默认配置
│   │   ├── index.ts               # 配置聚合
│   │   └── schema.ts              # 配置验证
│   │
│   ├── utils/                     # 工具函数
│   │   ├── logger.ts              # 日志工具
│   │   ├── errors.ts              # 错误类
│   │   └── validation.ts          # 验证工具
│   │
│   └── index.ts                   # 应用入口
│
├── tests/                         # 测试
│   ├── unit/                      # 单元测试
│   ├── integration/               # 集成测试
│   └── fixtures/                  # 测试数据
│
├── docs/                          # 文档
│   ├── api.md                     # API 文档
│   └── deployment.md              # 部署文档
│
├── data/                          # 数据目录
│   └── checkpoints/              # 文件检查点 (可选)
│
├── logs/                          # 日志目录
│
├── .env.example                   # 环境变量示例
├── .gitignore
├── package.json
├── tsconfig.json
├── Dockerfile
├── docker-compose.yml
├── ecosystem.config.js            # PM2 配置
└── README.md
```

### 9.2 开发工作流

```bash
# 1. 安装依赖
pnpm install

# 2. 设置环境变量
cp .env.example .env
# 编辑 .env 文件设置 DATABASE_URL

# 3. 初始化数据库
pnpm db:push     # 推送 schema 到数据库
# 或
pnpm db:migrate  # 运行迁移

# 4. 启动开发模式（热重载）
pnpm dev

# 5. 运行类型检查
pnpm type-check

# 6. 运行 Lint
pnpm lint

# 7. 运行测试
pnpm test

# 8. 构建
pnpm build

# 数据库相关命令:
pnpm db:generate  # 生成迁移文件
pnpm db:push      # 推送 schema 到数据库 (开发环境)
pnpm db:migrate   # 运行迁移 (生产环境)
pnpm db:studio     # 打开 Drizzle Studio
```

### 9.3 API 开发示例

#### 创建新路由

```typescript
// src/routes/custom.ts
import { FastifyInstance } from 'fastify';

export async function customRoutes(fastify: FastifyInstance) {
  // GET /api/v1/custom
  fastify.get('/custom', {
    preHandler: [fastify.authenticate, fastify.rateLimit],
    schema: {
      tags: ['Custom'],
      summary: 'Custom endpoint',
      response: {
        200: {
          type: 'object',
          properties: {
            message: { type: 'string' },
          },
        },
      },
    },
  }, async (request, reply) => {
    return { message: 'Hello from custom endpoint!' };
  });
}
```

#### 添加中间件

```typescript
// src/middleware/timing.ts
import { FastifyRequest, FastifyReply } from 'fastify';

export function timingMiddleware(
  request: FastifyRequest,
  reply: FastifyReply,
  done: () => void
) {
  const start = Date.now();

  reply.raw.on('finish', () => {
    const duration = Date.now() - start;
    request.log.info({ duration }, 'Request completed');
  });

  done();
}
```

### 9.4 测试策略

```typescript
// tests/integration/sessions.test.ts
import { buildApp } from '../helper';

describe('Sessions API', () => {
  let app: FastifyInstance;

  beforeAll(async () => {
    app = await buildApp();
  });

  afterAll(async () => {
    await app.close();
  });

  describe('POST /api/v1/sessions', () => {
    it('should create a new session', async () => {
      const response = await app.inject({
        method: 'POST',
        url: '/api/v1/sessions',
        headers: {
          'x-api-key': 'test-key',
        },
        payload: {
          initialPrompt: 'Hello Claude',
        },
      });

      expect(response.statusCode).toBe(201);
      expect(response.json()).toMatchObject({
        sessionId: expect.any(String),
        status: 'idle',
      });
    });
  });

  describe('GET /api/v1/sessions/:id', () => {
    it('should return session details', async () => {
      // 先创建会话
      const createResponse = await app.inject({
        method: 'POST',
        url: '/api/v1/sessions',
        headers: { 'x-api-key': 'test-key' },
      });

      const { sessionId } = createResponse.json();

      // 获取会话
      const response = await app.inject({
        method: 'GET',
        url: `/api/v1/sessions/${sessionId}`,
        headers: { 'x-api-key': 'test-key' },
      });

      expect(response.statusCode).toBe(200);
      expect(response.json()).toMatchObject({
        sessionId,
        status: 'idle',
      });
    });
  });
});
```

---

## 10. 监控与运维

### 10.1 日志管理

#### 结构化日志
```typescript
import pino from 'pino';

export const logger = pino({
  level: process.env.LOG_LEVEL || 'info',
  formatters: {
    level: (label) => {
      return { level: label };
    },
  },
  serializers: {
    req: pino.stdSerializers.req,
    res: pino.stdSerializers.res,
    err: pino.stdSerializers.err,
  },
  // 开发环境使用 pretty print
  transport:
    process.env.NODE_ENV === 'development'
      ? {
          target: 'pino-pretty',
          options: {
            colorize: true,
            translateTime: 'HH:MM:ss Z',
            ignore: 'pid,hostname',
          },
        }
      : undefined,
});
```

#### 日志输出示例
```json
{
  "level": "info",
  "time": 1711804800000,
  "msg": "Session created",
  "sessionId": "550e8400-e29b-41d4-a716-446655440000",
  "apiKey": "key1",
  "reqId": "req_123456"
}
```

### 10.2 指标收集

```typescript
import { Counter, Histogram, Registry } from 'prom-client';

export const metrics = {
  // 请求计数
  requestsTotal: new Counter({
    name: 'http_requests_total',
    help: 'Total number of HTTP requests',
    labelNames: ['method', 'route', 'status'],
  }),

  // 请求延迟
  requestDuration: new Histogram({
    name: 'http_request_duration_seconds',
    help: 'HTTP request duration',
    labelNames: ['method', 'route'],
    buckets: [0.1, 0.5, 1, 2, 5, 10],
  }),

  // 活跃会话数
  activeSessions: new Gauge({
    name: 'active_sessions_total',
    help: 'Number of active sessions',
  }),

  // 查询执行时间
  queryDuration: new Histogram({
    name: 'query_duration_seconds',
    help: 'Query execution duration',
    buckets: [1, 5, 10, 30, 60, 120, 300],
  }),
};
```

### 10.3 健康检查

```typescript
// 健康检查端点
fastify.get('/health', async (request, reply) => {
  const checks = {
    sdk: 'ok',
    storage: 'ok',
    redis: 'ok',
  };

  // 检查 SDK
  try {
    await sessionManager.healthCheck();
  } catch (err) {
    checks.sdk = 'error';
  }

  // 检查存储
  try {
    await storageService.healthCheck();
  } catch (err) {
    checks.storage = 'error';
  }

  // 检查 Redis
  if (redis) {
    try {
      await redis.ping();
    } catch (err) {
      checks.redis = 'error';
    }
  }

  const isHealthy = Object.values(checks).every(v => v === 'ok');

  return reply
    .status(isHealthy ? 200 : 503)
    .send({
      status: isHealthy ? 'healthy' : 'unhealthy',
      version: packageInfo.version,
      timestamp: Date.now(),
      checks,
    });
});
```

### 10.4 告警规则

```yaml
# prometheus/alerts.yml
groups:
  - name: claude-api
    interval: 30s
    rules:
      # 高错误率
      - alert: HighErrorRate
        expr: rate(http_requests_total{status=~"5.."}[5m]) > 0.05
        for: 5m
        annotations:
          summary: "High error rate detected"
          description: "Error rate is {{ $value }} for the last 5 minutes"

      # 慢查询
      - alert: SlowQueries
        expr: histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m])) > 5
        for: 10m
        annotations:
          summary: "Slow queries detected"
          description: "P95 latency is {{ $value }}s"

      # 高内存使用
      - alert: HighMemoryUsage
        expr: process_resident_memory_bytes / 1024 / 1024 > 1024
        for: 5m
        annotations:
          summary: "High memory usage"
          description: "Memory usage is {{ $value }}MB"
```

---

## 11. 测试策略

### 11.1 测试金字塔

```
        /\
       /E2E\          - 少量端到端测试
      /------\
     /  集成  \        - 中等数量集成测试
    /----------\
   /   单元测试  \     - 大量单元测试
  /--------------\
```

### 11.2 单元测试

```typescript
// tests/unit/SessionManager.test.ts
import { SessionManager } from '@/services/SessionManager';

describe('SessionManager', () => {
  let manager: SessionManager;

  beforeEach(() => {
    manager = new SessionManager({
      sessionsDir: '/tmp/test-sessions',
    });
  });

  afterEach(async () => {
    await manager.stop();
  });

  describe('createSession', () => {
    it('should create a session with valid input', async () => {
      const session = await manager.createSession({
        initialPrompt: 'Test prompt',
      });

      expect(session.sessionId).toMatch(/^[0-9a-f-]{36}$/);
      expect(session.status).toBe('idle');
      expect(session.summary).toBe('Test prompt');
    });

    it('should throw error when max sessions exceeded', async () => {
      const manager = new SessionManager({
        maxSessions: 1,
      });

      await manager.createSession({ initialPrompt: 'First' });

      await expect(
        manager.createSession({ initialPrompt: 'Second' })
      ).rejects.toThrow('MAX_SESSIONS_EXCEEDED');
    });
  });
});
```

### 11.3 集成测试

```typescript
// tests/integration/api.test.ts
import { buildApp } from './helper';

describe('API Integration Tests', () => {
  let app: FastifyInstance;

  beforeAll(async () => {
    app = await buildApp();
  });

  afterAll(async () => {
    await app.close();
  });

  describe('Query Flow', () => {
    it('should complete a full query cycle', async () => {
      // 1. 创建会话
      const createResponse = await app.inject({
        method: 'POST',
        url: '/api/v1/sessions',
        headers: { 'x-api-key': 'test-key' },
        payload: {
          initialPrompt: 'What is 2+2?',
          options: {
            permissionMode: 'bypassPermissions',
          },
        },
      });

      expect(createResponse.statusCode).toBe(201);
      const { sessionId } = createResponse.json();

      // 2. 获取会话状态
      const statusResponse = await app.inject({
        method: 'GET',
        url: `/api/v1/sessions/${sessionId}`,
        headers: { 'x-api-key': 'test-key' },
      });

      expect(statusResponse.statusCode).toBe(200);
      expect(statusResponse.json().status).toBe('idle');

      // 3. 获取消息
      const messagesResponse = await app.inject({
        method: 'GET',
        url: `/api/v1/sessions/${sessionId}/messages`,
        headers: { 'x-api-key': 'test-key' },
      });

      expect(messagesResponse.statusCode).toBe(200);
      const messages = messagesResponse.json().messages;
      expect(messages.length).toBeGreaterThan(0);

      // 4. 删除会话
      const deleteResponse = await app.inject({
        method: 'DELETE',
        url: `/api/v1/sessions/${sessionId}`,
        headers: { 'x-api-key': 'test-key' },
      });

      expect(deleteResponse.statusCode).toBe(204);
    });
  });
});
```

### 11.4 E2E 测试

```typescript
// tests/e2e/sse-flow.test.ts
import { EventSource } from 'eventsource';

describe('SSE Flow E2E', () => {
  it('should receive streamed messages', async () => {
    // 创建会话
    const createResponse = await fetch('http://localhost:3000/api/v1/sessions', {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'X-API-Key': 'test-key',
      },
      body: JSON.stringify({}),
    });

    const { sessionId } = await createResponse.json();

    // 连接 SSE
    const events: SDKMessage[] = [];
    const es = new EventSource(
      `http://localhost:3000/api/v1/sessions/${sessionId}/query`,
      {
        headers: { 'X-API-Key': 'test-key' },
      }
    );

    es.addEventListener('message', (e) => {
      events.push(JSON.parse(e.data));
    });

    // 等待完成
    await new Promise<void>((resolve) => {
      es.addEventListener('done', () => {
        es.close();
        resolve();
      });
    });

    // 验证
    expect(events.length).toBeGreaterThan(0);
    expect(events[0].type).toBe('system');
    expect(events[events.length - 1].type).toBe('result');
  }, 30000);
});
```

---

## 12. 附录

### 12.1 完整类型定义

```typescript
// src/types/api.ts
export interface CreateSessionRequest {
  initialPrompt?: string;
  options?: SessionOptions;
}

export interface SessionOptions {
  model?: string;
  agent?: string;
  permissionMode?: PermissionMode;
  allowedTools?: string[];
  disallowedTools?: string[];
  systemPrompt?: string | SystemPromptPreset;
  cwd?: string;
  enableFileCheckpointing?: boolean;
  maxBudgetUsd?: number;
  maxTurns?: number;
  effort?: 'low' | 'medium' | 'high' | 'max';
  thinking?: ThinkingConfig;
  outputFormat?: OutputFormat;
  agents?: Record<string, AgentDefinition>;
  mcpServers?: Record<string, McpServerConfig>;
}

export interface QueryRequest {
  prompt: string;
  options?: {
    maxTurns?: number;
    maxBudgetUsd?: number;
    effort?: 'low' | 'medium' | 'high' | 'max';
    thinking?: ThinkingConfig;
    outputFormat?: OutputFormat;
  };
}

export interface ErrorResponse {
  error: {
    code: ErrorCode;
    message: string;
    details?: unknown;
    requestId?: string;
    timestamp: number;
  };
}

export enum ErrorCode {
  UNKNOWN_ERROR = 'UNKNOWN_ERROR',
  INVALID_REQUEST = 'INVALID_REQUEST',
  UNAUTHORIZED = 'UNAUTHORIZED',
  FORBIDDEN = 'FORBIDDEN',
  NOT_FOUND = 'NOT_FOUND',
  RATE_LIMITED = 'RATE_LIMITED',
  INTERNAL_ERROR = 'INTERNAL_ERROR',
  SESSION_NOT_FOUND = 'SESSION_NOT_FOUND',
  SESSION_CLOSED = 'SESSION_CLOSED',
  SESSION_ACTIVE = 'SESSION_ACTIVE',
  QUERY_FAILED = 'QUERY_FAILED',
  QUERY_CANCELLED = 'QUERY_CANCELLED',
  QUERY_TIMEOUT = 'QUERY_TIMEOUT',
  SDK_ERROR = 'SDK_ERROR',
  SDK_AUTHENTICATION_FAILED = 'SDK_AUTHENTICATION_FAILED',
  SDK_RATE_LIMIT = 'SDK_RATE_LIMIT',
  SDK_QUOTA_EXCEEDED = 'SDK_QUOTA_EXCEEDED',
  MCP_SERVER_ERROR = 'MCP_SERVER_ERROR',
  MCP_SERVER_NOT_FOUND = 'MCP_SERVER_NOT_FOUND',
}
```

### 12.2 客户端使用示例

#### JavaScript/TypeScript

```typescript
// 客户端封装
class ClaudeAPIClient {
  constructor(
    private baseURL: string,
    private apiKey: string
  ) {}

  /**
   * 创建会话
   */
  async createSession(options: CreateSessionRequest = {}): Promise<Session> {
    const response = await fetch(`${this.baseURL}/sessions`, {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'X-API-Key': this.apiKey,
      },
      body: JSON.stringify(options),
    });

    if (!response.ok) {
      const error: ErrorResponse = await response.json();
      throw new Error(error.error.message);
    }

    return response.json();
  }

  /**
   * 流式查询
   */
  async *query(
    sessionId: string,
    prompt: string,
    options?: QueryOptions
  ): AsyncGenerator<SDKMessage> {
    const response = await fetch(`${this.baseURL}/sessions/${sessionId}/query`, {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'X-API-Key': this.apiKey,
        'Accept': 'text/event-stream',
      },
      body: JSON.stringify({ prompt, options }),
    });

    if (!response.ok) {
      const error: ErrorResponse = await response.json();
      throw new Error(error.error.message);
    }

    const reader = response.body!.getReader();
    const decoder = new TextDecoder();
    let buffer = '';

    while (true) {
      const { done, value } = await reader.read();
      if (done) break;

      buffer += decoder.decode(value, { stream: true });

      const lines = buffer.split('\n');
      buffer = lines.pop() || '';

      for (const line of lines) {
        if (line.startsWith('data: ')) {
          const data = line.slice(6);
          if (data === '[DONE]') continue;

          try {
            const message: SDKMessage = JSON.parse(data);
            yield message;
          } catch (err) {
            console.error('Failed to parse SSE data', { data, err });
          }
        }
      }
    }
  }

  /**
   * 获取会话消息
   */
  async getMessages(
    sessionId: string,
    options: { limit?: number; offset?: number } = {}
  ): Promise<SessionMessage[]> {
    const params = new URLSearchParams();
    if (options.limit) params.set('limit', options.limit.toString());
    if (options.offset) params.set('offset', options.offset.toString());

    const response = await fetch(
      `${this.baseURL}/sessions/${sessionId}/messages?${params}`,
      {
        headers: { 'X-API-Key': this.apiKey },
      }
    );

    if (!response.ok) {
      const error: ErrorResponse = await response.json();
      throw new Error(error.error.message);
    }

    const result: MessagesResponse = await response.json();
    return result.messages;
  }

  /**
   * 列出会话
   */
  async listSessions(
    options: { dir?: string; limit?: number } = {}
  ): Promise<Session[]> {
    const params = new URLSearchParams();
    if (options.dir) params.set('dir', options.dir);
    if (options.limit) params.set('limit', options.limit.toString());

    const response = await fetch(
      `${this.baseURL}/sessions?${params}`,
      {
        headers: { 'X-API-Key': this.apiKey },
      }
    );

    if (!response.ok) {
      const error: ErrorResponse = await response.json();
      throw new Error(error.error.message);
    }

    const result: SessionsList = await response.json();
    return result.sessions;
  }

  /**
   * 删除会话
   */
  async deleteSession(sessionId: string): Promise<void> {
    const response = await fetch(`${this.baseURL}/sessions/${sessionId}`, {
      method: 'DELETE',
      headers: { 'X-API-Key': this.apiKey },
    });

    if (!response.ok) {
      const error: ErrorResponse = await response.json();
      throw new Error(error.error.message);
    }
  }
}

// 使用示例
async function main() {
  const client = new ClaudeAPIClient(
    'https://api.example.com/api/v1',
    process.env.API_KEY!
  );

  // 创建会话
  const session = await client.createSession({
    initialPrompt: '你好 Claude',
    options: {
      model: 'claude-sonnet-4-20250514',
      permissionMode: 'bypassPermissions',
    },
  });

  console.log('Created session:', session.sessionId);

  // 流式查询
  for await (const message of client.query(session.sessionId, '介绍一下你自己')) {
    if (message.type === 'assistant') {
      console.log('Assistant:', message);
    } else if (message.type === 'result') {
      console.log('Result:', message.result);
      break;
    }
  }

  // 获取历史消息
  const messages = await client.getMessages(session.sessionId, { limit: 10 });
  console.log('Messages:', messages);

  // 删除会话
  await client.deleteSession(session.sessionId);
}
```

### 12.3 性能优化建议

| 优化项 | 说明 | 预期收益 |
|--------|------|----------|
| 连接复用 | 使用 HTTP Keep-Alive | 减少连接开销 |
| 响应压缩 | 启用 gzip/brotli | 减少传输大小 50-70% |
| 缓存策略 | Redis 缓存热会话 | 减少磁盘 I/O |
| 集群模式 | Node.js Cluster | 充分利用多核 |
| 负载均衡 | Nginx 反向代理 | 水平扩展 |
| 慢查询日志 | 记录超过阈值的查询 | 定位性能瓶颈 |

### 12.4 常见问题

#### Q: SSE 连接断开怎么办？
A: 实现自动重连机制：
```typescript
function createReconnectingSSE(url: string) {
  let es: EventSource | null = null;

  function connect() {
    es = new EventSource(url);

    es.addEventListener('error', () => {
      console.log('Connection lost, reconnecting in 3s...');
      setTimeout(connect, 3000);
    });
  }

  connect();

  return es;
}
```

#### Q: 如何处理大文件传输？
A: 对于大文件，使用分块传输或提供下载 URL，而非直接在响应中返回。

#### Q: 多实例部署如何共享会话？
A: 使用共享存储（NFS）和 Redis 缓存，确保会话可跨实例访问。

#### Q: 如何限制资源使用？
A: 配置 `maxTurns`、`maxBudgetUsd` 和 `enableFileCheckpointing` 选项。

---

## 版本历史

| 版本 | 日期 | 变更 |
|------|------|------|
| 1.0.0 | 2026-03-30 | 初始版本 |

---

**文档结束**
