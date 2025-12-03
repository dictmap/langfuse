# CLAUDE.md

## Project Overview

Langfuse is an open-source LLM engineering platform that helps teams collaboratively develop, monitor, evaluate, and debug AI applications.
The main feature areas are tracing, evals and prompt management. Langfuse consists of the web application (this repo), documentation, python SDK and javascript/typescript SDK.
This repo contains the web application, worker, and supporting packages but notably not the JS nor Python client SDKs.

## Claude Code Skills

This repository has specialized Claude Code skills available to help with development:

- **backend-dev-guidelines** - Comprehensive backend development guide covering tRPC routers, public APIs, BullMQ queues, services, middleware, database patterns, testing, and more. Use when working on backend features.
- **skill-developer** - Meta-skill for creating and managing Claude Code skills.

To activate a skill, it will be automatically triggered based on your work context, or you can reference it in your requests.

## Repository Structure

High level structure. There are more folders (eg for hooks etc).

```
langfuse/
├── .claude/                 # Claude Code configuration
│   ├── skills/             # Claude Code skills (backend-dev-guidelines, etc.)
│   ├── hooks/              # Development hooks
│   └── agents/             # Custom agents
├── web/                     # Next.js 14 frontend/backend application
│   ├── src/
│   │   ├── __tests__/      # Jest tests (sync, async, e2e)
│   │   ├── components/     # Reusable UI components (shadcn/ui)
│   │   ├── features/       # Feature-specific code organized by domain
│   │   ├── pages/          # Next.js pages (Pages Router)
│   │   │   └── api/        # API routes (tRPC, public REST APIs)
│   │   ├── server/         # tRPC API routes and server logic
│   │   │   ├── api/        # tRPC routers
│   │   │   ├── auth.ts     # NextAuth.js configuration
│   │   │   └── db.ts       # Database client
│   │   ├── env.mjs         # Environment config with Zod validation
│   │   └── instrumentation.ts  # OpenTelemetry setup
│   └── public/             # Static assets
├── worker/                  # Express.js background job processor
│   └── src/
│       ├── __tests__/      # Vitest tests
│       ├── queues/         # BullMQ queue processors
│       ├── features/       # Business logic
│       ├── backgroundMigrations/  # Background migration tasks
│       ├── app.ts          # Express setup + queue registration
│       ├── env.ts          # Environment config
│       └── instrumentation.ts  # OpenTelemetry setup
├── packages/
│   ├── shared/             # Shared types, schemas, and utilities
│   │   ├── prisma/         # Database schema and migrations
│   │   ├── clickhouse/     # ClickHouse migrations and schema
│   │   └── src/            # Shared TypeScript code
│   │       ├── server/     # Server-side utilities (queues, auth, services)
│   │       ├── features/   # Feature-specific shared code
│   │       ├── db.ts       # Prisma client
│   │       └── env.ts      # Environment config
│   ├── config-eslint/      # ESLint configuration
│   └── config-typescript/  # TypeScript configuration
├── ee/                     # Enterprise Edition features
├── fern/                   # API documentation and OpenAPI specs
├── scripts/                # Development and deployment scripts
└── docker-compose.*.yml    # Docker configurations for dev/prod
```

## Repository Architecture

This is a **pnpm + Turbo monorepo** with the following key packages:

### Core Applications
- **`/web/`** - Next.js 14 application (Pages Router) providing both frontend UI and backend APIs
  - Frontend: React components, pages, and client-side logic
  - Backend: tRPC procedures and public REST APIs
- **`/worker/`** - Express.js background job processing server
  - Processes BullMQ jobs for async operations (ingestion, evaluations, exports)
- **`/packages/shared/`** - Shared database schema, types, and utilities
  - Contains Prisma schema, ClickHouse schema, shared services, and utilities
  - Exposed via multiple export paths for frontend/backend separation

### Supporting Packages
- **`/ee/`** - Enterprise Edition features (separate licensing)
- **`/packages/config-eslint/`** - Shared ESLint configuration
- **`/packages/config-typescript/`** - Shared TypeScript configuration

### Layered Architecture

```
┌─ tRPC API (Web) ────────────┐   ┌─ Public REST API (Web) ────┐
│  HTTP Request               │   │  HTTP Request              │
│       ↓                     │   │       ↓                    │
│  tRPC Procedure             │   │  withMiddlewares +         │
│  (protectedProjectProcedure)│   │  createAuthedProjectAPIRoute│
│       ↓                     │   │       ↓                    │
│  Service (business logic)   │   │  Service (business logic)  │
│       ↓                     │   │       ↓                    │
│  Prisma / ClickHouse        │   │  Prisma / ClickHouse       │
└─────────────────────────────┘   └────────────────────────────┘
             ↓
        [Optional]: Publish to Redis BullMQ queue
             ↓
┌─ Worker Package (Express) ────────────────────────────────┐
│  BullMQ Queue Job                                         │
│       ↓                                                   │
│  Queue Processor (handles job)                            │
│       ↓                                                   │
│  Service (business logic)                                 │
│       ↓                                                   │
│  Prisma / ClickHouse                                      │
└───────────────────────────────────────────────────────────┘
```

## Development Commands

### Development
```sh
pnpm i               # Install dependencies
pnpm run dev         # Start all services (web + worker)
pnpm run dev:web     # Web app only (localhost:3000) - **used in most cases!**
pnpm run dev:worker  # Worker only
pnpm run dx          # Full initial setup: install deps, reset DBs, resets node modules, seed data, start dev. USE SPARINGLY AS IT WIPES THE DATABASE & node_modules
```

### Database Management
database commands are to be run in the `packages/shared/` folder.
```sh
pnpm run db:generate       # Build prisma models
pnpm run db:migrate        # Run Prisma migrations
pnpm run db:reset          # Reset and reseed databases
pnpm run db:seed           # Seed with example data
```

### Infrastructure
```sh
pnpm run infra:dev:up      # Start Docker services (PostgreSQL, ClickHouse, Redis, MinIO)
pnpm run infra:dev:down    # Stop Docker services
```

### Building
```sh
pnpm --filter=PACKAGE_NAME run build  # Runs the build command, will show real typescript errors etc.
```

### Testing in Web Package
The web package uses **Jest** for unit tests. Tests are located in `web/src/__tests__/` and organized into three projects:
- **sync-server**: Synchronous tests
- **async-server**: Integration tests (most API/backend tests)
- **client**: Client-side tests
- **e2e-server**: End-to-end tests (Playwright)

```sh
# From the web directory or root with filter:
pnpm test-sync --testPathPattern="$FILE_LOCATION_PATTERN" --testNamePattern="$TEST_NAME_PATTERN"

# For async/integration tests (most common for backend):
pnpm test -- --testPathPattern="$FILE_LOCATION_PATTERN" --testNamePattern="$TEST_NAME_PATTERN"

# For client tests:
pnpm test-client --testPathPattern="$FILE_LOCATION_PATTERN" --testNamePattern="$TEST_NAME_PATTERN"

# For E2E tests with Playwright:
pnpm test:e2e

# Watch mode for development:
pnpm test:watch
```

### Testing in the Worker Package
The worker uses **vitest** for unit tests.
```sh
# From root:
pnpm run test --filter=worker -- $TEST_FILE_NAME -t "$TEST_NAME"

# Run specific test:
pnpm run test --filter=worker -- llmConnections.test.ts -t "specific test name"

# Exclude LLM connection tests (faster):
pnpm run test:exclude-llm-connections --filter=worker
```

### Utilities
```bash
pnpm run format            # Format code across entire project
pnpm run nuke              # Remove all node_modules, build files, wipe database, docker containers. **USE WITH CAUTION**
```

## Technology Stack

### Web Application (`/web/`)
- **Framework**: Next.js 14 (Pages Router)
- **APIs**: tRPC (type-safe client-server communication) + REST APIs for public access
- **Authentication**: NextAuth.js/Auth.js
- **Database**: Prisma ORM with PostgreSQL
- **Analytics Database**: ClickHouse (high-volume trace data)
- **Validation**: Zod schemas, we use zodv4 (always import from `zod/v4`)
- **Styling**: Tailwind CSS with CSS variables for theming
- **Components**: shadcn/ui (Radix UI primitives)
- **State Management**: TanStack Query (React Query) + tRPC
- **Charts**: Tremor, Recharts

### Worker Application (`/worker/`)
- **Framework**: Express.js
- **Queue System**: BullMQ with Redis
- **Purpose**: Async processing (data ingestion, evaluations, exports, integrations)

### Infrastructure
- **Primary Database**: PostgreSQL (via Prisma ORM)
- **Analytics Database**: ClickHouse
- **Cache/Queues**: Redis
- **Blob Storage**: MinIO/S3

## Development Guidelines

### Frontend Features
- All new features go in `/web/src/features/[feature-name]/`
- Use tRPC for full-stack features (entry point: `web/src/server/api/root.ts`)
- Follow existing feature structure for consistency
- Use shadcn/ui components from `@/src/components/ui`
- Custom reusable components go in `@/src/components`

### Public API Development
- All public API routes in `/web/src/pages/api/public`
- Use `withMiddlewares.ts` wrapper
- Define types in `/web/src/features/public-api/types` with strict Zod v4 objects
- Add end-to-end tests (see `datasets-api.servertest.ts`)
- Manually update Fern API specs in `/fern/`, then regenerate OpenAPI spec via Fern CLI

### Authorization & RBAC
- Check `/web/src/features/rbac/README.md` for authorization patterns
- Implement proper entitlements checking (see `/web/src/features/entitlements/README.md`)

### Backend Development (tRPC, Public APIs, Services)

For comprehensive backend development guidance, the **backend-dev-guidelines** skill is available and covers:
- Creating tRPC routers and procedures
- Building public REST API endpoints
- Implementing BullMQ queue processors
- Service layer architecture and patterns
- Middleware patterns (tRPC and public API)
- Database access patterns (Prisma and ClickHouse)
- Testing strategies (Jest for web, vitest for worker)
- Observability with OpenTelemetry and DataDog

**Key Principles:**
- **Layered architecture**: tRPC/API routes → Services → Database
- **Service delegation**: Keep procedures/routes thin, delegate to services
- **Always filter by projectId** for tenant isolation in queries
- **Use env.mjs/env.ts** for config, NEVER direct `process.env` access
- **Validate with Zod v4** for all inputs
- **OpenTelemetry + DataDog** for observability (not Sentry for backend)

**Common Import Patterns:**

```typescript
// General types, schemas, constants (frontend + backend)
import { CloudConfigSchema, StringNoHTML, type APIScoreV2 } from "@langfuse/shared";

// Database - Prisma client (backend only)
import { prisma, Prisma } from "@langfuse/shared/src/db";

// Server utilities (backend only)
import {
  logger,
  instrumentAsync,
  traceException,
  getTracesTable,
  StorageService,
  recordIncrement,
} from "@langfuse/shared/src/server";

// API key management (backend only, specific path to avoid circular deps)
import { createAndAddApiKeysToDb } from "@langfuse/shared/src/server/auth/apiKeys";

// Encryption utilities (backend only)
import { encrypt, decrypt, sign, verify } from "@langfuse/shared/encryption";

// tRPC setup
import { z } from "zod/v4";
import { createTRPCRouter, protectedProjectProcedure } from "@/src/server/api/trpc";
import { TRPCError } from "@trpc/server";
```

**Example Feature Structure:**
```
web/src/features/datasets/
├── server/
│   ├── datasetsRouter.ts      # tRPC router
│   └── actions/               # Server actions/services
├── components/                # React components
└── types/                     # Feature types
```

### Database
- **Dual database system**: PostgreSQL (primary) + ClickHouse (analytics)
  - **PostgreSQL**: Prisma ORM for transactional data, metadata, user data
  - **ClickHouse**: Direct client for high-volume analytics (traces, observations, scores)
- **Migrations**:
  - PostgreSQL: Prisma migrations in `packages/shared/prisma/migrations/`
  - ClickHouse: SQL migrations in `packages/shared/clickhouse/migrations/`
- **Tenant isolation**: ALWAYS filter by `projectId` in all queries
- **Repository pattern**: Use repositories (`getTracesTable`, etc.) for complex queries
- Foreign key relationships may not be enforced in schema to allow unordered ingestion

### Testing
- Jest for API tests, Playwright for E2E tests
- For backend/API changes, tests must pass before pushes
- Add tests for new API endpoints and features
- When writing tests, focus on decoupling each `it` or `test` block to ensure that they can run independently and concurrently. Tests must never depend on the action or outcome of previous or subsequent tests.
- When writing tests, especially in the __tests__/async directory, ensure that you avoid `pruneDatabase` calls.

### Code Conventions
- **Pages Router** (not App Router)
- Follow conventional commits on main branch
- Use CSS variables for theming (supports auto dark/light mode)
- TypeScript throughout
- Zod v4 for all input validation

## Environment Setup

- **Node.js**: Version 24.6.0 (specified in `.nvmrc`)
- **Package Manager**: pnpm v9.5.0 (specified in `package.json`)
- **Database Dependencies**: Docker for local PostgreSQL, ClickHouse, Redis, MinIO
- **Environment**: Copy `.env.dev.example` to `.env`
- **Docker Compose Files**:
  - `docker-compose.dev.yml` - Standard local development (default)
  - `docker-compose.dev-azure.yml` - Development with Azure services
  - `docker-compose.dev-redis-cluster.yml` - Development with Redis cluster
  - `docker-compose.yml` - Production configuration

## Login for Development

When running locally with seed data:
- Username: `demo@langfuse.com`
- Password: `password`
- Demo project URL: `http://localhost:3000/project/7a88fb47-b4e2-43b8-a06c-a5ce950dc53a`

## Linear MCP
To get a project, use the `get_project` capability with the full project name as it is in the title.
- bad: message-placeholder-in-chat-messages-2beb6f02ec48
- good: Message placeholder in chat messages

## Front-end Tips

### Window Location Handling
- Whenever you want to use or do use window.location..., ensure that you also add proper handling for a custom basePath

## TypeScript Best Practices
- In TypeScript, if possible, don't use the `any` type

## General Coding Guidelines
- For easier code reviews, prefer not to move functions etc around within a file unless necessary or instructed to do so

## Development Tips
- Before trying to build the package, try running the linter once first
- Use the backend-dev-guidelines skill when working on backend features for comprehensive guidance
- Run tests in watch mode during development to catch issues early
- Use `pnpm run format` before committing to ensure consistent code style

## Observability

### OpenTelemetry + DataDog (Backend)
Langfuse backend uses **OpenTelemetry** for observability, with traces and logs sent to **DataDog**.

```typescript
import {
  logger,          // Winston logger with trace context
  traceException,  // Record exceptions to OpenTelemetry spans
  instrumentAsync, // Create instrumented spans
} from "@langfuse/shared/src/server";

// Structured logging (includes trace_id, span_id, dd.trace_id)
logger.info("Processing dataset", { datasetId, projectId });
logger.error("Failed to create dataset", { error: err.message });

// Record exceptions
try {
  await operation();
} catch (error) {
  traceException(error); // Records to current span
  throw error;
}

// Instrument operations
const result = await instrumentAsync(
  { name: "dataset.create" },
  async (span) => {
    span.setAttributes({ datasetId, projectId });
    return await createDataset();
  },
);
```

**Note**: Frontend uses **Sentry**, but backend (tRPC, API routes, services, worker) uses **OpenTelemetry + DataDog**.

## Common Anti-Patterns to Avoid

❌ Business logic in routes/procedures (delegate to services)
❌ Direct `process.env` usage (always use env.mjs/env.ts)
❌ Missing error handling or input validation
❌ Using `console.log` instead of `logger` for server-side logging
❌ Missing `projectId` filter on tenant-scoped queries
❌ Using `any` type in TypeScript
❌ Moving functions around unnecessarily (makes code review harder)
❌ Creating new files when editing existing ones would suffice
❌ Using `pruneDatabase` in tests (causes interference)

## Example Features to Reference

When implementing new features, reference these existing implementations:

- **Datasets** (`web/src/features/datasets/`) - Complete feature with tRPC router, public API, and service
- **Prompts** (`web/src/features/prompts/`) - Versioning and templates
- **Evaluations** (`web/src/features/evals/`) - Complex feature with worker integration
- **Public API** (`web/src/features/public-api/`) - Middleware and route patterns
- **Batch Exports** (`web/src/features/batch-exports/`) - Queue-based async processing
- **Automations** (`web/src/features/automations/`) - Event-driven workflows
