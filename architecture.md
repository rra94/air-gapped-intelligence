# Air-Gapped Intelligence: Technical Architecture

## System Overview

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              USER REQUEST                                   │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                           BRAIN (Claude API)                                │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐        │
│  │   Planner   │  │    GRPO     │  │   Reward    │  │   Prompts   │        │
│  │             │  │  Evaluator  │  │   Assigner  │  │             │        │
│  └─────────────┘  └─────────────┘  └─────────────┘  └─────────────┘        │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                    ┌─────────────────┼─────────────────┐
                    ▼                 ▼                 ▼
┌───────────────────────┐ ┌───────────────────┐ ┌───────────────────────────┐
│    AGENT REGISTRY     │ │  CONTEXT MANAGER  │ │   ROUTER / SCHEDULER      │
│  • Discovery          │ │  • TaskWorkspace  │ │  • Dispatch               │
│  • Validation         │ │  • Shared State   │ │  • Parallel Execution     │
│  • Model Updater      │ │  • Dependencies   │ │  • Sanitization           │
└───────────────────────┘ └───────────────────┘ └───────────────────────────┘
                    │                 │                 │
                    └─────────────────┼─────────────────┘
                                      │
        ┌─────────────┬───────────────┼───────────────┬─────────────┐
        ▼             ▼               ▼               ▼             ▼
┌─────────────┐ ┌─────────────┐ ┌─────────────┐ ┌─────────────┐ ┌─────────────┐
│ CodeExpert  │ │VisionExpert │ │SecurityExp  │ │ TestExpert  │ │ DataExpert  │
│qwen2.5-coder│ │ qwen2.5vl   │ │llama-guard3 │ │qwen2.5-coder│ │  qwen2.5    │
│    :14b     │ │    :7b      │ │    :8b      │ │    :14b     │ │    :14b     │
└─────────────┘ └─────────────┘ └─────────────┘ └─────────────┘ └─────────────┘
        │             │               │               │             │
        └─────────────┴───────────────┼───────────────┴─────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                           STORAGE LAYER                                     │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐             │
│  │  Conversation   │  │   Agent State   │  │   ChromaDB      │             │
│  │     Store       │  │     Store       │  │  Vector Store   │             │
│  │  (NO compact)   │  │  (checkpoints)  │  │ (semantic RAG)  │             │
│  └─────────────────┘  └─────────────────┘  └─────────────────┘             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Domain Expert Workers

Consolidated expert set to reduce resource requirements:

| Domain Expert | Primary Model | Fallback | Specialty |
|---------------|---------------|----------|-----------|
| **CodeExpert** | `qwen2.5-coder:14b` | `deepseek-coder-v2:16b` | Code, tests, security, refactoring |
| **VisionExpert** | `qwen2.5vl:7b` | `llava:13b` | Image analysis, OCR, UI screenshots |
| **DocExpert** | `qwen2.5:14b` | `phi4:14b` | Documentation, API specs |
| **DevOpsExpert** | `granite-code:20b` | `qwen2.5:14b` | CI/CD, Docker, K8s |
| **FastValidator** | `qwen2.5:3b` | `phi4:14b` | Quick syntax, linting |
| **EmbeddingExpert** | `nomic-embed-text` | `bge-m3` | Vector embeddings for ChromaDB |

**Hardware Requirements:**
- Minimum: 16GB RAM, 8GB VRAM (run one model at a time)
- Recommended: 32GB RAM, 16GB+ VRAM (keep models loaded)

---

## GRPO: Group Relative Policy Optimization

Self-improving evaluation without a separate critic model:

```
┌─────────────────────────────────────────────────────────────────┐
│                    GRPO EVALUATION LOOP                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  STEP 1: GROUP SAMPLING                                         │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │  Expert generates N solutions (e.g., N=5)               │    │
│  │  Task: "Refactor auth.py to use JWT"                    │    │
│  │  Solutions: [S1, S2, S3, S4, S5]                        │    │
│  └─────────────────────────────────────────────────────────┘    │
│                           ↓                                     │
│  STEP 2: REWARD ASSIGNMENT                                      │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │  Automated: tests pass, lint, type check, security      │    │
│  │  Brain-eval: correctness, efficiency, maintainability   │    │
│  │  Rewards: [R1=0.8, R2=0.6, R3=0.9, R4=0.5, R5=0.7]      │    │
│  └─────────────────────────────────────────────────────────┘    │
│                           ↓                                     │
│  STEP 3: BASELINE CALCULATION                                   │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │  Baseline = mean(R1, R2, R3, R4, R5) = 0.7              │    │
│  │  No external critic needed                              │    │
│  └─────────────────────────────────────────────────────────┘    │
│                           ↓                                     │
│  STEP 4: SELECTION & FEEDBACK                                   │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │  S3 (0.9): +0.2 → Best solution, use this               │    │
│  │  S1 (0.8): +0.1 → Good example                          │    │
│  │  S5 (0.7):  0.0 → Neutral                               │    │
│  │  S2 (0.6): -0.1 → Learn from mistakes                   │    │
│  │  S4 (0.5): -0.2 → Avoid this approach                   │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                 │
│  Output: Best solution + feedback for improvement               │
└─────────────────────────────────────────────────────────────────┘
```

**Reward Components:**
```
FINAL REWARD = 0.7 * automated + 0.3 * brain_evaluated

Automated (0-1 each, run locally):
  • Tests pass (0/1)
  • Lint score (0-1, normalized)
  • Type check pass (0/1)
  • Security scan clean (0/1)
  • Complexity delta (0-1, lower is better)
  • Test coverage delta (0-1, higher is better)

Brain-Evaluated (0-1 each, from sanitized summary):
  • Task completion: Did the summary indicate the task was done?
  • Scope adherence: Did it stay within requested changes?

NOTE: Brain does NOT evaluate code quality (correctness, efficiency).
      Code quality is measured by automated metrics only.
      Brain evaluates WHETHER the task was completed, not HOW WELL.
```

**Adaptive N Configuration:**
```
GRPO samples N solutions based on task complexity:

  simple:  N=1, skip_grpo=true   (lint fix, typo, rename)
  medium:  N=2, skip_grpo=false  (single function, bug fix)
  complex: N=3, skip_grpo=false  (multi-file refactor, new feature)
```

**GRPO Latency Expectations:**
```
┌──────────────────────────────────────────────────────────────────────┐
│  LATENCY BY HARDWARE TIER                                            │
├──────────────┬─────────────┬─────────────┬───────────────────────────┤
│ Hardware     │ Model       │ Per-Sample  │ N=3 Total (with overhead) │
├──────────────┼─────────────┼─────────────┼───────────────────────────┤
│ CPU only     │ 7B          │ 3-5 min     │ 10-15 min                 │
│ 8GB VRAM     │ 7B          │ 30-60 sec   │ 2-3 min                   │
│ 16GB VRAM    │ 14B         │ 45-90 sec   │ 3-5 min                   │
│ 24GB+ VRAM   │ 14B         │ 20-40 sec   │ 1-2 min                   │
└──────────────┴─────────────┴─────────────┴───────────────────────────┘

Latency Mitigation Strategies:
  1. Use smaller models (7B) for GRPO sampling, full model for final only
  2. Stream partial outputs - start evaluation before generation completes
  3. Cache model in VRAM between samples (Ollama keep_alive)
  4. Skip GRPO entirely for simple tasks (N=1)
  5. Parallel sampling if VRAM permits (24GB+ only)
```

**Error Propagation:**
```
┌──────────────────┬─────────────────────────────────┐
│ Failure          │ Action                          │
├──────────────────┼─────────────────────────────────┤
│ 1/N experts fail │ Continue with N-1, note failure │
│ All experts fail │ Escalate to Brain for re-plan   │
│ Partial output   │ Score as 0, include in baseline │
│ Timeout          │ Kill, use best completed result │
└──────────────────┴─────────────────────────────────┘
```

---

## Shared Context Manager

Workers share context through a centralized manager:

```
┌─────────────────────────────────────────────────────────────┐
│                    CONTEXT MANAGER                          │
├─────────────────────────────────────────────────────────────┤
│  TaskWorkspace (task_123):                                  │
│  ├─ file_state: {auth.py: v2, tests/: v1}                   │
│  ├─ expert_outputs: {CodeExpert: {...}, TestExpert: {...}}  │
│  ├─ shared_vars: {function_signature: "def login(...)"}     │
│  └─ dependencies: [CodeExpert → TestExpert]                 │
├─────────────────────────────────────────────────────────────┤
│  API:                                                       │
│  • get_context(task_id, worker_id) → relevant context       │
│  • set_output(task_id, worker_id, output) → store result    │
│  • share(task_id, from_worker, to_worker, data)             │
│  • get_file_state(task_id) → current file versions          │
│  • commit_changes(task_id) → persist to disk                │
└─────────────────────────────────────────────────────────────┘

Context Flow:
  CodeExpert writes function → Context Manager stores
                                       ↓
  TestExpert reads CodeExpert output from Context Manager
                                       ↓
  TestExpert generates tests → Context Manager stores
                                       ↓
  Brain evaluates both via GRPO
```

**Execution Model:**
```
SEQUENTIAL: File-modifying tasks (write, edit, delete)
  - One expert at a time for file changes
  - Prevents merge conflicts
  - Simpler state management

PARALLEL: Read/analysis tasks (review, analyze, search)
  - Multiple experts can read simultaneously
  - No conflict risk
  - Faster for exploration tasks
```

**DAG-Based Dependency Resolution:**
```
┌─────────────────────────────────────────────────────────────────┐
│  DEPENDENCY GRAPH ENGINE                                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Step 1: Build DAG from task dependencies                       │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │  CodeExpert ──────► TestExpert ──────► SecurityExpert   │    │
│  │       │                                     ▲           │    │
│  │       └─────────────► DocExpert ────────────┘           │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                 │
│  Step 2: Cycle Detection (Kahn's algorithm)                     │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │  IF cycle detected:                                     │    │
│  │    → Reject task plan, escalate to Brain for re-plan    │    │
│  │  ELSE:                                                  │    │
│  │    → Proceed with topological execution order           │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                 │
│  Step 3: Execute with fairness scheduling                       │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │  • Round-robin among ready tasks (no starvation)        │    │
│  │  • Priority boost for blocking dependencies             │    │
│  │  • Timeout per expert: 5 min (configurable)             │    │
│  │  • Max wait time: 10 min before forced escalation       │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘

Deadlock Prevention:
  • Cycle detection at plan time (not runtime)
  • No circular dependencies allowed
  • Brain must re-plan if cycle detected

Starvation Prevention:
  • Max 3 consecutive runs for any single expert
  • Aging: waiting tasks gain priority over time
  • Forced preemption after timeout
```

---

## Agent Registry

Dynamic discovery and management of domain experts:

```
┌─────────────────────────────────────────────────────────────────┐
│                       AGENT REGISTRY                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  DISCOVERY MODULE                                               │
│  • Scans Ollama for available models                            │
│  • Auto-registers new domain experts                            │
│  • Maintains discovery history & retry logic                    │
│                                                                 │
│  VALIDATION MODULE                                              │
│  • Probes agent capabilities (test requests)                    │
│  • Verifies model responds correctly                            │
│  • Generates capability metadata                                │
│                                                                 │
│  REGISTRY STORAGE                                               │
│  • Persistent SQLite storage                                    │
│  • Agent metadata: name, model, capabilities                    │
│  • Performance metrics & health status                          │
│                                                                 │
├─────────────────────────────────────────────────────────────────┤
│  API:                                                           │
│  • register(expert) → registers new domain expert               │
│  • discover() → scans for available models                      │
│  • lookup(domain) → finds best expert for domain                │
│  • health_check(expert_id) → verifies expert responsive         │
└─────────────────────────────────────────────────────────────────┘
```

---

## Model Updater

Automatic model updates when better versions available:

```
┌─────────────────────────────────────────────────────────────────┐
│                    MODEL UPDATER                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  VERSION CHECKER                                                │
│  • Polls Ollama library for new versions                        │
│  • Compares local vs available                                  │
│  • Identifies deprecated models                                 │
│                                                                 │
│  PERFORMANCE MONITOR                                            │
│  • Tracks per-expert success rates                              │
│  • Monitors inference latency                                   │
│  • Triggers swap when thresholds breached                       │
│                                                                 │
│  MODEL SWAPPER                                                  │
│  • Hot-swaps models without restart                             │
│  • Pulls new models via Ollama API                              │
│  • Rollback capability if new model fails                       │
│                                                                 │
├─────────────────────────────────────────────────────────────────┤
│  Update Flow:                                                   │
│  1. Scheduled check finds newer model                           │
│  2. Pull: `ollama pull deepseek-coder-v3:16b`                   │
│  3. Validation runs test prompts                                │
│  4. If pass → hot-swap; If fail → keep current                  │
└─────────────────────────────────────────────────────────────────┘

Auto-Update Policy:
  auto_update: true
  schedule: "weekly"
  require_validation: true
  rollback_on_failure: true
```

**Shadow Mode for Safe Updates:**
```
┌─────────────────────────────────────────────────────────────────┐
│  SHADOW MODE UPDATE PROTOCOL                                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Problem: New model might have different output format,         │
│           quality regression, or incompatible behavior          │
│                                                                 │
│  Solution: Run new model in "shadow mode" before swap           │
│                                                                 │
│  ┌───────────────────────────────────────────────────────┐      │
│  │  PHASE 1: Shadow Deployment                           │      │
│  │  • Pull new model: ollama pull qwen2.5-coder:latest   │      │
│  │  • Keep current model as primary                      │      │
│  │  • New model runs in parallel (shadow)                │      │
│  └───────────────────────────────────────────────────────┘      │
│                           ↓                                     │
│  ┌───────────────────────────────────────────────────────┐      │
│  │  PHASE 2: Comparison (N=10 real tasks)                │      │
│  │  • Both models process same inputs                    │      │
│  │  • Only current model output is used                  │      │
│  │  • Shadow output scored but discarded                 │      │
│  │  • Track: latency, quality score, error rate          │      │
│  └───────────────────────────────────────────────────────┘      │
│                           ↓                                     │
│  ┌───────────────────────────────────────────────────────┐      │
│  │  PHASE 3: Promotion Decision                          │      │
│  │  IF shadow_score >= current_score * 0.95:             │      │
│  │    AND shadow_errors <= current_errors:               │      │
│  │    AND shadow_latency <= current_latency * 1.2:       │      │
│  │      → PROMOTE shadow to primary                      │      │
│  │  ELSE:                                                │      │
│  │      → REJECT, keep current, log reason               │      │
│  └───────────────────────────────────────────────────────┘      │
│                                                                 │
│  Rollback: Keep previous model for 7 days after promotion       │
│            Instant rollback if error rate spikes                │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## Persistent Storage Layer

```
┌─────────────────────────────────────────────────────────────────┐
│                     STORAGE LAYER                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  CONVERSATION STORE (SQLite)                                    │
│  • Full Brain ↔ Expert message history                          │
│  • Indexed by task_id, timestamp                                │
│  • NEVER auto-deleted or compacted                              │
│  • Supports retrieval for mental model rebuild                  │
│                                                                 │
│  AGENT STATE STORE (SQLite)                                     │
│  • Per-expert state snapshots                                   │
│  • Checkpoint/restore capability                                │
│  • Tracks expert lifecycle (active, idle, error)                │
│                                                                 │
│  VECTOR STORE (ChromaDB)                                        │
│  • Collections: conversations, code_chunks, documentation       │
│  • Semantic search for RAG                                      │
│  • Local embeddings via nomic-embed-text                        │
│                                                                 │
│  MENTAL MODEL BUILDER                                           │
│  • Reconstructs context from conversation history               │
│  • Used ONLY when context window exhausted                      │
│  • Preserves key decisions, file states, outcomes               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘

Context Strategy:
  NORMAL: Brain keeps full context, all persisted to storage
  EXHAUSTED: Mental Model Builder creates summary from storage
             Brain receives summary + recent N messages
             Full history remains in storage (recoverable)
```

**Mental Model Builder Algorithm:**
```
┌─────────────────────────────────────────────────────────────────┐
│  MENTAL MODEL RECONSTRUCTION                                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  STEP 1: Extract Structured Data (always preserved)             │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │  • File states: {path: version, last_modified, status}  │    │
│  │  • Decisions: [{decision, rationale, timestamp}]        │    │
│  │  • Errors: [{error, resolution, expert}]                │    │
│  │  • Task outcomes: [{task_id, status, result_summary}]   │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                 │
│  STEP 2: Semantic Relevance Scoring                             │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │  For each conversation message:                         │    │
│  │    1. Embed message using EmbeddingExpert               │    │
│  │    2. Compare to current task embedding                 │    │
│  │    3. Score = cosine_similarity(msg, current_task)      │    │
│  │    4. Keep if score > 0.7 OR message contains decision  │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                 │
│  STEP 3: Compression (for kept messages)                        │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │  • Code blocks → "Modified {file}: {summary}"           │    │
│  │  • Long outputs → truncate to first/last 10 lines       │    │
│  │  • Repeated patterns → "...repeated N times..."         │    │
│  │  • Preserve: errors, decisions, user instructions       │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                 │
│  STEP 4: Assemble Mental Model                                  │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │  mental_model = {                                       │    │
│  │    "session_summary": <1 paragraph overview>,           │    │
│  │    "current_file_states": [...],                        │    │
│  │    "key_decisions": [...],                              │    │
│  │    "unresolved_issues": [...],                          │    │
│  │    "relevant_history": <compressed messages>,           │    │
│  │    "recent_messages": <last 20 raw messages>            │    │
│  │  }                                                      │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘

What Gets Dropped:
  • Intermediate outputs (superseded by final results)
  • Verbose logs and stack traces (keep summary only)
  • Duplicate/similar messages (keep representative)
  • Low-relevance context (score < 0.7, no decision markers)

What Is NEVER Dropped:
  • User instructions and preferences
  • Explicit decisions with rationale
  • Error resolutions (how problems were fixed)
  • File state changes
  • Task completion status
```

---

## Documentation Manager

Qwen-powered automatic documentation maintenance:

```
┌─────────────────────────────────────────────────────────────────┐
│                  DOCUMENTATION MANAGER                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  DOC WATCHER (watchdog)                                         │
│  • Monitors code changes (file system events)                   │
│  • Detects when docs may be stale                               │
│  • Tracks doc-to-code relationships                             │
│                                                                 │
│  DOC ANALYZER                                                   │
│  • Parses existing documentation                                │
│  • Identifies outdated sections                                 │
│  • Finds missing documentation                                  │
│                                                                 │
│  DOC GENERATOR (Qwen-powered)                                   │
│  • Generates new documentation from code                        │
│  • Updates existing docs to match changes                       │
│  • Maintains consistent style/format                            │
│                                                                 │
│  DOC CLEANER                                                    │
│  • Removes obsolete documentation                               │
│  • Fixes formatting issues                                      │
│  • Validates links and references                               │
│                                                                 │
├─────────────────────────────────────────────────────────────────┤
│  Modes: REACTIVE (on change), SCHEDULED, ON-DEMAND              │
└─────────────────────────────────────────────────────────────────┘
```

---

## Message Flow

```
USER → BRAIN (Claude)
         │
         ├─ Plans task decomposition
         ├─ Selects domain expert(s)
         └─ Defines GRPO criteria
         │
         ▼
    CONTEXT MANAGER ←──────────────────────────┐
         │                                     │
         ▼                                     │
    ROUTER/SCHEDULER                           │
         │                                     │
    ┌────┴────┬────────┬────────┐              │
    ▼         ▼        ▼        ▼              │
 CODE      VISION   SECURITY   TEST            │
 EXPERT    EXPERT   EXPERT     EXPERT          │
    │         │        │        │              │
    └────┬────┴────────┴────────┘              │
         │                                     │
         ▼                                     │
    SANITIZER ─────────────────────────────────┘
         │
         ▼
    BRAIN: GRPO EVALUATION
         │
    ┌────┴────┐
    ▼         ▼
  ACCEPT    RETRY
 (next)   (feedback)
```

---

## Project Structure

```
air-gapped-intelligence/
├── plan.md
├── architecture.md
├── pyproject.toml
├── src/air_gapped_intelligence/
│   ├── brain/
│   │   ├── client.py
│   │   ├── planner.py
│   │   ├── grpo/
│   │   │   ├── sampler.py
│   │   │   ├── evaluator.py
│   │   │   ├── baseline.py
│   │   │   └── selector.py
│   │   └── prompts.py
│   ├── hands/
│   │   ├── pool.py
│   │   ├── experts/
│   │   │   ├── base.py
│   │   │   ├── code.py
│   │   │   ├── vision.py
│   │   │   ├── security.py
│   │   │   └── ...
│   │   └── executor.py
│   ├── context/
│   │   ├── manager.py
│   │   ├── workspace.py
│   │   └── state.py
│   ├── storage/
│   │   ├── conversation.py
│   │   ├── agent_state.py
│   │   ├── vector.py
│   │   └── mental_model.py
│   ├── registry/
│   │   ├── registry.py
│   │   ├── discovery.py
│   │   ├── validation.py
│   │   └── updater/
│   │       ├── version_checker.py
│   │       └── swapper.py
│   ├── docs/
│   │   ├── manager.py
│   │   ├── watcher.py
│   │   └── generator.py
│   ├── router/
│   │   ├── orchestrator.py
│   │   ├── dispatcher.py
│   │   ├── scheduler.py
│   │   └── sanitizer.py
│   └── tools/
│       ├── base.py
│       ├── file_ops.py
│       └── shell.py
└── tests/
```

---

## Sanitizer

Transforms expert outputs for Brain while preserving GRPO scoring data:

```
┌────────────────────────────────────────┐
│  SANITIZER OUTPUT STRUCTURE            │
├────────────────────────────────────────┤
│  sanitized_result (sent to Brain):     │
│    summary: "Added JWT auth to login"  │
│    diff_stats: +45 lines, -12 lines    │
│    files_changed: [auth.py, config.py] │
│                                        │
│  grpo_metadata (local only):           │
│    raw_output: <full code>             │
│    test_results: pass/fail + coverage  │
│    lint_score: 9.2/10                  │
│    type_check: clean                   │
│    security_scan: 0 issues             │
└────────────────────────────────────────┘

Brain evaluates WHAT was done (summary)
Automated metrics score HOW WELL (metadata)
```

---

## Security Boundaries

1. **Data Sanitization**: Brain never sees raw code, only summaries/diffs
2. **Shell Allowlist**: Only approved commands can execute
3. **Path Validation**: File operations restricted to project scope
4. **Size Limits**: Large files truncated before processing
5. **Sandboxed Execution**: Experts run in isolated environments

---

## Operational Configuration

**Claude API Rate Limiting:**
```yaml
rate_limit:
  requests_per_minute: 50      # Tier 1 default
  retry_strategy: exponential_backoff
  max_retries: 3
  queue_overflow: persist_to_disk
```

**Ollama Connection Management:**
```yaml
ollama_config:
  keep_alive: "10m"            # Model stays loaded after request
  preload_models:              # Load on startup
    - qwen2.5-coder:14b
  max_concurrent: 2            # Parallel inference limit
  cold_start_timeout: 120s     # First load can be slow
```

---

## Key Design Decisions

| Decision | Rationale |
|----------|-----------|
| GRPO over fixed thresholds | Self-calibrating, no arbitrary cutoffs |
| ChromaDB for vectors | Local, no cloud dependency, good performance |
| No auto-compaction | Full history preserves context for rebuilds |
| Domain experts over generic | Specialized prompts yield better results |
| Hot-swap models | Upgrade without service interruption |
| Watchdog for docs | Real-time change detection |
| Sequential file writes | Prevents merge conflicts, simpler than locking |
| Adaptive GRPO N | Reduce inference cost for simple tasks |
