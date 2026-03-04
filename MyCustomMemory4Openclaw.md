# Custom Long-Term Memory with Time-Decaying Resolution for OpenClaw

This document traces the full implementation of integrating
[memory_ondisk](https://github.com/Zenodia/memory_ondisk) into the OpenClaw
workspace to provide hierarchical, time-decaying long-term memory.

---

## 1. Background & Goal

**Goal:** Add custom long-term memory with _time-decaying resolution_ to OpenClaw,
based on the hierarchical summarisation approach from
[github.com/Zenodia/memory_ondisk](https://github.com/Zenodia/memory_ondisk).

**Time-decaying resolution** means older memories are progressively compressed:

- Recent days → full transcript detail
- Last week → LLM-compressed daily summary
- Last month → further compressed weekly summary
- Older → monthly distillation

---

## 2. Research: How OpenClaw Memory Works

From the in-repo docs (`docs/concepts/memory.md`):

- Memory = **plain Markdown files** in `~/.openclaw/workspace/`
- Two default layers:
  - `memory/YYYY-MM-DD.md` — daily append-only logs (temporal decay applies in search)
  - `MEMORY.md` — curated long-term memory (only loaded in private/main sessions)
- Indexed by the `memory-core` plugin via SQLite + optional vector embeddings
- Two agent-facing tools: `memory_search` (semantic) and `memory_get` (targeted read)

### Built-in temporal decay (search scoring)

OpenClaw has a `temporalDecay` option that applies an exponential multiplier to
search scores based on file age:

```
decayedScore = score × e^(-λ × ageInDays)    where λ = ln(2) / halfLifeDays
```

With `halfLifeDays: 30`:

| Age     | Score weight |
| ------- | ------------ |
| Today   | 100%         |
| 7 days  | ~84%         |
| 30 days | 50%          |
| 90 days | ~12.5%       |

**Key insight:** Non-dated files (e.g. `memory/weekly-2026-W09.md`) are exempt
from temporal decay and always rank normally. This is what makes weekly/monthly
summary files permanently accessible regardless of age.

---

## 3. Research: How memory_ondisk Works

From [github.com/Zenodia/memory_ondisk](https://github.com/Zenodia/memory_ondisk):

- **`MemoryHandler`** — stores conversation turns as plain text with grep-friendly
  anchors (`<<<TURN:0001>>>`, `>>>USER:0001>>>`, `>>>BOT:0001>>>`, etc.)
- **`MemoryOps`** — higher-level wrapper with `process_message()` and background
  async summarisation
- **Hierarchical periods:** `DIRECT` → `DAILY` → `WEEKLY` → `MONTHLY`
- Summaries are generated every N turns (default: 10) via NVIDIA LLM
  (`nvidia/llama-3.3-nemotron-super-49b-v1.5`)
- Non-blocking: background `asyncio` tasks generate summaries without blocking
  conversation flow
- LLM prompts are first-person, period-specific (different focus/length for
  daily vs weekly vs monthly)

---

## 4. Integration Design

### The bridge

`memory_ondisk` stores raw text; OpenClaw consumes Markdown. The integration
**adapts the hierarchical summarisation pattern to write OpenClaw-compatible
Markdown files**.

### Resolution hierarchy

```
memory/YYYY-MM-DD.md        ← full detail     (temporal decay applies)
       ↓  after 7 days
memory/weekly-YYYY-WNN.md   ← LLM summary     (non-dated → NO temporal decay)
       ↓  after 35 days
memory/monthly-YYYY-MM.md   ← distilled        (non-dated → NO temporal decay)
```

- Daily files decay naturally in search scoring as they age
- Weekly/monthly summaries stay fully accessible forever (non-dated = no decay)
- Together this gives "time-decaying resolution": older memories have less detail
  but are still findable

---

## 5. Files Created

All files live in the **OpenClaw workspace** (`~/.openclaw/workspace/`), not in
the application source tree. They are workspace skills — Python scripts invoked
directly, independent of the `pnpm` build pipeline.

### `~/.openclaw/workspace/skills/memory-ondisk/consolidate_memory.py`

The core consolidation script. Two phases:

**Phase 1 — daily → weekly**

- Finds all `memory/YYYY-MM-DD.md` files older than 7 days
- Groups them by ISO week
- Uses NVIDIA LLM to summarise each week's content
- Writes `memory/weekly-YYYY-WNN.md`
- Skips weeks already consolidated (idempotent)

**Phase 2 — weekly → monthly**

- Finds all `weekly-*.md` files where the week started more than 35 days ago
- Groups by calendar month
- Summarises into `memory/monthly-YYYY-MM.md`
- Skips months already consolidated

LLM model: `nvidia/llama-3.1-nemotron-nano-8b-v1` (lightweight, configurable via
`NVIDIA_MEMORY_MODEL` env var). Falls back to first-lines extraction if no API key.

### `~/.openclaw/workspace/skills/memory-ondisk/SKILL.md`

Agent-readable instructions. Tells the agent:

- What the skill does and when to use it
- How to invoke the script
- How to enable OpenClaw temporal decay config

### `~/.openclaw/workspace/skills/memory-ondisk/requirements.txt`

```
langchain-nvidia-ai-endpoints>=0.1.0
langchain-core>=0.1.0
```

### `~/.openclaw/workspace/HEARTBEAT.md` (updated)

Added a periodic memory consolidation task so the agent checks for old daily
files during heartbeats and runs the script automatically.

---

## 6. Installation

### Python dependencies (WSL)

The `pnpm` build only manages Node.js packages. Python deps are installed
separately via `pip` inside WSL:

```bash
# Option A: direct install
pip install -r ~/.openclaw/workspace/skills/memory-ondisk/requirements.txt

# Option B: isolated venv (recommended)
python3 -m venv ~/.openclaw/venv
source ~/.openclaw/venv/bin/activate
pip install -r ~/.openclaw/workspace/skills/memory-ondisk/requirements.txt
```

### NVIDIA API key

```bash
export NVIDIA_API_KEY="your-key-here"
# Or persist it:
echo 'export NVIDIA_API_KEY="your-key-here"' >> ~/.bashrc
```

### Enable OpenClaw temporal decay (run in WSL)

```bash
openclaw config set agents.defaults.memorySearch.query.hybrid.enabled true
openclaw config set agents.defaults.memorySearch.query.hybrid.temporalDecay.enabled true
openclaw config set agents.defaults.memorySearch.query.hybrid.temporalDecay.halfLifeDays 30
openclaw config set agents.defaults.memorySearch.query.hybrid.mmr.enabled true
```

Then restart the gateway:

```bash
pkill -9 -f openclaw-gateway 2>/dev/null || true
openclaw gateway run
```

Verify config was written:

```bash
openclaw config get agents.defaults.memorySearch.query.hybrid
openclaw config file   # shows path to config.json
```

---

## 7. Running the Script

```bash
# Basic run
python3 ~/.openclaw/workspace/skills/memory-ondisk/consolidate_memory.py

# With venv
source ~/.openclaw/venv/bin/activate
python3 ~/.openclaw/workspace/skills/memory-ondisk/consolidate_memory.py

# Custom workspace path
python3 ~/.openclaw/workspace/skills/memory-ondisk/consolidate_memory.py \
    --workspace /path/to/workspace

# Force-regenerate existing summaries
python3 ~/.openclaw/workspace/skills/memory-ondisk/consolidate_memory.py --force
```

---

## 8. Testing

### Step 1 — Verify script runs

```bash
ls -la ~/.openclaw/workspace/memory/
python3 ~/.openclaw/workspace/skills/memory-ondisk/consolidate_memory.py
ls -la ~/.openclaw/workspace/memory/weekly-*.md 2>/dev/null || echo "no weekly files yet"
ls -la ~/.openclaw/workspace/memory/monthly-*.md 2>/dev/null || echo "no monthly files yet"
```

### Step 2 — Verify temporal decay is active

```bash
openclaw config get agents.defaults.memorySearch.query.hybrid
openclaw memory status --deep
```

### Step 3 — Ask the agent

```
What do you remember from this week?
What do you remember from last month?
What's the oldest thing in your memory?
Search your memory for anything from last week and tell me which file it came from.
Do you have detailed notes or just a summary for [date older than 7 days]?
```

```bash
# Shows file paths and scores — old dated files should score lower
openclaw memory search "your topic here" --verbose
```

### Step 4 — Synthetic test (if no old files exist yet)

```bash
# Create a fake old daily file
echo "## Test entry
- Had a meeting about project X
- Decided to use Python for the memory script
- Key insight: time-decaying memory is useful" \
> ~/.openclaw/workspace/memory/2026-02-20.md

# Run consolidation — should produce weekly-2026-W08.md
python3 ~/.openclaw/workspace/skills/memory-ondisk/consolidate_memory.py

# Inspect the generated weekly summary
cat ~/.openclaw/workspace/memory/weekly-2026-W08.md
```

Then ask the agent:

```
What do you know about project X or the Python memory script decision?
```

It should find the answer via the weekly summary, not the original dated file.

---

## 9. Key Design Decisions

| Decision                                 | Rationale                                                              |
| ---------------------------------------- | ---------------------------------------------------------------------- |
| Write Markdown not raw text              | OpenClaw only indexes `.md` files                                      |
| Weekly files named `weekly-YYYY-WNN.md`  | Non-dated pattern → exempt from temporal decay                         |
| Monthly files named `monthly-YYYY-MM.md` | Same — always findable regardless of age                               |
| Lightweight default LLM model            | `nemotron-nano-8b` is cheaper; override with `NVIDIA_MEMORY_MODEL`     |
| Idempotent (skip existing files)         | Safe to run in heartbeat repeatedly; use `--force` to regenerate       |
| Fallback without API key                 | First-lines extraction so script always produces something useful      |
| Separate from pnpm build                 | Python skill scripts are workspace tools, not compiled into the binary |

---

## 10. References

- [memory_ondisk repo](https://github.com/Zenodia/memory_ondisk)
- OpenClaw memory docs: `docs/concepts/memory.md` (in this repo)
- OpenClaw memory CLI: `docs/cli/memory.md` (in this repo)
- OpenClaw memory research notes: `docs/experiments/research/memory.md` (in this repo)
