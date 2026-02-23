# INFO_FILE_NAMES.md — Naming Conventions

## Purpose
Canonical file naming rules for this repository. All agents and extensions MUST follow these conventions.

---

## File Prefix Summary

| Prefix    | Usage                                                        | Location          |
|-----------|--------------------------------------------------------------|-------------------|
| `INFO_`   | Human-readable docs: rules, plans, architecture, explainers  | `docs/`           |
| `AGENT_`  | Agent operational files: state, handover, API trace          | `docs/`           |
| `DB_`     | App database files                                           | `data/db/`        |
| `MIG_`    | Schema migration files                                       | `data/db/migrations/` |
| `LIB_`    | Quick-access JSON libraries                                  | `data/lib/`       |
| `LIB_DB_` | Large queryable reference DBs                                | `data/libdb/`     |
| `CACHE_`  | Regeneratable derived artifacts                              | `data/cache/`     |
| `DEBUG_`  | Debug logs/dumps                                             | `logs/`           |
| `TEMP_`   | Throwaway scripts                                            | `temp/`           |
| `REVIEW_` | OpenAI review outputs (call_tag prefix)                      | `docs/reviews/`   |

---

## Naming Patterns

### Database Files
```
DB_<APP>_USER.sqlite          — user data
DB_<APP>_<PURPOSE>.sqlite     — other data
```

### Migration Files
```
MIG_<TARGET_DB>_<YYYYMMDD>_<SEQ>_<ACTION>.sql
Example: MIG_DB_APP_USER_20260223_001_ADD_SESSIONS.sql
```

### OpenAI Review Calls
```
call_tag: REVIEW_<TOPIC>_<YYYYMMDD>
output file: docs/reviews/REVIEW_<TOPIC>_<YYYYMMDD>.md
```

---

## Phase Header Standard (AGENT_HANDOVER.md)

All phase headers MUST follow this format:
```
## PHASE_<N> — <NAME>  [STATUS: <ACTIVE|COMPLETED|BLOCKED>]  [EXECUTOR: <AGENT_1|AGENT_2>]
```

---

_Last updated: 2026-02-23 by /boot_
