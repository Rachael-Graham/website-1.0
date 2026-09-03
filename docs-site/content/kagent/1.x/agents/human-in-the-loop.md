---
title: Human in the loop
description: Understand how an agent pauses to ask a question or to get a tool call approved, and what a client does to answer it.
weight: 50
author: kagent.dev
---

An agent that only answers questions can run unattended. An agent that takes action often should not. The human in the loop (HITL) mechanism lets an agent stop mid-turn, return a question or a pending tool call to a person, and continue once that person answers.

> [!IMPORTANT]
> HITL is negotiated by the **client**, per call, rather than configured on an {{< gloss "AgentTemplate" >}}AgentTemplate{{< /gloss >}}. There is no field to switch it on. A client that does not ask for the extension still gets the pause: the agent stops, and the task waits. What it loses is the ability to answer, because the request reaches it as bare text with no correlation `id`. This differs from kagent 0.x, where a `requireApproval` list on the `Agent` resource decided which tools paused.

## How a pause works

The following diagram traces one turn in which the agent stops for a person.

```mermaid
flowchart TB
    caller["Client sends a message<br>requesting the HITL extension"]
    working["Agent works on the turn"]
    decision{"Does the agent need<br>a person?"}
    pause["Task state becomes<br>INPUT_REQUIRED"]
    request["Status message carries a<br>tool_approval_request<br>or ask_user_request"]
    answer["Client sends a response message<br>on the same task"]
    done["Agent finishes the turn"]

    caller --> working
    working --> decision
    decision -->|no| done
    decision -->|yes| pause
    pause --> request
    request --> answer
    answer --> working
```

The client opens the turn by sending a message that requests the HITL extension. The agent works until it either finishes, in which case the turn ends, or needs a person. When it needs a person, the task moves to `INPUT_REQUIRED` and its status message carries either a `tool_approval_request` or an `ask_user_request`. The client answers by sending a response message on the same task, and the agent resumes the turn where it left off.

## Pause kinds

An agent pauses either to get permission before it acts or to ask a question. Each case raises its own request.

| Request | Raised when | The client answers with |
| ------- | ----------- | ----------------------- |
| `tool_approval_request` | The agent wants to call a tool that asked for confirmation before it runs. | `tool_approval_response` |
| `ask_user_request` | The agent calls the built-in `ask_user` tool because it needs information only a person has. | `ask_user_response` |

Both use the same pause and resume mechanism, so a client that handles one can handle the other with a different payload.

The runtime's tool-confirmation mechanism decides which tool calls raise an approval, rather than kagent configuration. kagent does not keep a list of tools that need approval.

## Negotiate the extension

HITL is an [A2A](https://a2a-protocol.org) message extension, identified by a versioned URI. A client requests it by setting that URI as the `A2A-Extensions` header on the call that sends a message.

```http
A2A-Extensions: https://kagent.dev/extensions/hitl/v1
```

kagent activates the extension only for calls that request it, and echoes the activated URI back. A client that never requests the extension sees ordinary turns until the agent needs a person. The turn then pauses like any other, and that client has no way to answer the request.

A call from outside the cluster addresses the agent with two more headers, because the gateway routes on metadata rather than on a path. Port-forward the controller's gRPC port first, as in [Install kagent]({{< link path="setup/installation" >}}).

```bash
grpcurl -plaintext \
  -H 'A2A-Extensions: https://kagent.dev/extensions/hitl/v1' \
  -H 'x-kagent-agent-instance-namespace: kagent' \
  -H 'x-kagent-agent-instance-id: <instance-id>' \
  -d '{
    "message": {
      "message_id": "msg-1",
      "role": "ROLE_USER",
      "parts": [{"text": "Delete the obsolete pod in the production namespace."}]
    }
  }' localhost:8084 lf.a2a.v1.A2AService/SendStreamingMessage
```

When the agent pauses, the payload arrives in the status message's `metadata`, keyed by the extension URI. The URI is also listed in the message's `extensions` array. Each payload carries a `type` field that specifies its shape.

| Type | Direction |
| ---- | --------- |
| `tool_approval_request` | Agent to client |
| `ask_user_request` | Agent to client |
| `tool_approval_response` | Client to agent |
| `ask_user_response` | Client to agent |

> [!WARNING]
> In case of failure, both halves of this negotiation fail silently, and neither failure reports anything.
- **A send that omits the header produces a pause that cannot be answered.** The turn still stops, but its status message carries the question as prose, with no `metadata` and no correlation `id`, so there is nothing to render and no `id` to answer with. Re-reading that task with the header does not recover it, because the payload was never attached. Send the header on every call: it is harmless on a read, and unrecoverable if missed on a send. An attached payload is stored with the task, so a later read returns it whether or not that read requests the extension.
- **A response that omits the `extensions` array is delivered as ordinary text.** kagent ignores the `metadata` payload unless the message itself lists the extension URI in `extensions`. The task resumes and the agent replies, so the call looks like it worked, but the structured decision never reached the agent.

### Approving or rejecting a tool

A `tool_approval_request` lists the pending calls, each with an `id`, the tool `name`, and the `args` the agent chose. The response decides every listed call.

```json
{
  "type": "tool_approval_response",
  "approvals": [
    { "id": "<tool-id>", "approved": true },
    { "id": "<other-tool-id>", "approved": false, "rejection_reason": "Deleting that namespace is out of scope." }
  ]
}
```

A response must decide every call in the request. A rejection reason is optional but worth sending, because the agent receives it and can adapt rather than simply failing.

### Answering a question

An `ask_user_request` carries an `id` and a list of `questions`. The response echoes the same `id` and answers them in order.

```json
{
  "type": "ask_user_response",
  "id": "<request-id>",
  "answers": [
    { "answer": ["us-east-1"] }
  ]
}
```

## Resume a paused task

A paused task waits. To resume, the client sends a message on the same task and context, carrying the response payload. kagent rejects a resume attempt on a task that is not waiting, with `task is not waiting for input`.

Because the {{< gloss "Transcript" >}}transcript{{< /gloss >}} only grows, the question and the answer both stay in the task history, so a later reader can see what was asked and what a person decided.

## Task states

An A2A task moves through several states over its life. Two of them mean that the task has stopped and is waiting on a person, rather than working.

| State | Meaning |
| ----- | ------- |
| `INPUT_REQUIRED` | The agent is waiting for a person. This is the state that a tool approval or a question produces. |
| `AUTH_REQUIRED` | The agent is waiting for credentials. kagent's own runtimes never set this state, but its gateway accepts a resume from it, so a `byo` runtime that produces it works. |

## Agents bound as tools

An agent that a parent binds as a tool can raise a pause of its own. The request then carries a `nested` block naming the subagent, along with its task and context, so a client can tell the person which agent is actually asking rather than attributing it to the parent.

## Client support

Because the client negotiates HITL rather than the agent offering it, what a person can do with a pause depends on which client raised the turn.

| Client | HITL |
| ------ | ---- |
| Your own A2A client | Full. Request the extension URI and handle the four payload types. |
| An MCP client that supports tasks | Supported. `invoke_agent_instance` returns a task, and input requests surface as MCP elicitations. |
| The kagent CLI | Not supported, and a turn that pauses is stranded. `kagent invoke` does not request the extension, so an agent that needs a person parks the task at `INPUT_REQUIRED` with nothing to answer it by. The CLI reports `Input required to continue this AgentInstance.` and stops there. Send the turn again from a client that requests the extension. |

## Next steps

{{< cards >}}
  {{< card link=`{{< link path="skills-and-mcp/about-tools" >}}` title="About tools" subtitle="Bind the tools that an approval request would cover." >}}
  {{< card link=`{{< link path="agents/system-prompts" >}}` title="System prompts" subtitle="Tell an agent when to ask rather than act." >}}
{{< /cards >}}
