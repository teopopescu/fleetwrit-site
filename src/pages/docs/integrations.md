---
layout: ../../layouts/DocsLayout.astro
title: Integrations
description: Each integration wraps a framework's own pause mechanism, so your team keeps its checkpointing and adds Fleetwrit's queue, identity and record.
---

> **Status:** **LangGraph**, **OpenAI Agents**, and **Amazon Bedrock AgentCore** ship today; the LlamaIndex adapter and the interrupt/webhook resume mode are planned.

## How integrations work

Fleetwrit does not replace your framework's pause-and-resume — it wraps it. The integration turns a native interrupt into a Fleetwrit request, waits for a decision, verifies the receipt, and resumes the framework exactly where it stopped. Your checkpointing is unchanged.

Each integration ships as a PyPI extra so the core stays dependency-free:

```bash
pip install "fleetwrit[langchain] @ git+https://github.com/teopopescu/fleetwrit-python.git"
# langchain, openai-agents and agentcore ship today; llamaindex is planned
```

## Agent runtimes

| Integration | Hooks into | What you write |
| --- | --- | --- |
| LangGraph / LangChain | `interrupt()` and `Command(resume=...)`; LangChain HITL middleware | `FleetwritMiddleware(...)` or `fleetwrit_node()` |
| LlamaIndex Workflows | `InputRequiredEvent` / `HumanResponseEvent` | `FleetwritHITL(workflow)` |
| OpenAI Agents SDK | `needs_approval`, `result.interruptions`, `RunState` | `await fleetwrit_run(agent, input)` |
| Amazon Bedrock AgentCore | In-agent tool gating, or a Gateway Lambda interceptor | `gate(fw, ...)(tool)` or `gateway_handler(fw, ...)` |

## LangGraph example

```python
from langgraph.graph import StateGraph
from fleetwrit.integrations.langchain import FleetwritMiddleware

middleware = FleetwritMiddleware(tools={"rollback_deployment": True})

graph = StateGraph(State)
graph.add_node("act", middleware.wrap(act_node))
app = graph.compile(checkpointer=checkpointer)
```

When `act` calls a gated tool, the graph pauses with a native `interrupt()`, Fleetwrit creates the request, and the graph resumes via `Command(resume=decision)` once a reviewer decides. Kill the process mid-wait and it re-attaches on restart.

## Bedrock AgentCore

The AgentCore adapter is implemented and tested in fleetwrit 0.0.2. Install it as an extra:

```bash
pip install "fleetwrit[agentcore]"
```

`fleetwrit.integrations.agentcore` exposes `gate`, `gateway_handler` and `tool_input`. You choose where to gate: inside the agent, or at the AgentCore Gateway.

### In-agent gating on AgentCore Runtime

Decorate the tool — no gateway needed. `gate` pauses for a human, runs only the approved action, and carries a signed receipt.

```python
from fleetwrit import Client, action
from fleetwrit.integrations.agentcore import gate

fw = Client()  # reads FLEETWRIT_URL / FLEETWRIT_AGENT_ID / FLEETWRIT_ENVIRONMENT

@action(type="deploy.rollback", title="Roll back", risk="high",
        reversible=False, editable=["version"])
def roll_back(service: str, version: int) -> str:
    ...  # your AWS/kubectl call

# register this as the agent's tool: it pauses for a human, runs only the
# approved action, and carries a signed receipt.
gated_roll_back = gate(
    fw, build_action=lambda service, version: roll_back.action(service=service, version=version),
)(roll_back)
```

`gate` supports both sync and `async def` tools. On rejection the wrapped tool returns a short `"Rejected by reviewer: …"` string the agent can read and act on.

### Gateway Lambda interceptor

Gate any tool at the AgentCore Gateway with no agent-code changes. Deploy `handler` as the Lambda behind an AgentCore Gateway target.

```python
from fleetwrit import Client
from fleetwrit.integrations.agentcore import gateway_handler
from mytools import roll_back, roll_back_def   # your impl + its @action definition

fw = Client()

# Deploy `handler` as the Lambda behind an AgentCore Gateway target.
handler = gateway_handler(
    fw,
    build_action=lambda service, version: roll_back_def.action(service=service, version=version),
    invoke=roll_back,
)
```

The handler reads the tool args from the Gateway event, requires a Fleetwrit decision, and:

- on approval runs `invoke(**approved_args)` and returns `{"statusCode": 200, "approved": true, "result": ..., "receipt": "<JWS>"}`;
- on rejection returns `{"statusCode": 403, "approved": false, "reason": "..."}` and the tool never runs.

There is also `tool_input(event)`, which extracts the tool arguments from the common Gateway event shapes (`input` / `arguments` / `parameters` / `body`).

The SDK pieces — `gate`, `gateway_handler` and `tool_input` — are implemented and tested. Deploying the handler as a Lambda and wiring the AgentCore Gateway target (SAM/CloudFormation) is standard AWS setup you own; there is no one-click deploy.

## Blocking vs interrupt mode

- **Blocking** (default): the SDK long-polls and survives a restart. Best for long-running processes.
- **Interrupt**: the SDK raises the runtime's own interrupt and resumes from a signed webhook. Best for checkpointing runtimes that tear down between steps.

## Contract tests

Every integration ships a contract suite that:

1. pauses, decides, and resumes on that one terminal decision;
2. **kills the agent process mid-wait** and checks the decision is delivered and consumed at most once;
3. exercises the rejection path and the expiry path.

Run them against the oldest supported and the latest framework version:

```bash
nox -s contract-langchain
```

## Status

The live list of runtimes, identity providers, policy engines and notification channels — with `available` / `next` / `planned` status — is on the [integrations section of the site](/fleetwrit-site/#integrations). Vote for what you need next on the linked GitHub issues.

## Next steps

- [Identity](/fleetwrit-site/docs/identity) — connect the IdP reviewers sign in with.
- [Asking a human](/fleetwrit-site/docs/asking) — the calls the integrations wrap.
