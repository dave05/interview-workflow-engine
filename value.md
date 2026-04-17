# Sola Interview Prep — ML Platform Engineer

## The Company

- **Sola** — AI-native RPA. Record a task once, AI agent runs/adapts/self-heals at scale.
- **Founded** 2023, YC, Jersey City NJ. **Funding**: $21M (a16z, Conviction/Sarah Guo).
- **Founders**: Jessica Wu (CEO, ex-Citadel/Thrive, youngest quant at billion-dollar hedge fund), Neil Deshmukh (CTO, MIT CV/LLM research, multimodal RL).
- **Team**: ~33 employees. David Fullerton (ex-CTO Heap, Stack Overflow).
- **Growth**: 5x revenue in 2025, workflow volume doubling MoM, 30%+ quarterly growth.
- **Customers**: Fortune 100s. Logistics, insurance, finance, legal, healthcare.
- **Stack**: Uses Claude Opus 4.6 for agent orchestration + CUA in production.
- **Competitors**: UiPath (incumbent), Automation Anywhere, Microsoft Power Automate, Kore.ai, Moveworks.
- **Threat**: Anthropic launched Claude Managed Agents (April 2026) with enterprise features (RBAC, sandboxing, governance). Allianz adopted. $0.08/hr + tokens.
- **Sola's moat**: Vertical depth, recording UX, data flywheel, customer relationships. Better models help Sola if they own the product layer.

---

## The Repo — 8 Files

```
src/
  types.py                 ← Data model
  workflow_definition.py   ← Invoice workflow (DATA, not code)
  workflow.py              ← Execution loop (ORCHESTRATOR)
  activities.py            ← Actual work (SIDE EFFECTS)
  util/llm.py              ← 3 LLM stubs (EXTENSION POINTS)
  util/session.py           ← Fake browser (HTTP + lxml)
  run/client.py            ← Entry point
  run/worker.py            ← Polls Temporal
```

---

## Core Concepts

**Temporal** = durable workflow orchestration. Workflows survive crashes, auto-retry, persist state.

- **Workflow** = deterministic orchestrator. No side effects. (`workflow.py`)
- **Activity** = side-effecting work unit. HTTP, scraping, LLM. (`activities.py`)
- **Worker** = process polling Temporal for tasks. (`run/worker.py`)
- **Task Queue** = `"workflow-engine-py"` — routes work to workers.
- **Critical rule**: Workflows must be deterministic (Temporal replays them). All side effects go in activities.

---

## Type System (`types.py`)

**5 Actions** (browser ops inside WebNode):
`OpenAction` | `ClickAction` | `ScrapeAction` | `ScrapeAllAction` | `InputAction`

**4 Node types** (workflow steps):
`WebNode` | `ConditionalNode` | `SendEmailNode` | `OutputNode`

**Graph**: `WorkflowDefinition` = nodes + edges + variables
**Edge**: source → target + optional `branch_handle` ("yes"/"no")
**Contract**: `NodeResult` = `{ next_node_id: str|None, variables: dict }` — every activity returns this.

---

## Workflow Definition (`workflow_definition.py`)

```
login → scrape-list → scrape-detail → check-critical
                                           │
                                    yes ───┤─── no
                                           │         │
                                      send-email     │
                                           └──► output-csv ◄┘
```

Variables: `invoicehubUrl`, `username`, `password` → accumulates `invoiceNumbers[]`, `invoiceAmounts[]`, `daysOverdues[]`, `invoiceAmount`, `daysOverdue`, `customerName`, `contactEmail`, `notified`, `output`

---

## The Execution Loop (`workflow.py:40-105`)

```python
variables = {**definition.variables, **input.inputs}
root = find_node_with_no_incoming_edges()
current = root.id

while current is not None:
    match node:
        case WebNode():         → execute_web_node         # line 74
        case ConditionalNode(): → execute_conditional_node  # line 80
        case SendEmailNode():   → execute_send_email_node   # line 86
        case OutputNode():      → execute_output_node       # line 92
        case _:                 → raise                     # line 98
    variables = result.variables
    current = result.next_node_id   # None = done
```

**Extension point**: Add new node type → add `case` here.

Each activity: 60s timeout, 2 max attempts, 1s initial backoff.

---

## Two Critical Helpers (`activities.py`)

**`resolve_template(template, vars)`** — `"Hello {{name}}"` → `"Hello Dave"`. Arrays join with commas. Missing → empty string.

**`determine_next_node(edges, node_id, branch?)`** — Walks the graph. With `branch="yes"` finds matching `branch_handle` edge. Returns `None` at end.

---

## Activities (`activities.py`)

**`execute_web_node`**: Creates `Session()`, loops actions: Open→navigate, Click→click, Scrape→store, ScrapeAll→store as list, Input→pass.

**`execute_conditional_node`**: `resolve_template(prompt)` → `evaluate_branch(prompt, vars)` → follows "yes"/"no" edge. Stores decision in `output_variable`.

**`execute_send_email_node`**: Resolves to/subject/body templates. Logs. Doesn't actually send.

**`execute_output_node`**: Builds CSV. Arrays → multiple rows. Scalars → first row only. Stores in output variable.

---

## LLM Stubs (`util/llm.py`)

| Function | Status | What it would do in production |
|---|---|---|
| `evaluate_branch()` | **WIRED** (conditional node calls it) | LLM reads prompt + data → "yes"/"no" |
| `generate_xpath()` | NOT wired | LLM reads HTML + description → new XPath selector |
| `transform_data()` | NOT wired | LLM transforms/extracts structured data |

Current mock: `evaluate_branch` checks `"critical" in prompt`, then `amount > 5000 or days > 60`.

---

## Session (`util/session.py`)

```python
Session.navigate(url)     # httpx GET → lxml parse
Session.scrape(xpath)     # first match → text_content()
Session.scrape_all(xpath) # all matches → list[str]
Session.click(xpath)      # <a> → follow href; <button> → follow form action
```

---

## Runtime Trace

```
client.py → Temporal → worker.py → ExecuteWorkflow.run()
  → execute_web_node("login")         → vars unchanged
  → execute_web_node("scrape-list")   → +invoiceNumbers[], invoiceAmounts[], daysOverdues[]
  → execute_web_node("scrape-detail") → +invoiceAmount, daysOverdue, customerName, contactEmail
  → execute_conditional_node          → 12450>5000 → "yes" → +notified="yes"
  → execute_send_email_node           → logs email
  → execute_output_node               → builds CSV → +output="..."  → next=None (DONE)
```

---

## Likely Coding Questions (25-30 min)

### #1: Add a new node type (e.g. TransformNode) — 70% likely
Touch: `types.py` → `activities.py` → `workflow.py` match → `worker.py` → test

### #2: Implement self-healing XPath — 60% likely
Touch: `activities.py:execute_web_node` try/except → fallback to `generate_xpath()` → cache

### #3: Refactor LLM to clean abstraction — 50% likely
Touch: `util/llm.py` Protocol/interface → mock vs real → dependency injection → tests stay green

### #4: Fan-out (process all invoices) — 30% likely
Touch: Temporal child workflows or loop inside activity

### Pattern for adding a new node type:
```
1. types.py     → add DataClass + Node, update WorkflowNode union
2. activities.py → add @activity.defn function returning NodeResult
3. workflow.py  → add case in match statement (line 73)
4. worker.py    → register activity in activities list
5. test         → write test with fixture
```

---

## System Design Answers

### "Scale to 10k concurrent workflows?"
1. Workers scale horizontally (K8s HPA on queue depth)
2. Separate worker pools by activity type (web/LLM/output)
3. Rate limit per target system + per LLM provider (Redis token bucket)
4. Temporal namespaces per customer for isolation
5. Temporal itself handles 10k fine; LLM rate limits are the real bottleneck

### "Handle LLM failures in production?"
Layered defense:
1. **Retry with backoff** — Temporal retry policy, don't retry non-retryable errors
2. **Structured output validation** — force JSON schema, Pydantic validate
3. **Fallback** — try different model → simpler prompt → deterministic fallback
4. **Circuit breaker** — >30% error rate → stop hitting provider
5. **Timeout** — 30s hard limit per LLM call
6. **Human-in-the-loop** — low confidence → route to human
7. **Cost guardrails** — per-workflow token budget, daily cap

### "Evaluate LLM conditional node over time?"
1. **Golden dataset** — labeled invoices with expected classifications, include boundary cases
2. **Offline eval** — pytest parametrize, assert accuracy > 95%, run on prompt changes + nightly
3. **Metrics** — accuracy, precision, recall (false negatives worse than false positives), latency, cost
4. **Online eval** — shadow mode (compare new vs old), sample 1% for human review, canary deploys
5. **Prompt versioning** — version prompts, eval before promote, can rollback
6. **Drift detection** — weekly accuracy, distribution of yes/no, alert on 5%+ week-over-week drop
7. **Data flywheel** — production failures → human labels → grow golden dataset

---

## Smart Q&A Questions

1. "Anthropic launched Managed Agents with enterprise features this month. How are you thinking about the strategic response — infrastructure you'd build on, or direct competition?"
2. "What's the most common failure mode — UI changes, LLM hallucinations, or data edge cases?"
3. "How does self-healing actually work in production — retry loop with vision, or proactive validation?"
4. "What's the engineering split between recording layer (CV + LLM) and execution layer (Temporal)?"
5. "How do you think about the tradeoff between deterministic rules and LLM decisions in customer workflows?"

---

## "Why Sola?" Answer

"Sola is at the intersection where I want to work — LLMs aren't a bolt-on, they're core to why the product works. Traditional RPA has been brittle for 20 years, and you're the first team using agentic AI to solve that. The 5x revenue growth in logistics — one of the most legacy-heavy industries — tells me the product is real. And the engineering problem is fascinating: durable execution with Temporal, self-healing selectors, LLM reasoning in the loop, all wrapped in something business users can use. Better models from Anthropic/OpenAI are infrastructure that makes Sola's product stronger, not weaker — as long as you own the customer relationship and vertical depth."

---

## Interview Reminders

- **Use Claude Code openly.** They literally told you to. It's how they work.
- **Drive independently.** Don't wait for hints. Ask clarifying questions upfront, then execute.
- **Think out loud.** They're scoring reasoning and implementation choices, not just output.
- **Test your code.** Writing a test shows senior-level practice.
- **Framework then details.** For design questions: name the problem categories first, then address each.
- **Know the 5 things cold:**
  1. `match` at `workflow.py:73` — where you extend
  2. `NodeResult` — the universal contract
  3. `resolve_template()` — the glue
  4. `determine_next_node()` — graph traversal
  5. `util/llm.py` — 3 stubs, only `evaluate_branch` is live
