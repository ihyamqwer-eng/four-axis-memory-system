# Update Logic

Triggered automatically after each reply.

## Core Flow

1. **Session Axis** (mandatory)
   - Source: conversation info fields
     - `chat_id` → channel identifier
     - `message_id` → message ID
     - `timestamp` → message time
   - Format: `{ts, who, msg, refs, source}`

2. **God Agent Axis** (on demand)
   - Mood/location/activity changes → update
   - No change → skip

3. **Timeline Axis** (on demand)
   - Important events (config change, channel migration, task creation) → append
   - Casual chat → skip

4. **Object Axis** (on demand)
   - Task progress/ability usage → update
   - None → skip

## Hard Rules

- Step 1 is mandatory, regardless of changes
- Steps 2~4 are judged autonomously by the agent, no secondary editing

## Automation Scripts

| Script | Function |
|---|---|
| `autofour_v2.py` | Four-axis writer (with auto-state LLM) |
| `manage_static_files.py` | Auto-scan inventory and important dates |
| `memory_recall.py` | Semantic retrieval |
| `wake_inject.sh` | Context injection on wake |

## Source Field Format

```json
{
  "chat_id": "channel user ID",
  "message_id": "channel message ID",
  "timestamp": "message time"
}
```
