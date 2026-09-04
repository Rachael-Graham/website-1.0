---
title: Call an agent over A2A
description: Use the A2A service that the kagent controller serves to read an AgentInstance's agent card, send it a message, and stream a reply.
weight: 30
author: kagent.dev
---

Every {{< gloss "AgentInstance" >}}AgentInstance{{< /gloss >}} is reachable over the {{< gloss "A2A" >}}A2A{{< /gloss >}} (Agent-to-Agent) protocol through the kagent controller. A2A is not a side door; it is how kagent talks to its own agents. The CLI, the [MCP server]({{< link path="examples/agents-via-mcp" >}}), and any client you write all take the same path.

This example uses [grpcurl](https://github.com/fullstorydev/grpcurl) to show the requests and replies directly. Real callers use an A2A client library rather than assembling requests by hand.

## About the kagent A2A service

The controller serves `lf.a2a.v1.A2AService` on its gRPC port, `8084`, which is the same port and service that the kagent CLI uses.

An AgentInstance is not addressed by a URL path. A caller names the instance in two pieces of request metadata, and the controller routes the call to that instance's Actor.

| Metadata header | Value |
| --------------- | ----- |
| `x-kagent-agent-instance-namespace` | The namespace holding the AgentInstance. |
| `x-kagent-agent-instance-id` | The AgentInstance's ID. |

> [!NOTE]
> Header routing replaces the `/api/a2a/<namespace>/<agent-name>/` URL paths that kagent 0.x served over HTTP. The unit you address also changed: a 0.x caller addressed an agent, while a 1.x caller addresses one AgentInstance, which is one conversation with that agent.

> [!WARNING]
> The open source build does not authenticate this port. Any caller that can reach it can invoke any AgentInstance, so do not expose port `8084` outside the cluster. For what the open source build does guarantee, see [Identity]({{< link path="substrate-runtime/identity" >}}).

## Before you begin

1. [Install kagent]({{< link path="setup/installation" >}}).
2. [Create your first agent]({{< link path="get-started/your-first-agent" >}}), and save the AgentInstance's ID.
   ```bash
   export INSTANCE_ID=<your-agent-instance-id>
   ```

3. Install [grpcurl](https://github.com/fullstorydev/grpcurl), and confirm that your kagent installation sets `controller.grpc.reflection=true`. Reflection lets grpcurl discover the service without a local copy of the A2A protocol buffer definitions.

4. Port-forward the controller's gRPC port, and leave the command running.
   ```bash
   kubectl port-forward -n kagent svc/kagent-controller 8084:8084
   ```

## Read the agent card

An A2A client normally starts by reading the agent card, which tells it what the agent is and which protocol features the agent supports.

1. Fetch the card for your AgentInstance.
   ```bash
   grpcurl -plaintext \
     -H "x-kagent-agent-instance-namespace: kagent" \
     -H "x-kagent-agent-instance-id: $INSTANCE_ID" \
     localhost:8084 lf.a2a.v1.A2AService/GetExtendedAgentCard
   ```

   Example output:
   ```json
   {
     "name": "my_first_agent",
     "description": "My first kagent agent",
     "supportedInterfaces": [
       {
         "url": "http://kagent-controller.kagent.svc:8084",
         "protocolBinding": "GRPC",
         "protocolVersion": "1.0"
       }
     ],
     "version": "v1",
     "capabilities": {
       "streaming": true,
       "pushNotifications": false,
       "extensions": [
         {
           "uri": "https://kagent.dev/extensions/hitl/v1",
           "description": "Human in the loop for tool approval, ask user, and nested subagents"
         }
       ],
       "extendedAgentCard": true
     },
     "defaultInputModes": ["text"],
     "defaultOutputModes": ["text"]
   }
   ```

2. Read the card for the three things a caller acts on. The `name` field is the {{< gloss "AgentTemplate" >}}AgentTemplate{{< /gloss >}}'s name with hyphens replaced by underscores, and `description` comes from the AgentTemplate's `spec.description`, so the description a caller sees is the one you wrote. The `supportedInterfaces` URL is the controller's in-cluster address rather than the Actor's, because a caller reaches the agent through the controller. The `extensions` list advertises human-in-the-loop support, which a client opts into per call.

> [!NOTE]
> The card carries no `skills`. kagent 0.x let you declare agent card skills in an `a2aConfig` block, and v1alpha3 has no such field, so kagent generates the card from the AgentTemplate's name and description alone. An AgentTemplate's `spec.skills` field is a different feature: those are [Agent Skills]({{< link path="skills-and-mcp/skills" >}}) that the agent can use, not advertisements to a caller.

## Send a message

`SendMessage` blocks until the agent finishes, then returns the whole task. A message needs its own ID, a role, and at least one part.

1. Send a message to the AgentInstance.
   ```bash
   grpcurl -plaintext \
     -H "x-kagent-agent-instance-namespace: kagent" \
     -H "x-kagent-agent-instance-id: $INSTANCE_ID" \
     -d '{"message":{"messageId":"'"$(uuidgen)"'","role":"ROLE_USER","parts":[{"text":"What is 7 times 6? Answer with just the number."}]}}' \
     localhost:8084 lf.a2a.v1.A2AService/SendMessage
   ```

   The reply text arrives in `artifacts`, not in `status`. Example output, with the message history omitted:
   ```json
   {
     "task": {
       "id": "01a06cfb-a9ae-7ddb-be98-baaf17414998",
       "contextId": "01a068e3-aeb6-7abc-8d6f-5ba9becd3143",
       "status": {
         "state": "TASK_STATE_COMPLETED",
         "timestamp": "2026-09-04T15:13:52.990137169Z"
       },
       "artifacts": [
         {
           "artifactId": "01a06cfb-bf2c-70ea-8e65-e3ef15133a96",
           "parts": [{ "text": "42" }]
         }
       ],
       "history": [ ]
     }
   }
   ```

2. Save the task ID to read the task again later.
   ```bash
   export TASK_ID=<your-task-id>
   ```

Two identifiers come back, and they mean different things. The `id` is one turn, and a new one appears on every message. The `contextId` is the conversation, and it is the AgentInstance's own ID, which is why a second message to the same instance continues the conversation rather than starting a new one. Each artifact also carries runtime metadata under `adk_` keys, including the token counts for that turn.

## Stream a reply

`SendStreamingMessage` takes the same request and returns a sequence of events instead of one result. A caller can then show a reply as the agent produces it.

Send a message on the streaming method.
```bash
grpcurl -plaintext \
  -H "x-kagent-agent-instance-namespace: kagent" \
  -H "x-kagent-agent-instance-id: $INSTANCE_ID" \
  -d '{"message":{"messageId":"'"$(uuidgen)"'","role":"ROLE_USER","parts":[{"text":"Count from 1 to 3."}]}}' \
  localhost:8084 lf.a2a.v1.A2AService/SendStreamingMessage
```

The stream opens with the task at `TASK_STATE_SUBMITTED`, moves to `TASK_STATE_WORKING`, and then emits an artifact update for each chunk of the reply. Every chunk shares one `artifactId`, so a client appends them into a single artifact rather than treating each as a separate answer. Example output, abbreviated to the text of each event:
```console
"state": "TASK_STATE_SUBMITTED"
"state": "TASK_STATE_WORKING"
"artifactId": "01a06cfd-1c3c-7e65-8650-2e86f5d7f5eb", "text": "1"
"artifactId": "01a06cfd-1c3c-7e65-8650-2e86f5d7f5eb", "text": ","
"artifactId": "01a06cfd-1c3c-7e65-8650-2e86f5d7f5eb", "text": " "
"artifactId": "01a06cfd-1c3c-7e65-8650-2e86f5d7f5eb", "text": "2"
```

## Read a task later

A task outlives the call that created it, so a caller that lost its connection can read the result rather than asking the agent again.

Read the task by ID.
```bash
grpcurl -plaintext \
  -H "x-kagent-agent-instance-namespace: kagent" \
  -H "x-kagent-agent-instance-id: $INSTANCE_ID" \
  -d '{"id":"'"$TASK_ID"'"}' \
  localhost:8084 lf.a2a.v1.A2AService/GetTask
```

The task comes back with the same `status`, `artifacts`, and `history` that `SendMessage` returned. A task that is still running reports `TASK_STATE_WORKING` and has no artifacts yet, and `CancelTask` takes the same `id` to stop it.

## When an agent needs a person

An agent can stop mid-task to ask a question or to request approval for a tool call. The task then reports `TASK_STATE_INPUT_REQUIRED` and waits, and the caller answers by sending a message that carries the same task ID.

A client only sees these pauses if it requests the human-in-the-loop extension that the agent card advertises. A client that never requests it is never interrupted. For the pause types, the approval model, and what the agent receives back, see [Human in the loop]({{< link path="agents/human-in-the-loop" >}}).

## Clean up

This example creates no Kubernetes resources, so there is nothing to delete. Stop the port-forward with `Ctrl+C`. The tasks that your messages created stay on the AgentInstance as part of its conversation, and deleting the AgentInstance removes them with it.

## A2A method reference

These are the methods that this example uses. The service defines more, including `ListTasks` and the push notification configuration calls, but an agent card that reports `pushNotifications` as `false` does not support being called back.

| Method | What it does |
| ------ | ------------ |
| `GetExtendedAgentCard` | Returns the agent card describing the instance. |
| `SendMessage` | Sends a message and returns the finished task. |
| `SendStreamingMessage` | Sends a message and streams events as the agent works. |
| `GetTask` | Reads a task that a previous call created. |
| `CancelTask` | Stops a task that is still running. |

## Next steps

{{< cards >}}
  {{< card link=`{{< link path="examples/agents-via-mcp" >}}` title="Use agents from an MCP client" subtitle="Reach the same agents through MCP instead, from Claude Code or Cursor." >}}
  {{< card link=`{{< link path="agents/human-in-the-loop" >}}` title="Human in the loop" subtitle="Handle an agent that pauses to ask a question or request approval." >}}
  {{< card link=`{{< link path="observability/tracing" >}}` title="Tracing" subtitle="Follow one A2A call from the controller through to the Actor that served it." >}}
{{< /cards >}}
