# Architecture Overview

## Core Distinction: Event-Driven vs State-Driven

The four axes are grouped by their persistence strategy, not just by their content type.

```
┌─────────────────────────────────────────────────────────────┐
│                    Four Axes (Raw Data Layer)                │
│                                                             │
│  ┌──── EVENT-DRIVEN ─────┐  ┌──── STATE-DRIVEN ──────┐    │
│  │                       │  │                         │    │
│  │     Timeline          │  │      God Agent          │    │
│  │   append-only         │  │   last-write-wins       │    │
│  │   immutable history   │  │   current snapshot      │    │
│  │   "what happened"     │  │   "how I'm feeling"     │    │
│  ├───────────────────────┤  ├─────────────────────────┤    │
│  │     Session           │  │      Object             │    │
│  │   append-only         │  │   append changelog      │    │
│  │   raw dialogue        │  │   task transitions      │    │
│  │   "exact words"       │  │   "project status"      │    │
│  └───────────────────────┘  └─────────────────────────┘    │
└──────────────────────────┬──────────────────────────────────┘
                           │
                           ▼
┌──────────────────────────────────────────────────────────────┐
│              LanceDB (Semantic Retrieval Layer)               │
│                                                              │
│  Cross-axis semantic search: "I remember something about..." │
│  extract-fact → embed → search                               │
│                                                              │
│  When you don't remember which axis it's in                  │
└──────────────────────────┬───────────────────────────────────┘
                           │
                           ▼
┌──────────────────────────────────────────────────────────────┐
│              Wiki Vault (Knowledge Base Layer)                │
│                                                              │
│  Auto-sedimented knowledge from conversation                 │
│  importance ≥ 4 → auto wiki page                             │
│                                                              │
│  concepts/ · syntheses/ · entities/ · sources/               │
└──────────────────────────────────────────────────────────────┘
```

## Data Flow

```
User Message
   ↓ Always write
Session Axis (raw dialogue archive)
   ↓ Auto-evaluate
┌── State change? → God Agent Axis (last-write-wins)
├── Important event? → Timeline Axis (append-only)
├── Task progress? → Object Axis (changelog)
└── Knowledge to extract? → extract-fact
        ↓
LanceDB vectorization (cross-axis semantic search)
        ↓
  importance ≥ 4? → Wiki Vault (readable knowledge)
```

## Update Rules

| Axis   | Driving | When                | How                | Strategy       |
|--------|---------|---------------------|--------------------|----------------|
| Session| Event   | Every reply         | `--msg "text"`     | Append-only    |
| God    | State   | On state change     | `--state '{}'`     | Last-write-wins|
| Timeline| Event  | On important event  | `--event "name"`   | Append-only    |
| Object | State   | On task progress    | Manual jsonl write | Append changelog|
| Static | —       | Auto scan           | `check-static`     | Append-only    |

## One Command

```bash
python3 autofour_v2.py --mode auto-update \
  --channel "channel" \
  --who "agent" \
  --msg "dialog text" \
  --auto-state
```
