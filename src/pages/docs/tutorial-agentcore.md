---
layout: ../../layouts/DocsLayout.astro
title: "Tutorial: Amazon Bedrock AgentCore"
description: Gate an AgentCore Runtime tool behind a human decision.
---

You'll take one consequential tool — terminating EC2 instances — and gate it behind a named human's decision. When the agent calls it, the request waits for a reviewer to approve, decline or edit, and the agent resumes on exactly that decision with a signed receipt. You'll run it offline first, then route it to a live reviewer.

## 1. Install

```bash
pip install "fleetwrit[agentcore]"
```

## 2. Define the action

An action is declared once, next to the function that performs it. The definition drives the fingerprint, the reviewer's screen and the catalog.

```python
from fleetwrit import action

@action(type="infra.terminate", title="Terminate instances", risk="high",
        reversible=False, queue="sre-oncall", editable=["count"],
        summary="Terminate {count} {instance_type} instances in {asg}")
def terminate_instances(asg: str, instance_type: str, count: int) -> str:
    print(f"  -> terminating {count} {instance_type} instances in {asg}")
    return f"terminated:{asg}:{count}"
```

## 3. Gate it

`gate` wraps the tool so a call pauses for a human, runs only the approved action, and carries a signed receipt. In an AgentCore Runtime agent you register `gated` as the tool. `build_action` maps the call arguments onto the action fingerprint.

```python
from fleetwrit.integrations.agentcore import gate

gated = gate(client, build_action=lambda asg, instance_type, count:
             terminate_instances.action(asg=asg, instance_type=instance_type, count=count))(terminate_instances)
```

## 4. The full script

```python
import os
import fleetwrit
from fleetwrit import action, testing
from fleetwrit.integrations.agentcore import gate

@action(type="infra.terminate", title="Terminate instances", risk="high",
        reversible=False, queue="sre-oncall", editable=["count"],
        summary="Terminate {count} {instance_type} instances in {asg}")
def terminate_instances(asg: str, instance_type: str, count: int) -> str:
    print(f"  -> terminating {count} {instance_type} instances in {asg}")
    return f"terminated:{asg}:{count}"

client = (fleetwrit.Client(agent_id="sre-remediation", environment="prod")
          if os.getenv("FLEETWRIT_URL")
          else testing.client(testing.auto_approve(), agent_id="sre-remediation", environment="prod"))

# in an AgentCore Runtime agent, register `gated` as the tool
gated = gate(client, build_action=lambda asg, instance_type, count:
             terminate_instances.action(asg=asg, instance_type=instance_type, count=count))(terminate_instances)

print("result:", gated(asg="web-prod", instance_type="m5.large", count=8))
```

## 5. Test it (offline, no server, no API key)

Save the script and run it:

```bash
python tutorial_agentcore.py
```

Expected output:

```
result: terminated:web-prod:8
```

(the `  -> terminating 8 m5.large instances in web-prod` line prints just before it.)

`fleetwrit.testing` is what makes this runnable with no server and no LLM: `testing.client(testing.auto_approve(), ...)` returns an in-memory client whose fake reviewer auto-approves every request, so the whole gate-and-resume loop runs in-process. The same script ships in the SDK repo at `examples/agentcore_tutorial.py` and is verified in CI.

## 6. Go live (a human decides)

Start the local stack — an empty server plus the dashboard on `http://localhost:4100`:

```bash
pip install "fleetwrit[dev-server]"
fleetwrit dev
```

In a second terminal, point the client at it and re-run:

```bash
export FLEETWRIT_URL=http://localhost:4100
python tutorial_agentcore.py
```

Because `FLEETWRIT_URL` is now set, the script uses a real `fleetwrit.Client` instead of the in-memory fake. The call blocks: the request waits in the dashboard for a human to **approve**, **decline** or **edit** the `count`. When the reviewer decides, the agent resumes on that exact decision — running the approved arguments, not necessarily the ones proposed — and carries a signed receipt.

## Gateway interceptor (experimental)

`gate` above is the recommended, fully-tested path. Alternatively, `gateway_handler` builds an AWS Lambda handler meant to gate at the AgentCore **Gateway** with no agent-code changes — but the exact Gateway→Lambda event shape is **not yet validated on live AWS**. Treat it as a pattern to adapt to your Gateway target, not a drop-in; prefer `gate` above unless you specifically need gateway-level gating. See [Integrations](/fleetwrit-site/docs/integrations) for details.

## Next steps

- [Integrations](/fleetwrit-site/docs/integrations) — how each adapter wraps its framework's own pause-and-resume.
- [Defining actions](/fleetwrit-site/docs/actions) — risk, summaries, display hints, redaction and versioning.
