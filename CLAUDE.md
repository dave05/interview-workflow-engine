# Workflow Engine

Temporal-based workflow engine that executes graph-defined workflows for web scraping, data processing, and automation. Python implementation in `workflow-engine-py/`.

## Architecture

```
src/
  types.py                 ← Data model (actions, nodes, edges, results)
  workflow_definition.py   ← Invoice export workflow (data, not code)
  workflow.py              ← Execution loop — walks the node graph
  activities.py            ← Side-effecting work (scraping, LLM, email, CSV)
  util/llm.py              ← 3 LLM stubs (evaluate_branch, generate_xpath, transform_data)
  util/session.py          ← Simulated browser (HTTP + lxml, not Selenium)
  run/client.py            ← Triggers workflows via Temporal client
  run/worker.py            ← Polls Temporal, executes workflows + activities
```

## Key Concepts

- **Workflow** (`workflow.py`) = deterministic orchestrator. No side effects. Temporal replays these on crash recovery.
- **Activity** (`activities.py`) = side-effecting work. HTTP, scraping, LLM calls. Wrapped with retry + timeout by Temporal.
- **Variables** = shared `dict[str, VariableValue]` flowing through every node. Each activity reads/updates and passes forward.
- **NodeResult** = `{ next_node_id: str|None, variables: dict }` — universal return type from every activity. `None` = workflow done.

## Type System (`types.py`)

**Actions** (browser ops inside WebNode):
`OpenAction` | `ClickAction` | `ScrapeAction` | `ScrapeAllAction` | `InputAction`

**Nodes** (workflow steps):
`WebNode` | `ConditionalNode` | `SendEmailNode` | `OutputNode`

**Graph**: `WorkflowDefinition` = nodes + edges + initial variables
**Edge**: `WorkflowEdge` = source → target + optional `branch_handle` ("yes"/"no")

## Execution Loop (`workflow.py:56-102`)

```python
while current_node_id is not None:
    match node:
        case WebNode():         → execute_web_node
        case ConditionalNode(): → execute_conditional_node
        case SendEmailNode():   → execute_send_email_node
        case OutputNode():      → execute_output_node
    variables = result.variables
    current_node_id = result.next_node_id
```

Each activity: 60s timeout, 2 max attempts, 1s initial backoff.

## Two Critical Helpers (`activities.py`)

**`resolve_template(template, vars)`** — `"Hello {{name}}"` → `"Hello Dave"`. Arrays join with commas. Missing vars → empty string.

**`determine_next_node(edges, node_id, branch?)`** — Finds next node from edge list. With `branch="yes"` matches `branch_handle`. Returns `None` at graph end.

## LLM Stubs (`util/llm.py`)

| Function | Status | Purpose |
|---|---|---|
| `evaluate_branch(prompt, variables)` | **Wired** — called by `execute_conditional_node` | Decides "yes"/"no" branch. Currently hardcoded: `amount > 5000 or days > 60` |
| `generate_xpath(description, html)` | Not wired | Self-healing: generate new XPath when scrape fails |
| `transform_data(prompt, data)` | Not wired | LLM-powered data transformation/extraction |

## Adding a New Node Type

```
1. types.py      → add DataClass + Node, update WorkflowNode union
2. activities.py  → add @activity.defn function returning NodeResult
3. workflow.py    → add case in match statement (line 73)
4. worker.py      → register activity in activities list
5. test           → write test with fixture
```

## Every Activity Has the Same Signature

```python
@activity.defn
async def execute_*_node(
    data: *NodeData,                    # node-specific config
    variables: dict[str, VariableValue], # full variable state
    edges: list[WorkflowEdge],          # graph edges for traversal
    node_id: str,                       # current node's id
) -> NodeResult:                        # {next_node_id, variables}
```

Exception: `execute_web_node` takes `actions: list[Action]` instead of a data object.

## Variable Flow Through the Workflow

```
Start:     {invoicehubUrl, username, password}
login:     unchanged
scrape-list:    + invoiceNumbers[], invoiceAmounts[], daysOverdues[]
scrape-detail:  + invoiceAmount, daysOverdue, customerName, contactEmail
check-critical: + notified="yes"|"no"
send-email:     unchanged
output-csv:     + output="Invoice Number,Amount,Days Overdue\n..."
```

## Session API (`util/session.py`)

```python
Session.navigate(url)     # httpx GET → lxml parse
Session.scrape(xpath)     # first match → text_content(), empty string if no match
Session.scrape_all(xpath) # all matches → list[str]
Session.click(xpath)      # <a> → follow href; <button> in <form> → follow form action; else raise
```

Scrape returns empty string on no match (silent failure). Click raises on no match.

## Running

```bash
docker compose up -d                          # Temporal + InvoiceHub
cd workflow-engine-py
python3.12 -m venv .venv && source .venv/bin/activate
pip install ".[dev]"
python -m src.run.worker                      # terminal 1
python -m src.run.client --hello              # terminal 2 (verify)
python -m src.run.client                      # run invoice workflow
pytest                                        # run tests
```

## Infrastructure

- **Temporal Server**: port 7233 (PostgreSQL + Elasticsearch backend)
- **Temporal UI**: http://localhost:8080
- **InvoiceHub**: http://localhost:3000 (Express.js mock billing app, 12 invoices, admin/admin123)
- **Task Queue**: `"workflow-engine-py"`

## Testing Patterns (`test_activities.py`)

- Uses embedded `http.server` with `FIXTURE_HTML` (3-row invoice table)
- `base_url` fixture provides the test server URL
- Activities are tested directly (not through Temporal) — just call the async function
- `caplog` captures `activity.logger.info` output for assertion
- Edge cases: missing variables resolve to empty, no outgoing edge returns None
