---
title: kagent architecture
description: See how a Harness and AgentTemplate become a running conversation, across kagent's two authorization planes.
weight: 30
author: kagent.dev
---

The previous page defined the [core concepts]({{< link path="about/core-concepts" >}}) of Harness, AgentTemplate, AgentInstance, and Actor. This page connects them into one system: how applying a Harness and AgentTemplate leads to a running conversation, and which parts of that path Kubernetes governs versus which parts kagent governs itself.

## Two authorization planes

kagent 1.0 splits authorization across two planes:

- The **Kubernetes plane** governs the {{< gloss "Harness" >}}Harness{{< /gloss >}} and {{< gloss "AgentTemplate" >}}AgentTemplate{{< /gloss >}} custom resources. Kubernetes Role-Based Access Control (RBAC) decides who can create, read, or edit the resources, exactly as it would for any other Custom Resource Definition (CRD).
- The **kagent plane** governs any interactions involving {{< gloss "AgentInstance" >}}AgentInstances{{< /gloss >}}, such as creating, suspending, resuming, sharing, deleting, and holding a conversation with an AgentInstance. kagent's own gRPC authentication and authorization decide who can complete these interactions, independent of Kubernetes RBAC.

Someone with Kubernetes RBAC access to apply a Harness and AgentTemplate does not automatically have access to create or talk to AgentInstances that use them, and the reverse is also true. The following diagram shows where the boundary between the two planes falls.
</br></br>

```mermaid
flowchart TB

    subgraph k8s["Kubernetes plane (RBAC)"]
        operator["Operator<br>kubectl apply"]
        harness["Harness"]
        template["AgentTemplate"]
        controller["kagent controller"]
        actortemplate["ActorTemplate (Substrate)"]
        operator --> harness
        operator --> template
        harness --> controller
        template --> controller
        controller -->|compiles the pair into| actortemplate
    end

    subgraph kagentplane["kagent plane (gRPC auth)"]
        caller["Caller"]
        gateway["A2A gateway"]
        instance["AgentInstance"]
        actor["Actor (Substrate)"]
        caller -->|A2A conversation| gateway
        caller -->|CreateAgentInstance| instance
        gateway -->|routes to| actor
        instance -->|runs on| actor
    end

    %% Declared outside both subgraphs on purpose: a node belongs to whichever
    %% subgraph first references it, so putting this edge inside the kagent plane
    %% would pull ActorTemplate out of the Kubernetes plane.
    actortemplate -->|instantiated as| instance
    %% Invisible link: forces the kagent plane to sit fully below the Kubernetes
    %% plane. Without it, the layout engine staggers the two planes diagonally.
    %% actortemplate ~~~ caller

    classDef crd stroke:#a78bfa,stroke-width:2px
    class harness,template crd
```

Follow the **Kubernetes plane** first. An operator applies a Harness and an AgentTemplate, governed by Kubernetes RBAC. The kagent controller watches for a valid pair with a matching `allowedAgentTemplates` selector, and compiles it into an {{< gloss "ActorTemplate" >}}ActorTemplate{{< /gloss >}} on Substrate.

The **kagent plane** starts once that ActorTemplate exists. A caller, who may or may not be the same person as the operator, calls `CreateAgentInstance` through kagent's gRPC API. This call is governed by kagent's own authentication and authorization, not by Kubernetes RBAC. kagent creates the AgentInstance from the newest ActorTemplate that compiled successfully, and that AgentInstance runs on an {{< gloss "Actor" >}}Actor{{< /gloss >}}.

From there, the caller holds a conversation with the AgentInstance over the A2A (Agent-to-Agent) protocol. The A2A gateway routes each request to the Actor running behind the target AgentInstance. This means that the caller only ever needs to know an AgentInstance's identity, never which Actor or {{< gloss "Worker" >}}Worker{{< /gloss >}} is behind it.

## Why two planes

Kubernetes RBAC is designed to authorize configuration changes: who can create a Deployment, edit a ConfigMap, or in this case, apply a Harness or AgentTemplate. It is not designed to authorize a running conversation, share access to it with another user, or scope who can suspend it. kagent's gRPC plane exists to authorize exactly those actions, at the granularity of a single AgentInstance rather than a namespace or a resource kind.

This split also keeps the two lifecycles independent. Editing a Harness or AgentTemplate does not affect AgentInstances already running against the ActorTemplate that they were created from. It only affects new AgentInstances, created after the edit is compiled.
