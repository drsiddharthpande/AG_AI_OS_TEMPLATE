# INFO_IDE_EXTENSIONS_ROLE.md — IDE Extension Contract

## Purpose
Defines the identification, default behavior, and logging rules for IDE extensions (Agent 0A, 0B, 0C) operating in this repository.

---

## Extension Identification

| ID     | Role                              | Example Tools              |
|--------|-----------------------------------|----------------------------|
| 0A     | Linter / Static Analyzer          | Pylance, ESLint, Ruff      |
| 0B     | Code Reviewer / Suggester         | GitHub Copilot, Codeium    |
| 0C     | Formatter / Auto-fixer            | Black, Prettier            |

---

## Default Behavior

1. **Advisory only by default** — extensions MUST NOT edit production files unless explicitly requested by the user.
2. **All suggestions** go to `docs/AGENT_AGENT0_OUTBOX.md` as append-only entries.
3. **Agent 1** reads the outbox each run and decides whether to promote suggestions into the active phase plan.
4. **Agent 2** treats the outbox as advisory only unless tasks are explicitly promoted into the ACTIVE PHASE.

---

## Editing Files (Only on Explicit User Request)

If an extension edits files, it MUST:
1. Append an `[EXT_EDIT_LOG]` block inside `docs/AGENT_HANDOVER.md` under the active/relevant phase.
2. Append a matching summary entry in `docs/AGENT_AGENT0_OUTBOX.md`.

### EXT_EDIT_LOG Format
```
[EXT_EDIT_LOG]
Executor: 0A | 0B | 0C
Files Changed: <list>
What/Why: <description>
Commands/Tests Run: <list>
Result: PASS | FAIL | NOT RUN
Rollback: <steps>
```

---

## Logging Rules

- Every extension suggestion or edit MUST be logged in `docs/AGENT_AGENT0_OUTBOX.md`.
- Log format: `[AGENT_0X | YYYY-MM-DD] <summary of suggestion or edit>`
- Do NOT log trivial formatting suggestions (e.g., trailing whitespace fixes) unless they are part of a larger change.

---

_Last updated: 2026-02-23 by /boot_
