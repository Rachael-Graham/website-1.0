---
title: About tools
description: Give an agent tools by binding Model Context Protocol servers or other agents to an AgentTemplate.
weight: 10
author: kagent.dev
---

An {{< gloss "AgentTemplate" >}}AgentTemplate{{< /gloss >}}'s `spec.tools` list defines what an agent can do beyond its system prompt. Each entry is a **{{< gloss "Tool binding" >}}tool binding{{< /gloss >}}**, and every binding selects exactly one source: a {{< gloss "Model Context Protocol" >}}Model Context Protocol{{< /gloss >}} (MCP) server, or another AgentTemplate used as a tool. A binding that names both, or neither, is rejected.

- `mcp`: Binds tools from an MCP server.
- `agent`: Binds another AgentTemplate, so that the agent can hand work to it.

Both kinds resolve within the AgentTemplate's own namespace, so a binding cannot reach a server or a template in another namespace.

## MCP tools

An `mcp` binding names a server and the tools to take from it. Naming the tools is required. A binding takes between 1 and 50 named tools, never the server's whole catalog.

```yaml
tools:
  - mcp:
      server:
        kind: RemoteMCPServer
        name: my-mcp-server
      tools:
        - search_docs
        - fetch_page
```

| Field | Description |
| ----- | ----------- |
| `mcp.server.kind` | The kind of server resource. `RemoteMCPServer` is the only accepted value. |
| `mcp.server.name` | The server's name, in the AgentTemplate's namespace. |
| `mcp.tools` | The names of the tools to bind, between 1 and 50. Duplicates in the list are collapsed. |

## Agents as tools

An `agent` binding points at another AgentTemplate, which lets one agent route work to another. The model reads the `description` when it decides whether to route work here, so a description that states plainly what the bound agent is for matters more than the detail of its configuration.

```yaml
tools:
  - agent:
      name: log-searcher
      description: Search application logs for a time range and a query string.
      templateRef:
        name: log-search-agent
      isolation: Shared
```

| Field | Description |
| ----- | ----------- |
| `agent.name` | The name that the model sees for this binding. |
| `agent.description` | The text that tells the parent agent when to route work here. |
| `agent.templateRef.name` | The AgentTemplate to bind, in the same namespace. |
| `agent.isolation` | The [isolation mode](#shared-and-dedicated-isolation) for the bound agent. Currently, `Shared` is the only supported value. |

### Shared and Dedicated isolation

The isolation setting determines whether a bound agent runs inside its parent's runtime boundary, or runs within a boundary of its own.

- **`Shared`**: The bound agent runs inside the parent's {{< gloss "Actor" >}}Actor{{< /gloss >}}. Nesting costs no extra compute, and the two agents share one sandbox.
- **`Dedicated`**: The bound agent would run in its own Actor, with its own sandbox and its own suspend and resume cycle.

> [!WARNING]
> `Dedicated` is not currently implemented. The AgentTemplate schema accepts the value, but compiling a binding that uses it fails with `Dedicated AgentTemplate tools are not supported yet`, and the pair does not become ready. Use `Shared`, which is the default.

### What a Shared tree allows

A `Shared` binding nests one agent inside another's runtime, so kagent constrains the shape of the resulting tree. The compiler enforces each of the following rules, and a violation surfaces as a failed {{< gloss "Revision" >}}revision{{< /gloss >}} rather than a failure at run time.

- **One level of nesting.** A bound agent cannot itself bind another agent. A second consecutive binding is rejected as exceeding the kagent runtime boundary.
- **No cycles.** An AgentTemplate cannot reach itself through a chain of bindings.
- **No reuse within one tree.** The same AgentTemplate cannot appear twice in the same tree.
- **Unique binding names.** Two bindings on one AgentTemplate cannot share a `name`.
- **The bound template must be admitted too.** A nested AgentTemplate has to match the same {{< gloss "Harness" >}}Harness{{< /gloss >}}'s `allowedAgentTemplates` selector. Binding a template that the Harness does not admit is rejected.
