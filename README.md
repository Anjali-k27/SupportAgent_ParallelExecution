# Enterprise AI Support Platform
### Session 9 of 12 — Shared Scratchpads & Consensus

A production-grade AI customer support system built progressively over 12 sessions using **LangGraph**, **Google Gemini 2.5 Flash**, and **FastAPI**. Each session adds one architectural layer on top of the last — nothing is ever removed or replaced.

---

## Quick Start

```bash
# 1. Clone and enter the project
git clone https://github.com/waseemkhan606/phase3-session8
cd phase3-session8

# 2. Create and activate a virtual environment
python -m venv .venv
source .venv/bin/activate        # Windows: .venv\Scripts\activate

# 3. Install dependencies
pip install -r requirements.txt
python -m spacy download en_core_web_lg

# 4. Set your Google API key
echo "GOOGLE_API_KEY=your-key-here" > .env

# 5. Run the CLI verification
python support_agent.py

# 6. Start the web server
python api.py
# → Open http://localhost:8000
```

---

## Project Structure

```
session9/
├── support_agent.py   # Core agent logic — LangGraph graph, all nodes, state
├── api.py             # FastAPI server — REST + Server-Sent Events streaming
├── index.html         # Single-file frontend — no build step required
├── requirements.txt   # Pinned Python dependencies
├── .env               # GOOGLE_API_KEY (never commit this)
└── support.db         # SQLite checkpoint store (auto-created on first run)
```

---

## Architecture Overview

The system runs two execution paths inside a single LangGraph master graph, chosen automatically based on ticket complexity:

```
User Ticket
    │
    ▼
┌─────────────────────────────┐
│   TRIAGE SUBGRAPH           │  ← ingress security scan + classification
│   ingress_node              │
│   classify_node             │
└────────────┬────────────────┘
             │
    route_after_triage()
             │
    ┌────────┴────────────┐
    │                     │
    ▼                     ▼
SUPERVISOR PATH      DISPATCHER PATH
(single-intent)      (multi-issue / fraud)
    │                     │
    ▼                     ▼
supervisor_node      dispatcher_node
    │                [list[Send] fan-out]
    ▼                     │
tech_support  ◄───┐  ┌────┼────────────┐
fraud_handler     │  ▼    ▼            ▼
general_handler   │ tech  billing   fraud
    │             │ analysis analysis analysis
    └─────────────┘  (parallel superstep)
    (hub-and-spoke)       │
                    synthesizer_node
                          │
                         END
```

### Path 1 — Supervisor (Sequential)
Handles simple, single-intent tickets. The supervisor LLM reads the full state and routes one worker at a time in a hub-and-spoke pattern until it decides `FINISH`.

### Path 2 — Dispatcher (Parallel)
Fires when the ticket contains multiple issues or fraud signals. The dispatcher uses LangGraph's **Send API** to fan out to up to three specialist agents simultaneously. After all agents complete their superstep, the synthesizer merges findings into one response.

---

## Session-by-Session Build Log

Each session introduced one new architectural concept. All prior code is permanent.

### Session 1 — The Blueprint
**Concept:** State schema and graph skeleton.

- Defined `SharedState` (17 fields) using LangGraph `TypedDict`
- Wired a basic classify → route → handler graph
- Established the `raw_input → final_response` contract
- Key file: `support_agent.py` — `SharedState`, `build_initial_state()`

### Session 2 — Tool Binding
**Concept:** LLM function calling with real tools.

- Added `get_customer_details` — CRM lookup (mock data, `C-XXXX` IDs)
- Added `search_knowledge_base` — keyword search over KB articles
- Bound tools to the LLM with `llm.bind_tools(TOOLS)`
- Key: `agent_node` upgraded from stub to tool-calling loop

### Session 3 — The ReAct Architecture
**Concept:** Observe → Reason → Act loop with safety limits.

- `agent_node` became a full ReAct loop: LLM calls tool → reads result → calls again if needed
- `MAX_ITERATIONS = 5` circuit breaker — fires escalation if exceeded
- `check_fraud_signals` tool added for fraud analysis
- Duplicate tool-call detection via fingerprinting
- Key: `get_tool_fingerprint()`, `build_escalation_response()`

### Session 4 — Persistence & Threading
**Concept:** SQLite checkpointer for multi-turn conversations.

- `SqliteSaver` wired as LangGraph checkpointer → every state transition persisted
- `thread_id` parameter on `run_ticket()` — same ID continues a conversation
- `/api/threads` and `/api/history/{thread_id}` endpoints added
- Key: `support.db`, `get_conversation_history()`, `get_active_threads()`

### Session 5 — Context Management
**Concept:** Automatic summarization to prevent context window overflow.

- `SUMMARY_THRESHOLD = 8` — when message count exceeds this, summarization fires
- `summarization_node` compresses old messages into `system_summary` string
- `deduplicate_messages` custom reducer replaces `add_messages` to handle `RemoveMessage`
- Key: `summarization_node()`, `SUMMARIZATION_PROMPT`, `deduplicate_messages()`

### Session 6 — Guardrails & Bounding
**Concept:** Input and output security with Microsoft Presidio.

- `ingress_node` — scans every ticket for PII (7 entity types) and injection patterns (14 regex rules)
- PII is anonymized before reaching any LLM; injection attempts are blocked
- `egress_node` — scans final response for PII leakage and uncertainty markers
- `blocked_response_node` — returns a pre-written refusal with zero LLM tokens
- Key: `ingress_node()`, `egress_node()`, `INJECTION_PATTERNS`, `PII_ENTITIES`

### Session 7 — Multi-Agent Topologies
**Concept:** Subgraph compilation for modular graph composition.

- `triage_subgraph` compiled independently: `ingress → classify`
- `tech_support_subgraph` compiled independently: `ReAct loop + egress`
- `operator.add` established as the reducer for `internal_notes` and `tool_results` — parallel-write safe
- `demonstrate_silent_overwrite_bug()` shows what happens without `operator.add`
- Key: `build_triage_subgraph()`, `build_tech_support_subgraph()`

### Session 8 — The Supervisor Orchestrator
**Concept:** LLM-powered dynamic routing in a hub-and-spoke topology.

- `SupervisorDecision` Pydantic model — enforces `Literal` values on structured output
- `supervisor_node` — calls LLM on every delegation, writes routing decision to `internal_notes`
- `supervisor_router` — pure Python, reads `next_worker` from state
- `MAX_DELEGATIONS = 5` safety limit — fires `FINISH` if exceeded
- `general_handler` upgraded from stub to LLM call (stub caused infinite supervisor loops)
- Key: `supervisor_node()`, `supervisor_router()`, `SupervisorDecision`

### Session 9 — Shared Scratchpads & Consensus *(current)*
**Concept:** Parallel execution via the LangGraph Send API.

- `dispatcher_node` — reads category + fraud keywords, returns `list[Send]` for fan-out
- Three specialist agents run **simultaneously** as a superstep:
  - `tech_analysis_agent` — scoped to `search_knowledge_base` only
  - `billing_analysis_agent` — scoped to `get_customer_details` only
  - `fraud_analysis_agent` — scoped to `check_fraud_signals` only
- Each agent writes a structured `finding` dict to `internal_notes` (preserved by `operator.add`)
- `synthesizer_node` — waits for all agents, filters by severity, calls LLM for unified response
- `route_after_triage` — decides which path: supervisor, dispatcher, or terminal
- Key: `dispatcher_node()`, `synthesizer_node()`, `SEVERITY_ORDER`, `Send`

---

## State Schema (`SharedState`)

All 17 fields declared in Session 1. Each field is either populated by its owning node or carried forward unchanged.

| Field | Type | Owner | Purpose |
|-------|------|-------|---------|
| `raw_input` | `str` | Entry | Original user message, never modified |
| `sanitized_input` | `str` | `ingress_node` | PII-masked version |
| `category` | `str` | `classify_node` | `technical / billing / fraud / general` |
| `messages` | `list` | `agent_node` | Full conversation history (custom reducer) |
| `customer_data` | `dict` | Tool | CRM data from `get_customer_details` |
| `tool_results` | `list` | `agent_node` | Tool call fingerprints (operator.add) |
| `pii_detected` | `bool` | `ingress_node` | Whether PII was found and masked |
| `injection_detected` | `bool` | `ingress_node` | Whether injection pattern matched |
| `is_safe` | `bool` | `ingress_node` | False blocks the request |
| `system_summary` | `str` | `summarization_node` | Compressed history for long threads |
| `iteration_count` | `int` | `agent_node` | ReAct loop counter for circuit breaker |
| `internal_notes` | `list` | All agents | Shared scratchpad (operator.add — parallel safe) |
| `delegation_count` | `int` | `supervisor_node` | How many times supervisor has routed |
| `next_worker` | `str` | `supervisor_node` | Last routing decision |
| `github_draft` | `dict` | *(Session 10)* | Proposed GitHub issue before creation |
| `github_issue_url` | `str` | *(Session 10)* | URL after issue is created |
| `final_response` | `str` | Various | The customer-facing answer |

---

## Tools

Three tools are registered with the LLM. Each has a full docstring specifying WHAT, WHEN, FORMAT, and RETURN contract.

| Tool | Args | Returns | Used by |
|------|------|---------|---------|
| `get_customer_details` | `customer_id: str` (C-XXXX) | name, billing status, balance, transactions | `agent_node`, `billing_analysis_agent` |
| `search_knowledge_base` | `query: str` | matched KB articles or fallback guidance | `agent_node`, `tech_analysis_agent` |
| `check_fraud_signals` | `account_id: str` (ACC-FXXX) | risk score, flagged patterns, recommendation | `fraud_handler`, `fraud_analysis_agent` |

Scoped bindings for parallel agents:
```python
tech_analysis_llm    = llm.bind_tools([search_knowledge_base])   # cannot call CRM or fraud
billing_analysis_llm = llm.bind_tools([get_customer_details])    # cannot call KB or fraud
fraud_analysis_llm   = llm.bind_tools([check_fraud_signals])     # cannot call KB or CRM
```

---

## Parallel Agent Finding Schema

Each parallel agent writes one `finding` dict to `internal_notes`. The schema is enforced by `extract_finding_from_response()`, which falls back to `build_empty_finding()` on parse failure.

```python
finding = {
    'agent':       'tech_analysis' | 'billing_analysis' | 'fraud_analysis',
    'finding':     str,   # what was discovered
    'action':      str,   # recommended next step
    'severity':    'critical' | 'high' | 'medium' | 'low' | 'none',
    'data':        dict,  # raw tool output (optional)
    'tool_called': str,   # name of tool invoked or 'none'
}
```

Severity order used by the synthesizer (critical findings addressed first):
```python
SEVERITY_ORDER = {'critical': 0, 'high': 1, 'medium': 2, 'low': 3, 'none': 4}
```

---

## API Reference

### `POST /api/run`
Runs a ticket through the full graph synchronously.

**Request:**
```json
{ "ticket": "string", "thread_id": "optional-uuid", "return_existing": false }
```

**Response (key fields):**
```json
{
  "category": "technical",
  "final_response": "...",
  "delegation_count": 2,
  "next_worker": "FINISH",
  "parallel_executed": true,
  "agent_findings": [
    { "agent": "tech_analysis", "severity": "high", "finding": "..." },
    { "agent": "billing_analysis", "severity": "high", "finding": "..." }
  ],
  "supervisor_notes": [...],
  "tool_calls_log": [...],
  "thread_id": "uuid"
}
```

### `POST /api/stream`
Streams graph execution as Server-Sent Events.

**SSE event fields (Session 9 additions):**
```json
{ "node": "dispatcher", "subgraph": "parallel" }
{ "node": "tech_analysis_agent", "parallel": true, "agent_name": "tech_analysis_agent", "finding": {...} }
{ "node": "synthesizer", "synthesis_complete": true, "findings_merged": 2 }
```

### `POST /api/verify`
Runs the 5-check automated verification suite.

### `GET /api/threads`
Returns all thread IDs with at least one checkpoint.

### `GET /api/history/{thread_id}`
Returns full checkpoint history for a conversation thread.

### `GET /health`
```json
{
  "session": 9,
  "parallel_agents": 3,
  "architecture": "hybrid_parallel_sequential"
}
```

---

## Key Design Decisions

**Why `operator.add` on `internal_notes`?**
LangGraph merges state from parallel branches by calling the field's reducer. The default reducer overwrites — so if tech and billing agents both write to `internal_notes`, only one would survive. `operator.add` appends both, preserving every finding.

**Why scoped LLM bindings per parallel agent?**
Each agent is given only the one tool it needs. A billing agent that can also call `check_fraud_signals` might hallucinate fraud risk based on billing data. Scoping enforces domain separation at the API level, not just the prompt level.

**Why is dispatcher a no-op node with a conditional edge?**
LangGraph's `Send` API is designed for conditional edge routing functions that return `list[Send]`. Registering `dispatcher_node` as the conditional edge function (rather than the node function itself) is the pattern that's cleanly supported across LangGraph 1.x.

**Why does route_after_triage exist?**
In Session 8, every ticket went to the supervisor, even blocked requests. The supervisor would then return `FINISH` immediately after seeing `final_response` already set. `route_after_triage` short-circuits this: blocked → `terminal` (zero LLM tokens), multi-issue → `dispatcher`, simple → `supervisor`.

---

## Security Model

Every ticket passes through two security gates:

**Ingress (before any LLM call):**
1. PII detection via Microsoft Presidio (7 entity types, confidence > 0.7) — PII is anonymized but the request proceeds
2. Injection pattern matching (14 regex rules) — matched requests are blocked with a reference number, zero tokens consumed

**Egress (after LLM response):**
1. PII leakage scan on the outgoing response — flagged and logged
2. Uncertainty marker detection (9 patterns like "I think", "maybe") — flagged and logged

Both gates use `is_safe` in state to communicate downstream. Injection blocks; PII alone does not.

---

## Running the Verification Suite

### CLI
```bash
python support_agent.py
```
Runs 5 test cases then the 5-check verification suite. Expected output:
```
SESSION 9 COMPLETE — 5/5 checks passed in ~35000ms
  ✅ PASS  Multi-issue ticket → dispatcher fires, ≥2 findings
  ✅ PASS  All findings preserved — operator.add working
  ✅ PASS  Synthesizer produces coherent unified response
  ✅ PASS  Fraud ticket → fraud_analysis_agent fires
  ✅ PASS  Simple ticket still uses supervisor path
```

### Browser
1. Start the server: `python api.py`
2. Open `http://localhost:8000`
3. Scroll to **🧪 Session 9 — Verification Test** at the bottom
4. Click **▶ Run Verification Test**
5. All 5 rows should show ✅ and the footer should read:
   > 🚀 Session 10 is unblocked — System API Integration

---

## Session 10 Preview — System API Integration

The next session adds write access. On top of Session 9's parallel path:

- `tech_analysis_agent` will populate `github_draft` when severity is `high` or `critical`
- New `create_github_issue(title, body, labels)` tool with SHA-256 idempotency keys
- New `github_tool_node` reads `github_draft`, checks idempotency, calls the API, writes `github_issue_url`
- Zero-Trust credential injection at runtime

All Session 9 infrastructure (dispatcher, parallel agents, synthesizer) remains unchanged.

---

## Technology Stack

| Layer | Technology | Version |
|-------|-----------|---------|
| LLM | Google Gemini 2.5 Flash | via `langchain-google-genai` |
| Agent framework | LangGraph | 1.2.0 |
| Tool binding | LangChain Core | 1.4.0 |
| Persistence | SQLite + `langgraph-checkpoint-sqlite` | — |
| PII / security | Microsoft Presidio | 2.2.x |
| NLP model | spaCy `en_core_web_lg` | 3.8.0 |
| API server | FastAPI + Uvicorn | 0.136.1 |
| Frontend | Vanilla JS + SSE | no build step |
| Environment | Python | 3.10+ |
