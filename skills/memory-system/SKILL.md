---
name: memory-system
description: Use for reading/writing structured memory blocks, hash-chain audit, session logs. Replaces flat STATE.MD with typed JSON blocks and provenance chain. Load when starting/ending sessions, recording decisions, or verifying facts.
---

> Structured memory system with semantic typing and hash-chain provenance.
> Replaces STATE.MD with JSON blocks that agents read/write programmatically.

## ARCHITECTURE

```
.opencode/memory/
├── state.json              # Current focus, pending decisions (small, fast)
├── chain.jsonl             # Global hash-chain: every block append-only
├── blocks/
│   ├── 0001.fact.json      # Semantic-typed memory blocks
│   ├── 0002.logic.json
│   └── ...
└── sessions/
    ├── 2026-07-13.jsonl    # Session logs (append-only)
    └── ...
```

## MEMORY BLOCKS

Each block is a JSON file. Filename: `{id}.{register}.json`

```json
{
  "id": "0042",
  "register": "FACT",
  "prev_hash": "a3f2c...",
  "hash": "7b1d9...",
  "timestamp": "2026-07-14T12:00:00Z",
  "project": "DSA",
  "content": "DSA 3D attenuation verified: near=0.25 far=0.0025",
  "source": "bench run 2026-07-13",
  "confidence": 0.95,
  "tags": ["3d", "attenuation", "verified"],
  "links": ["0038", "0040"]
}
```

### Registers (semantic types)

| Register | Use for | Example |
|---|---|---|
| `SENSE` | Raw observations, user input, logs | "recvfrom captured 264/324 pages" |
| `FACT` | Verified claims with sources | "Cipher = stream cipher [source: taint trap]" |
| `LOGIC` | Inferences from FACT/SENSE | "xRead key needed for full decrypt" |
| `OPINION` | Subjective assessments | "Option (a) preferred for RE approach" |
| `ACTION` | Execution plans, next steps | "RE logiovfs xRead next session" |

### Hash-chain

Every block references the previous block's hash:
```
block_N.prev_hash = block_{N-1}.hash
block_N.hash = sha256(id + register + content + prev_hash)
```

This creates a tamper-evident chain. If any block is modified, all subsequent hashes break.

## HOW TO READ

### Current state
Read `state.json` — it has focus, pending decisions, session count.

### Find blocks by project
Search `blocks/` for files containing `"project": "DSA"`.

### Find blocks by register
Search `blocks/` for files named `*.fact.json` etc.

### Verify chain integrity
Walk chain.jsonl, recompute each hash, check it matches.

## HOW TO WRITE

### New block
1. Get next ID from chain.jsonl (last entry's id + 1)
2. Set prev_hash = last entry's hash
3. Compute hash = sha256(id + register + content + prev_hash)
4. Write block file: `blocks/{id}.{register}.json`
5. Append to chain.jsonl: `{"id": "0043", "hash": "...", "prev_hash": "...", "timestamp": "..."}`

### Update state.json
- Update `focus` array (max 3 items)
- Update `last_session`
- Append to `pending_decisions` or resolve existing

### Session log
Append to `sessions/{date}.jsonl`:
```json
{"time": "12:00", "event": "session_start", "agent": "sint-main"}
{"time": "12:01", "event": "block_written", "id": "0043", "register": "FACT"}
{"time": "12:30", "event": "session_end", "blocks_created": 3, "decisions_resolved": 1}
```

## SESSION PROTOCOL

### Start
1. Read `state.json`
2. Read last 10 entries from `chain.jsonl`
3. Load relevant project blocks
4. Present: "Last session: [summary]. Continuing with [focus]."

### During
- Write blocks for significant findings (FACT/LOGIC)
- Write ACTION blocks for plans
- Update state.json focus if task changes

### End
1. Write session summary to `sessions/{date}.jsonl`
2. Update `state.json` focus and pending decisions
3. Move verified patterns to appropriate skill files
4. Present: "Session saved. Next: [continuation prompt]."

## HASH COMPUTATION

For block hashing, use this Python snippet:
```python
import hashlib, json

def compute_hash(block_id, register, content, prev_hash):
    payload = f"{block_id}:{register}:{content}:{prev_hash}"
    return hashlib.sha256(payload.encode()).hexdigest()[:16]
```

Truncate to 16 hex chars (64 bits) — sufficient for tamper detection, compact.

## ANTI-PATTERNS

- Never modify an existing block — create a new one with `links` to the old
- Never delete blocks — mark as deprecated with a new block
- Never skip the hash chain — it's the provenance backbone
- Never put raw user input in FACT without source tagging
