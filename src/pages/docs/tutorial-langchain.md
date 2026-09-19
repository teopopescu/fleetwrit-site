---
layout: ../../layouts/DocsLayout.astro
title: "Tutorial: LangChain / LangGraph"
description: Gate a LangGraph node behind a human decision.
---

You'll take one consequential action — issuing a customer refund — and gate it behind a named human's decision inside a LangGraph node. When the graph reaches the node, the request waits for a reviewer to approve, decline or edit, and the graph resumes on exactly that decision with a signed receipt. You'll run it offline first, then route it to a live reviewer.

## 1. Install

```bash
pip install "fleetwrit[langchain]"
```

## 2. Define the action

An action is declared once, next to the function that performs it. The definition drives the fingerprint, the reviewer's screen and the catalog. The `display` hint tells the dashboard to render `amount` as money in the charge's currency.

```python
from fleetwrit import Money, action

@action(type="refund.issue", title="Issue refund", risk="high", reversible=False,
        queue="finance-ops", editable=["amount"],
        summary="Refund {amount} on charge {charge}",
        display={"amount": Money(currency_field="currency")})
def issue_refund(charge: str, amount: int, currency: str) -> str:
    print(f"  -> executing refund of {amount} {currency} on {charge}")
    return f"re_{charge}_{amount}"
```

## 3. Gate it

`fleetwrit_node` returns a LangGraph node that pauses for a human, runs only the approved action via `run`, and writes the outcome back into graph state. `build_action` maps state onto the action fingerprint; `context` attaches reviewer-facing context.

```python
from fleetwrit.integrations.langchain import fleetwrit_node

gate = fleetwrit_node(
    client,
    build_action=lambda s: issue_refund.action(charge=s["charge"], amount=s["amount"], currency=s["currency"]),
    run=lambda args: issue_refund(**args),
    context=lambda s: {"ticket": "ZD-99120"},
)
```

## 4. The full script

```python
import os
from typing import Any, TypedDict
import fleetwrit
from fleetwrit import Money, action, testing
from fleetwrit.integrations.langchain import fleetwrit_node
from langgraph.graph import END, START, StateGraph

@action(type="refund.issue", title="Issue refund", risk="high", reversible=False,
        queue="finance-ops", editable=["amount"],
        summary="Refund {amount} on charge {charge}",
        display={"amount": Money(currency_field="currency")})
def issue_refund(charge: str, amount: int, currency: str) -> str:
    print(f"  -> executing refund of {amount} {currency} on {charge}")
    return f"re_{charge}_{amount}"

client = (fleetwrit.Client(agent_id="support-refunds", environment="prod")
          if os.getenv("FLEETWRIT_URL")
          else testing.client(testing.auto_approve(), agent_id="support-refunds", environment="prod"))

class State(TypedDict, total=False):
    charge: str
    amount: int
    currency: str
    result: Any
    fleetwrit: dict

gate = fleetwrit_node(
    client,
    build_action=lambda s: issue_refund.action(charge=s["charge"], amount=s["amount"], currency=s["currency"]),
    run=lambda args: issue_refund(**args),
    context=lambda s: {"ticket": "ZD-99120"},
)

graph = StateGraph(State)
graph.add_node("gate", gate)
graph.add_edge(START, "gate")
graph.add_edge("gate", END)
app = graph.compile()

out = app.invoke({"charge": "ch_123", "amount": 400000, "currency": "gbp"})
print("result:", out["result"], "| receipt:", out["fleetwrit"]["receipt"] is not None)
```

## 5. Test it (offline, no server, no API key)

Save the script and run it:

```bash
python tutorial_langchain.py
```

Expected output:

```
result: re_ch_123_400000 | receipt: True
```

(the `  -> executing refund of 400000 gbp on ch_123` line prints just before it.)

`fleetwrit.testing` is what makes this runnable with no server and no LLM: `testing.client(testing.auto_approve(), ...)` returns an in-memory client whose fake reviewer auto-approves every request, so the whole gate-and-resume loop runs in-process. The same script ships in the SDK repo at `examples/langchain_tutorial.py` and is verified in CI.

The node writes its outcome back into state under `out["fleetwrit"]`, which carries `approved`, `modified`, `args`, `receipt` and `reviewer`. On rejection `result` is `None` and `fleetwrit.approved` is `False`.

## 6. Go live (a human decides)

Start the local stack — an empty server plus the dashboard on `http://localhost:4100`:

```bash
pip install "fleetwrit[dev-server]"
fleetwrit dev
```

In a second terminal, point the client at it and re-run:

```bash
export FLEETWRIT_URL=http://localhost:4100
python tutorial_langchain.py
```

Because `FLEETWRIT_URL` is now set, the script uses a real `fleetwrit.Client` instead of the in-memory fake. The graph blocks at the `gate` node: the request waits in the dashboard for a human to **approve**, **decline** or **edit** the `amount`. When the reviewer decides, the graph resumes on that exact decision — running the approved arguments, not necessarily the ones proposed — and carries a signed receipt.

## Next steps

- [Integrations](/fleetwrit-site/docs/integrations) — how each adapter wraps its framework's own pause-and-resume.
- [Defining actions](/fleetwrit-site/docs/actions) — risk, summaries, display hints, redaction and versioning.
