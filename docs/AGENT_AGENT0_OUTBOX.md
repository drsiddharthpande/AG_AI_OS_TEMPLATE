# AGENT_AGENT0_OUTBOX.md — IDE Extension Outbox

## Purpose
Append-only log where IDE extensions (Agent 0A, 0B, 0C) write their suggestions and edit logs. Agent 1 reads this each run. Agent 2 treats this as advisory only unless tasks are promoted into the ACTIVE PHASE.

---

## How to Append
Use this format for each entry:
```
[AGENT_0X | YYYY-MM-DD] <summary of suggestion or edit>
Details: <optional details>
```

---

## Log

_No entries yet. Extensions should append below this line._

---

_Last read by Agent 1: 2026-02-23_
