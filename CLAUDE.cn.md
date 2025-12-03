# CLAUDE.md

## 项目概述

Langfuse 是一个开源的 LLM 工程平台，帮助团队协作开发、监控、评估和调试 AI 应用程序。
主要功能领域包括追踪（tracing）、评估（evals）和提示管理（prompt management）。Langfuse 由 Web 应用（本仓库）、文档、Python SDK 和 JavaScript/TypeScript SDK 组成。
本仓库包含 Web 应用、Worker 和支持包，但不包括 JS 或 Python 客户端 SDK。

## Claude Code 技能

本仓库提供了专门的 Claude Code 技能来辅助开发：

- **backend-dev-guidelines** - 全面的后端开发指南，涵盖 tRPC 路由、公共 API、BullMQ 队列、服务、中间件、数据库模式、测试等内容。在进行后端功能开发时使用。
- **skill-developer** - 用于创建和管理 Claude Code 技能的元技能。

技能会根据你的工作上下文自动激活，你也可以在请求中引用它们。

## 仓库结构

高层结构。还有更多文件夹（例如 hooks 等）。

```
langfuse/
├── .claude/                 # Claude Code 配置
│   ├── skills/             # Claude Code 技能（backend-dev-guidelines 等）
│   ├── hooks/              # 开发钩子
│   └── agents/             # 自定义代理
├── web/                     # Next.js 14 前端/后端应用
│   ├── src/
│   │   ├── __tests__/      # Jest 测试（sync, async, e2e）
│   │   ├── components/     # 可复用 UI 组件（shadcn/ui）
│   │   ├── features/       # 按领域组织的功能特定代码
│   │   ├── pages/          # Next.js 页面（Pages Router）
│   │   │   └── api/        # API 路由（tRPC、公共 REST API）
│   │   ├── server/         # tRPC API 路由和服务器逻辑
│   │   │   ├── api/        # tRPC 路由器
│   │   │   ├── auth.ts     # NextAuth.js 配置
│   │   │   └── db.ts       # 数据库客户端
│   │   ├── env.mjs         # 环境配置（Zod 验证）
│   │   └── instrumentation.ts  # OpenTelemetry 设置
│   └── public/             # 静态资源
├── worker/                  # Express.js 后台任务处理器
│   └── src/
│       ├── __tests__/      # Vitest 测试
│       ├── queues/         # BullMQ 队列处理器
│       ├── features/       # 业务逻辑
│       ├── backgroundMigrations/  # 后台迁移任务
│       ├── app.ts          # Express 设置 + 队列注册
│       ├── env.ts          # 环境配置
│       └── instrumentation.ts  # OpenTelemetry 设置
├── packages/
│   ├── shared/             # 共享类型、模式和工具
│   │   ├── prisma/         # 数据库模式和迁移
│   │   ├── clickhouse/     # ClickHouse 迁移和模式
│   │   └── src/            # 共享 TypeScript 代码
│   │       ├── server/     # 服务器端工具（队列、认证、服务）
│   │       ├── features/   # 功能特定的共享代码
│   │       ├── db.ts       # Prisma 客户端
│   │       └── env.ts      # 环境配置
│   ├── config-eslint/      # ESLint 配置
│   └── config-typescript/  # TypeScript 配置
├── ee/                     # 企业版功能
├── fern/                   # API 文档和 OpenAPI 规范
├── scripts/                # 开发和部署脚本
└── docker-compose.*.yml    # 开发/生产的 Docker 配置
```

## 仓库架构

这是一个 **pnpm + Turbo monorepo**，包含以下关键包：

### 核心应用
- **`/web/`** - Next.js 14 应用（Pages Router），提供前端 UI 和后端 API
  - 前端：React 组件、页面和客户端逻辑
  - 后端：tRPC 过程和公共 REST API
- **`/worker/`** - Express.js 后台任务处理服务器
  - 处理异步操作的 BullMQ 任务（数据摄取、评估、导出）
- **`/packages/shared/`** - 共享数据库模式、类型和工具
  - 包含 Prisma 模式、ClickHouse 模式、共享服务和工具
  - 通过多个导出路径暴露，用于前端/后端分离

### 支持包
- **`/ee/`** - 企业版功能（单独授权）
- **`/packages/config-eslint/`** - 共享 ESLint 配置
- **`/packages/config-typescript/`** - 共享 TypeScript 配置

### 分层架构

```
┌─ tRPC API (Web) ────────────┐   ┌─ 公共 REST API (Web) ───────┐
│  HTTP 请求                  │   │  HTTP 请求                  │
│       ↓                     │   │       ↓                     │
│  tRPC 过程                  │   │  withMiddlewares +          │
│  (protectedProjectProcedure)│   │  createAuthedProjectAPIRoute│
│       ↓                     │   │       ↓                     │
│  服务层（业务逻辑）          │   │  服务层（业务逻辑）          │
│       ↓                     │   │       ↓                     │
│  Prisma / ClickHouse        │   │  Prisma / ClickHouse        │
└─────────────────────────────┘   └─────────────────────────────┘
             ↓
        [可选]：发布到 Redis BullMQ 队列
             ↓
┌─ Worker 包（Express）────────────────────────────────────────┐
│  BullMQ 队列任务                                             │
│       ↓                                                      │
│  队列处理器（处理任务）                                        │
│       ↓                                                      │
│  服务层（业务逻辑）                                            │
│       ↓                                                      │
│  Prisma / ClickHouse                                         │
└──────────────────────────────────────────────────────────────┘
```

## 开发命令

### 开发
```sh
pnpm i               # 安装依赖
pnpm run dev         # 启动所有服务（web + worker）
pnpm run dev:web     # 仅启动 Web 应用（localhost:3000）- **大多数情况下使用！**
pnpm run dev:worker  # 仅启动 Worker
pnpm run dx          # 完整的初始化设置：安装依赖、重置数据库、重置 node_modules、填充数据、启动开发。谨慎使用，因为它会清空数据库和 node_modules
```

### 数据库管理
数据库命令需要在 `packages/shared/` 文件夹中运行。
```sh
pnpm run db:generate       # 构建 Prisma 模型
pnpm run db:migrate        # 运行 Prisma 迁移
pnpm run db:reset          # 重置并重新填充数据库
pnpm run db:seed           # 填充示例数据
```

### 基础设施
```sh
pnpm run infra:dev:up      # 启动 Docker 服务（PostgreSQL、ClickHouse、Redis、MinIO）
pnpm run infra:dev:down    # 停止 Docker 服务
```

### 构建
```sh
pnpm --filter=PACKAGE_NAME run build  # 运行构建命令，会显示真正的 TypeScript 错误等
```

### Web 包测试
Web 包使用 **Jest** 进行单元测试。测试位于 `web/src/__tests__/`，分为三个项目：
- **sync-server**：同步测试
- **async-server**：集成测试（大多数 API/后端测试）
- **client**：客户端测试
- **e2e-server**：端到端测试（Playwright）

```sh
# 从 web 目录或使用过滤器从根目录运行：
pnpm test-sync --testPathPattern="$FILE_LOCATION_PATTERN" --testNamePattern="$TEST_NAME_PATTERN"

# 用于异步/集成测试（后端最常用）：
pnpm test -- --testPathPattern="$FILE_LOCATION_PATTERN" --testNamePattern="$TEST_NAME_PATTERN"

# 用于客户端测试：
pnpm test-client --testPathPattern="$FILE_LOCATION_PATTERN" --testNamePattern="$TEST_NAME_PATTERN"

# 用于 Playwright E2E 测试：
pnpm test:e2e

# 开发时的监视模式：
pnpm test:watch
```

### Worker 包测试
Worker 使用 **vitest** 进行单元测试。
```sh
# 从根目录：
pnpm run test --filter=worker -- $TEST_FILE_NAME -t "$TEST_NAME"

# 运行特定测试：
pnpm run test --filter=worker -- llmConnections.test.ts -t "specific test name"

# 排除 LLM 连接测试（更快）：
pnpm run test:exclude-llm-connections --filter=worker
```

### 工具
```bash
pnpm run format            # 格式化整个项目的代码
pnpm run nuke              # 删除所有 node_modules、构建文件、清空数据库、Docker 容器。**谨慎使用**
```

## 技术栈

### Web 应用（`/web/`）
- **框架**：Next.js 14（Pages Router）
- **API**：tRPC（类型安全的客户端-服务器通信）+ 公共访问的 REST API
- **身份验证**：NextAuth.js/Auth.js
- **数据库**：Prisma ORM with PostgreSQL
- **分析数据库**：ClickHouse（大容量追踪数据）
- **验证**：Zod 模式，我们使用 zod v4（始终从 `zod/v4` 导入）
- **样式**：Tailwind CSS，使用 CSS 变量支持主题
- **组件**：shadcn/ui（Radix UI 原语）
- **状态管理**：TanStack Query（React Query）+ tRPC
- **图表**：Tremor、Recharts

### Worker 应用（`/worker/`）
- **框架**：Express.js
- **队列系统**：BullMQ with Redis
- **用途**：异步处理（数据摄取、评估、导出、集成）

### 基础设施
- **主数据库**：PostgreSQL（通过 Prisma ORM）
- **分析数据库**：ClickHouse
- **缓存/队列**：Redis
- **对象存储**：MinIO/S3

## 开发指南

### 前端功能
- 所有新功能都放在 `/web/src/features/[feature-name]/`
- 使用 tRPC 进行全栈功能开发（入口点：`web/src/server/api/root.ts`）
- 遵循现有功能结构以保持一致性
- 使用来自 `@/src/components/ui` 的 shadcn/ui 组件
- 自定义可复用组件放在 `@/src/components`

### 公共 API 开发
- 所有公共 API 路由在 `/web/src/pages/api/public`
- 使用 `withMiddlewares.ts` 包装器
- 在 `/web/src/features/public-api/types` 中使用严格的 Zod v4 对象定义类型
- 添加端到端测试（参见 `datasets-api.servertest.ts`）
- 手动更新 `/fern/` 中的 Fern API 规范，然后通过 Fern CLI 重新生成 OpenAPI 规范

### 授权和 RBAC
- 查看 `/web/src/features/rbac/README.md` 了解授权模式
- 实施适当的权限检查（参见 `/web/src/features/entitlements/README.md`）

### 后端开发（tRPC、公共 API、服务）

提供了全面的后端开发指南，**backend-dev-guidelines** 技能涵盖：
- 创建 tRPC 路由器和过程
- 构建公共 REST API 端点
- 实现 BullMQ 队列处理器
- 服务层架构和模式
- 中间件模式（tRPC 和公共 API）
- 数据库访问模式（Prisma 和 ClickHouse）
- 测试策略（Web 使用 Jest，Worker 使用 vitest）
- 使用 OpenTelemetry 和 DataDog 进行可观测性

**核心原则：**
- **分层架构**：tRPC/API 路由 → 服务 → 数据库
- **服务委托**：保持过程/路由精简，委托给服务
- **始终按 projectId 过滤**，以实现查询中的租户隔离
- **使用 env.mjs/env.ts** 进行配置，永远不要直接访问 `process.env`
- **使用 Zod v4 验证**所有输入
- **OpenTelemetry + DataDog** 用于可观测性（后端不使用 Sentry）

**常见导入模式：**

```typescript
// 通用类型、模式、常量（前端 + 后端）
import { CloudConfigSchema, StringNoHTML, type APIScoreV2 } from "@langfuse/shared";

// 数据库 - Prisma 客户端（仅后端）
import { prisma, Prisma } from "@langfuse/shared/src/db";

// 服务器工具（仅后端）
import {
  logger,
  instrumentAsync,
  traceException,
  getTracesTable,
  StorageService,
  recordIncrement,
} from "@langfuse/shared/src/server";

// API 密钥管理（仅后端，特定路径以避免循环依赖）
import { createAndAddApiKeysToDb } from "@langfuse/shared/src/server/auth/apiKeys";

// 加密工具（仅后端）
import { encrypt, decrypt, sign, verify } from "@langfuse/shared/encryption";

// tRPC 设置
import { z } from "zod/v4";
import { createTRPCRouter, protectedProjectProcedure } from "@/src/server/api/trpc";
import { TRPCError } from "@trpc/server";
```

**示例功能结构：**
```
web/src/features/datasets/
├── server/
│   ├── datasetsRouter.ts      # tRPC 路由器
│   └── actions/               # 服务器操作/服务
├── components/                # React 组件
└── types/                     # 功能类型
```

### 数据库
- **双数据库系统**：PostgreSQL（主）+ ClickHouse（分析）
  - **PostgreSQL**：Prisma ORM 用于事务数据、元数据、用户数据
  - **ClickHouse**：直接客户端用于大容量分析（追踪、观察、评分）
- **迁移**：
  - PostgreSQL：`packages/shared/prisma/migrations/` 中的 Prisma 迁移
  - ClickHouse：`packages/shared/clickhouse/migrations/` 中的 SQL 迁移
- **租户隔离**：所有查询必须按 `projectId` 过滤
- **仓库模式**：对复杂查询使用仓库（`getTracesTable` 等）
- 为了允许无序摄取，模式中可能不强制执行外键关系

### 测试
- API 测试使用 Jest，E2E 测试使用 Playwright
- 对于后端/API 更改，推送前测试必须通过
- 为新的 API 端点和功能添加测试
- 编写测试时，专注于解耦每个 `it` 或 `test` 块，以确保它们可以独立和并发运行。测试不得依赖于先前或后续测试的操作或结果。
- 编写测试时，特别是在 __tests__/async 目录中，确保避免 `pruneDatabase` 调用。

### 代码约定
- **Pages Router**（不是 App Router）
- 在主分支上遵循约定式提交
- 使用 CSS 变量支持主题（支持自动深色/浅色模式）
- 全面使用 TypeScript
- 所有输入验证使用 Zod v4

## 环境设置

- **Node.js**：版本 24.6.0（在 `.nvmrc` 中指定）
- **包管理器**：pnpm v9.5.0（在 `package.json` 中指定）
- **数据库依赖**：用于本地 PostgreSQL、ClickHouse、Redis、MinIO 的 Docker
- **环境变量**：将 `.env.dev.example` 复制为 `.env`
- **Docker Compose 文件**：
  - `docker-compose.dev.yml` - 标准本地开发（默认）
  - `docker-compose.dev-azure.yml` - 使用 Azure 服务的开发
  - `docker-compose.dev-redis-cluster.yml` - 使用 Redis 集群的开发
  - `docker-compose.yml` - 生产配置

## 开发登录

使用种子数据在本地运行时：
- 用户名：`demo@langfuse.com`
- 密码：`password`
- 演示项目 URL：`http://localhost:3000/project/7a88fb47-b4e2-43b8-a06c-a5ce950dc53a`

## Linear MCP
要获取项目，请使用 `get_project` 功能，使用标题中的完整项目名称。
- 错误：message-placeholder-in-chat-messages-2beb6f02ec48
- 正确：Message placeholder in chat messages

## 前端提示

### Window Location 处理
- 每当你想使用或使用 window.location... 时，确保同时为自定义 basePath 添加适当的处理

## TypeScript 最佳实践
- 在 TypeScript 中，如果可能，不要使用 `any` 类型

## 通用编码指南
- 为了更容易进行代码审查，除非必要或被指示这样做，否则不要在文件中移动函数等

## 开发提示
- 在尝试构建包之前，先尝试运行一次 linter
- 在处理后端功能时使用 backend-dev-guidelines 技能以获得全面指导
- 在开发期间以监视模式运行测试，以尽早发现问题
- 在提交前使用 `pnpm run format` 确保代码风格一致

## 可观测性

### OpenTelemetry + DataDog（后端）
Langfuse 后端使用 **OpenTelemetry** 进行可观测性，追踪和日志发送到 **DataDog**。

```typescript
import {
  logger,          // 带追踪上下文的 Winston 日志记录器
  traceException,  // 将异常记录到 OpenTelemetry spans
  instrumentAsync, // 创建仪表化 spans
} from "@langfuse/shared/src/server";

// 结构化日志（包括 trace_id、span_id、dd.trace_id）
logger.info("Processing dataset", { datasetId, projectId });
logger.error("Failed to create dataset", { error: err.message });

// 记录异常
try {
  await operation();
} catch (error) {
  traceException(error); // 记录到当前 span
  throw error;
}

// 仪表化操作
const result = await instrumentAsync(
  { name: "dataset.create" },
  async (span) => {
    span.setAttributes({ datasetId, projectId });
    return await createDataset();
  },
);
```

**注意**：前端使用 **Sentry**，但后端（tRPC、API 路由、服务、worker）使用 **OpenTelemetry + DataDog**。

## 要避免的常见反模式

❌ 在路由/过程中放置业务逻辑（应委托给服务）
❌ 直接使用 `process.env`（始终使用 env.mjs/env.ts）
❌ 缺少错误处理或输入验证
❌ 在服务器端使用 `console.log` 而不是 `logger` 进行日志记录
❌ 在租户范围的查询中缺少 `projectId` 过滤器
❌ 使用 TypeScript 的 `any` 类型
❌ 不必要地在文件中移动函数（使代码审查更困难）
❌ 在可以编辑现有文件时创建新文件
❌ 在测试中使用 `pruneDatabase`（导致干扰）

## 参考功能示例

实现新功能时，参考这些现有实现：

- **Datasets**（`web/src/features/datasets/`）- 完整功能，包含 tRPC 路由器、公共 API 和服务
- **Prompts**（`web/src/features/prompts/`）- 版本控制和模板
- **Evaluations**（`web/src/features/evals/`）- 与 worker 集成的复杂功能
- **Public API**（`web/src/features/public-api/`）- 中间件和路由模式
- **Batch Exports**（`web/src/features/batch-exports/`）- 基于队列的异步处理
- **Automations**（`web/src/features/automations/`）- 事件驱动的工作流
