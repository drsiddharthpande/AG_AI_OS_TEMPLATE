# 🛑 INFO_AI_RULES.md — PROJECT RULES (DO NOT IGNORE)
This file contains **critical rules** that must be followed by all AI Agents working on this project/workspace.

> **Scope + Relationship to Global Rules / Workflows**
- This is the **workspace doctrine** (human-readable + agent-enforceable).
- Your IDE-level **Global Rules** should force agents to:
  1) follow `docs/AGENT_*.md` protocol files, and  
  2) follow this file for OpenAI endpoint usage and review standards.
- This file is designed to work with your naming conventions:
  - **INFO_*** = human-readable plans/rules/changes you show to devs
  - **AGENT_*** = agent baton / phase state / agent logs
  - **DB_*** = actual data stores
  - **MIG_<TARGET_DB>_*** = migrations
  - **LIB_*** = small fast-access JSON libraries (quick lookup)
  - **LIB_DB_*** = huge libraries stored as .db for UI retrieval
  - **External API call naming prefixes** (request “names”): `REVIEW_…`, `PLAN_…`, etc.
  - **API trace log**: `docs/AGENT_API_TRACE.jsonl`

> **Important: Antigravity reserved names**
- Don’t create files like `/boot` (reserved). Use: `docs/AGENT_BOOT.md` or `docs/AGENT_WORKFLOW.md`.

---

## 0. AGENT 1 OPERATING STRATEGIES (TASK-LEVEL) — WHERE “STRATEGY A/B” LIVES
You asked: *“Where have we written Strategy A and Strategy B for Agent 1?”*  
Answer: right here — **task-level Strategy A/B** (separate from the review strategies below).

### Strategy A (Low-Risk / Default Execution)
**Use when:** UI tweaks, docs updates, small logic changes, single-file edits, refactors < ~50 LOC, non-security changes.  
**Agent 1 behavior:**
1. Read the relevant `AGENT_*.md` baton files and `docs/CHANGE_LOG.md`.
2. Produce a **clear Implementation Plan** in phases if needed (even 1 phase).
3. Hand off Phase 1 to Agent 2.
4. Validate with basic reasoning + quick sanity checks.

### Strategy B (High-Risk / Mandatory Deep Review Loop)
**Use when:** auth/security, database schema, migrations, payment flows, permission systems, refactors > ~50 LOC, performance-critical code, “I’m stuck”, or any high-impact change.  
**Agent 1 behavior (inviolable):**
1. Draft plan + threat model / failure modes (as relevant).
2. Run **GPT-5.2 xHigh review** using Responses API (see Section 1) — standard or Star Topology depending on size.
3. Update:
   - `docs/AGENT_HANDOVER.md` (human baton)
   - `docs/AGENT_STATUS.json` (machine state)
   - `docs/AGENT_DECISIONS.md` (decision record)
   - `docs/CHANGE_LOG.md` (what changed / why / when)
4. Only then unlock Phase 1 for Agent 2.

---

## 1. GPT-5.2 xHigh Review Configuration (Responses API)
**GOAL:** Use the stateful Responses API with gpt-5.2 to ensure deep reasoning for Architectural and code reviews.  
**Status:** MANDATORY / INVIOLABLE  
**Context:** When running an Architectural Review script (e.g., `ops/architect_review.py`).

### Strategy A: Standard Review (Single Context)
**The Rule:**  
You MUST use the `/v1/responses` endpoint with this specific configuration:
```python
# ✅ CORRECT - Responses API Pattern
response = client.responses.create(
    model="gpt-5.2",                 # Standard Enterprise Model ($1.75/1M)
    reasoning={"effort": "xhigh"},   # MUST be a nested object
    input="[VERBOSITY: HIGH] ...",   # Put verbosity instructions in the prompt
    store=True                       # Mandatory for caching/statefulness
)

## Why? (Responses API + xHigh)

- **Deep Reasoning:** `xhigh` triggers the deepest “System 2” thinking chain, catching edge cases standard reviews miss.  
- **Star Topology:** Using `store=True` allows forking reviews using `previous_response_id`.  
  Reference: OpenAI Responses API docs — https://platform.openai.com/docs/api-reference/responses

## Penalty
Using `chat.completions` **or** flat parameters (e.g., `reasoning_effort="xhigh"`) triggers immediate failure.

> **Note (official parameters)**
- In the Responses API, `reasoning` is a **nested object** (e.g., `reasoning: {"effort": "xhigh"}`).
- `store` defaults to `true` unless changed.  
  Reference: https://platform.openai.com/docs/api-reference/responses

---

### Strategy B: Star Topology “Map-Reduce” (Massive Files)

**Context:** For massive files, standard xHigh reasoning degrades as context fills. Use this “Global Context Root + Local Chunk Branches” strategy.

**The Concept:**
1. **The Root (Map):** Holds “Architectural Standards” and “Global Project Context”. (Full cost ~2k tokens). **Do NOT put code here.**
2. **The Branches (Reduce):** Split file into logical chunks (e.g., 200–300 lines). Each chunk is a separate xHigh branch linking to Root. (Enables cached-context savings via `previous_response_id`.)

## Implementation Pattern (ops/large_file_review.py):

import os
import math
import asyncio
from openai import AsyncOpenAI

# 🟢 CONFIG
API_KEY = os.getenv("OPENAI_API_KEY")
client = AsyncOpenAI(api_key=API_KEY)
MODEL = "gpt-5.2"

def chunk_file_by_lines(file_path, chunk_size=300):
    """Splits a large file into smaller overlapping chunks."""
    with open(file_path, 'r') as f:
        lines = f.readlines()
    
    chunks = []
    total_lines = len(lines)
    for i in range(0, total_lines, chunk_size):
        # Create chunk with strict line numbers for reference
        segment = "".join(lines[i : i + chunk_size])
        chunks.append({
            "index": i // chunk_size + 1,
            "start_line": i + 1,
            "end_line": min(i + chunk_size, total_lines),
            "content": segment
        })
    return chunks

async def review_large_file(file_path):
    print(f"🚀 Starting Large File Review: {file_path}")
    
    # ---------------------------------------------------------
    # STEP 1: ESTABLISH ROOT (Policy & Global Context)
    # ---------------------------------------------------------
    print("🌱 Creating Root Context...")
    
    # We include the 'File Skeleton' (mocked here) so branches aren't blind
    root_prompt = """
    You are a Security Architect. We are reviewing a LARGE legacy file.
    
    ## GLOBAL CONTEXT
    - This file handles: User Authentication & Session Management.
    - Critical Rules: No raw SQL, no hardcoded secrets, verify all JWT tokens.
    
    ## YOUR JOB
    I will send you a *segment* of this file. 
    Review it in isolation using xHigh reasoning.
    
    ## OUTPUT FORMAT (JSON)
    {
      "critical_issues": [],
      "refactoring_suggestions": []
    }
    """
    
    root_response = await client.responses.create(
        model=MODEL,
        input=root_prompt,
        store=True  # ✅ MANDATORY for Star Topology
    )
    ROOT_ID = root_response.id
    print(f"✅ Root Established: {ROOT_ID}")

    # ---------------------------------------------------------
    # STEP 2: FORK BRANCHES (Process Chunks in Parallel)
    # ---------------------------------------------------------
    chunks = chunk_file_by_lines(file_path, chunk_size=200) # Small chunks = Better reasoning
    print(f"📂 Split file into {len(chunks)} branches.")
    
    semaphore = asyncio.Semaphore(5) # Rate limit safety

    async def process_chunk(chunk):
        async with semaphore:
            print(f"   👉 Forking Branch {chunk['index']} (Lines {chunk['start_line']}-{chunk['end_line']})...")
            
            response = await client.responses.create(
                model=MODEL,
                # 🟢 USE xHigh REASONING
                reasoning={"effort": "xhigh"},
                
                # 🟢 STAR TOPOLOGY (Link to Root)
                previous_response_id=ROOT_ID,
                
                input=f"""
                ## SEGMENT REVIEW
                Lines: {chunk['start_line']} to {chunk['end_line']}
                
                ```python
                {chunk['content']}
                ```
                """
            )
            return {
                "chunk": chunk['index'],
                "lines": f"{chunk['start_line']}-{chunk['end_line']}",
                "analysis": response.output_text
            }

    # Run all branches
    results = await asyncio.gather(*[process_chunk(c) for c in chunks])

    # ---------------------------------------------------------
    # STEP 3: AGGREGATE RESULTS
    # ---------------------------------------------------------
    print("\n🏁 REVIEW COMPLETE. Findings:")
    for res in results:
        print(f"\n--- Segment {res['lines']} ---")
        print(res['analysis'][:300] + "...") # Preview

if __name__ == "__main__":
    asyncio.run(review_large_file("src/huge_legacy_controller.py"))


## Critical Advice for “Blind Branches” (Star Topology)

Chunk 2 cannot see Chunk 1. Limit hallucinations by adding this to the **Root Prompt**:

> “You are reviewing a fragment of a file. Do not flag ‘undefined variables’ or ‘missing imports’ unless you are certain they are not defined elsewhere. Focus strictly on logic errors, security flaws, and syntax bad practices within the code provided.”

### Important clarification (pricing + caching)

- GPT-5.2 has a separate **cached input** price tier.  
  https://platform.openai.com/docs/models/gpt-5.2

- `previous_response_id` is the **conversation state** mechanism (multi-turn context) supported by the Responses API.  
  https://platform.openai.com/docs/api-reference/responses

- Prompt caching is additionally controlled via fields like `prompt_cache_key` / `prompt_cache_retention`  
  (don’t assume every `previous_response_id` automatically bills at cached rates).  
  https://platform.openai.com/docs/api-reference/responses

---

## 2. GPT-5.2 Token Limits & Pricing (Tier 1)

**Reference Date:** January 2026

### Token Limits

| Parameter | Value |
| --- | --- |
| **Context Window** | 400,000 tokens |
| **Max Output Tokens** | 128,000 tokens |
| **Tokens Per Day (TPD)** | 900,000 |

> **Correction / Reality check:**  
> Rate limits (RPM/TPM/Batch queue) are tier- and project-dependent; don’t treat “TPD 900,000” as universal. Check your project’s limits page.  
> https://platform.openai.com/docs/models/gpt-5.2

### Pricing (The “Star Topology” Advantage)

| Token Type | Cost per Million | Condition |
| --- | --- | --- |
| **Standard Input** | $1.75 | First call (Root Context) |
| **Cached Input** | $0.175 | 90% Discount (via `previous_response_id`) |
| **Output Tokens** | $14.00 | Generated reasoning + code |
| **Pro Input** | $21.00 | ⚠️ DANGER: `gpt-5.2-pro` model only |

> **Source of truth (GPT-5.2 pricing + cached input):**  
> https://platform.openai.com/docs/models/gpt-5.2

### API Config & Reasoning Tiers

| API Config | Use Case | Approx Cost/Request |
| --- | --- | --- |
| `model="gpt-5.2"` `reasoning={"effort": "none"}` | Quick Q&A | ~$0.01 |
| `model="gpt-5.2"` `reasoning={"effort": "medium"}` | Logic/Refactoring | ~$0.15 |
| `model="gpt-5.2"` `reasoning={"effort": "xhigh"}` | Architectural Reviews | ~$1.00 - $3.00 |
| `model="gpt-5.2-pro"` `reasoning={"effort": "xhigh"}` | Emergency / Security | **$20.00+** |

---

## 3. GPT-5.2 Temperature & Reasoning Rules (CRITICAL)

**Status:** MANDATORY — API will reject invalid combinations

### 1. Parameter Syntax (Nested vs Flat)

The Responses API requires the Nested Object syntax.

- ✅ **CORRECT:** `reasoning={"effort": "xhigh"}`
- ❌ **WRONG:** `reasoning_effort="xhigh"` (Legacy flat param = 400 Error)

### 2. Temperature Lock

When reasoning is enabled, the model manages its own entropy. You cannot set temperature manually.

> **Evidence:** In the latest model guidance, temperature/top_p are only supported when reasoning is disabled / `reasoning.effort="none"`.  
> https://platform.openai.com/docs/guides/latest-model?utm_source=chatgpt.com

#### Scenario A: Reasoning Enabled

```python
# ✅ CORRECT - No Temperature Set
response = client.responses.create(
    model="gpt-5.2",
    reasoning={"effort": "xhigh"},  # or "medium"
    input="..."
    # temperature=1.0  <-- DO NOT INCLUDE THIS.
    # The API locks strictness automatically.
)

#### Scenario A: Reasoning Disabled
# ✅ CORRECT - Temperature Allowed
response = client.responses.create(
    model="gpt-5.2",
    # reasoning param omitted entirely
    temperature=0.5,
    input="..."
)

Implementation Pattern: 
# Correct pattern for our codebase (ops/architect_review.py)
api_params = {
    "model": "gpt-5.2",
    "input": prompt_text,
    "store": True
}

if intensity in ["medium", "high", "xhigh"]:
    # Reasoning Mode: Add nested reasoning, NO temperature
    api_params["reasoning"] = {"effort": intensity}
else:
    # Standard Mode: Add temperature
    api_params["temperature"] = 0.5

await client.responses.create(**api_params)

## 4. External OpenAI Call Tag Discipline (MANDATORY)

**Hard rule:** Every external OpenAI call (review, research, planning, debugging) MUST use a `call_tag` that starts with:

- `REVIEW_...`

### 4.1 Canonical `call_tag` format
Use one consistent pattern (adjust the middle fields as needed, but keep `REVIEW_` and timestamp):

- `REVIEW_<PROJECT>_<SCOPE>_<TARGET>_<INTENT>_<EFFORT>_<YYYYMMDD_HHMMSS>_IST`

**Examples**
- `REVIEW_RESEARCHWALA_ARCH_STUDY_DESIGN_XHIGH_20260125_231500_IST`
- `REVIEW_GLENOID_PLAN_ENROLLMENT_FLOW_MEDIUM_20260125_231700_IST`
- `REVIEW_APP_DEBUG_LOGIN_ISSUE_HIGH_20260125_232000_IST`

### 4.2 Where the `call_tag` must appear (NO EXCEPTIONS)
1) **In the request metadata** (Responses API `metadata`) as:
   - `call_tag: "REVIEW_..."`

2) **In the saved output filename**, using the same tag:
   - `docs/reviews/<call_tag>.md` (human-readable)
   - or `docs/reviews/<call_tag>.json` (structured output)

3) **In the trace log** (append-only):
   - `docs/AGENT_API_TRACE.jsonl` MUST include the same `call_tag`.

### 4.3 Star Topology naming (Root + Branches)
- Root: `REVIEW_<...>_ROOT_<YYYYMMDD_HHMMSS>_IST`
- Branches:
  - `REVIEW_<...>_B01_L0001-0300_<YYYYMMDD_HHMMSS>_IST`
  - `REVIEW_<...>_B02_L0301-0600_<YYYYMMDD_HHMMSS>_IST`
  - etc.

**No exceptions:** Do NOT introduce `PLAN_` or `RESEARCH_` prefixes in this system.  
All external calls remain under `REVIEW_` for uniform traceability + caching.

---

## 4.4 docs/AGENT_API_TRACE.jsonl (Required)

Append **one JSONL line per external request** containing:

- timestamp (ISO)
- call_tag (must start with `REVIEW_...`)
- response_id (OpenAI response id)
- previous_response_id (if used)
- model, reasoning.effort
- file targets (if review)
- agent (Architect vs Coder)
- success/failure + error summary

---

## 5. Conversation State vs Stored Responses vs ZDR (MANDATORY)

- Responses API supports `previous_response_id` for multi-turn state:
  https://platform.openai.com/docs/api-reference/responses

- `store` defaults to true; set explicitly for clarity:
  https://platform.openai.com/docs/api-reference/responses

- If your org uses Zero Data Retention (ZDR) or you set `store=false`, you may need
  `reasoning.encrypted_content` to preserve reasoning across turns statelessly:
  https://platform.openai.com/docs/api-reference/responses

- Retention / data handling is governed by OpenAI data controls policies — treat those as the source of truth:
  https://platform.openai.com/docs/guides/your-data



## 8. STOP REPEATING API KEY ERRORS (CRITICAL)
**Status:** MANDATORY — This error has occurred 10+ times

### The Problem
AI agents keep writing code like this:
```python
# ❌ WRONG - Will fail if env var not set
client = OpenAI(api_key=os.environ.get("OPENAI_API_KEY"))
The Solution
ALWAYS use the project's LLMClient class which has the API key embedded:

# ✅ CORRECT - Uses project's LLMClient with embedded key
from llm_client import LLMClient
llm_wrapper = LLMClient()
client = llm_wrapper.client  # Get underlying OpenAI client
# Then use client.chat.completions.create(...)
Why LLMClient?
Has fallback API key hardcoded (line 38 of llm_client.py)

Handles timeouts properly (60s configured)

Has retry logic built in

Security note (non-negotiable):
Even if you use LLMClient for reliability, never commit real keys to public repos. OpenAI guidance is to keep keys secret and out of source control.
Reference: https://platform.openai.com/docs/api-reference/introduction?utm_source=chatgpt.com

If LLMClient truly hardcodes a production key, treat the repo as private and schedule migration to a secrets manager / project key store.

If You MUST Use Raw OpenAI
# ✅ Acceptable fallback - load from secrets file
import json
with open("secrets.json") as f:
    api_key = json.load(f).get("OPENAI_API_KEY")
client = OpenAI(api_key=api_key)
NEVER use os.environ.get("OPENAI_API_KEY") alone — it will fail!


9. GPT-5.2 xHigh Timeout Prevention (CRITICAL)

Status: MANDATORY - This timeout has occurred 8+ times

The Problem

GPT-5.2 with reasoning_effort="xhigh" takes 30-120 seconds to respond. Default timeouts cause failures:

ERROR: Request timed out.

The Solution

When calling GPT-5.2 with xhigh reasoning, configure extended timeouts:

from openai import OpenAI
# ✅ CORRECT - Extended timeout for xhigh reasoning
client = OpenAI(
    api_key=api_key,
    timeout=180.0  # 3 minutes for xhigh
)
response = client.chat.completions.create(
    model="gpt-5.2",
    reasoning_effort="xhigh",
    messages=[...]
)

Timeout Guidelines by Reasoning Effort
reasoning_effort	Recommended Timeout	Typical Response Time
"none"	30s (default)	1-5 seconds
"medium"	60s	5-20 seconds
"high"	120s	15-45 seconds
"xhigh"	240-600s (4-10 min)	60-300 seconds

xhigh Timeout Selection:

Simple queries: 240s (4 min)

Medium complexity (single branch review): 360s (6 min)

High complexity (full codebase review): 600s (10 min)

Implementation Pattern (With Request Tracing)
import uuid
from llm_client import LLMClient
from openai import OpenAI

# 1. Get API key from LLMClient
llm = LLMClient()
api_key = llm.api_key

# 2. Create client with 20-MINUTE timeout for xhigh
client = OpenAI(
    api_key=api_key,
    timeout=1200.0  # 20 minutes for xhigh reasoning
)

# 3. Generate traceable request ID BEFORE sending
trace_id = f"researchwala-xhigh-{uuid.uuid4()}"
print(f"📡 Trace ID: {trace_id}")

try:
    response = client.chat.completions.create(
        model="gpt-5.2",
        reasoning_effort="xhigh",
        messages=[...],
        # 4. Inject trace header for OpenAI support debugging
        extra_headers={
            "X-Client-Request-Id": trace_id
        }
    )
    print(f"✓ Server Receipt: {response.id}")
except Exception as e:
    # 5. On failure, you have trace ID for OpenAI support
    print(f"❌ Request Failed. Trace ID for support: {trace_id}")
    raise


CRITICAL FOR xhigh:

Timeout MUST be 1200s (20 min) for complex prompts

ALWAYS log the trace_id BEFORE the request

If timeout occurs, share trace_id with OpenAI support

Extra (recommended): Background mode
If you’re doing very long reviews, consider Responses API background=true instead of pushing timeouts to absurd numbers.

11. Chat Completions vs Responses API (CRITICAL for GPT-5.2)

Status: VERIFIED (January 2026)
Impact: Token savings, reasoning model support, stateful conversations

The Two OpenAI Endpoints
Feature	/v1/chat/completions (Legacy)	/v1/responses (New/Agentic)
Era	GPT-3.5 / GPT-4 / GPT-4o	GPT-5.2 / o3 / Reasoning Models
State	Stateless (manual history)	Stateful (previous_response_id)
History	Send ALL messages every call	Only send new input
Reasoning Param	Flat: reasoning_effort="xhigh"	Nested: reasoning: {"effort": "xhigh"}
Batch API Support	❌ Rejects reasoning param	✅ Full support
Built-in Tools	❌ Manual implementation	✅ Web search, file search, code exec
Token Cost	High (re-send history)	Low (server-side context)

Notes anchored to official API reference:

previous_response_id, store, and reasoning are native in Responses API.

GPT-5.2 supports both Chat Completions and Responses endpoints.

Why /v1/responses is Better for GPT-5.2

Native Reasoning Support: The reasoning parameter works correctly

Stateful Conversations: Use previous_response_id to maintain context without re-sending history

30-Day Storage: Responses stored on OpenAI servers (opt-out with store=False)

Token Savings: For multi-turn chats, can save 50-80% tokens

Reminder: Storage/retention behavior is controlled by OpenAI policy and your org settings; don’t hardcode assumptions into compliance docs.

Batch API: Must Use /v1/responses for xhigh Reasoning
# ❌ FAILS with /v1/chat/completions (uses nested object = wrong)
batch_request = {
    "url": "/v1/chat/completions",
    "body": {
        "model": "gpt-5.2",
        "reasoning": {"effort": "xhigh"},  # Error: unknown_parameter (wrong format!)
        "messages": [...]
    }
}

# ⚠️ chat/completions COULD work with FLAT format, but xhigh may be silently ignored
batch_request = {
    "url": "/v1/chat/completions",
    "body": {
        "model": "gpt-5.2",
        "reasoning_effort": "xhigh",  # Flat string format
        "messages": [...]
    }
}

# ✅ CORRECT: Use /v1/responses with NESTED format
batch_request = {
    "url": "/v1/responses",
    "body": {
        "model": "gpt-5.2",
        "reasoning": {"effort": "xhigh"},  # ✅ Nested object format
        "input": "Your prompt here..."  # Note: "input" not "messages"
    }
}
}

Synchronous API: Both Work (Different Patterns)
# Chat Completions (Legacy) - Still works for sync calls
response = client.chat.completions.create(
    model="gpt-5.2",
    reasoning_effort="xhigh",  # SDK converts to nested format
    messages=[{"role": "user", "content": "..."}]
)

# Responses API (New) - Recommended for multi-turn
response = client.responses.create(
    model="gpt-5.2",
    reasoning={"effort": "xhigh"},
    input="Your prompt here...",
    store=True  # Enable stateful mode
)

# Continue conversation (saves tokens!)
follow_up = client.responses.create(
    model="gpt-5.2",
    input="Follow-up question...",
    previous_response_id=response.id  # No need to re-send history!
)

12. Dependencies on the Global Two-Agent Workflow (READ THIS ONCE)

This AI rules file assumes your workspace also contains:

docs/AGENT_HANDOVER.md (human baton)

docs/AGENT_DECISIONS.md (why X over Y)

docs/AGENT_STATUS.json (machine-readable phase state)

docs/CHANGE_LOG.md (dated history of major changes)

docs/AGENT_API_TRACE.jsonl (every external API call trace)

Agent 1 must update those before Agent 2 is allowed to execute high-risk phases (Strategy B).