---
title: Sandboxing
description: Learn how Agent Substrate isolates each Actor in its own sandbox, and what that sandbox separates.
weight: 10
author: kagent.dev
---

An agent is a program that decides at run time what to do next. It runs the commands that a model asks for, and it calls the tools that it was given. [Agent Substrate]({{< link path="about/agent-substrate" >}}) therefore does not run an Actor as an ordinary container process. It runs each Actor inside its own sandbox, on a Worker that hosts one Actor at a time. This page explains what selects a sandbox, what the sandbox separates, and how traffic reaches an Actor through it.

## Sandbox classes

A **sandbox class** is the sandbox runtime family that a Worker uses. Agent Substrate supports two.

- **`gvisor`**: The default. [gVisor](https://gvisor.dev) runs a user-space kernel that intercepts the sandboxed program's system calls, so the workload does not call the host kernel directly.
- **`microvm`**: Runs the workload inside a lightweight virtual machine, which places a hypervisor boundary between the workload and the host.

A WorkerPool selects its class through the `sandboxClass` field, which defaults to `gvisor`. The choice is not only a runtime preference. It also determines the shape of the Worker pods that Agent Substrate creates for that pool, including the virtualization device mounts and node placement that a micro-VM needs, and it determines which sandbox configurations the pool can draw on.

> [!NOTE]
> kagent generates ActorTemplates that use the `gvisor` class. Keep a WorkerPool that backs kagent Harnesses on `gvisor`.

## Sandbox configuration

A **SandboxConfig** is a cluster-scoped resource that holds the material needed to start one sandbox runtime family. It carries the runtime assets that the node agent fetches, keyed by processor architecture, along with the pause image that holds the sandbox's namespaces as its root container. One SandboxConfig can be marked as the cluster default for its class, and a WorkerPool that names no configuration explicitly resolves to that default.

Holding these assets in a cluster resource is what lets one configuration pin a runtime version for many ActorTemplates at once, rather than each template carrying its own copy. A default installation creates a single `gvisor-default` configuration.

## What the sandbox separates

The sandbox draws a boundary in three places.

- **Process and kernel**: The Actor's processes run against the sandbox runtime rather than the Worker node's kernel. A system call that the workload makes is handled by gVisor's user-space kernel, or by the guest kernel inside a micro-VM, instead of reaching the host directly.
- **Filesystem**: The Actor sees the filesystem assembled from its container image, plus whatever durable volume its ActorTemplate declares. Writes to the root filesystem are a layer on top of the image, captured in a `Full` snapshot and discarded by a `Data` one. See [Suspend and resume]({{< link path="substrate-runtime/suspend-and-resume" >}}) for what each scope keeps.
- **Network**: The Actor does not share the Worker pod's network position. The node agent gives the active Actor a private, point-to-point virtual network inside the Worker pod, so reaching the Actor means going through Agent Substrate's own network path rather than connecting to the Worker directly.

## How traffic reaches a sandboxed Actor

Every Actor is addressed by name, at `<actor-name>.<atespace>.actors.resources.substrate.ate.dev`. Reaching it involves several hops, and each one is what keeps a sandboxed Actor addressable without exposing the Worker that it happens to be running on.

1. Agent Substrate runs its own Domain Name System (DNS) service that answers queries for that address pattern with the address of the router, rather than any individual Worker.
2. The router reads the Actor name and atespace from the request, asks the Agent Substrate API to resume that Actor and report which Worker it is now assigned to, then selects that Worker as the destination.
3. The router connects to a listener on the Worker over mutual Transport Layer Security (mTLS). The listener validates that the caller is the router, and forwards traffic only to the Actor currently assigned to that Worker.

Because the router resolves the Worker assignment on every request, an Actor keeps a stable address across suspends, resumes, and moves between Workers.

Traffic in the other direction leaves through a separate egress gateway rather than going straight out from the Worker. Routing Actor egress through one gateway is what gives Agent Substrate a single place to apply outbound controls.

## Default network posture

Agent Substrate creates a Kubernetes NetworkPolicy for each WorkerPool, selecting that pool's Worker pods. The policy restricts **ingress** to the Agent Substrate router alone. No other pod in the cluster can open a connection to a Worker, so an Actor is not reachable by anything that bypasses the routing path.

That policy governs inbound traffic only. It does not constrain what an Actor may reach outbound, so outbound access is whatever the surrounding cluster and its infrastructure already allow. Treat network egress as something to configure deliberately for your environment rather than as something the WorkerPool policy settles.

## Next steps

{{< cards >}}
  {{< card link=`{{< link path="substrate-runtime/suspend-and-resume" >}}` title="Suspend and resume" subtitle="See what a snapshot captures and how an idle Actor comes back." >}}
  {{< card link=`{{< link path="about/agent-substrate" >}}` title="Architecture: Agent Substrate" subtitle="Review how Workers, Actors, and ActorTemplates fit together." >}}
{{< /cards >}}
