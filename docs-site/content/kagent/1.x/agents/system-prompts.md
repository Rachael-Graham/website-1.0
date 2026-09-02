---
title: System prompts
description: Set an agent's system prompt inline or from a ConfigMap, and template it with values that kagent resolves at compile time.
weight: 30
author: kagent.dev
---

An {{< gloss "AgentTemplate" >}}AgentTemplate{{< /gloss >}}'s system prompt defines the agent's role and how it should behave. kagent resolves the prompt when it compiles a {{< gloss "Revision" >}}revision{{< /gloss >}}, so the text an agent runs with is fixed for the life of an {{< gloss "AgentInstance" >}}AgentInstance{{< /gloss >}}: editing the prompt affects instances created after the edit compiles, not ones already running.

## Write an effective prompt

A prompt that works tends to carry four things, in roughly this order.

- **Role.** What the agent is, in a sentence. `You are a Kubernetes assistant.`
- **Scope.** What it should and should not take on, which matters more as you give it more tools.
- **Instructions.** How to behave in the cases you care about: when to ask for clarification, what to do when a tool fails, when to refuse.
- **Response format.** What a good answer looks like. Ask for Markdown, or for a summary before detail, if that is what you want.

Two things are worth stating explicitly, because models otherwise guess: what the agent should do when it does not know an answer, and whether it should act or ask first when an action is consequential. For approval gates that the runtime enforces rather than the prompt, see [Human in the loop]({{< link path="agents/human-in-the-loop" >}}).

## Set the prompt inline

```yaml
spec:
  systemPrompt: |-
    You are a Kubernetes assistant. You help users understand what is running in their cluster.

    # Instructions
    - Ask for clarification before running a tool when a request is ambiguous.
    - Answer from tool output rather than from memory of how clusters usually look.
    - Say so plainly when a question cannot be answered with the tools you have.
```

## Store the prompt in a ConfigMap

Use `systemPromptFrom` to keep the prompt outside the AgentTemplate, which lets several AgentTemplates share one prompt, or lets a prompt change without editing the agent.

1. Create a ConfigMap holding the prompt.
   ```yaml
   kubectl apply -f - <<EOF
   apiVersion: v1
   kind: ConfigMap
   metadata:
     name: shared-prompts
     namespace: kagent
   data:
     kubernetes-assistant: |-
       You are a Kubernetes assistant. You help users understand what is running in their cluster.
   EOF
   ```

2. Reference the key from the AgentTemplate.
   ```yaml
   spec:
     systemPromptFrom:
       name: shared-prompts
       key: kubernetes-assistant
   ```

| Field | Description |
| ----- | ----------- |
| `systemPromptFrom.name` | The ConfigMap holding the prompt, in the AgentTemplate's namespace. |
| `systemPromptFrom.key` | The key within that ConfigMap. |

`systemPrompt` and `systemPromptFrom` are mutually exclusive, and an AgentTemplate that sets both is rejected.

> [!NOTE]
> A prompt can come only from a ConfigMap. Earlier versions of kagent also accepted a Secret, through a `systemMessageFrom.type` field that v1alpha3 does not have. A system prompt is not a credential, so keep secrets out of it and pass them to the runtime as [Harness environment variables]({{< link path="agents/agent-harness" >}}) instead.

## Template the prompt

Set `spec.promptTemplate` to run the prompt through [Go templates](https://pkg.go.dev/text/template) before it reaches the model. Templating applies to whichever prompt you set, inline or from a ConfigMap.

```yaml
spec:
  description: Answers questions about a Kubernetes cluster.
  systemPrompt: |-
    You are {{ .AgentTemplateName }}. {{ .Description }}

    You have these tools available: {{ .ToolNames }}

    {{ include "shared-prompts/response-format" }}
  promptTemplate:
    dataSources:
      - name: shared-prompts
```

### Values available to a template

| Value | Description |
| ----- | ----------- |
| `.AgentTemplateName` | The AgentTemplate's `metadata.name`. |
| `.AgentTemplateNamespace` | The AgentTemplate's `metadata.namespace`. |
| `.Description` | The AgentTemplate's `spec.description`. |
| `.ToolNames` | The tool names selected from every MCP server the AgentTemplate binds. Agents bound as tools are not included. |

`.ToolNames` is worth using rather than listing tools by hand, because it cannot drift from the bindings that the AgentTemplate actually declares.

> [!NOTE]
> A prompt template can reach only these values and the ConfigMaps that `dataSources` names. Templates cannot read arbitrary Kubernetes objects.

### Include text from a ConfigMap

The `include` function pulls in one key from a ConfigMap that `dataSources` lists. Its argument is `"<source>/<key>"`, where the source is the ConfigMap's name.

```yaml
spec:
  promptTemplate:
    dataSources:
      - name: shared-prompts
      - name: house-style
        alias: style
```

| Field | Description |
| ----- | ----------- |
| `dataSources[].name` | A ConfigMap in the AgentTemplate's namespace. Up to 20 entries. |
| `dataSources[].alias` | An alternative identifier to use in `include` paths. The ConfigMap is still looked up by `name`. |

An alias only changes what you type. With the preceding configuration, `include "house-style/tone"` fails and `include "style/tone"` succeeds.

Every key in every listed ConfigMap becomes available, so two sources that share a key name collide. kagent rejects that at compile time rather than picking one.

## Troubleshooting

Prompt problems surface on the AgentTemplate's `ResolvedRefs` condition, because they are reference failures rather than runtime errors. No revision compiles, so no new AgentInstance can start.

| Message | Cause |
| ------- | ----- |
| `resolve systemPromptFrom: ConfigMap "x" not found` | The ConfigMap does not exist in the AgentTemplate's namespace. |
| `resolve systemPromptFrom: ConfigMap "x" does not contain key "y"` | The ConfigMap exists but has no such key. |
| `resolve prompt source "x": ConfigMap not found` | A `dataSources` entry names a ConfigMap that does not exist. |
| `duplicate prompt template identifier "x/y"` | Two data sources expose the same `source/key` path. Give one of them an `alias`. |
| `prompt template "x/y" not found, available: [...]` | An `include` path does not match any available key. The error lists every path that is available. |
| `parse system message template: ...` | The template is not valid Go template syntax. |

## Next steps

{{< cards >}}
  {{< card link=`{{< link path="skills-and-mcp/about-tools" >}}` title="About tools" subtitle="Bind the tools that a prompt can refer to." >}}
  {{< card link=`{{< link path="agents/agent-memory" >}}` title="Agent memory" subtitle="Let an agent carry what it learned into later conversations." >}}
{{< /cards >}}
