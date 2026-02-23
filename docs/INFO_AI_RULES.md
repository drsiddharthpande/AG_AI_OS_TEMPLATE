# INFO_AI_RULES.md — Workspace AI Doctrine

## Purpose
This file defines the AI operating rules for this repository. All agents (Agent 1, Agent 2, and IDE extensions) MUST read this file at the start of every run.

## Workspace Info
- **Repo**: AG_AI_OS_TEMPLATE
- **Role**: Template repository for all new projects using the AI OS architecture.
- **Owner**: Dr. Siddharth Pande

---

## Core Rules (Inherited from Global)

All global AI OS rules apply to this repo. See the GLOBAL AI OS RULES in the agent's system memory for the full doctrine. This file captures any **repo-specific overrides** below.

---

## Repo Workflow Overrides

_None defined yet. Add repo-specific overrides here when needed._

---

## Phase Execution Policy

- Default policy applies: Agent 2 executes ONLY the ACTIVE phase as defined in `AGENT_STATUS.json`.
- Phase 4 (Cleanup Notes) is NOT executable work; it is documentation only and executes ONLY during `/ship`.

---

## Agent Roles in This Repo

| Agent    | Role                                                                 |
|----------|----------------------------------------------------------------------|
| Agent 1  | Architect: plans phases, writes/updates INFO_ + AGENT_ docs         |
| Agent 2  | Coder: implements ACTIVE phase, runs tests, updates AGENT_ files     |
| Agent 0A/0B/0C | IDE extensions: advisory by default; log to AGENT_AGENT0_OUTBOX.md |

---

## File & Folder Policy

Follows the standard AI OS structure:
```
docs/     — all AI OS control + info files
ops/      — review/automation scripts
src/      — application source code
data/     — db, lib, libdb, cache
logs/     — log files
temp/     — throwaway scripts (TEMP_*)
.agents/  — workflow files
```

---

_Last updated: 2026-02-23 by /boot_
