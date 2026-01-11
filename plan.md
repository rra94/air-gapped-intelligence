# Air-Gapped Intelligence: Project Plan

## Overview

A **Manager-Worker Swarm architecture** where Claude (Brain) handles high-level reasoning while a pool of specialized local Qwen workers (Hands) execute sensitive operations. Data sovereignty is maintained by never sending raw code/data to the cloud.

**Stack:** Python + Anthropic SDK + Ollama + MCP + ChromaDB

---

## Core Concept: Brain vs Hands

| Component | Role | Location |
|-----------|------|----------|
| **Brain** (Claude) | High-level reasoning, planning, GRPO evaluation | Cloud API |
| **Hands** (Qwen) | Code execution, file ops, image processing | Local (Ollama) |
| **Router** | Intercepts, dispatches, sanitizes | Local |

---

## Implementation Steps

### Phase 1: Foundation
- [ ] Project scaffolding (`pyproject.toml`, `.env.example`)
- [ ] Configuration layer (Pydantic Settings)
- [ ] **plan.md** and **architecture.md** documentation

### Phase 2: Brain Layer
- [ ] Claude API client with tool injection
- [ ] Task planner and decomposition
- [ ] GRPO evaluation system (sampler, evaluator, selector)

### Phase 3: Hands Layer (Domain Experts)
- [ ] Base domain expert interface
- [ ] Worker pool manager (lifecycle, spin up/down)
- [ ] Specialized experts: Code, Vision, Security, Test, Data, DevOps
- [ ] Ollama client wrapper

### Phase 4: Orchestration
- [ ] Router/Dispatcher (routes to experts)
- [ ] Parallel scheduler
- [ ] Sanitizer (transforms outputs for Brain)
- [ ] Main orchestration loop

### Phase 5: Context & Storage
- [ ] Shared Context Manager (worker-to-worker)
- [ ] Conversation Store (full history, NO auto-compact)
- [ ] Agent State Store (checkpoints)
- [ ] ChromaDB Vector Store (semantic search)
- [ ] Mental Model Builder (context reconstruction)

### Phase 6: Registry & Updates
- [ ] Agent Registry (discovery, validation, storage)
- [ ] Model Updater (version checker, hot-swap, rollback)

### Phase 7: Documentation Manager
- [ ] Doc Watcher (file system events)
- [ ] Doc Analyzer (staleness detection)
- [ ] Doc Generator (Qwen-powered)

### Phase 8: Integration
- [ ] CLI entry point
- [ ] MCP tool definitions
- [ ] Tests and examples

---

## Pros & Cons

### Advantages

| Benefit | Description |
|---------|-------------|
| **Data Sovereignty** | Sensitive code never leaves local machine |
| **Cost Efficiency** | Local Qwen handles heavy lifting (free tokens) |
| **GRPO Evaluation** | Self-calibrating, no critic model needed |
| **Domain Experts** | Right model for the job |
| **Parallel Execution** | Multiple experts run simultaneously |
| **Full History** | Conversation persisted, never lost |
| **Semantic Search** | ChromaDB enables RAG |
| **Auto Documentation** | Docs stay current with code |
| **Model Auto-Update** | Registry keeps experts on best models |

### Challenges

| Challenge | Mitigation |
|-----------|------------|
| Context Desync | Full history storage, mental model rebuild |
| Local GPU Required | Ollama supports CPU, smaller models available |
| Complexity | Clean architecture, comprehensive tests |
| Storage Growth | Periodic archival, configurable retention |

---

## Best Practices

### 1. Data Sanitization
```
Brain sees:     Sanitized summaries, diffs, metrics
Brain NEVER sees: Raw source code, credentials, full file contents
```

### 2. GRPO Evaluation
```
1. Generate N solutions (group sampling)
2. Score each solution (automated + Brain eval)
3. Calculate baseline = mean(rewards)
4. Select best, learn from above/below baseline
```

### 3. Error Recovery
- **RETRYABLE**: Network issues → exponential backoff
- **RECOVERABLE**: Tool failures → Brain re-plans
- **FATAL**: Permission denied → stop and report

### 4. Security
- Shell command allowlist
- Path validation for file operations
- Size limits for file reads
- Sandboxed execution

---

## Dependencies

```toml
dependencies = [
    "anthropic>=0.40.0,<1.0.0",       # Claude API
    "ollama>=0.4.0,<1.0.0",           # Ollama client
    "mcp>=1.25.0,<2.0.0",             # Model Context Protocol (pin major)
    "pydantic>=2.0.0,<3.0.0",         # Data validation
    "pydantic-settings>=2.0.0,<3.0.0",# Configuration
    "chromadb>=0.4.0,<0.6.0",         # Vector database (API changed in 0.5)
    "watchdog>=4.0.0,<6.0.0",         # File system monitoring
    "rich>=13.0.0,<15.0.0",           # CLI output
    "structlog>=24.0.0,<26.0.0",      # Structured logging
]
```

---

## Timeline

This is a phased implementation. Each phase can be developed and tested independently:

1. **Foundation + Brain**: Core infrastructure
2. **Hands + Orchestration**: Expert pool and routing
3. **Storage + Context**: Persistence and state management
4. **Registry + Docs**: Dynamic discovery and auto-documentation

---

## Success Criteria

- [ ] Brain can dispatch tasks to appropriate domain experts
- [ ] Experts execute locally without sending sensitive data to cloud
- [ ] GRPO evaluation selects best solution from group
- [ ] Context persists across sessions (no data loss)
- [ ] Model updates happen automatically when better versions available
- [ ] Documentation stays synchronized with code changes
