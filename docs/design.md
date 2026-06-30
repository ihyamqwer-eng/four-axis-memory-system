# Design Document: Four-Axis Memory System

A personal knowledge management system that decomposes "memory" into four dimensions: time, state, project, and dialogue. Covers the full knowledge lifecycle from raw data to vector retrieval to readable knowledge entries.

Not a replacement for vector databases—but an organizational framework on top of them.

---

## Why "Four Axes"

Problems with existing memory systems:

| Approach      | Problem                                      |
|---------------|----------------------------------------------|
| Pure vector   | Loses temporal order, can't separate "state" and "event" |
| MemGPT        | Limited context, no upper-level organization for long-term memory |
| Graph DB      | Heavy extraction overhead, overkill for personal use |
| Flat logs     | Low retrieval efficiency, hard to find "that thing from before" |

Four axes separate memory by **purpose**, not by format:

| Axis      | Purpose                      | Human analogy                    |
|-----------|------------------------------|----------------------------------|
| Timeline  | Time index, "what happened"  | "What happened that day"         |
| God Agent | Live state, "who I am"       | "What I'm wearing, how I feel"   |
| Object    | Task/project tracking        | "How's that project going"       |
| Session   | Raw dialogue archive         | "What exactly did they say"      |

---

## Three-Layer Architecture

```
┌──────────────────────────────────────────────────────────────┐
│              Layer 1: Four Axes (Raw Data)                    │
│                                                              │
│  Timeline · God Agent · Object · Session                     │
│  (jsonl files, append-only)                                   │
└──────────────────────────┬───────────────────────────────────┘
                           │
                           ▼
┌──────────────────────────────────────────────────────────────┐
│         Layer 2: Semantic Retrieval (LanceDB + Embedding)     │
│                                                              │
│  extract-fact → LLM extraction → embedding model → LanceDB  │
│  memory_recall.py → cross-axis semantic search                │
│                                                              │
│  What plain text search can't do:                            │
│  "happy" matches "feeling great today"                        │
│  cross-axis join is automatic                                 │
└──────────────────────────┬───────────────────────────────────┘
                           │
                           ▼
┌──────────────────────────────────────────────────────────────┐
│              Layer 3: Wiki Vault (Knowledge Base)             │
│                                                              │
│  concepts/   → conceptual definitions                        │
│  syntheses/  → distilled knowledge from dialogue              │
│  entities/   → entity profiles (tools, people, config items)  │
│  sources/    → fact provenance tracking                       │
│                                                              │
│  auto lint · auto provenance · wiki native support           │
└──────────────────────────────────────────────────────────────┘
```

## Design Principles

1. **Append-only, never mutate** — history is immutable
2. **Separate by responsibility, retrieve on demand** — don't overload one dimension
3. **Local-first, API-enhanced** — runs on filesystem alone
4. **Auto-sediment, grow organically** — knowledge grows from conversation naturally

---

## Axis Specifications

### Timeline (`timeline.jsonl`)

Temporal event index. One entry per event.

```json
{"ts": "2026-06-21T22:54:45+08:00", "event": "System Upgrade", "level": "milestone", "summary": "All three layers automated"}
```

Fields:
- `ts`: ISO 8601 timestamp with timezone
- `event`: event name
- `level`: daily | important | milestone
- `summary`: event description

### God Agent (`god/{agent}.jsonl`)

Live state snapshot chain. Appends a full snapshot on each state change.

```json
{"ts": "2026-06-22T09:21:19+08:00", "type": "state_update", "state": {"mood": "sleepy", "location": "home", "activity": "coding"}}
```

Concurrency: **last-write-wins**. Single-user, no collisions.

### Session (`session/YYYY-MM-DD-channel/reply-01.jsonl`)

Raw dialogue archive, organized by date and channel.

```json
{"ts": "2026-06-22T09:21:00+08:00", "who": "user", "msg": "Good morning", "channel": "telegram"}
```

Highest precision layer. Used for exact recall. Mandatory write.

### Object (`object/{name}.jsonl`)

Independent task/role state tracking. Created on task start, archived on completion.

```json
{"ts": "2026-06-20T01:19:04+08:00", "event": "task_created", "name": "doc_design_v2", "state": "in_progress", "detail": "Adding three-layer architecture"}
```

---

## Update Model

| Axis      | When                  | How                     | Concurrency       |
|-----------|-----------------------|-------------------------|-------------------|
| Timeline  | Important event       | `--event "name"`        | Append-only       |
| God Agent | State change          | `--state '{}'`          | Last-write-wins   |
| Session   | Every reply (forced)  | `--msg "text"`          | Append-only       |
| Object    | Task progress         | Manual jsonl write      | Append-only       |
| Semantic  | Every reply           | `extract-fact` auto     | Upsert            |
| Static    | Auto scan             | `check-static`          | Append-only       |
| Wiki      | Important facts       | `extract-fact` auto sync| Append or create  |

### One Command

```bash
python3 autofour_v2.py --mode auto-update \
  --channel "channel" \
  --who "speaker" \
  --msg "dialog text" \
  --auto-state
```

---

## Query Model

| Query                              | Path                           |
|------------------------------------|--------------------------------|
| "What happened recently?"          | Last N entries in timeline     |
| "How is she feeling?"              | Last 1 entry in god agent      |
| "What'd they say that day?"        | Session file for that date     |
| "How's that project going?"        | Object file for that task      |
| "I remember something about..."    | Semantic search                |
| "What's the system architecture?"  | Wiki concepts                  |
| "We discussed this knowledge"      | Wiki syntheses                 |

---

## Knowledge Lifecycle

### Three-Layer Sedimentation

```
Raw facts (four axes)
   ↓ LLM extract-fact
Vectorized knowledge (LanceDB)
   ↓ importance ≥ 4 auto-sync
Readable entries (Wiki Vault)
```

### extract-fact Types

| Type            | Example                     | Storage                |
|-----------------|-----------------------------|------------------------|
| Tech config     | API key change, tool usage  | LanceDB + Wiki concepts|
| User preference | Likes, habits               | LanceDB + Wiki syntheses|
| Rules           | Behavior rules, safety words| LanceDB + Wiki syntheses|
| Skills          | What was learned            | LanceDB + Wiki syntheses|
| Relationship    | Emotional changes           | LanceDB + Wiki syntheses|
| Persona         | Background, traits          | LanceDB + Wiki entities|

---

## Comparison

|                   | Four Axes | MemGPT | Pure Vector | Graph DB |
|-------------------|-----------|--------|-------------|----------|
| Time awareness    | ✅ Timeline| ✅ rec. summary| ❌      | ❌       |
| State tracking    | ✅ God    | ❌      | ❌          | ❌       |
| Project isolation | ✅ Object | ❌      | ❌          | ❌       |
| Raw dialogue      | ✅ Session| ❌ lost | ✅ (unstructured)| ❌  |
| Semantic search   | ⚠️ Optional| ❌    | ✅ Core     | ❌       |
| Readable KB       | ✅ Wiki   | ❌      | ❌          | ❌       |
| Deploy complexity | Low (FS)  | Medium  | Medium      | High     |
| Single user       | ✅ Perfect| ✅      | ✅ (heavy)  | ❌ (heavy)|

---

## Limitations

- No formal field definitions (engineering language, not academic)
- No large-scale validation (single-user scenario)
- Semantic retrieval and state inference depend on external LLM/embedding models
- No distributed/multi-tenant support
- Wiki writes are trigger-based (extract-fact), not real-time streaming

---

## Directory Structure

```
memory/
├── timeline.jsonl           # Event index
├── god/
│   ├── agent.jsonl          # State snapshots
│   └── user.jsonl
├── session/
│   └── YYYY-MM-DD-channel/  # Dialogue archives
├── object/
│   └── {name}.jsonl         # Task/role state
├── inventory.md             # Auto-maintained item log
└── important_dates.md       # Auto-maintained dates

wiki/
├── concepts/                # Conceptual definitions
├── syntheses/               # Distilled knowledge entries
├── entities/                # Entity profiles
├── sources/                 # Source tracking
└── reports/                 # Lint reports
```

---

**License:** MIT  
**Author:** Anonymous  
**Last updated:** 2026-06-30
