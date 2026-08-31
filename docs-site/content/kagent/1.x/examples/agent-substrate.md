---
title: Agent Substrate
description: Watch an agent's Actor suspend between turns, pin the conversation with a checkpoint, and fork it into two independent branches.
weight: 10
author: kagent.dev
---

[Agent Substrate]({{< link path="about/agent-substrate" >}}) runs every agent as an Actor: a sandboxed unit of compute that holds a {{< gloss "Worker" >}}Worker{{< /gloss >}} only while a turn is in progress, and whose state you can pin and branch. This example follows one agent through all three behaviors.

## Before you begin

1. Complete [Your first agent]({{< link path="get-started/your-first-agent" >}}). The steps on this page continues from the Harness, AgentTemplate, and AgentInstance that the agent guide creates, and assumes that you have sent the agent at least one message.

2. If you have not already, save the AgentInstance's ID to an environment variable. To find the ID, run `kagent get agent-instance` to list your AgentInstances and copy the value from the `ID` column.
   ```bash
   export INSTANCE_ID=<your-agent-instance-id>
   ```

3. Install [grpcurl](https://github.com/fullstorydev/grpcurl), and confirm that your kagent installation sets `controller.grpc.reflection`. Checkpoints and forks have no kagent CLI commands yet, so this example calls `CheckpointService` directly.

4. Port-forward the controller's gRPC port to your local machine.
   ```bash
   kubectl port-forward -n kagent svc/kagent-controller 8084:8084
   ```

## Watch the Actor suspend between turns

1. List the Actors in your namespace's {{< gloss "Atespace" >}}atespace{{< /gloss >}}. kagent names an AgentInstance's Actor `ai-<agent-instance-id>`.
   ```bash
   kubectl ate get actors --atespace kagent
   ```

   Between turns, the Actor reports `ACTOR_STATE_SUSPENDED` and holds no Worker, so the `ATEOM POD` column reads `<none>`. Example output:
   ```console
   ATESPACE   NAME                                        TEMPLATE                                                 STATE                   ATEOM POD   ATEOM IP   VERSION   AGE
   kagent     ai-0198c3d7-4f2a-7b61-9c3e-5d8f7a2b4e10     kagent/my-first-agent-my-first-harness-5f2b3c1a9e8d      ACTOR_STATE_SUSPENDED   <none>                 4         6m
   ```

   That Actor is also the isolation boundary. Every Actor runs in its own gVisor sandbox rather than sharing one with its neighbors, which is what makes it safe to let a model run tools and execute commands. For what the sandbox blocks, see [Sandboxing]({{< link path="substrate-runtime/sandboxing" >}}).

2. Send the agent another message. Nothing in the command acknowledges that the Actor was suspended, because resuming is automatic.
   ```bash
   kagent invoke --agent-instance $INSTANCE_ID --task "Summarize this conversation so far."
   ```

3. List the Actors again while the turn is running, and the same Actor reports `ACTOR_STATE_RUNNING` against a real Worker pod. Once the turn finishes it returns to `ACTOR_STATE_SUSPENDED`.
   ```bash
   kubectl ate get actors --atespace kagent
   ```

The AgentInstance stays `READY` throughout all of this. Suspension is a property of the Actor underneath the conversation, not of the conversation, which is why a suspended agent is still listed and still readable. For the full cycle, see [Suspend and resume]({{< link path="substrate-runtime/suspend-and-resume" >}}).

## Pin the conversation with a checkpoint

Each suspend writes a snapshot, and Agent Substrate is free to collect that snapshot once a newer one supersedes it. A {{< gloss "Checkpoint" >}}checkpoint{{< /gloss >}} pins one so that you can come back to it.

1. Create a checkpoint. The `requestId` field is a required idempotency key of 1 to 128 characters, so reusing it returns the same checkpoint rather than creating a second one.
   ```bash
   grpcurl -plaintext \
     -d '{"namespace":"kagent","agentInstanceId":"'"$INSTANCE_ID"'","requestId":"'"$(uuidgen)"'"}' \
     localhost:8084 kagent.api.v1alpha1.CheckpointService/CreateCheckpoint
   ```

   The checkpoint records the snapshot it pinned and how far the transcript had advanced. Example output:
   ```json
   {
     "checkpoint": {
       "id": "0198c3e2-8a41-7d05-b6c2-1f4e9a7b3c58",
       "namespace": "kagent",
       "agentInstanceId": "0198c3d7-4f2a-7b61-9c3e-5d8f7a2b4e10",
       "headTaskId": "0198c3d9-b7e3-7a24-8f10-6c2d5e8a1b47",
       "historySequence": "4",
       "state": "CHECKPOINT_STATE_READY",
       "createdAt": "2026-08-31T15:12:44Z"
     }
   }
   ```

2. Save the checkpoint's `id`, to fork from it in the next section.
   ```bash
   export CHECKPOINT_ID=<your-checkpoint-id>
   ```

3. List the checkpoints on the AgentInstance at any time. Omit `limit` for the default page of 50, up to a maximum of 100.
   ```bash
   grpcurl -plaintext \
     -d '{"namespace":"kagent","agentInstanceId":"'"$INSTANCE_ID"'","page":{"limit":50}}' \
     localhost:8084 kagent.api.v1alpha1.CheckpointService/ListCheckpoints
   ```

Underneath, the checkpoint attaches an ActorSnapshotTag named `checkpoint-<checkpoint-id>` to the snapshot, and Agent Substrate does not collect a snapshot while a tag names it. You can see the tag with `kubectl ate get actor-snapshot-tag`.

> [!NOTE]
> A checkpoint captures a turn boundary, so two conditions have to hold and a request that fails either one reports `AgentInstance has no quiescent turn boundary`. The AgentInstance must be `READY` with no lifecycle operation in flight, and at least one turn must have reached a quiescent state. Send the request again once the turn finishes.

## Fork the conversation into a second agent

Forking creates a second AgentInstance that starts from the pinned snapshot, with the transcript up to the checkpoint's position already in place. The original is untouched, so the two conversations diverge from that point.

1. Fork the checkpoint.
   ```bash
   grpcurl -plaintext \
     -d '{"namespace":"kagent","checkpointId":"'"$CHECKPOINT_ID"'","requestId":"'"$(uuidgen)"'"}' \
     localhost:8084 kagent.api.v1alpha1.CheckpointService/ForkAgentInstance
   ```

   The response carries a new AgentInstance with its own ID. Example output:
   ```json
   {
     "agentInstance": {
       "id": "0198c3e5-1d62-7f38-a904-8b3c7e2f5d16",
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

2. Save the fork's ID, then send it down a different path than the original.
   ```bash
   export FORK_ID=<your-fork-agent-instance-id>
   kagent invoke --agent-instance $FORK_ID --task "What did I ask you first?"
   ```

   The fork answers from the transcript it inherited, which is what distinguishes a fork from a new AgentInstance that happens to use the same AgentTemplate.

3. List your AgentInstances, and both branches appear, each with its own Actor.
   ```bash
   kagent get agent-instance
   ```

A fork runs the compiled revision that its checkpoint was taken on, not whatever revision the AgentTemplate resolves to now. Editing the AgentTemplate after checkpointing does not change what a fork of that checkpoint runs, which is what makes a fork a faithful continuation rather than a fresh start with an old transcript.

> [!NOTE]
> A checkpoint can only be forked when its snapshot captured durable data alone. kagent compiles every ActorTemplate to take a `Data`-scope snapshot on commit, so a checkpoint taken on a suspended AgentInstance is forkable. A checkpoint whose snapshot also captured process state is rejected with `Checkpoint includes process state and cannot be forked`, because process memory belongs to the one Actor that produced it.

## Clean up

1. Delete the checkpoint. Deleting removes the ActorSnapshotTag and releases the pin, and Agent Substrate can collect the snapshot once no tag names it.
   ```bash
   grpcurl -plaintext \
     -d '{"namespace":"kagent","checkpointId":"'"$CHECKPOINT_ID"'"}' \
     localhost:8084 kagent.api.v1alpha1.CheckpointService/DeleteCheckpoint
   ```

2. Delete both AgentInstances.
   ```bash
   kagent delete agent-instance $FORK_ID
   kagent delete agent-instance $INSTANCE_ID
   ```

3. Delete the AgentTemplate and the Harness.
   ```bash
   kubectl delete agenttemplate my-first-agent -n kagent
   kubectl delete harness my-first-harness -n kagent
   ```

## Next steps

{{< cards >}}
  {{< card link=`{{< link path="substrate-runtime/suspend-and-resume" >}}` title="Suspend and resume" subtitle="Understand the snapshot cycle that checkpoints pin." >}}
  {{< card link=`{{< link path="substrate-runtime/sandboxing" >}}` title="Sandboxing" subtitle="See what the sandbox around each Actor isolates." >}}
  {{< card link=`{{< link path="substrate-runtime/identity" >}}` title="Identity" subtitle="See who owns an AgentInstance and the checkpoints taken on it." >}}
{{< /cards >}}
