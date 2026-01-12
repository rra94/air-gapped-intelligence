# Air-Gapped Intelligence: Technical Architecture

## System Overview

```bash
┌─────────────────────────────────────────────────────────────────────────────┐
│                              USER REQUEST                                   │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                    PII SCRUBBER (Input) ──► CENTRAL VAULT                   │
│  Tokenize sensitive data before Brain: john@acme.com → {{EMAIL_a1b2}}       │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                     FAST-PATH CLASSIFIER (Rule-Based)                       │
│  Trivial tasks bypass Brain entirely → direct to single expert              │
└─────────────────────────────────────────────────────────────────────────────┘
                    │                               │
            TRIVIAL │                               │ NEEDS_PLANNING
                    ▼                               ▼
┌───────────────────────────┐   ┌─────────────────────────────────────────────┐
│  FAST-PATH EXECUTOR       │   │                BRAIN (Claude API)           │
│  • Single expert          │   │  ┌─────────┐ ┌──────────┐ ┌─────────┐       │
│  • Skip GRPO (N=1)        │   │  │ Planner │ │   GRPO   │ │ Prompts │       │
│  • Auto checks → done     │   │  └─────────┘ └──────────┘ └─────────┘       │
└───────────────────────────┘   └─────────────────────────────────────────────┘
           │                                        │
           │ (on failure)                           │
           └─────────────────►──────────────────────┤
                                                    ▼
                    ┌─────────────────┬─────────────────┐
                    ▼                 ▼                 ▼
┌───────────────────────┐ ┌───────────────────┐ ┌───────────────────────────┐
│    AGENT REGISTRY     │ │  CONTEXT MANAGER  │ │   ROUTER / SCHEDULER      │
│  • Discovery          │ │  • TaskWorkspace  │ │  • Dispatch               │
│  • Validation         │ │  • Shared State   │ │  • Parallel Execution     │
│                       │ │  • Dependencies   │ │  • PII Scrub + Sanitize   │
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
│                   PII SCRUBBER (Output) ──► CENTRAL VAULT                   │
│  Tokenize expert output before Brain, local experts restore from vault      │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                           STORAGE LAYER                                     │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐              │
│  │  Conversation   │  │   Agent State   │  │   ChromaDB      │              │
│  │     Store       │  │     Store       │  │  Vector Store   │              │
│  │  (NO compact)   │  │  (checkpoints)  │  │ (semantic RAG)  │              │
│  └─────────────────┘  └─────────────────┘  └─────────────────┘              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Domain Expert Workers

Consolidated expert set matching the system diagram:

| Domain Expert | Primary Model | Fallback | Specialty |
| --------------- | --------------- | ---------- | ----------- |
| **CodeExpert** | `qwen2.5-coder:14b` | `deepseek-coder-v2:16b` | Code generation, refactoring, bug fixes |
| **VisionExpert** | `qwen2.5vl:7b` | `llava:13b` | Image analysis, OCR, UI screenshots |
| **SecurityExpert** | `llama-guard3:8b` | `qwen2.5-coder:14b` | Security scanning, vulnerability detection |
| **TestExpert** | `qwen2.5-coder:14b` | `deepseek-coder-v2:16b` | Test generation, coverage analysis |
| **DataExpert** | `qwen2.5:14b` | `phi4:14b` | Data analysis, transformations |

**Embedding Model:** `nomic-embed-text` (for ChromaDB vector store)

**Hardware Requirements:**

- Recommended: 32GB RAM, 16GB+ VRAM (keep models loaded)

---

## GRPO: Group Relative Policy Optimization

Self-improving evaluation without a separate critic model:

```bash
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
│  │  Brain-eval: task completion, scope adherence           │    │
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

```bash
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

```bash
GRPO samples N solutions based on task complexity:

  simple:  N=1, skip_grpo=true   (lint fix, typo, rename)
  medium:  N=2, skip_grpo=false  (single function, bug fix)
  complex: N=3, skip_grpo=false  (multi-file refactor, new feature)
```

**GRPO Latency Expectations:**

```bash
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

```bash
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

```bash
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
│  • complete_task(task_id) → finalize + trigger hooks        │
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

```bash
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

```bash
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

**Post-Task Hooks:**

Triggered by `complete_task(task_id)`:

| Hook | Blocking | Purpose |
| ------ | ---------- | --------- |
| `audit_finalize` | Yes | Seal audit chain |
| `doc_generation` | No (defer until GPU idle) | Queue doc updates |
| `metrics_update` | No | Record performance |
| `workspace_cleanup` | Yes | Free resources |

---

## Agent Registry

Dynamic discovery and management of domain experts:

```bash
┌─────────────────────────────────────────────────────────────────┐
│                       AGENT REGISTRY                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  REGISTRY STORAGE                                               │
│  • Persistent SQLite storage                                    │
│  • Agent metadata: name, model, capabilities                    │
│  • Performance metrics & health status                          │
│                                                                 │
├─────────────────────────────────────────────────────────────────┤
│  API:                                                           │
│  • register(expert) → registers new domain expert               │
└─────────────────────────────────────────────────────────────────┘
```

---

## Persistent Storage Layer

```bash
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
│  CENTRAL PII VAULT (SQLite, AES-256 encrypted)                  │
│  • Token mapping: {{EMAIL_a1b2}} → encrypted(john@acme.com)     │
│  • Shared across input + output scrubbing (single source)       │
│  • Local experts restore originals via vault.resolve(token)     │
│  • Scoped per-session, cleared on session end                   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘

Context Strategy:
  NORMAL: Brain keeps full context, all persisted to storage
  EXHAUSTED: Mental Model Builder creates summary from storage
             Brain receives summary + recent N messages
             Full history remains in storage (recoverable)
```

**Mental Model Builder:**

Used only when context window exhausted. Reconstructs from storage:

```bash
1. Extract structured data (file states, decisions, errors, outcomes)
2. Score relevance (embed + cosine similarity > 0.7)
3. Compress (code → summary, truncate long outputs)
4. Assemble: session_summary + file_states + decisions + recent_messages
```

| Dropped | Never Dropped |
| --------- | --------------- |
| Intermediate outputs | User instructions |
| Verbose logs | Decisions with rationale |
| Duplicates | Error resolutions |
| Low-relevance (< 0.7) | File state changes |

---

## Documentation Manager

Deferred documentation generation using the Audit Log as change manifest.
No file watcher needed - audit log already tracks every change.

```bash
DURING WORK                      END OF TASK (via post-task hook)
═══════════                      ═══════════════════════════════

Experts work ──► Audit Log       Audit Log ──► Manifest ──► Doc Generator
(full GPU)      (records all)    (GPU free)    (what changed) (batch update)
```

**Components:**

```yaml
┌─────────────────────────────────────────────────────────────────┐
│  MANIFEST BUILDER     Query audit log, group by affected docs   │
│  DOC ANALYZER         Map code → doc relationships              │
│  DOC GENERATOR        Qwen-powered, full GPU, batch processing  │
│  DOC CLEANER          Remove stale docs, fix broken links       │
└─────────────────────────────────────────────────────────────────┘
```

**Triggers:**

| Trigger | When |
| --------- | ------ |
| END_OF_TASK | Post-task hook (deferred until GPU idle) |
| END_OF_SESSION | User exits / timeout |
| ON_DEMAND | User command |
| SCHEDULED | Cron (nightly) |

---

## Fast-Path Routing

Rule-based classification that bypasses full planning for trivial tasks:

```yaml
┌─────────────────────────────────────────────────────────────────────────────┐
│                              USER REQUEST                                   │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                      FAST-PATH CLASSIFIER (Rule-Based):                     │
│                                                                             │
│  Pattern matching against known trivial task types:                         │
│  • fix typo, rename X to Y, add comment, format file                        │
│  • update version, change string literal, toggle flag                       │
│                                                                             │
│  Decision: TRIVIAL | NEEDS_PLANNING                                         │
└─────────────────────────────────────────────────────────────────────────────┘
                    │                               │
            TRIVIAL │                               │ NEEDS_PLANNING
                    ▼                               ▼
┌───────────────────────────────┐   ┌─────────────────────────────────────────┐
│        FAST PATH:             │   │           STANDARD PATH:                │
│                               │   │                                         │
│  • Skip Brain planning        │   │  • Full Brain planning                  │
│  • Direct to single expert    │   │  • Multi-expert coordination            │
│  • Skip GRPO (N=1)            │   │  • GRPO evaluation (N=2-3)              │
│  • Minimal latency            │   │  • Full workflow                        │
└───────────────────────────────┘   └─────────────────────────────────────────┘
```

**Classification Rules:**

```yaml
TRIVIAL PATTERNS → Fast Path:
  • fix typo/spelling      → CodeExpert
  • rename X to Y          → CodeExpert
  • add comment            → CodeExpert
  • format file/code       → CodeExpert
  • update version         → CodeExpert
  • change string literal  → CodeExpert
  • remove unused import   → CodeExpert
  • fix lint error         → FastValidator

COMPLEXITY ESCALATORS → Standard Path:
  • refactor, implement, add feature
  • fix bug (needs investigation)
  • security (needs SecurityExpert)
  • test (needs TestExpert)
  • multiple files, across the codebase

DEFAULT: If no pattern matches → Standard Path (safe)
```

**Fast-Path Execution Flow:**

```bash
TRIVIAL TASK DETECTED
         │
         ▼
┌─────────────────────────────────────────────────────────────────┐
│  FAST-PATH EXECUTOR                                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. Select expert from classifier result                        │
│  2. Build minimal context (file content only)                   │
│  3. Execute single expert, single solution (N=1)                │
│  4. Run automated checks (lint, type check)                     │
│  5. If checks pass → commit changes                             │
│  6. If checks fail → escalate to standard path                  │
│                                                                 │
│  NO Brain API call for planning                                 │
│  NO GRPO multi-sampling                                         │
│  NO multi-expert coordination                                   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Escalation from Fast-Path:**

```yaml
┌──────────────────────┬─────────────────────────────────────────────┐
│ Condition            │ Action                                      │
├──────────────────────┼─────────────────────────────────────────────┤
│ Automated checks fail│ Escalate to standard path with context      │
│ Expert returns error │ Escalate to standard path for re-planning   │
│ Multiple files found │ Escalate (was misclassified)                │
│ User explicitly asks │ Always use standard path if user requests   │
│ Confidence < 0.8     │ Escalate to be safe                         │
└──────────────────────┴─────────────────────────────────────────────┘
```

**Benefits:**

- **Latency**: ~2-5s for trivial tasks vs 30s+ for full planning
- **Cost**: Zero Claude API calls for trivial tasks
- **Safety**: Automated checks catch failures, escalation preserves correctness

---

## Sanitizer

Transforms expert outputs for Brain while preserving GRPO scoring data.
PII Scrubber runs first (regex + NER detection), then Sanitizer extracts summaries:

```yaml
              ┌──────────────────────────────────────┐
              │         CENTRAL PII VAULT            │
              │  {{TOKEN}} ↔ encrypted(original)     │
              └──────────────────────────────────────┘
                     ▲                    ▲
                     │                    │
    ┌────────────────┴───┐          ┌─────┴────────────────┐
    │  INPUT SCRUBBING   │          │  OUTPUT SCRUBBING    │
    │  User → tokenize   │          │  Expert → tokenize   │
    │  → Brain           │          │  → Sanitizer → Brain │
    └────────────────────┘          └──────────────────────┘
                                          │
                                          ▼
                               Local experts resolve
                               tokens via vault lookup
```

**PII Scrubber** runs on both inputs and outputs with a central vault:

- **Input path**: User request → scrub → Brain sees tokens
- **Output path**: Expert output → scrub → Sanitizer → Brain sees tokens
- **Restore path**: Brain response → local experts resolve tokens from vault
- Regex: API keys, credentials, emails, SSN, phones, IPs, private keys
- NER: Person names, organizations, addresses (local spaCy model)
- Tokenization: `john@acme.com` → `{{EMAIL_a1b2}}`, `sk-abc123` → `{{KEY_c3d4}}`

```yaml
┌────────────────────────────────────────┐
│  SANITIZER OUTPUT STRUCTURE            │
├────────────────────────────────────────┤
│  sanitized_result (sent to Brain):     │
│    summary: "Added JWT auth to login"  │
│    diff_stats: +45 lines, -12 lines    │
│    files_changed: [auth.py, config.py] │
│    pii_scrubbed: true                  │
│                                        │
│  grpo_metadata (local only):           │
│    raw_output: <full code>             │
│    test_results: pass/fail + coverage  │
│    lint_score: 9.2/10                  │
│    type_check: clean                   │
│    security_scan: 0 issues             │
│    pii_vault_ref: task_123             │
└────────────────────────────────────────┘

Brain evaluates WHAT was done (summary)
Automated metrics score HOW WELL (metadata)
Local experts can restore PII from vault when needed
```

### Local Audit Log

Cryptographic audit trail for sanitization and reward integrity (stored in `audit.db`, never sent to cloud):

```yaml
┌─────────────────────────────────────────────────────────────────┐
│  AUDIT RECORD                                                   │
├─────────────────────────────────────────────────────────────────┤
│  Identity:     record_id, task_id, expert_id, timestamp         │
│  Hashes:       raw_input, sanitized_output, grpo_metadata       │
│  PII:          detections_count, vault_ref, scrub_method        │
│  Transform:    version, rules_applied                           │
│  Reward:       inputs_hash, score, breakdown                    │
│  Chain:        previous_hash, hmac_signature                    │
└─────────────────────────────────────────────────────────────────┘

Flow: Expert Output → PII Scrubber (detect, tokenize, vault) → Sanitizer → Brain
      Brain sees tokens and "Added auth for {{EMAIL_a1b2}}"
      Local experts call vault.resolve() to restore originals
      Reward Calculator → Append to audit → Re-sign
```

**Capabilities:**

| Query | Purpose |
| ------- | --------- |
| Verify record | Check signature and hash integrity |
| Verify chain | Detect tampering in history |
| Detect leakage | Scan outgoing messages for raw data |
| Trace task | Follow data flow for a specific task |

**Retention:** 90 days, daily integrity check, alert on chain break

---

## Security Boundaries

1. **PII Scrubbing**: Regex + NER tokenizes inputs and outputs before Brain
2. **Central Vault**: Single source for all PII tokens, local experts restore via lookup
3. **Data Sanitization**: Brain never sees raw code, only summaries/diffs
4. **Path Validation**: File operations restricted to project scope
5. **Size Limits**: Large files truncated before processing
6. **Sandboxed Execution**: Experts run in isolated environments

---

## Project Structure

```yaml
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
│   │   │   ├── test.py
│   │   │   └── data.py
│   │   └── executor.py
│   ├── context/
│   │   ├── manager.py
│   │   ├── workspace.py
│   │   ├── state.py
│   │   └── hooks.py            # Post-task hook system
│   ├── storage/
│   │   ├── conversation.py
│   │   ├── agent_state.py
│   │   ├── vector.py
│   │   ├── mental_model.py
│   │   ├── pii_vault.py        # Encrypted PII storage for local restoration
│   │   └── audit.py            # Local audit log for sanitizer/rewards
│   ├── registry/
│   │   ├── registry.py
│   │   ├── discovery.py
│   │   └── validation.py
│   ├── docs/
│   │   ├── manager.py
│   │   ├── manifest.py         # Builds change manifest from audit log
│   │   ├── analyzer.py         # Maps code → doc relationships
│   │   └── generator.py
│   ├── router/
│   │   ├── orchestrator.py
│   │   ├── dispatcher.py
│   │   ├── scheduler.py
│   │   ├── pii_scrubber.py       # Regex + NER PII detection/redaction
│   │   ├── sanitizer.py
│   │   └── fast_path/
│   │       ├── classifier.py      # Rule-based task classification
│   │       └── executor.py        # Fast-path execution logic
│   └── tools/
│       ├── base.py
│       ├── file_ops.py
│       └── shell.py
└── tests/
```

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
| ---------- | ----------- |
| GRPO over fixed thresholds | Self-calibrating, no arbitrary cutoffs |
| ChromaDB for vectors | Local, no cloud dependency, good performance |
| No auto-compaction | Full history preserves context for rebuilds |
| Domain experts over generic | Specialized prompts yield better results |
| Deferred doc generation | No VRAM contention, uses audit log as manifest |
| Sequential file writes | Prevents merge conflicts, simpler than locking |
| Adaptive GRPO N | Reduce inference cost for simple tasks |
| Fast-path routing | Zero API calls for trivial tasks |
| Post-task hooks | Clean trigger for deferred operations |
| Central PII Vault | Tokenize in/out; single source for local expert restoration |
