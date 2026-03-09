# Progressive Disclosure

Memory retrieval uses a **three-layer progressive disclosure model**, all handled autonomously by the **memory-recall skill** running in a forked subagent context. Claude invokes the skill when it judges the user's question needs historical context -- no manual intervention required.

```mermaid
graph TD
    SKILL["memory-recall skill<br/>(context: fork subagent)"]
    SKILL --> L1["L1: Search<br/>(memsearch search)"]
    L1 --> L2["L2: Expand<br/>(memsearch expand)"]
    L2 --> L3["L3: Transcript drill-down<br/>(memsearch transcript)"]
    L3 --> RETURN["Curated summary<br/>→ main agent"]

    style SKILL fill:#2a3a5c,stroke:#6ba3d6,color:#a8b2c1
    style L1 fill:#2a3a5c,stroke:#6ba3d6,color:#a8b2c1
    style L2 fill:#2a3a5c,stroke:#e0976b,color:#a8b2c1
    style L3 fill:#2a3a5c,stroke:#d66b6b,color:#a8b2c1
    style RETURN fill:#2a3a5c,stroke:#7bc67e,color:#a8b2c1
```

## How the Skill Works

When Claude detects that a user's question could benefit from past context, it automatically invokes the `memory-recall` skill. The skill runs in a **forked subagent context** (`context: fork`), meaning it has its own context window and does not pollute the main conversation. The subagent:

1. **Searches** for relevant memories using `memsearch search`
2. **Evaluates** which results are truly relevant (skips noise)
3. **Expands** promising results with `memsearch expand` to get full markdown sections
4. **Drills into transcripts** when needed with `memsearch transcript`
5. **Returns a curated summary** to the main agent

The main agent only sees the final summary -- all intermediate search results, raw expand output, and transcript parsing happen inside the subagent.

Users can also manually invoke the skill with `/memory-recall <query>` if Claude doesn't trigger it automatically.

## L1: Search

The subagent runs `memsearch search` to find relevant chunks from the indexed memory files.

## L2: Expand

For promising search results, the subagent runs `memsearch expand` to retrieve the **full markdown section** surrounding a chunk:

```bash
$ memsearch expand 7a3f9b21e4c08d56
```

**Example output:**

```
Source: .memsearch/memory/2026-02-10.md (lines 12-32)
Heading: 09:15
Session: abc123de-f456-7890-abcd-ef1234567890
Turn: def456ab-cdef-1234-5678-90abcdef1234
Transcript: /home/user/.claude/projects/.../abc123de...7890.jsonl

### 08:50
<!-- session:abc123de... turn:aaa11122... transcript:/.../abc123de...7890.jsonl -->
- Set up project scaffolding for the new API service
- Configured FastAPI with uvicorn, added health check endpoint
- Connected to PostgreSQL via SQLAlchemy async engine

### 09:15
<!-- session:abc123de... turn:def456ab... transcript:/.../abc123de...7890.jsonl -->
- Added Redis caching middleware to API with 5-minute TTL
- Used redis-py async client with connection pooling (max 10 connections)
- Cache key format: `api:v1:{endpoint}:{hash(params)}`
- Added cache hit/miss Prometheus counters for monitoring
- Wrote integration tests with fakeredis
```

The subagent sees the full context including neighboring sections. The embedded `<!-- session:... -->` anchors link to the original conversation -- if the subagent needs to go even deeper, it moves to L3.

Additional flags:

```bash
# JSON output with anchor metadata (for programmatic L3 drill-down)
memsearch expand 47b5475122b992b6 --json-output

# Show N lines of context before/after instead of the full section
memsearch expand 47b5475122b992b6 --lines 10
```

## L3: Transcript Drill-Down

When Claude needs the original conversation verbatim -- for instance, to recall exact code snippets, error messages, or tool outputs -- it drills into the JSONL transcript.

**List all turns** in a session:

```bash
$ memsearch transcript /path/to/session.jsonl
```

```
All turns (73):

  6d6210b7-b84  08:50:14  Set up the project scaffolding for...          [12 tools]
  3075ee94-0f6  09:05:22  Can you add a health check endpoint?
  8e45ce0d-9a0  09:15:03  Add a Redis caching layer to the API...        [8 tools]
  53f5cac3-6d9  09:32:41  The cache TTL should be configurable...         [3 tools]
  c708b40c-8f8  09:45:18  Let's add Prometheus metrics for cache...      [10 tools]
```

Each line shows the turn UUID prefix, timestamp, content preview, and how many tool calls occurred.

**Drill into a specific turn** with surrounding context:

```bash
$ memsearch transcript /path/to/session.jsonl --turn 8e45ce0d --context 1
```

```
Showing 2 turns around 8e45ce0d:

>>> [09:05:22] 3075ee94
Can you add a health check endpoint?

**Assistant**: Sure, I'll add a `/health` endpoint that checks the database
connection and returns the service version.

>>> [09:15:03] 8e45ce0d
Add a Redis caching layer to the API with a 5-minute TTL.

**Assistant**: I'll add Redis caching middleware. Let me first check
your current dependencies and middleware setup.
  [Read] requirements.txt
  [Read] src/middleware/__init__.py
  [Write] src/middleware/cache.py
  [Edit] src/main.py — added cache middleware to app
```

This recovers the full original conversation -- user messages, assistant responses, and tool call summaries -- so Claude can recall exactly what happened during a past session.

```bash
# JSON output for programmatic use
memsearch transcript /path/to/session.jsonl --turn 6d6210b7 --json-output
```

## What the JSONL Looks Like

The transcript files are standard [JSON Lines](https://jsonlines.org/) -- one JSON object per line. Claude Code writes every message, tool call, and tool result as a separate line. Here is what the key message types look like (abbreviated for readability):

**User message** (human input):

```json
{
  "type": "user",
  "uuid": "6d6210b7-b841-4cd7-a97f-e3c8bb185d06",
  "parentUuid": "8404eaca-3926-4765-bcb9-6ca4befae466",
  "sessionId": "433f8bc3-a5a8-46a2-8285-71941dc96ad0",
  "timestamp": "2026-02-11T15:15:14.284Z",
  "message": {
    "role": "user",
    "content": "Add a Redis caching layer to the API with a 5-minute TTL."
  }
}
```

**Assistant message** (text response):

```json
{
  "type": "assistant",
  "uuid": "32da9357-1efe-4985-8a6e-4864bbf58951",
  "parentUuid": "d99f255c-6ac7-43fa-bcc8-c0dabc4c65cf",
  "sessionId": "433f8bc3-a5a8-46a2-8285-71941dc96ad0",
  "timestamp": "2026-02-11T15:15:36.510Z",
  "message": {
    "role": "assistant",
    "content": [
      {"type": "text", "text": "I'll add Redis caching middleware. Let me check your current setup."}
    ]
  }
}
```

**Assistant message** (tool call):

```json
{
  "type": "assistant",
  "uuid": "35fa9333-02ff-4b07-9036-ec0e3e290602",
  "parentUuid": "7ab167db-9a57-4f51-b5d3-eb63a2e6a5ad",
  "sessionId": "433f8bc3-a5a8-46a2-8285-71941dc96ad0",
  "timestamp": "2026-02-11T15:15:20.992Z",
  "message": {
    "role": "assistant",
    "content": [
      {
        "type": "tool_use",
        "id": "toolu_014CPfherKZMyYbbG5VT4dyX",
        "name": "Read",
        "input": {"file_path": "/path/to/src/middleware/__init__.py"}
      }
    ]
  }
}
```

**Tool result** (returned to assistant as a user message):

```json
{
  "type": "user",
  "uuid": "7dd5ac66-c848-4e39-952a-511c94ac66f2",
  "parentUuid": "35fa9333-02ff-4b07-9036-ec0e3e290602",
  "sessionId": "433f8bc3-a5a8-46a2-8285-71941dc96ad0",
  "timestamp": "2026-02-11T15:15:21.005Z",
  "message": {
    "role": "user",
    "content": [
      {
        "type": "tool_result",
        "tool_use_id": "toolu_014CPfherKZMyYbbG5VT4dyX",
        "content": "     1→from .logging import LoggingMiddleware\n     2→\n     3→..."
      }
    ]
  }
}
```

Key fields:

| Field | Description |
|-------|-------------|
| `type` | Message type: `user`, `assistant`, `progress`, `system`, `file-history-snapshot` |
| `uuid` | Unique ID for this message |
| `parentUuid` | ID of the previous message (forms a linked chain) |
| `sessionId` | Session ID (matches the JSONL filename) |
| `timestamp` | ISO 8601 timestamp |
| `message.content` | String for user text, or array of `text` / `tool_use` / `tool_result` blocks |

!!! tip "You don't need to parse JSONL manually"
    The `memsearch transcript` command handles all the parsing, truncation, and formatting. The JSONL structure is documented here for transparency -- most users will never need to read these files directly.

## Session Anchors

Each memory summary includes an HTML comment anchor that links the chunk back to its source session, enabling the L2-to-L3 drill-down:

```markdown
### 14:30
<!-- session:abc123def turn:ghi789jkl transcript:/home/user/.claude/projects/.../abc123def.jsonl -->
- Implemented caching system with Redis L1 and in-process LRU L2
- Fixed N+1 query issue in order-service using selectinload
- Decided to use Prometheus counters for cache hit/miss metrics
```

The anchor contains three fields:

| Field | Description |
|-------|-------------|
| `session` | Claude Code session ID (also the JSONL filename without extension) |
| `turn` | UUID of the last user turn in the session |
| `transcript` | Absolute path to the JSONL transcript file |

Claude extracts these fields from `memsearch expand --json-output` and uses them to call `memsearch transcript` for L3 access.
