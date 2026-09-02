---
title: Google Vertex AI
description: Understand why the Google Vertex AI providers do not currently run on a kagent Harness.
weight: 20
author: kagent.dev
---

The `GeminiVertexAI` and `AnthropicVertexAI` providers reach Gemini and Claude models through Google Cloud Vertex AI. Both providers exist in the {{< gloss "ModelConfig" >}}ModelConfig{{< /gloss >}} schema, but neither currently runs on a kagent {{< gloss "Harness" >}}Harness{{< /gloss >}}.

> [!WARNING]
> **Vertex AI is not currently supported.** Vertex AI authenticates with a Google Application Default Credentials file rather than with an API key. When `apiKeySecret` is set, kagent mounts that credentials file into the agent, and mounting a file is not something an agent running on {{< gloss "Agent Substrate" >}}Agent Substrate{{< /gloss >}} can do. The AgentTemplate reports the `Compatible` condition as `False` with the message `ModelConfig requires volume mounts unsupported by Substrate ActorTemplate`, and no runtime {{< gloss "Revision" >}}revision{{< /gloss >}} is compiled.

## Why the configuration fails

kagent passes model credentials to an agent as environment variables. Every provider whose credential is a string, such as an API key, works on a Harness. Vertex AI is different: the Google credentials are a JSON file, so kagent sets `GOOGLE_APPLICATION_CREDENTIALS` to a path and mounts the Secret at that path. An agent runs as a Substrate Actor rather than as a pod that kagent controls, so there is nowhere to mount it.

Omitting `apiKeySecret` avoids the mount, because kagent then expects Google credentials to be already present in the agent's environment. An Actor's sandbox does not inherit cloud workload identity from the node, so that path does not authenticate either.

For the full explanation of which configurations a Harness can run, see [About model providers]({{< link path="setup/model-providers/about-model-providers" >}}).

## What to use instead

| Goal | Alternative |
| ---- | ----------- |
| Gemini models | Use the [Gemini]({{< link path="setup/model-providers/gemini" >}}) provider, which reaches the same model family through the Google AI Studio API with an API key. |
| Claude models | Use the [Anthropic]({{< link path="setup/model-providers/anthropic" >}}) provider, or reach Claude through [Amazon Bedrock]({{< link path="setup/model-providers/amazon-bedrock" >}}). |
| Vertex AI specifically | Route Vertex AI through a gateway that presents an OpenAI-compatible API and authenticates to Google itself, then point an [OpenAI-compatible endpoint]({{< link path="setup/model-providers/byo-openai" >}}) ModelConfig at the gateway. |

## Next steps

{{< cards >}}
  {{< card link=`{{< link path="setup/model-providers/gemini" >}}` title="Gemini" subtitle="Reach Gemini models with an API key instead." >}}
  {{< card link=`{{< link path="setup/model-providers/about-model-providers" >}}` title="About model providers" subtitle="Understand which provider configurations a Harness can run." >}}
{{< /cards >}}
