---
description: Run a GPT review using REVIEW_ call_tag discipline + Star Topology; save output to docs/reviews and append response_id trace to AGENT_API_TRACE.jsonl.
---

# /review — Architecture / Security Review

This workflow runs a structured OpenAI API review using the Star Topology pattern.

## Steps

1. **Confirm the review scope** with the user:
   - Which files to review?
   - What intensity? (`low` | `medium` | `high` | `xhigh`)
   - What focus? (security | architecture | code quality | all)

2. **Generate a `call_tag`** following the naming convention:
   ```
   REVIEW_<TOPIC>_<YYYYMMDD>
   Example: REVIEW_SECURITY_20260223
   ```

3. **Run the review script**:
   ```
   python ops/architect_review.py --files <file1> <file2> --intensity <level> --call_tag <tag>
   ```
   - If `ops/architect_review.py` doesn't exist, create it first using the standard template.

4. **Save output** to:
   ```
   docs/reviews/<call_tag>.md
   ```

5. **Append JSONL trace** to `docs/AGENT_API_TRACE.jsonl`:
   ```json
   {"timestamp": "...", "call_tag": "REVIEW_...", "model": "...", "effort": "...", "store": true, "response_id": "...", "previous_response_id": null, "script": "ops/architect_review.py", "inputs": [...], "summary": "..."}
   ```

6. **Reference response_id** inside `docs/CHANGE_LOG.md` under the active phase.

7. **Report to user**: Share the key findings and link to the saved review file.

## Star Topology (Multi-file reviews)
- Run individual reviews per file first (leaf calls).
- Then run a synthesis call referencing all leaf `response_id`s.
- Store synthesis output as `docs/reviews/<call_tag>_SYNTHESIS.md`.
