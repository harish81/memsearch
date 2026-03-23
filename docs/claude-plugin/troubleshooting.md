# Troubleshooting

The plugin provides several observability mechanisms, from always-on status lines to opt-in debug logging. Work from the top down -- most issues are resolved by the first two sections.

| Mechanism | Always On? | What You See | Best For |
|-----------|-----------|--------------|----------|
| [SessionStart status line](#1-sessionstart-status-line) | Yes | `[memsearch v0.1.11] embedding: openai/... | milvus: ...` | Config errors, version checks |
| [Debug mode](#2-debug-mode-debug) | No | Full hook JSON in `~/.claude/logs/` | Hook execution, additionalContext |
| [CLI diagnostic commands](#3-cli-diagnostic-commands) | Manual | Config, index stats, search results | Config verification, search testing |
| [Watch process](#4-watch-process) | Yes (background) | PID file at `.memsearch/.watch.pid` | Index sync issues |
| [Skill execution](#5-skill-execution-progressive-disclosure) | Yes (in UI) | Skill invocation + Bash tool calls | Memory recall debugging |
| [Memory files](#6-memory-files) | Yes | `.memsearch/memory/YYYY-MM-DD.md` | Stop hook, summary quality |
| [First-time model download](#7-first-time-model-download) | First run only | Model download progress | First session hangs |

---

## 1. SessionStart Status Line

Every session starts with a status line in `systemMessage`. This is the first thing to check when something seems wrong.

Hooks communicate with Claude Code by returning JSON. Two key fields:

- **`systemMessage`** — A **visible** info line shown in the terminal, like a status bar.
- **`additionalContext`** — **Invisible** to the user; injected into Claude's context silently. Only appears in debug logs (`claude --debug`).

Here is what a session looks like with the plugin installed:

```
   ✻
   |
  ▟█▙     Claude Code v2.x.x
▐▛███▜▌   Model · Plan
▝▜█████▛▘  ~/my-project
 ▘▘ ▝▝
 ⎿  SessionStart:startup says: [memsearch v0.1.11]        ← systemMessage
    embedding: openai/text-embedding-3-small | milvus:       (SessionStart hook)
    ~/.memsearch/milvus.db

❯ How does the caching layer work?

 ⎿  UserPromptSubmit says: [memsearch] Memory available    ← systemMessage
                                                             (UserPromptSubmit hook)
✶ Thinking…
```

The SessionStart hook also loads the 2 most recent daily logs as `additionalContext` — Claude reads this silently to decide when to invoke the memory-recall skill, but you won't see it in the terminal.

**Normal:**

```
[memsearch v0.1.11] embedding: openai/text-embedding-3-small | milvus: ~/.memsearch/milvus.db
```

**API key missing:**

```
[memsearch v0.1.11] embedding: openai/text-embedding-3-small | milvus: ~/.memsearch/milvus.db | ERROR: OPENAI_API_KEY not set — memory search disabled
```

**Update available:**

```
[memsearch v0.1.11] embedding: openai/text-embedding-3-small | milvus: ~/.memsearch/milvus.db | UPDATE: v0.1.12 available
```

### "ERROR: \<KEY\> not set -- memory search disabled"

The plugin checks for the required API key at session start. If missing, memory recording still writes `.md` files, but semantic search and indexing are disabled.

| Provider | Required environment variable |
|----------|------------------------------|
| `onnx` (ccplugin default) | None (local, CPU) |
| `openai` (Python API default) | `OPENAI_API_KEY` |
| `google` | `GOOGLE_API_KEY` |
| `voyage` | `VOYAGE_API_KEY` |
| `ollama` | None (local) |
| `local` | None (local) |

**Fix:** export the key for your configured provider:

```bash
# The ccplugin defaults to onnx (no key needed). If you use OpenAI:
export OPENAI_API_KEY="sk-..."
memsearch config set embedding.provider openai

# Or switch to free local embedding (no key needed)
memsearch config set embedding.provider onnx
```

To make it permanent, add the export to your `~/.bashrc`, `~/.zshrc`, or equivalent.

### "UPDATE: v0.x.x available"

The plugin checks PyPI at session start (2s timeout) and shows this hint when a newer version exists. The hint now includes the exact upgrade command for your installation method:

```
UPDATE: v0.1.15 available — run: pip install --upgrade memsearch
UPDATE: v0.1.15 available — run: uv tool upgrade memsearch
```

The plugin auto-detects how memsearch was installed:

| Install method | Upgrade command shown |
|---|---|
| `pip install memsearch` | `pip install --upgrade memsearch` |
| `uv tool install memsearch` | `uv tool upgrade memsearch` |
| `uvx` (auto) | `uvx --upgrade memsearch --version` |

!!! note
    `uvx` users get automatic upgrades -- the plugin runs `uvx --upgrade` on every bootstrap. The `UPDATE` hint primarily helps `pip`/`uv tool` users who have no automatic update mechanism.

---

## 2. Debug Mode (`--debug`)

Claude Code's `--debug` flag enables verbose logging for all hooks.

**Start Claude Code with debug logging:**

```bash
claude --debug
```

**Log location:** `~/.claude/logs/` (timestamped files)

**What to look for in the logs:**

```bash
# See all hook outputs (additionalContext, systemMessage, etc.)
grep -A 5 'hook' ~/.claude/logs/*.log

# Check SessionStart output specifically
grep -A 10 'SessionStart' ~/.claude/logs/*.log

# See what additionalContext was injected
grep 'additionalContext' ~/.claude/logs/*.log
```

Each hook outputs JSON to stdout. In debug mode, you can see the raw JSON -- useful for verifying that `additionalContext` (cold-start memories) and `systemMessage` (status line) are being returned correctly.

---

## 3. CLI Diagnostic Commands

These commands work outside of Claude Code -- run them directly in your terminal.

**Verify resolved configuration:**

```bash
memsearch config list --resolved
```

Shows the effective config after merging all layers (defaults → `~/.memsearch/config.toml` → `.memsearch.toml` → env vars). Check that `embedding.provider`, `embedding.model`, and `milvus.uri` are what you expect.

**Check index health:**

```bash
memsearch stats
```

Shows collection name, chunk count, and embedding dimensions. If the count is 0 or unexpectedly low, re-index:

```bash
memsearch index .memsearch/memory/ --force
```

**Test search manually:**

```bash
memsearch search "your query here" --top-k 5
```

If this returns no results but `stats` shows chunks exist, the issue is likely with embeddings (wrong API key, different model than what was used for indexing).

**Expand a specific chunk:**

```bash
memsearch expand <chunk_hash>
```

Retrieves the full markdown section surrounding a chunk, including session anchors. Useful for verifying that the L2 expand layer works.

**Trace back to original conversation:**

```bash
memsearch transcript /path/to/session.jsonl
memsearch transcript /path/to/session.jsonl --turn <uuid> --context 3
```

Lists all turns or drills into a specific turn. The transcript path is embedded in session anchors (the `<!-- session:... transcript:... -->` HTML comments in memory files).

---

## 4. Watch Process

The `memsearch watch` singleton runs in the background, auto-re-indexing when memory files change.

**PID file location:** `.memsearch/.watch.pid`

**Check if it's running:**

```bash
cat .memsearch/.watch.pid && kill -0 $(cat .memsearch/.watch.pid) 2>/dev/null && echo "running" || echo "not running"
```

**Restart manually:**

```bash
# Kill existing watch (if any) and start fresh
kill $(cat .memsearch/.watch.pid) 2>/dev/null; rm -f .memsearch/.watch.pid
memsearch watch .memsearch/memory/ &
echo $! > .memsearch/.watch.pid
```

**Sweep for orphaned processes:**

```bash
pgrep -f "memsearch watch" && echo "found orphans" || echo "clean"
```

The watch process is started by `SessionStart` and stopped by `SessionEnd`. If Claude Code crashes or is killed with SIGKILL, the `SessionEnd` hook won't fire and the process may become orphaned. The next `SessionStart` always stops any existing watch before starting a new one.

> **Note:** Milvus Lite does not support concurrent access, so the plugin falls back to one-time indexing at session start instead of a persistent watcher. For real-time indexing, use [Milvus Server or Zilliz Cloud](../getting-started.md#milvus-backends).

---

## 5. Skill Execution & Progressive Disclosure

When Claude decides past context is needed, it invokes the `memory-recall` skill. You can observe the three progressive disclosure layers in the Claude Code UI:

```
╭─ memory-recall                                          ─╮
│                                                          │
│  ● Searching for relevant memories...                    │
│                                                          │
│  $ memsearch search "redis caching" --top-k 5           │
│    → 3 results found                                     │
│                                                          │
│  $ memsearch expand 7a3f9b21e4c08d56                     │
│    → Full section from 2026-02-10.md                     │
│                                                          │
│  Summary: Found relevant context about Redis caching...  │
│                                                          │
╰──────────────────────────────────────────────────────────╯
```

The skill runs in a forked subagent (`context: fork`), so its intermediate work does not pollute your main conversation context.

**Force a skill invocation for debugging:**

```
/memory-recall <your query>
```

This manually triggers the skill, bypassing Claude's judgment about whether memory is needed.

**Skill not triggering automatically?** Possible reasons:

- Claude judged that the question doesn't need historical context -- this is by design
- The `UserPromptSubmit` hint (`[memsearch] Memory available`) didn't fire -- check that the prompt is ≥ 10 characters
- `memsearch` is not installed or not in PATH -- the `UserPromptSubmit` hook returns `{}` when `MEMSEARCH_CMD` is empty

---

## 6. Memory Files

All memories are stored as plain markdown in `.memsearch/memory/`.

**Directory location:** `.memsearch/memory/` (project-scoped)

**File format:** One file per day, named `YYYY-MM-DD.md`:

```markdown
## Session 14:30

### 14:30
<!-- session:abc123def turn:ghi789jkl transcript:/home/user/.claude/projects/.../abc123def.jsonl -->
- Implemented caching system with Redis L1 and in-process LRU L2
- Fixed N+1 query issue in order-service using selectinload
```

**Verify the Stop hook is working:**

```bash
# Check if today's file exists and has content
cat .memsearch/memory/$(date +%Y-%m-%d).md

# Check if recent sessions have summaries (not just headings)
tail -20 .memsearch/memory/$(date +%Y-%m-%d).md
```

If you see `## Session HH:MM` headings but no `### HH:MM` sub-headings with bullet points underneath, the Stop hook is not completing successfully. Common causes:

- `claude` CLI not found -- the Stop hook calls `claude -p --model haiku` to summarize
- API key missing -- the Stop hook skips summarization when the embedding provider key is not set
- Transcript too short -- sessions with fewer than 3 JSONL lines are skipped

---

## Common Issues

| Symptom | Check | Section |
|---------|-------|---------|
| "ERROR: \<KEY\> not set" in status line | Export the required API key for your provider | [§1](#1-sessionstart-status-line) |
| "UPDATE: v0.x.x available" in status line | Upgrade memsearch | [§1](#1-sessionstart-status-line) |
| Search returns no results | Run `memsearch stats` and `memsearch search` manually | [§3](#3-cli-diagnostic-commands) |
| New memories not being indexed | Check watch process is running | [§4](#4-watch-process) |
| Claude never invokes memory recall | Try `/memory-recall <query>` manually | [§5](#5-skill-execution-progressive-disclosure) |
| First session hangs or memory search unavailable | The ONNX model (~558 MB) is downloading in the background. See [§7](#7-first-time-model-download) | [§7](#7-first-time-model-download) |
| Session summaries missing from memory files | Check `claude` CLI is available and API key is set | [§6](#6-memory-files) |

---

## 7. First-Time Model Download

The plugin defaults to the **ONNX bge-m3 int8** embedding model, which runs locally on CPU with no API key required. On the very first session, this model (~558 MB) needs to be downloaded from HuggingFace Hub. The download runs in the background during session start, and during this time memory search may be temporarily unavailable.

**Symptoms:**

- First session appears to hang after sending a prompt (the background download is blocking Milvus Lite)
- `[memsearch] Memory available` hint appears but memory recall returns no results
- `memsearch search` or `memsearch index` commands hang on first run

**Pre-download the model manually:**

```bash
# This triggers the model download without starting a Claude session
uvx --from 'memsearch[onnx]' memsearch search --provider onnx "warmup" 2>/dev/null || true
```

**If the download is slow or stuck:**

HuggingFace Hub may be slow or inaccessible from certain networks. Set the `HF_ENDPOINT` environment variable to use a mirror:

```bash
export HF_ENDPOINT=https://hf-mirror.com
uvx --from 'memsearch[onnx]' memsearch search --provider onnx "warmup" 2>/dev/null || true
```

To make this permanent, add `export HF_ENDPOINT=https://hf-mirror.com` to your `~/.bashrc` or `~/.zshrc`.

**After the first download:** The model is cached locally at `~/.cache/huggingface/hub/` and all subsequent sessions load it instantly from disk with no network access required.
