---
title: Your first agent
description: Apply a Harness and AgentTemplate, then create and talk to the AgentInstance they produce.
weight: 1
author: kagent.dev
---

This guide walks through the full path from applying a Harness and an AgentTemplate to holding a conversation with the AgentInstance they produce. [Core concepts]({{< link path="about/core-concepts" >}}) defines each of these terms, and [Architecture]({{< link path="about/architecture" >}}) covers how they fit together.

## Before you begin creating your first agent

- Install kagent with a WorkerPool provisioned. See [Installation]({{< link path="setup/installation" >}}).
- Install [grpcurl](https://github.com/fullstorydev/grpcurl).
- Enable gRPC reflection on the controller, so grpcurl can discover its methods without a local copy of kagent's proto files. Add `--set controller.grpc.reflection=true` to your Helm install or upgrade command.
- Port-forward the controller's gRPC port to your local machine:

  ```shell
  kubectl port-forward -n kagent svc/kagent-controller 8084:8084
  ```

The kagent CLI does not yet have commands for Harness, AgentTemplate, or AgentInstance. Until it does, this guide uses `kubectl` and `grpcurl` directly.

## Apply a Harness and an AgentTemplate

1. Apply a `ModelConfig` that names your model provider:

   ```yaml
   apiVersion: kagent.dev/v1alpha3
   kind: ModelConfig
   metadata:
     name: default-model-config
     namespace: kagent
   spec:
     provider: "OpenAI"
     model: "gpt-4.1-mini"
     apiKeySecret: kagent-openai
     apiKeySecretKey: OPENAI_API_KEY
   ```

   If you installed kagent with the Helm chart's default model provider settings, this `ModelConfig` already exists. Skip this step in that case.

2. Apply a `Harness` that uses kagent's native runtime:

   ```yaml
   apiVersion: kagent.dev/v1alpha3
   kind: Harness
   metadata:
     name: my-first-harness
     namespace: kagent
   spec:
     kagent: {}
     workload:
       image: <runtime-image>@sha256:<digest> # pin this to your kagent release's runtime image
     substrate:
       workerPoolRef:
         name: kagent-default
       snapshotPolicy:
         location: gs://<your-bucket>/kagent/
     allowedAgentTemplates:
       selector:
         matchLabels:
           kagent.dev/harness: my-first-harness
   ```

3. Apply an `AgentTemplate` labeled to match that selector:

   ```yaml
   apiVersion: kagent.dev/v1alpha3
   kind: AgentTemplate
   metadata:
     name: my-first-agent
     namespace: kagent
     labels:
       kagent.dev/harness: my-first-harness
   spec:
     description: My first kagent agent
     modelConfig:
       name: default-model-config
     systemPrompt: You are a concise, helpful assistant.
   ```

   An `AgentTemplate` has no field naming a Harness. The `kagent.dev/harness` label is only a convention this guide uses to match the selector above. Choose any label key and value, as long as the Harness selector and the AgentTemplate's labels agree.

4. Confirm the pair is ready:

   ```shell
   kubectl get agenttemplate my-first-agent -n kagent -o jsonpath='{.status.harnesses}'
   ```

   A ready pair reports a `Ready` condition for `my-first-harness`. If the status is empty, wait a few seconds for the kagent controller to reconcile, then check again.

## Create the AgentInstance

Once the pair is ready, create an AgentInstance from it:

```shell
grpcurl -plaintext \
  -d '{"namespace":"kagent","harness":"my-first-harness","agentTemplate":"my-first-agent","requestId":"'"$(uuidgen)"'"}' \
  localhost:8084 kagent.api.v1alpha1.AgentInstanceService/CreateAgentInstance
```

The response includes an `id` field. Save it. The next step needs it to address the AgentInstance you just created.

## Talk to your agent

Send a message to the AgentInstance over the A2A (Agent-to-Agent) protocol, using the `id` from the previous step:

```shell
grpcurl -plaintext \
  -H "x-kagent-agent-instance-namespace: kagent" \
  -H "x-kagent-agent-instance-id: <id-from-the-previous-step>" \
  -d '{"message":{"messageId":"'"$(uuidgen)"'","role":"ROLE_USER","parts":[{"text":"What is 2+2?"}]}}' \
  localhost:8084 lf.a2a.v1.A2AService/SendMessage
```

The response carries the agent's reply in the same `parts` shape as the request.

A future kagent CLI release will wrap the create-and-converse steps above into a single command.

## Clean up your first agent

```shell
kubectl delete agenttemplate my-first-agent -n kagent
kubectl delete harness my-first-harness -n kagent
```

Deleting the Harness and AgentTemplate does not delete the AgentInstance you created from them. Delete it directly through the same `AgentInstanceService` you used to create it.

## Next steps

- [Architecture: Agent Substrate]({{< link path="about/agent-substrate" >}}): what happens to your AgentInstance's Actor when it sits idle.
- [Agent harness]({{< link path="agents/agent-harness" >}}): the full set of Harness runtime options, including Claude Code and Codex.
- [Skills]({{< link path="skills-mcp/skills" >}}): give your agent capabilities beyond its system prompt.
