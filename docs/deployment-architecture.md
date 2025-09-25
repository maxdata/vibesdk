# vibesdk Deployment Architecture

## Overview

The vibesdk uses a **serverless, edge-first architecture** entirely built on Cloudflare's platform. This document provides a comprehensive overview of how the system is deployed and operates across Cloudflare's global network.

`★ Insight ─────────────────────────────────────`
• Zero traditional servers - everything runs on Cloudflare's edge infrastructure
• Durable Objects provide stateful compute for AI code generation
• Containers run within Workers for secure user code execution
`─────────────────────────────────────────────────`

## High-Level Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    Cloudflare Edge                         │
├─────────────────────────────────────────────────────────────┤
│  Frontend (Static Assets)    │   Workers (TypeScript)       │
│  • React + Vite              │   • API Controllers           │
│  • Compiled to /dist         │   • Authentication            │
│  • Served via Workers Assets │   • WebSocket handlers        │
├─────────────────────────────────────────────────────────────┤
│                    Durable Objects                         │
│  • CodeGeneratorAgent        │   • UserAppSandboxService     │
│  • Stateful AI operations    │   • Container management      │
│  • WebSocket persistence     │   • Code execution            │
├─────────────────────────────────────────────────────────────┤
│                    Storage Layer                           │
│  • D1 Database               │   • R2 Object Storage         │
│  • KV Store                  │   • AI Gateway Analytics      │
└─────────────────────────────────────────────────────────────┘
```

## Core Components

### 1. Frontend Deployment

**Technology**: React + Vite → Static Assets
**Location**: Served via Cloudflare Workers Assets

```bash
# Build process
npm run build  # Compiles React to /dist directory

# Deployment
# Static files served through Workers Assets binding
# Configured in wrangler.jsonc:
{
  "assets": {
    "directory": "dist",
    "not_found_handling": "single-page-application",
    "run_worker_first": true
  }
}
```

**Benefits**:
- **Global CDN**: Assets served from 200+ edge locations
- **Zero cold starts**: Static assets always available
- **SPA routing**: Client-side routing with fallback handling

### 2. Backend (Workers) Deployment

**Technology**: TypeScript → Cloudflare Workers
**Entry Point**: `worker/index.ts`

```typescript
// Main worker entry point
export default {
  async fetch(request, env, ctx) {
    // Route handling for API endpoints
    // WebSocket upgrade for real-time communication
    // Asset serving for frontend
  }
}
```

**API Structure**:
```
/api/agent/:id/ws     # WebSocket for real-time AI generation
/api/apps            # Application management
/api/auth            # OAuth authentication
/api/analytics       # Usage analytics
/api/screenshots     # Preview generation
/api/github-export   # GitHub integration
```

**Deployment Commands**:
```bash
npm run deploy       # Full production deployment
npm run local        # Local development
npm run dev:remote   # Development with remote bindings
```

### 3. Durable Objects

#### CodeGeneratorAgent
**Purpose**: Stateful AI-powered code generation
**Location**: `worker/agents/core/simpleGeneratorAgent.ts`

```typescript
export class CodeGeneratorAgent {
  constructor(state: DurableObjectState, env: Env) {
    // Persistent WebSocket connections
    // AI conversation state
    // Code generation pipeline
  }
  
  async webSocketMessage(ws: WebSocket, message: any) {
    // Real-time code generation updates
    // File streaming via SCOF protocol
    // Error handling and recovery
  }
}
```

**Key Features**:
- **Persistent state**: Maintains conversation and generation context
- **WebSocket handling**: Real-time bidirectional communication
- **AI orchestration**: Manages multiple AI model interactions
- **SCOF streaming**: Structured code output format

#### UserAppSandboxService
**Purpose**: Secure container-based code execution
**Location**: Integrated with `@cloudflare/sandbox`

```typescript
export { Sandbox as UserAppSandboxService } from "@cloudflare/sandbox";
```

**Capabilities**:
- **Isolated execution**: Secure sandboxed environments
- **Container management**: Docker-based user app execution
- **Process monitoring**: Real-time error detection and logging
- **Resource management**: CPU, memory, and disk quotas

### 4. Storage Architecture

#### D1 Database (SQLite-based)
**Purpose**: Relational data storage
**Schema**: `worker/database/schema.ts`

```typescript
// Key tables
const users = sqliteTable('users', { ... });
const apps = sqliteTable('apps', { ... });
const sessions = sqliteTable('sessions', { ... });
const analytics = sqliteTable('analytics', { ... });
```

**Migration System**:
```bash
npm run db:generate      # Create migrations
npm run db:migrate:local # Apply locally
npm run db:migrate:remote # Apply to production
```

#### R2 Object Storage
**Purpose**: Large file and template storage

```jsonc
{
  "r2_buckets": [
    {
      "binding": "TEMPLATES_BUCKET",
      "bucket_name": "vibesdk-templates"
    }
  ]
}
```

**Use Cases**:
- Template repository storage
- Generated application assets
- User file uploads
- Deployment artifacts

#### KV Store
**Purpose**: Fast key-value storage

```jsonc
{
  "kv_namespaces": [
    {
      "binding": "VibecoderStore",
      "id": "f066f3c2e4824981b48e8586c04db9c1"
    }
  ]
}
```

**Use Cases**:
- Session management
- Cache layer
- Configuration storage
- Rate limiting data

### 5. Container Deployment

**Architecture**: Cloudflare Workers Containers
**Base Images**: 
- `SandboxDockerfile` - User code execution environment
- `DeployerDockerfile` - Deployment operations

**Configuration**:
```jsonc
{
  "containers": [
    {
      "class_name": "UserAppSandboxService",
      "image": "./SandboxDockerfile",
      "max_instances": 2900,
      "instance_type": {
        "vcpu": 4,
        "memory_mib": 4096,
        "disk_mb": 10240
      }
    }
  ]
}
```

**Features**:
- **Horizontal scaling**: Up to 2900 container instances
- **Resource isolation**: 4 vCPU, 4GB RAM per instance
- **Process monitoring**: Built-in error detection and logging
- **Secure execution**: Sandboxed user code execution

## Deployment Environments

### Production
**Domain**: `build.cloudflare.dev`
**Configuration**: `wrangler.jsonc`

```jsonc
{
  "name": "vibesdk-production",
  "routes": [
    {
      "pattern": "build.cloudflare.dev",
      "custom_domain": true
    }
  ],
  "workers_dev": false
}
```

### Development
**Local**: `localhost:5173`
**Remote**: Development with production bindings

```bash
# Local development (with local DB)
npm run local

# Remote development (with production bindings)
npm run dev:remote
```

## Security Architecture

### Authentication
**Strategy**: OAuth + JWT sessions
**Providers**: Google, GitHub (configurable)

```typescript
// JWT-based session management
const session = await verifyJWT(token, env.JWT_SECRET);
```

### Rate Limiting
**Implementation**: Durable Objects + KV-based limiting

```jsonc
{
  "unsafe": {
    "bindings": [
      {
        "name": "API_RATE_LIMITER",
        "type": "ratelimit",
        "simple": { "limit": 1000, "period": 60 }
      }
    ]
  }
}
```

### Container Security
- **Sandboxed execution**: Strong process isolation
- **Resource limits**: CPU, memory, disk quotas
- **Network restrictions**: Controlled outbound access
- **File system isolation**: Restricted directory access

## AI Integration

### Multi-Provider Support
**Configured Providers**:
- Anthropic (Claude)
- OpenAI (GPT models)
- Google AI Studio
- Cerebras Systems

**AI Gateway Integration**:
```jsonc
{
  "vars": {
    "CLOUDFLARE_AI_GATEWAY": "vibesdk-gateway"
  }
}
```

**Benefits**:
- **Request caching**: Reduced AI API costs
- **Analytics**: Detailed usage tracking
- **Rate limiting**: Provider-specific limits
- **Fallback handling**: Multi-provider resilience

## Monitoring & Observability

### Cloudflare Analytics
```jsonc
{
  "observability": {
    "enabled": true,
    "head_sampling_rate": 1
  }
}
```

### Error Tracking
**Integration**: Sentry for error monitoring
**Endpoint**: `/api/sentry` - Proxied error reporting

### Performance Monitoring
- **Workers Analytics**: Request latency and error rates
- **Durable Objects Metrics**: State size and operation counts
- **AI Gateway Analytics**: Token usage and model performance
- **Container Monitoring**: Resource utilization and error patterns

## Deployment Process

### 1. Build Phase
```bash
# TypeScript compilation + Frontend build
npm run build
```

### 2. Database Migrations
```bash
# Generate and apply schema changes
npm run db:generate
npm run db:migrate:remote
```

### 3. Worker Deployment
```bash
# Deploy with environment variables and secrets
bun --env-file .prod.vars scripts/deploy.ts
```

### 4. Post-Deployment
- Template repository sync to R2
- Container registry updates  
- Environment variable updates
- Health checks and validation

## Environment Configuration

### Required Variables
```bash
# Core services
CLOUDFLARE_API_TOKEN=xxx
CLOUDFLARE_ACCOUNT_ID=xxx
JWT_SECRET=xxx

# AI Providers
ANTHROPIC_API_KEY=xxx
OPENAI_API_KEY=xxx
GOOGLE_AI_STUDIO_API_KEY=xxx

# OAuth
GOOGLE_CLIENT_ID=xxx
GOOGLE_CLIENT_SECRET=xxx
GITHUB_CLIENT_ID=xxx
GITHUB_CLIENT_SECRET=xxx
```

### Wrangler Configuration
```jsonc
{
  "vars": {
    "TEMPLATES_REPOSITORY": "https://github.com/cloudflare/vibesdk-templates",
    "MAX_SANDBOX_INSTANCES": "10",
    "SANDBOX_INSTANCE_TYPE": "standard",
    "CLOUDFLARE_AI_GATEWAY": "vibesdk-gateway"
  }
}
```

## Scaling Characteristics

### Automatic Scaling
- **Workers**: Scales to zero, handles massive concurrency
- **Durable Objects**: Per-object isolation, horizontal scaling
- **Containers**: Up to 2900 instances, resource-based scaling
- **Storage**: Unlimited capacity (D1, R2, KV)

### Performance Characteristics
- **Global latency**: <50ms response times worldwide
- **Cold start**: <100ms for Workers
- **Container startup**: 2-5 seconds for first request
- **Database**: SQLite performance with global replication

## Disaster Recovery

### Data Persistence
- **D1**: Automatic backups and point-in-time recovery
- **R2**: 99.999999999% durability with versioning
- **Durable Objects**: Automatic state persistence and migration

### Failover
- **Multi-region**: Automatic failover across Cloudflare's network
- **Container recovery**: Automatic restart on failure
- **Database**: Read replicas and automatic failover

## Cost Model

### Pricing Factors
- **Workers**: Request-based pricing
- **Durable Objects**: GB-seconds + requests
- **Containers**: vCPU-seconds + memory usage
- **Storage**: GB-month for D1/R2/KV
- **AI Gateway**: Request volume (caching reduces costs)

## Related Documentation

- [Python Deployment](./python-deployment.md) - Debug tools (not deployed)
- [Container Deployment](./container-deployment.md) - Container-specific details
- [Architecture Diagrams](./architecture-diagrams.md) - Visual representations
- [Project Report](./projectreport.md) - Implementation details