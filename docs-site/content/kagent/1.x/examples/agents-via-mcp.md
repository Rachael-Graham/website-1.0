---
title: Use agents from an MCP client
description: Connect Claude Code, Cursor, or another agent to kagent's MCP server, then discover and invoke your AgentInstances as tools.
weight: 20
author: kagent.dev
---

The kagent controller runs a {{< gloss "Model Context Protocol" >}}Model Context Protocol{{< /gloss >}} (MCP) server that exposes your {{< gloss "AgentInstance" >}}AgentInstances{{< /gloss >}} as tools. Any MCP client can then discover the agents in your cluster and delegate work to them. This mechanism allows one agent to orchestrate another as a sub-agent.

This example runs in the opposite direction to [Your first MCP tool]({{< link path="get-started/your-first-mcp-tool" >}}). There, kagent is the MCP client and an external server provides the tools. Here, kagent is the MCP server and your agents are the tools.

## Before you begin

1. [Install kagent]({{< link path="setup/installation" >}}).
2. [Create your first agent]({{< link path="get-started/your-first-agent" >}}), so that you have at least one AgentInstance in the `READY` state. The MCP server lists ready instances only.

## About the endpoint

The MCP server is part of the controller's HTTP port rather than a separate deployment, so a default installation already serves it at `/mcp` on port `8083`.

- **Transport**: Streamable HTTP only. Server-Sent Events (SSE) as a standalone transport and stdio are both unsupported, so a client that offers a transport choice has to use Streamable HTTP.
- **Sessions**: The handler is stateless, so each request stands alone and no session has to be established first.
- **Extensions**: The server advertises the `io.modelcontextprotocol/tasks` extension, which changes how invocations behave. For more information, see [Invoke without waiting](#invoke-without-waiting).

> [!WARNING]
> The open source build does not authenticate this endpoint. Every request is accepted, and the caller's identity is read from an `X-User-Id` header that the caller sets itself, defaulting to `admin@kagent.dev`. Because the endpoint can invoke agents, create checkpoints, and create AgentInstances, do not expose port `8083` outside the cluster. For the wider identity model and what the open source build does guarantee, see [Identity]({{< link path="substrate-runtime/identity" >}}).

## Connect a client

1. Forward the controller's HTTP port, and leave the command running.
   ```bash
   kubectl port-forward -n kagent svc/kagent-controller 8083:8083
   ```

2. Add `http://localhost:8083/mcp` to your client.
   {{< tabs >}}
   {{% tab name="Claude Code" %}}
   ```bash
   claude mcp add --transport http kagent http://localhost:8083/mcp
   ```
   Add `--scope project` to limit the entry to the current project rather than your user configuration.
   {{% /tab %}}
   {{% tab name="Cursor" %}}
   Add the server to your Cursor MCP settings.
   ```json
   {
     "mcpServers": {
       "kagent": {
         "url": "http://localhost:8083/mcp"
       }
     }
   }
   ```
   {{% /tab %}}
   {{% tab name="curl" %}}
   Useful for confirming the endpoint before you configure a client. Streamable HTTP replies are framed as events, so each response arrives on a `data:` line.
   ```bash
   curl -s -X POST http://localhost:8083/mcp \
     -H 'Content-Type: application/json' \
     -H 'Accept: application/json, text/event-stream' \
     -d '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2025-06-18","capabilities":{},"clientInfo":{"name":"curl","version":"0"}}}'
   ```
   Example output:
   ```console
   event: message
   data: {"jsonrpc":"2.0","id":1,"result":{"capabilities":{"extensions":{"io.modelcontextprotocol/tasks":{}},"tools":{"listChanged":true}},"protocolVersion":"2025-06-18","serverInfo":{"name":"kagent","version":"v1.0.0"}}}
   ```
   {{% /tab %}}
   {{< /tabs >}}

## Tools

The server exposes five tools. Two cover discovery and conversation, and three expose the {{< gloss "Checkpoint" >}}checkpoint{{< /gloss >}} operations, so a client can pin and branch an agent's state as well as talk to it. Every tool takes a `namespace`, because an AgentInstance is scoped to one.

| Tool | Required arguments | What it does |
| ---- | ------------------ | ------------ |
| `list_agent_instances` | `namespace` | Lists the ready AgentInstances that the caller created. Takes `match_labels`, `page_size`, and `page_token`. |
| `invoke_agent_instance` | `namespace`, `agent_instance_id`, `message` | Sends a message and returns the agent's reply. Takes `message_id` for idempotency. |
| `create_agent_instance_checkpoint` | `namespace`, `agent_instance_id` | Pins the conversation at a turn boundary. Takes `request_id` for idempotency. |
| `list_agent_instance_checkpoints` | `namespace`, `agent_instance_id` | Lists that instance's checkpoints. Takes `page_size` and `page_token`. |
| `fork_agent_instance` | `namespace`, `checkpoint_id` | Creates a new AgentInstance from a checkpoint. Takes `request_id` for idempotency. |

> [!NOTE]
> The tools take no session or conversation argument, because an AgentInstance **is** the conversation. Sending a second message to the same `agent_instance_id` continues where the first left off, and the reply's `context_id` matches the instance's own ID. To hold two independent conversations on one AgentTemplate, create two AgentInstances.

## List and invoke an agent

1. Ask your client to list the agents in the `kagent` namespace. A client calls `list_agent_instances`, which returns one line per instance plus the same data as structured content. Example output:
   ```console
   kagent/01a068e3-aeb6-7abc-8d6f-5ba9becd3143 (my-first-agent via my-first-harness)
   ```

   An empty list where you expect instances is almost always creator scoping rather than a missing agent. The tool returns only the instances that the calling identity created, and it has no option to widen that. The kagent command line interface and this endpoint both default to `admin@kagent.dev`, so they see each other's instances. If you created the instance with `kagent --user-id <someone-else>`, send a matching `X-User-Id` header from the client.

2. Ask the agent a question, naming the instance you want. A client calls `invoke_agent_instance` and blocks until the agent answers.
   ```json
   {
     "namespace": "kagent",
     "agent_instance_id": "01a068e3-aeb6-7abc-8d6f-5ba9becd3143",
     "message": "What is 2+2? Answer with just the number."
   }
   ```

   The reply text comes back as the tool's content, with the task identifiers alongside it. Example output:
   ```json
   {
     "namespace": "kagent",
     "agent_instance_id": "01a068e3-aeb6-7abc-8d6f-5ba9becd3143",
     "task_id": "01a0690e-65d2-7d62-b69d-2ecdcc8064b2",
     "context_id": "01a068e3-aeb6-7abc-8d6f-5ba9becd3143",
     "state": "TASK_STATE_COMPLETED",
     "text": "4"
   }
   ```

3. Ask a follow-up question against the same instance to confirm that the conversation continues. The agent answers with the earlier turns in context, because the transcript belongs to the AgentInstance rather than to the client.

## Invoke without waiting

A client that declares the `io.modelcontextprotocol/tasks` extension gets a different result from the same tool. Rather than blocking, `invoke_agent_instance` returns immediately with a task to poll. This behavior keeps a long agent run from holding a request open.

The task carries an opaque ID of the form `v1.<encoded reference>` that identifies the namespace, the instance, and the A2A task together, so pass it back verbatim rather than parsing it. Example output:
```json
{
  "taskId": "v1.eyJuYW1lc3BhY2UiOiJrYWdlbnQiLCJpbnN0YW5jZUlkIjoi...",
  "status": "working",
  "createdAt": "2026-09-03T20:56:27.277853198Z",
  "lastUpdatedAt": "2026-09-03T20:56:27.277853198Z",
  "ttlMs": null,
  "pollIntervalMs": 1000,
  "resultType": "task"
}
```

Poll it with `tasks/get`. The status reports `working`, `input_required`, `completed`, or `cancelled`, and `pollIntervalMs` suggests waiting a second between calls. Once the run finishes, `resultType` changes to `complete` and a `result` field holds the same content that a blocking call would have returned.
```json
{
  "status": "completed",
  "statusMessage": "One, two, three.",
  "resultType": "complete",
  "result": {
    "content": [{ "type": "text", "text": "One, two, three." }]
  }
}
```

Two more methods complete the set. `tasks/cancel` stops a run that is still working, and `tasks/update` answers an agent that is waiting on a person.

> [!NOTE]
> A `working` task's `statusMessage` holds the raw task record rather than a readable sentence, because the agent has not produced any text yet. Read `status` to decide whether to keep polling, and take the answer from `result` once `resultType` is `complete`.

## Answer an agent's question

When an agent pauses to ask something, the task's status becomes `input_required` and `tasks/get` returns an `inputRequests` entry describing what the agent needs. kagent renders that as an MCP elicitation, so a client presents its own prompt and sends the answer back with `tasks/update`.

The schema kagent builds depends on what the agent asked for.

- **A question** becomes one string field per question, named `response` when the agent asks one question and `response_1`, `response_2`, and so on when it asks more than one. A question with a fixed set of choices restricts the field to those values, and one that accepts more than one answer takes an array.
- **A tool approval** becomes one boolean field per tool, named `approve_1`, `approve_2`, and so on. A response has to decide every tool in the request.

An elicitation result of `accept` sends the answers on, while `decline` and `cancel` tell the agent that the person refused, which the agent can then adapt to rather than treating as an error.

This is the same pause that any other client sees, reached through MCP instead of A2A. For the pause types, the approval model, and what the agent receives, see [Human in the loop]({{< link path="agents/human-in-the-loop" >}}).

> [!IMPORTANT]
> Only a client that declares the tasks extension can answer an agent. A blocking `invoke_agent_instance` call has nowhere to surface the question, so an agent that pauses leaves that call waiting.

## Checkpoint and fork from a client

The three checkpoint tools give an MCP client the same operations that the [Agent Substrate example]({{< link path="examples/agent-substrate" >}}) performs from the command line. A client might pin a known-good point before letting an agent try something risky, then start a second agent from that point later, on the revision the checkpoint captured rather than whatever the AgentTemplate has become.

Note that a fork begins its own conversation rather than continuing the original's, so plan for the two AgentInstances to share a starting state and nothing else. For what a fork does and does not inherit, see the [Agent Substrate example]({{< link path="examples/agent-substrate" >}}).

1. Create a checkpoint with `create_agent_instance_checkpoint`. The result identifies the checkpoint and the turn it pinned. Example output:
   ```json
   {
     "checkpoint": {
       "id": "01a0690f-5548-7935-b7ca-70919fc9c221",
       "namespace": "kagent",
       "agent_instance_id": "01a068e3-aeb6-7abc-8d6f-5ba9becd3143",
       "head_task_id": "01a0690f-058d-7d29-a880-9b5d6d30b772",
       "history_sequence": 91,
       "state": "CHECKPOINT_STATE_READY"
     }
   }
   ```

2. Fork it with `fork_agent_instance`, passing the `checkpoint_id`. The result is a new AgentInstance with its own ID, already `READY`. Example output:
   ```json
   {
     "agent_instance": {
       "id": "01a0690f-6df2-78fc-b956-5fd1d42f02c2",
       "namespace": "kagent",
       "harness": "my-first-harness",
       "agent_template": "my-first-agent",
       "state": "AGENT_INSTANCE_STATE_READY"
     }
   }
   ```

3. Invoke the fork with `invoke_agent_instance` and its new ID. The fork runs the {{< gloss "Revision" >}}revision{{< /gloss >}} its checkpoint was taken on, so editing the AgentTemplate afterwards does not change what the fork runs.

For what a checkpoint captures and why a checkpoint taken on a suspended instance is the forkable kind, see [Suspend and resume]({{< link path="substrate-runtime/suspend-and-resume" >}}).

## Next steps

{{< cards >}}
  {{< card link=`{{< link path="examples/agent-substrate" >}}` title="Agent Substrate" subtitle="Follow suspend, checkpoint, and fork from the command line instead." >}}
  {{< card link=`{{< link path="agents/human-in-the-loop" >}}` title="Human in the loop" subtitle="Understand the pauses an agent can raise and how a client answers them." >}}
  {{< card link=`{{< link path="get-started/your-first-mcp-tool" >}}` title="Your first MCP tool" subtitle="Point kagent at an MCP server so that your agent gains tools of its own." >}}
{{< /cards >}}
