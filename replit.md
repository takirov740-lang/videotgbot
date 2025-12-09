# Overview

This is a Mastra-based agent automation system built for Replit. The project enables users to create AI-powered workflows and agents that respond to time-based (cron) triggers and webhook events (Telegram, Slack, etc.). It uses the Mastra framework for agent orchestration, Inngest for durable workflow execution, and supports multiple storage backends (PostgreSQL, LibSQL) for persistence and memory.

The primary use case is building conversational AI bots and automated workflows that can interact with external services, maintain conversation history, and perform complex multi-step operations with built-in reliability and observability.

# User Preferences

Preferred communication style: Simple, everyday language.

# System Architecture

## Core Framework Architecture

**Mastra Framework**: The application is built on Mastra v0.20.0, a TypeScript framework for building AI applications. The main Mastra instance is initialized in `src/mastra/index.ts` and serves as the central orchestrator for agents, workflows, tools, and integrations.

**Why Mastra?** Provides a unified interface for agent management, workflow orchestration, memory systems, and tool integration with built-in support for streaming, suspend/resume, and multi-step operations.

## Agent System

**Agent Architecture**: Agents are defined using the `Agent` class from `@mastra/core/agent` and configured with:
- System instructions (personality/behavior)
- LLM model (OpenAI, Anthropic, OpenRouter support via AI SDK)
- Tools (functions the agent can execute)
- Memory (conversation history and working memory)
- Guardrails (input/output processors for content moderation)

**Example Agent**: `videoRouletteAgent` is registered in the system and demonstrates the pattern for creating conversational AI agents.

**Why this approach?** Separates agent configuration from business logic, enables reusability, and provides consistent interfaces for memory, tools, and model selection.

## Workflow Orchestration

**Workflow Engine**: Workflows are created using `createWorkflow` and `createStep` from `@mastra/core/workflows`. Each step defines:
- Input/output schemas (using Zod for type safety)
- Execute function (business logic)
- Optional retry configuration
- Optional suspend/resume points

**Example Workflow**: `videoRouletteWorkflow` demonstrates multi-step orchestration with type-safe data flow between steps.

**Control Flow**: Supports sequential execution (`.then()`), parallel execution (`.parallel()`), conditional branching (`.if()`), and suspend/resume for human-in-the-loop interactions.

**Why workflows over agents?** When task execution requires explicit control, deterministic step ordering, or clear data transformations, workflows provide more reliability than agent reasoning alone.

## Durability Layer (Inngest Integration)

**Inngest for Persistence**: Custom integration in `src/mastra/inngest/` provides:
- Step-by-step execution with automatic memoization
- Retry logic for failed steps
- Workflow state persistence
- Resume from failure points without re-executing completed steps

**Integration Points**:
- `inngest` client exported from `src/mastra/inngest/client`
- `inngestServe` function for registering workflows
- Custom workflow registration via `registerCronWorkflow` and `registerApiRoute`

**Why Inngest?** Provides production-grade durability without requiring infrastructure management. Workflows can be paused for hours/days and resumed exactly where they left off.

## Trigger System

**Time-Based Triggers**: Cron expressions trigger workflows on schedule via `registerCronTrigger` in `src/triggers/cronTriggers.ts`. These are registered before Mastra initialization and don't create HTTP endpoints.

**Webhook Triggers**: External services (Telegram, Slack, Linear, etc.) trigger workflows via HTTP webhooks registered through `registerApiRoute`. Each connector has its own trigger file in `src/triggers/`.

**Trigger Pattern**:
1. Webhook receives payload at `/webhooks/{provider}/{action}`
2. Inngest event routing forwards to appropriate handler
3. Handler validates payload and starts workflow with `workflow.createRunAsync()`
4. Workflow executes with full Inngest orchestration

**Why this pattern?** Decouples event reception from workflow execution, enables testing without external services, and provides consistent error handling across all trigger types.

## Memory and Context Management

**Memory Types**:
- **Conversation History**: Last N messages from current thread (default: 10)
- **Semantic Recall**: RAG-based vector search for relevant past messages
- **Working Memory**: Persistent structured data (preferences, facts) stored as Markdown or Zod schemas

**Memory Scoping**:
- **Thread-scoped**: Isolated per conversation (default)
- **Resource-scoped**: Shared across all threads for same user

**Storage Requirement**: Memory requires a storage adapter (LibSQL, PostgreSQL, Upstash) configured on the main Mastra instance.

**Why this architecture?** Enables agents to maintain long-term context without hitting token limits, while keeping relevant information accessible through semantic search.

## Logger Configuration

**Custom Logger**: `ProductionPinoLogger` extends `MastraLogger` to provide structured JSON logging with Pino. Configured in `src/mastra/index.ts` with log levels and formatting.

**Why Pino?** High-performance logging with minimal overhead, suitable for production environments with structured log aggregation.

## Storage Strategy

**Shared Storage**: Single storage instance (`sharedPostgresStorage` from `src/mastra/storage`) used for:
- Workflow state persistence
- Agent memory (threads, messages, working memory)
- Snapshot storage for suspend/resume

**Storage Adapters**: Supports multiple backends:
- `@mastra/pg` - PostgreSQL with pgvector
- `@mastra/libsql` - LibSQL/Turso
- `@mastra/upstash` - Upstash Redis + Vector

**Why shared storage?** Simplifies configuration and ensures consistent data access across all agents and workflows.

## Development vs Production

**Development Mode**: Uses `mastra dev` command to start local server with:
- Mastra Playground UI for testing agents/workflows
- Hot reload for code changes
- Local Inngest dev server for workflow debugging

**Production Mode**: Uses `mastra build` to compile for deployment with:
- OpenTelemetry instrumentation for observability
- Production-optimized logging
- Inngest Cloud for durable execution

**Build System**: TypeScript compilation with ES2022 target, ES module format, and bundler resolution. Output goes to `.mastra/output/`.

# External Dependencies

## AI/LLM Providers

- **OpenAI** (`@ai-sdk/openai`): Primary LLM provider for GPT-4o models
- **OpenRouter** (`@openrouter/ai-sdk-provider`): Alternative model provider for diverse model access
- **Vercel AI SDK** (`ai` v4.3.16): Core AI SDK for model interactions, streaming, and tool calling

**Required Environment Variables**: `OPENAI_API_KEY`, optionally `OPENROUTER_API_KEY`

## Workflow & Orchestration

- **Inngest** (`inngest` v3.40.2, `@inngest/realtime`): Durable workflow execution with step memoization and retry logic
- **Mastra Inngest Integration** (`@mastra/inngest`): Custom bridge between Mastra workflows and Inngest execution engine

**Required Environment Variables**: Inngest credentials for production deployment (auto-configured in dev mode)

## Storage & Persistence

- **PostgreSQL** (`pg` v8.16.3, `@mastra/pg`): Relational database with pgvector extension for memory storage
- **LibSQL** (`@mastra/libsql`): SQLite-compatible database for local/edge deployments
- **Mastra Memory** (`@mastra/memory`): Memory abstraction layer supporting multiple storage backends

**Required Environment Variables**: `DATABASE_URL` for PostgreSQL or local file paths for LibSQL

## External Service Integrations

- **Telegram** (via Telegram Bot API): Webhook-based bot integration for receiving/sending messages
- **Slack** (`@slack/web-api`): Web API client for Slack bot interactions
- **Exa Search** (`exa-js`): Web search API integration for information retrieval

**Required Environment Variables**: 
- `TELEGRAM_BOT_TOKEN` for Telegram integration
- `SLACK_BOT_TOKEN` for Slack integration
- `EXA_API_KEY` for search capabilities

## Logging & Observability

- **Pino** (`pino` v9.9.4): High-performance JSON logger
- **Mastra Loggers** (`@mastra/loggers`): Logger abstraction for Mastra framework
- **OpenTelemetry**: Auto-instrumentation for production observability (configured in build output)

## Development Tools

- **TypeScript** (v5.9.3): Type checking and compilation
- **tsx** (v4.20.3): TypeScript execution for development
- **Prettier** (v3.6.2): Code formatting
- **Mastra CLI** (`mastra` v0.14.0): Development server and build tooling

## Schema Validation

- **Zod** (v3.25.76): Runtime type validation for workflow schemas, tool inputs/outputs, and memory structures