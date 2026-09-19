---
layout: ../../layouts/DocsLayout.astro
title: "Tutorial: OpenAI Agents SDK"
description: Gate an OpenAI Agents SDK tool behind a human decision, end to end.
---

You'll take one consequential tool — a production rollback — and gate it behind a named human's decision. When the agent calls it, the request waits for a reviewer to approve, decline or edit, and the agent resumes on exactly that decision with a signed receipt. You'll run the whole thing offline first, then flip a switch and route it to a live reviewer.

## 1. Install

```bash
pip install "fleetwrit[openai-agents]"
```

## 2. Define the action

An action is declared once, next to the function that performs it. The definition drives the fingerprint, the reviewer's screen and the catalog.

```python
from fleetwrit import action

@action(type="deploy.rollback", title="Roll back deployment", risk="high",
        reversible=False, queue="sre-oncall", editable=["version"],
        summary="Roll back {service} to version {version}")
def roll_back(service: str, version: int) -> str:
    print(f"  -> rolling back {service} to v{version}")
    return f"{service}@v{version}"
```

## 3. Gate it

`gated_tool` wraps the function so a call pauses for a human, runs only the approved action, and carries a signed receipt. `build_action` maps the call arguments onto the action fingerprint.

```python
from fleetwrit.integrations.openai_agents import gated_tool

gated = gated_tool(
    client, build_action=lambda service, version: roll_back.action(service=service, version=version),
)(roll_back)
```

## 4. The full script

```python
import os
import fleetwrit
from fleetwrit import action, testing
from fleetwrit.integrations.openai_agents import gated_tool

@action(type="deploy.rollback", title="Roll back deployment", risk="high",
        reversible=False, queue="sre-oncall", editable=["version"],
        summary="Roll back {service} to version {version}")
def roll_back(service: str, version: int) -> str:
    print(f"  -> rolling back {service} to v{version}")
    return f"{service}@v{version}"

# offline auto-approve unless FLEETWRIT_URL points at a live server
client = (fleetwrit.Client(agent_id="sre-remediation", environment="prod")
          if os.getenv("FLEETWRIT_URL")
          else testing.client(testing.auto_approve(), agent_id="sre-remediation", environment="prod"))

gated = gated_tool(
    client, build_action=lambda service, version: roll_back.action(service=service, version=version),
)(roll_back)

# call it directly here; in a real agent, register it as a tool (see below)
print("result:", gated(service="payments-api", version=42))
```

## 5. Test it (offline, no server, no API key)

Save the script and run it:

```bash
python tutorial_openai_agents.py
```

Expected output:

```
result: payments-api@v42
```

(the `  -> rolling back payments-api to v42` line prints just before it.)

`fleetwrit.testing` is what makes this runnable with no server and no LLM: `testing.client(testing.auto_approve(), ...)` returns an in-memory client whose fake reviewer auto-approves every request, so the whole gate-and-resume loop runs in-process. The same script ships in the SDK repo at `examples/openai_agents_tutorial.py` and is verified in CI.

## 6. Go live (a human decides)

Start the local stack — an empty server plus the dashboard on `http://localhost:4100`:

```bash
pip install "fleetwrit[dev-server]"
fleetwrit dev
```

In a second terminal, point the client at it and re-run:

```bash
export FLEETWRIT_URL=http://localhost:4100
python tutorial_openai_agents.py
```

Because `FLEETWRIT_URL` is now set, the script uses a real `fleetwrit.Client` instead of the in-memory fake. The call blocks: the request waits in the dashboard for a human to **approve**, **decline** or **edit** the `version`. When the reviewer decides, the agent resumes on that exact decision — running the approved arguments, not necessarily the ones proposed — and carries a signed receipt.

## In a real agent

In practice you register `gated` as a tool rather than calling it directly (needs `agents` and `OPENAI_API_KEY`):

```python
from agents import Agent, Runner, function_tool
agent = Agent(name="SRE", instructions="Roll payments-api to version 42.", tools=[function_tool(gated)])
print(Runner.run_sync(agent, "Roll back payments-api to version 42").final_output)
```

`gated_tool` preserves the tool signature and supports async tools; on rejection it returns a short `"Rejected by reviewer: …"` string the agent reads.

## Next steps

- [Integrations](/fleetwrit-site/docs/integrations) — how each adapter wraps its framework's own pause-and-resume.
- [Defining actions](/fleetwrit-site/docs/actions) — risk, summaries, display hints, redaction and versioning.
