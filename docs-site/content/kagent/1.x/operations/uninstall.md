---
title: Uninstall
description: Remove kagent and Agent Substrate from a cluster, including the identity material and storage that Helm does not own.
weight: 50
author: kagent.dev
---

A kagent installation has two layers, and removing it reverses the [install]({{< link path="setup/installation" >}}) in order: kagent first, then Agent Substrate underneath it. Helm removes most of each layer, but the identity material and some storage were never Helm's to begin with, so a complete uninstall ends with a manual pass. This page covers both.

> [!CAUTION]
> Uninstalling deletes every kagent resource in every namespace, along with all agent conversation state and all stored snapshots. None of it can be recovered afterward. Back up anything you want to keep before you start.

## Before you begin

- Confirm that you have administrative access to the cluster.
- Back up any Harness, AgentTemplate, and ModelConfig definitions that you want to keep.
- Confirm that nothing outside kagent depends on the agents that you are about to remove.

## What an uninstall removes

Helm removes what its releases own, and the split matters because the material it leaves behind is exactly what blocks a later reinstall.

| Removed by Helm | Left behind |
| --------------- | ----------- |
| Every `kagent.dev` and `ate.dev` custom resource definition, and with them every Harness, AgentTemplate, WorkerPool, and SandboxConfig in the cluster | The certificate authority and JSON Web Token pools that the install created with `kubectl ate` |
| kagent's bundled PostgreSQL volume, holding all conversation state | Agent Substrate's PostgreSQL volume, `data-postgres-0`, because a StatefulSet volume claim outlives its release |
| The object storage volume holding every Actor snapshot | The `kagent`, `ate-system`, and `podcertificate-controller-system` namespaces |

## Uninstall kagent

Remove the kagent release before the CRDs release, because deleting the definitions first strands the controller.

1. Uninstall the kagent chart.
   ```bash
   helm uninstall kagent -n kagent
   ```

2. Uninstall the CRDs chart. This step deletes every `kagent.dev` custom resource definition, and Kubernetes deletes every resource of those kinds across all namespaces with them.
   ```bash
   helm uninstall kagent-crds -n kagent
   ```

> [!NOTE]
> The `kagent uninstall` command removes the same two releases in the same order, and it is a convenience rather than a different path. It does not touch Agent Substrate, so the rest of this page still applies. Prefer Helm, for the same reason that the install guide does: the CLI does not manage the Agent Substrate layer.

## Uninstall Agent Substrate

Agent Substrate is a separate installation in the `ate-system` namespace, and no kagent command removes it.

1. Uninstall the Agent Substrate chart.
   ```bash
   helm uninstall substrate -n ate-system
   ```

2. Uninstall the Agent Substrate CRDs chart, which deletes the `workerpools`, `sandboxconfigs`, and `csidriverconfigs` definitions.
   ```bash
   helm uninstall substrate-crds -n ate-system
   ```

## Remove the identity material

The identity material is the part most often left behind, and leaving it behind breaks the next installation rather than the current one.

The install created certificate authority and JSON Web Token pools with the `kubectl ate` plugin instead of Helm, so no release owns them and no uninstall removes them. A later install that tries to create a pool that already exists fails with a message naming the secret.

```console
Error: while uploading pool state to secret: secrets "service-dns-ca-pool" already exists
```

1. Delete the actor identity pools and the material derived from them.
   ```bash
   kubectl delete secret actor-id-jwt-pool actor-id-ca-pool actor-id-ca-certs -n ate-system
   kubectl delete configmap ate-api-authentication -n ate-system
   ```

2. Delete the certificate authority pools.
   ```bash
   kubectl delete secret service-dns-ca-pool pod-identity-ca-pool \
     -n podcertificate-controller-system
   ```

3. Delete the Agent Substrate database volume, which a StatefulSet volume claim keeps alive after its release is gone.
   ```bash
   kubectl delete pvc data-postgres-0 -n ate-system
   ```

> [!IMPORTANT]
> Deleting `data-postgres-0` matters even if you plan to reinstall immediately. Agent Substrate folds its schema changes into a single baseline migration before release, so a database that survives from an earlier version keeps that migration marked as applied and never picks up the new schema. The cluster then looks healthy and fails later, at the first checkpoint operation.

## Remove the namespaces

Deleting the namespaces removes anything that the preceding steps missed, including volumes left by optional components.

```bash
kubectl delete namespace kagent ate-system podcertificate-controller-system
```

Confirm that nothing remains.

```bash
kubectl get crd | grep -E 'kagent\.dev|ate\.dev'
kubectl get ns kagent ate-system podcertificate-controller-system
```

Both commands should report that they found nothing.
