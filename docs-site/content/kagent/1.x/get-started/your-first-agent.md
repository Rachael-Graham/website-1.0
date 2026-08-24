---
title: Your first agent
description: Create and communicate with your first agent by using the kagent project.
weight: 10
author: kagent.dev
---

This guide walks you through creating an agent, from applying a Harness and an AgentTemplate to holding a conversation with the AgentInstance that they produce. For definitions of each of these components, review the [core concepts]({{< link path="about/core-concepts" >}}). For an overview of how each component fits together in kagent, review the [architecture]({{< link path="about/architecture" >}}).

## Before you begin

1. [Install kagent with a WorkerPool provisioned]({{< link path="setup/installation" >}}). Be sure to add `--set controller.grpc.reflection=true` to the Helm install command so that grpcurl can discover the controller's gRPC methods without a local copy of kagent's proto files.
2. Install [grpcurl](https://github.com/fullstorydev/grpcurl).
3. Port-forward the controller's gRPC port to your local machine.
   ```shell
   kubectl port-forward -n kagent svc/kagent-controller 8084:8084
   ```

> [!NOTE]
> This guide uses `kubectl` and `grpcurl` directly, as the kagent CLI does not yet have commands for Harness, AgentTemplate, or AgentInstance.

## Create a Harness and an AgentTemplate

1. Apply a `Harness` that uses kagent's native runtime.
   ```yaml
   apiVersion: kagent.dev/v1alpha3
   kind: Harness
   metadata:
     name: my-first-harness
     namespace: kagent
   spec:
     kagent: {}
     workload:
       # Your kagent release's runtime image
       image: <runtime-image>@sha256:<digest>
     substrate:
       workerPoolRef:
         name: kagent-default
       snapshotPolicy:
         # The object storage location your cluster's Substrate installation uses for Actor snapshots
         location: gs://<your-bucket>/kagent/
     allowedAgentTemplates:
       selector:
         matchLabels:
           # Selector to match the AgentTemplate label
           kagent.dev/harness: my-first-harness
   ```
   > [!NOTE]
   > An `AgentTemplate` has no field naming this Harness. The `kagent.dev/harness: my-first-harness` selector is a convention that this guide uses to match the `kagent.dev/harness` label in the next step. However, you can choose any label key and value, as long as the Harness selector and the AgentTemplate's labels match.

2. Apply an `AgentTemplate` that is labeled to match the Harness's `allowedAgentTemplates` selector. The `ModelConfig` field references the `default-model-config` that was automatically created for the model provider API key that you provided during kagent installation.
   ```yaml
   apiVersion: kagent.dev/v1alpha3
   kind: AgentTemplate
   metadata:
     name: my-first-agent
     namespace: kagent
     labels:
       # Label matching the Harness selector
       kagent.dev/harness: my-first-harness
   spec:
     description: My first kagent agent
     modelConfig:
       # Default config created by the kagent install guide
       name: default-model-config
     systemPrompt: You are a concise, helpful assistant.
   ```

3. Confirm that the pair is ready.
   ```shell
   kubectl get agenttemplate my-first-agent -n kagent -o jsonpath='{.status.harnesses}' | jq .
   ```

   A ready pair reports a `Ready` condition for `my-first-harness`. If the status is empty, wait a few seconds for the kagent controller to reconcile, then check again.
   ```json
   [
     {
       "harness": "my-first-harness",
       "desiredRevision": "sha256:5f2b3c1a9e8d",
       "latestSuccessfulRevision": "sha256:5f2b3c1a9e8d",
       "conditions": [
         {
           "type": "Ready",
           "status": "True",
           "reason": "Ready",
           "message": "ActorTemplate is ready",
           "lastTransitionTime": "2026-08-24T15:02:10Z"
         }
       ]
     }
   ]
   ```

## Create the AgentInstance

1. Create an AgentInstance from the Harness and AgentTemplate pair.
   ```shell
   RESPONSE=$(grpcurl -plaintext \
     -d '{"namespace":"kagent","harness":"my-first-harness","agentTemplate":"my-first-agent","requestId":"'"$(uuidgen)"'"}' \
     localhost:8084 kagent.api.v1alpha1.AgentInstanceService/CreateAgentInstance)
   echo "$RESPONSE"
   ```

   A successful response includes the new AgentInstance and its `id`.
   ```json
   {
     "agentInstance": {
       "id": "8f14e45f-ceea-4a37-b0f1-2b5c4d3a9c6e",
       "namespace": "kagent",
       "harness": {
         "namespace": "kagent",
         "name": "my-first-harness"
       },
       "agentTemplate": {
         "namespace": "kagent",
         "name": "my-first-agent"
       },
       "state": "AGENT_INSTANCE_STATE_READY"
     }
   }
   ```

2. Save the AgentInstance's `id` to an environment variable. The next step needs it to address the AgentInstance you created.
   ```shell
   export INSTANCE_ID=$(echo "$RESPONSE" | jq -r '.agentInstance.id')
   ```

## Talk to your agent

Send a message to the AgentInstance over the A2A (Agent-to-Agent) protocol.

```shell
grpcurl -plaintext \
  -H "x-kagent-agent-instance-namespace: kagent" \
  -H "x-kagent-agent-instance-id: $INSTANCE_ID" \
  -d '{"message":{"messageId":"'"$(uuidgen)"'","role":"ROLE_USER","parts":[{"text":"What is 2+2?"}]}}' \
  localhost:8084 lf.a2a.v1.A2AService/SendMessage
```

The response carries the agent's reply in the same `parts` shape as the request.
```json
{
  "message": {
    "messageId": "b2b1e2b4-5c3a-4f8e-9d1a-7e6f5c4b3a2d",
    "role": "ROLE_AGENT",
    "parts": [
      {
        "text": "4"
      }
    ]
  }
}
```

## Clean up

1. Delete the Harness and AgentTemplate.
   ```shell
   kubectl delete agenttemplate my-first-agent -n kagent
   kubectl delete harness my-first-harness -n kagent
   ```

2. Delete the AgentInstance directly through the same `AgentInstanceService` you used to create it, as deleting the Harness and AgentTemplate does not delete the AgentInstance you created from them.
   ```shell
   grpcurl -plaintext \
     -d '{"namespace":"kagent","agentInstanceId":"'"$INSTANCE_ID"'"}' \
     localhost:8084 kagent.api.v1alpha1.AgentInstanceService/DeleteAgentInstance
   ```

## Next steps

{{< cards >}}
  {{< card link=`{{< link path="about/agent-substrate" >}}` title="Architecture: Agent Substrate" subtitle="Understand what happens to your AgentInstance's Actor when it sits idle." >}}
  {{< card link=`{{< link path="agents/agent-harness" >}}` title="Agent harness" subtitle="Choose from the full set of Harness runtime options, including Claude Code and Codex." >}}
  {{< card link=`{{< link path="skills-mcp/skills" >}}` title="Skills" subtitle="Give your agent capabilities beyond its system prompt." >}}
{{< /cards >}}
