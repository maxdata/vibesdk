# Python in vibesdk: Debug Tools Only

## Overview

**Important**: Python is **NOT deployed** as part of the vibesdk production system. Python exists solely as local development and debugging tools to help analyze the Durable Objects state and AI inference patterns.

## Architecture Context

The vibesdk uses a **serverless, edge-first architecture** powered entirely by Cloudflare's platform:
- **Frontend**: React + Vite (compiled to static assets)
- **Backend**: Cloudflare Workers (JavaScript/TypeScript runtime)
- **Database**: Cloudflare D1 (SQLite)
- **Storage**: Cloudflare R2 + KV
- **Compute**: Durable Objects + Containers

**No traditional servers, no Python runtime in production.**

## Python Debug Tools Location

All Python tools are located in `/debug-tools/`:

```
debug-tools/
├── ai_request_analyzer_v2.py     # AI inference request analysis
├── conversation_analyzer.py      # Conversation message size analysis  
├── migration_tester.py          # State migration testing
└── state_analyzer.py            # Durable Object state debugging
```

## Tool Descriptions

### 1. AI Request Analyzer (`ai_request_analyzer_v2.py`)

**Purpose**: Analyzes AI inference requests from PhaseImplementation operations to optimize token usage and reduce costs.

**Key Features**:
- Parses SCOF (Structured Code Output Format) from AI responses
- Tracks dependencies, blueprint schemas, and template variables
- Detects cross-message duplication and calculates compression potential
- Provides recommendations for prompt optimization

**Usage**:
```bash
python debug-tools/ai_request_analyzer_v2.py path/to/sample-request.json --detailed
```

**When to use**: When analyzing high token usage or investigating AI prompt efficiency issues.

### 2. Conversation Analyzer (`conversation_analyzer.py`)

**Purpose**: Analyzes the `conversationMessages` property in Durable Object state to understand why state storage is large.

**Key Features**:
- Breaks down message sizes by type and content
- Identifies the largest messages causing state bloat
- Provides specific cleanup recommendations
- Helps debug Durable Objects storage limits

**Usage**:
```bash
python debug-tools/conversation_analyzer.py
```

**When to use**: When Durable Objects hit state size limits or storage costs are unexpectedly high.

### 3. Migration Tester (`migration_tester.py`)

**Purpose**: Tests the TypeScript migration algorithm in Python to validate state cleanup logic without running the full Worker.

**Key Features**:
- Simulates the exact deduplication logic from TypeScript
- Validates conversation message cleanup (50 message limit)
- Tests migration edge cases and data integrity
- Provides before/after analytics

**Usage**:
```bash
python debug-tools/migration_tester.py
```

**When to use**: Before deploying state migration changes to validate the cleanup algorithm works correctly.

### 4. State Analyzer (`state_analyzer.py`)

**Purpose**: Parses error messages from Durable Object `setState` failures to identify what's causing state growth.

**Key Features**:
- Analyzes size of each state property when serialized
- Shows differences between old and new states
- Identifies main contributors to state growth
- Critical for debugging SQL storage issues in Durable Objects

**Usage**:
```bash
python debug-tools/state_analyzer.py path/to/error-log.txt
```

**When to use**: When setState operations fail due to size limits or when investigating state growth patterns.

## Dependencies

These Python scripts use **only standard library modules**:
- `json`, `sys`, `re`, `pathlib`
- `dataclasses`, `typing`, `collections`
- `argparse`, `math`, `enum`, `abc`

**No external dependencies** - they run with any Python 3.7+ installation.

## Development Workflow

### When State Issues Occur:

1. **Check Durable Object logs** for setState failures
2. **Run State Analyzer** on error logs to identify size contributors  
3. **Use Conversation Analyzer** if conversationMessages are the issue
4. **Test migration logic** with Migration Tester before deploying fixes
5. **Optimize AI prompts** using AI Request Analyzer to reduce future state growth

### Typical Investigation Process:

```bash
# 1. Analyze current state size issues
python debug-tools/state_analyzer.py logs/durable-object-error.log

# 2. If conversation messages are large, get detailed breakdown  
python debug-tools/conversation_analyzer.py

# 3. Test any state cleanup before deployment
python debug-tools/migration_tester.py

# 4. Optimize AI requests to prevent future issues
python debug-tools/ai_request_analyzer_v2.py logs/ai-request.json --detailed
```

## Integration with TypeScript Code

The TypeScript codebase includes Python command recognition in:
- `worker/agents/utils/common.ts:108` - Recognizes `python`, `pip`, `conda` commands
- `worker/agents/inferutils/core.ts:172` - Handles Python/YAML content in optimization

This allows the AI agents to work with Python code when generated, but doesn't execute Python in production.

## Key Points

✅ **Python is debug-only** - not deployed or executed in production  
✅ **Self-contained tools** - no external dependencies required  
✅ **Cloudflare-native stack** - Workers, Durable Objects, D1, R2  
✅ **Edge deployment** - no traditional servers or containers for Python  
✅ **State debugging focus** - all tools help debug Durable Object issues  

❌ **No Python runtime** in Cloudflare Workers  
❌ **No Python deployment** process needed  
❌ **No Python dependencies** to manage in production  

## Related Documentation

- [Container Deployment](./container-deployment.md) - How containers are actually deployed
- [Deployment Architecture](./deployment-architecture.md) - Overall system architecture
- [Architecture Diagrams](./architecture-diagrams.md) - Visual system overview