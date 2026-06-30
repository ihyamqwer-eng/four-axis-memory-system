# Architecture Overview

```
┌─────────────────────────────────────────────────────────────┐
│                    Four Axes (Raw Data Layer)                │
│                                                             │
│  ┌────────────────┐  ┌─────────────────┐                    │
│  │    Timeline    │  │    God Agent    │                    │
│  │  time.jsonl    │  │  agent.jsonl     │                    │
│  │  append-only   │  │  last-write-wins│                    │
│  │  event index   │  │  state snapshot │                    │
│  └───────┬────────┘  └────────┬─────────┘                    │
│          │                    │                              │
│  ┌───────┴────────┐  ┌───────┴─────────┐                    │
│  │    Object      │  │    Session      │                    │
│  │  tasks.jsonl   │  │  YYYY-MM-DD/    │                    │
│  │  task tracking │  │  raw dialogue   │                    │
│  │  create→archive│  │  highest detail │                    │
│  └───────┬────────┘  └────────┬─────────┘                    │
│          │                    │                              │
└──────────┼────────────────────┼──────────────────────────────┘
           │                    │
           ▼                    ▼
┌──────────────────────────────────────────────────────────────┐
│              LanceDB (Semantic Retrieval Layer)               │
│                                                              │
│  extract-fact → embed → search                              │
│  memory_recall.py → top-3 semantic results                  │
└──────────────────────────┬───────────────────────────────────┘
                           │
                           ▼
┌──────────────────────────────────────────────────────────────┐
│              Wiki Vault (Knowledge Base Layer)                │
│                                                              │
│  concepts/ · syntheses/ · sources/ · reports/               │
│  auto lint · auto provenance tracking                        │
└──────────────────────────────────────────────────────────────┘
```

## Data Flow

```
User Message
   ↓ Write
Session Axis (raw dialogue archive)
   ↓
extract-fact (LLM extracts key facts)
   ↓
LanceDB vectorization (for semantic retrieval)
   ↓
State changes → God Agent Axis
Important events → Timeline Axis
Task progress → Object Axis

On wake:
  wake_inject → last 5 timeline + god agent snapshot + top-3 semantic
```

## Update Rules

| Axis   | When                | How                       | Strategy       |
|--------|---------------------|---------------------------|----------------|
| Session| Every reply         | `--msg "text"`            | Append-only    |
| God    | On state change     | `--state '{}'`            | Last-write-wins|
| Timeline| On important event | `--event "name"`          | Append-only    |
| Object | On task progress   | Manual jsonl write        | Append-only    |
| Static | Auto scan           | `check-static`            | Append-only    |

## One Command

```bash
python3 autofour_v2.py --mode auto-update \
  --channel "channel" \
  --who "agent" \
  --msg "dialog text" \
  --auto-state
```
