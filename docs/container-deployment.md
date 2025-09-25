# Container Deployment in vibesdk

## Overview

The vibesdk uses **Cloudflare Workers Containers** to provide secure, isolated environments for executing user-generated code. This is different from traditional Docker deployment - containers run within Cloudflare's edge infrastructure as part of the Workers platform.

`★ Insight ─────────────────────────────────────`
• Containers in vibesdk are NOT traditional Docker containers on servers
• They're Cloudflare Workers Containers - serverless, edge-deployed compute environments
• Each container is a Durable Object that can execute user code safely in isolation
`─────────────────────────────────────────────────`

## Architecture

### Container Types

The system uses **two different Docker images** for different purposes:

#### 1. **SandboxDockerfile** (Primary Sandbox Environment)
```dockerfile
FROM docker.io/cloudflare/sandbox:0.1.3
```

**Purpose**: Provides secure execution environment for user-generated applications

**Key Features**:
- **Git integration**: Pre-configured git with Cloudflare build agent credentials
- **Cloudflared tunnel**: Enables secure connections to the internet
- **Process monitoring**: Comprehensive error tracking and process management
- **Bun runtime**: Fast JavaScript/TypeScript execution environment

**Installed Components**:
- System dependencies: `git`, `ca-certificates`, `curl`, `procps`, `net-tools`
- Cloudflared binary for secure tunneling
- Process monitoring system (`/app/container/`)
- Bun package manager and runtime

#### 2. **DeployerDockerfile** (Deployment Environment)
```dockerfile
FROM docker.io/cloudflare/sandbox:0.1.3
```

**Purpose**: Handles deployment operations and Wrangler CLI tasks

**Key Features**:
- **Wrangler CLI**: Latest version for Cloudflare deployments
- **Minimal footprint**: Only includes deployment essentials
- **Same base image**: Consistent environment with sandbox

## Durable Objects Integration

Containers are managed by **Durable Object classes** that provide stateful, persistent execution:

### UserAppSandboxService
```typescript
// From wrangler.jsonc
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
```

**Resource Allocation**:
- **CPU**: 4 vCPUs per instance
- **Memory**: 4GB RAM per instance  
- **Storage**: ~10GB disk per instance
- **Max Instances**: 2900 (configurable via `MAX_SANDBOX_INSTANCES`)

**Instance Distribution**:
- Uses **consistent hashing** to distribute sessions across containers
- **Hash-based allocation**: `hash(sessionId) % max_instances`
- **Persistent mapping**: Same session always goes to same container

### Session-to-Container Mapping
```typescript
function getAutoAllocatedSandbox(sessionId: string): string {
  let hash = 0;
  for (let i = 0; i < sessionId.length; i++) {
    const char = sessionId.charCodeAt(i);
    hash = ((hash << 5) - hash) + char;
    hash = hash & hash;
  }
  hash = Math.abs(hash);
  
  const max_instances = env.MAX_SANDBOX_INSTANCES || 10;
  const containerIndex = hash % max_instances;
  return `container-pool-${containerIndex}`;
}
```

## Process Monitoring System

Each container includes a sophisticated **monitoring system** located in `/container/`:

### Components

#### 1. **CLI Tools** (`cli-tools.ts`)
- Command-line interface for container management
- Process lifecycle control (start, stop, monitor)
- Error reporting and log collection
- Symlinked as `/usr/local/bin/monitor-cli` for easy access

#### 2. **Process Monitor** (`process-monitor.ts`)
```typescript
// Enhanced monitoring with error patterns and framework support
class ProcessMonitor extends EventEmitter {
  // Real-time process monitoring
  // Error pattern detection
  // Framework-specific log parsing
  // Automatic crash recovery
}
```

**Features**:
- **Real-time monitoring**: Live process state tracking
- **Error detection**: Pattern-based error classification
- **Log management**: Rolling logs with size limits
- **Framework support**: Vite, React, Next.js optimized parsing

#### 3. **Storage System** (`storage.ts`)
```typescript
// File-based storage for process data and error tracking
class StorageManager {
  // Error persistence
  // Process state management
  // Log rotation and cleanup
}
```

## Deployment Process

### 1. **Container Build & Registry**
```bash
# Containers are built and pushed to Cloudflare's registry
# Referenced in wrangler.jsonc as "./SandboxDockerfile"
```

### 2. **Wrangler Configuration**
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
      },
      "rollout_step_percentage": 100
    }
  ]
}
```

### 3. **Environment Variables**
```jsonc
{
  "vars": {
    "MAX_SANDBOX_INSTANCES": "10",
    "SANDBOX_INSTANCE_TYPE": "standard"
  }
}
```

### 4. **Deployment Commands**
```bash
# Deploy entire stack including containers
npm run deploy

# Local development with containers
npm run local

# Remote development
npm run dev:remote
```

## Container Lifecycle

### 1. **Instance Creation**
- **Triggered by**: First request to `CodeGeneratorAgent` with new sessionId
- **Allocation**: Consistent hash-based container selection
- **Initialization**: Process monitor starts, environment prepared

### 2. **Code Execution**
```typescript
// Execute user code in isolated container
const response = await sandboxClient.executeCommands([
  'npm install',
  'npm run build',
  'npm run dev'
]);
```

### 3. **State Persistence**
- **Durable Object state**: Maintains container metadata and session info
- **File system**: Persists between requests within same session
- **Process monitoring**: Continuous error tracking and log collection

### 4. **Cleanup & Shutdown**
- **Automatic**: After inactivity timeout
- **Manual**: Via API shutdown commands
- **Resource recovery**: Memory and disk space reclaimed

## Security Model

### Container Isolation
- **Cloudflare's sandbox**: Built on strong isolation primitives
- **Network restrictions**: Limited outbound access through Cloudflared tunnels
- **Resource limits**: CPU, memory, and disk quotas enforced
- **Process isolation**: Each session runs in separate container instance

### Code Execution Safety
- **No privileged operations**: Containers run without root access
- **Limited file system access**: Restricted to application directories
- **Network tunneling**: All external connections through Cloudflare tunnels
- **Monitoring**: All process activity logged and monitored

## Monitoring & Observability

### Process Health
```bash
# Check container status
bun run cli-tools.ts process status

# View error logs  
bun run cli-tools.ts errors

# Monitor process logs
bun run cli-tools.ts logs --follow
```

### Error Detection
- **Pattern-based**: Recognizes common framework error patterns
- **Real-time alerts**: Immediate notification of critical errors
- **Categorization**: Error severity and type classification
- **Recovery**: Automatic restart on certain error conditions

### Performance Metrics
- **Resource usage**: CPU, memory, disk utilization
- **Process uptime**: Container instance lifetime tracking
- **Request latency**: Code execution response times
- **Error rates**: Failure frequency and patterns

## Configuration

### Instance Types
```typescript
// Configurable through environment variables
const instanceConfig = {
  standard: { vcpu: 4, memory_mib: 4096, disk_mb: 10240 },
  large: { vcpu: 8, memory_mib: 8192, disk_mb: 20480 }
};
```

### Scaling Parameters
- **MAX_SANDBOX_INSTANCES**: Maximum number of container instances
- **SANDBOX_INSTANCE_TYPE**: Resource allocation profile
- **Rollout percentage**: Gradual deployment control

## Integration Points

### With Durable Objects
- **StateManager**: Persists container state and session data
- **WebSocket**: Real-time communication with frontend
- **Event streaming**: Live updates on process status

### With Workers Platform
- **Service bindings**: Integration with other Cloudflare services
- **Environment variables**: Configuration through Workers environment
- **Observability**: Metrics collection and alerting

### With Frontend
- **Real-time updates**: Process status and error reporting
- **File synchronization**: Code changes reflected in container
- **Preview URLs**: Live application access through Cloudflare tunnels

## Key Differences from Traditional Docker

| Traditional Docker | Cloudflare Containers |
|-------------------|----------------------|
| Server-based | Edge-deployed |
| Manual scaling | Auto-scaling |
| Network configuration | Cloudflare tunnels |
| External registry | Integrated platform |
| Separate orchestration | Built-in management |
| Infrastructure management | Fully managed |

## Related Documentation

- [Python Deployment](./python-deployment.md) - Why Python isn't deployed
- [Deployment Architecture](./deployment-architecture.md) - Overall system design
- [Architecture Diagrams](./architecture-diagrams.md) - Visual overview