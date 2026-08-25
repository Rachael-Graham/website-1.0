---
title: Install kagent
description: Install kagent 1.0 and Agent Substrate on a Kubernetes cluster.
weight: 10
author: kagent.dev
---

kagent 1.0 runs every agent on [Agent Substrate]({{< link path="about/agent-substrate" >}}), so an installation sets up two systems in the same cluster. Agent Substrate provides the sandboxed compute that agents run on, and kagent provides the Harness, AgentTemplate, and AgentInstance API that you author against.

## About installing kagent

Installation has three phases, and the order matters.

- **Install Agent Substrate**: Deploys the Agent Substrate control plane and data plane into the `ate-system` namespace.
- **Bootstrap Agent Substrate identity**: Creates the certificate authority (CA) pools, the JSON Web Token (JWT) authority pool, and the authentication configuration that Agent Substrate components use to prove identity to each other and to kagent.
- **Install kagent**: Deploys the kagent controller with the Agent Substrate integration enabled, and provisions the WorkerPool that agents run on.

> [!IMPORTANT]
> The bootstrap phase is required, and no Helm chart performs it for you. Until you complete it, every pod in the `ate-system` namespace stays in `ContainerCreating` and reports `FailedMount` events for the `podidentity` and `servicedns` volumes. Agent Substrate authenticates its components with mutual Transport Layer Security (mTLS), and the identity material that mTLS depends on is created by the `kubectl-ate` plugin, not by Helm.

> [!NOTE]
> Install kagent 1.0 with Helm. The `kagent install` command does not provision Agent Substrate and cannot produce a working 1.0 installation.

## Before you begin

You need the following tools and cluster capabilities.

- **Kubernetes 1.35 or later**: Agent Substrate requires the `ClusterTrustBundle`, `ClusterTrustBundleProjection`, and `PodCertificateRequest` feature gates, along with the `certificates.k8s.io/v1beta1` API. Kubernetes 1.35 enables all of them by default. On an earlier version, enable them explicitly on the API server.
- **Helm 3 and `kubectl`**: Both must be on your `PATH`.
- **`jq` and `openssl`**: The bootstrap phase uses both to extract a root certificate from a generated CA pool.
- **A model provider API key**: The examples on this page use OpenAI. For other providers, see [Configure model providers]({{< link path="setup/configure-model-providers" >}}).

### Prepare a cluster

Confirm that your cluster exposes the certificate APIs that Agent Substrate depends on.

{{< tabs >}}
{{% tab name="Existing cluster" %}}
Check that both resources are present.
```bash
kubectl api-resources --api-group=certificates.k8s.io
```

Example output, truncated to the two resources that matter.
```console
NAME                     APIVERSION                    NAMESPACED   KIND
clustertrustbundles      certificates.k8s.io/v1beta1   false        ClusterTrustBundle
podcertificaterequests   certificates.k8s.io/v1beta1   true         PodCertificateRequest
```

If either resource is missing, enable the feature gates on your API server before you continue.
{{% /tab %}}
{{% tab name="Local kind cluster" %}}
Create a kind cluster on a node image that includes the certificate APIs.
```bash
kind create cluster --name kagent --image kindest/node:v1.35.0
```

An earlier node image does not expose `certificates.k8s.io/v1beta1`, and Agent Substrate cannot start on it.
{{% /tab %}}
{{< /tabs >}}

### Install the kubectl-ate plugin

Agent Substrate ships `kubectl-ate` as a standalone binary that is published with each Agent Substrate release. Download the build that matches your operating system and architecture, then put it on your `PATH` so that `kubectl` finds it as a plugin.
```bash
curl -fsSL -o kubectl-ate \
  "https://github.com/kagent-dev/substrate/releases/download/v{{< reuse "versions/agent-substrate-1x.md" >}}/kubectl-ate-$(uname -s | tr '[:upper:]' '[:lower:]')-$(uname -m | sed 's/x86_64/amd64/; s/aarch64/arm64/')"
chmod +x kubectl-ate
sudo mv kubectl-ate /usr/local/bin/
```

Confirm that `kubectl` picks the plugin up.
```bash
kubectl ate --help
```

## Install Agent Substrate

1. Install the Agent Substrate custom resource definitions (CRDs).
   ```bash
   helm upgrade --install substrate-crds \
     oci://ghcr.io/kagent-dev/substrate/helm/substrate-crds \
     --version {{< reuse "versions/agent-substrate-1x.md" >}} \
     --namespace ate-system --create-namespace
   ```
2. Install the Agent Substrate control plane and data plane. Do not add `--wait` to this command, because the pods cannot become ready until you bootstrap identity in the next section.
   ```bash
   helm upgrade --install substrate \
     oci://ghcr.io/kagent-dev/substrate/helm/substrate \
     --version {{< reuse "versions/agent-substrate-1x.md" >}} \
     --namespace ate-system
   ```

## Bootstrap Agent Substrate identity

Agent Substrate signs pod identities and service certificates from CA pools that you generate, and it authenticates callers against a JWT authority pool. The `kubectl-ate admin` commands create that material as Kubernetes secrets, and a ConfigMap tells the Agent Substrate API server which token issuer to trust.

1. Create the CA pools that sign service DNS and pod identity certificates.
   ```bash
   kubectl ate admin make-ca-pool --ca-id=1 \
     --name=service-dns-ca-pool \
     --secret-namespace=podcertificate-controller-system
   kubectl ate admin make-ca-pool --ca-id=1 \
     --name=pod-identity-ca-pool \
     --secret-namespace=podcertificate-controller-system
   ```
2. Create the actor identity pools that Agent Substrate uses to issue and verify actor credentials.
   ```bash
   kubectl ate admin make-jwt-pool --key-id=1 \
     --name=actor-id-jwt-pool \
     --secret-namespace=ate-system
   kubectl ate admin make-ca-pool --ca-id=1 \
     --name=actor-id-ca-pool \
     --secret-namespace=ate-system
   ```
3. Extract the actor identity root certificate and store it in the secret that the Agent Substrate API server reads.
   ```bash
   actor_id_ca_root="$(kubectl get secret actor-id-ca-pool -n ate-system \
     -o jsonpath='{.data.pool}' | base64 --decode \
     | jq -r '.CAs[0].RootCertificateDER' | base64 --decode \
     | openssl x509 -inform der -outform pem)"

   kubectl create secret generic actor-id-ca-certs -n ate-system \
     --from-literal=ca.crt="${actor_id_ca_root}"
   ```
4. Create the authentication configuration. The `kubernetes` provider accepts Kubernetes ServiceAccount tokens that are issued for the Agent Substrate API server audience.
   ```bash
   kubectl create configmap ate-api-authentication -n ate-system \
     --from-literal=authentication.yaml='actorIdentityJWTProvider: kubernetes
   jwtProviders:
   - name: kubernetes
     issuer: https://kubernetes.default.svc
     audiences: [api.ate-system.svc]
     certificateAuthorityFile: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
     discoveryTokenFile: /var/run/secrets/kubernetes.io/serviceaccount/token
   '
   ```
5. Roll Agent Substrate out again so that its pods mount the identity material, and wait for them to become ready.
   ```bash
   helm upgrade substrate \
     oci://ghcr.io/kagent-dev/substrate/helm/substrate \
     --version {{< reuse "versions/agent-substrate-1x.md" >}} \
     --namespace ate-system --reuse-values --wait --timeout 10m
   ```
6. Verify that Agent Substrate is running.
   ```bash
   kubectl get pods -n ate-system
   ```
   Example output.
   ```console
   NAME                              READY   STATUS      RESTARTS   AGE
   ate-api-server-59fccdf6dc-f77h6   1/1     Running     3          9m
   ate-api-server-59fccdf6dc-q49hv   1/1     Running     3          9m
   ate-controller-6c788456f8-zh2rm   1/1     Running     0          9m
   atelet-wxm5s                      1/1     Running     0          9m
   atenet-egress-66f5699886-6rgg9    2/2     Running     0          9m
   atenet-router-645bd98bdd-dlrv2    2/2     Running     0          9m
   dns-6bf4fff5bb-zqsnm              2/2     Running     0          9m
   postgres-0                        1/1     Running     0          9m
   rustfs-56cdbc9dcb-2ntck           1/1     Running     0          9m
   rustfs-bucket-init-4pxgt          0/1     Completed   0          9m
   ```

## Install kagent

The kagent chart connects the controller to Agent Substrate and creates a WorkerPool for agents to run on. A WorkerPool is platform capacity that you provision once, and every Harness references it. No Harness can run until a WorkerPool exists.

1. Install the kagent CRDs.
   ```bash
   helm upgrade --install kagent-crds \
     oci://ghcr.io/kagent-dev/kagent/helm/kagent-crds \
     --version <kagent-version> \
     --namespace kagent --create-namespace --wait
   ```
2. Set your model provider API key.
   ```bash
   export OPENAI_API_KEY="your-api-key-here"
   ```
3. Install kagent with the Agent Substrate integration enabled. Set `substrateWorkerPool.ateomImage` explicitly, because the chart has no default for it and the install fails without it whenever `substrateWorkerPool.create` is `true`.
   ```bash
   helm upgrade --install kagent \
     oci://ghcr.io/kagent-dev/kagent/helm/kagent \
     --version <kagent-version> \
     --namespace kagent --create-namespace --timeout 10m \
     --set providers.default=openAI \
     --set providers.openAI.apiKey="${OPENAI_API_KEY}" \
     --set controller.substrate.enabled=true \
     --set controller.substrate.ateApiEndpoint=dns:///api.ate-system.svc:443 \
     --set controller.substrate.atenetRouterURL=http://atenet-router.ate-system.svc:80 \
     --set controller.substrate.defaultWorkerPool.name=kagent-default \
     --set substrateWorkerPool.create=true \
     --set substrateWorkerPool.replicas=1 \
     --set-string substrateWorkerPool.ateomImage=ghcr.io/kagent-dev/substrate/ateom-gvisor:v{{< reuse "versions/agent-substrate-1x.md" >}}
   ```
4. Wait for the controller to roll out.
   ```bash
   kubectl rollout status deployment/kagent-controller -n kagent --timeout=300s
   ```

> [!NOTE]
> The kagent controller can restart a few times during a first install while it waits for its bundled PostgreSQL database to accept connections. The controller logs `dial tcp ...:5432: connect: connection refused` and then recovers on its own. A restart loop that reports an `ate-api` dial failure instead points at an incomplete identity bootstrap.

## Verify the installation

1. Confirm that the kagent pods are running.
   ```bash
   kubectl get pods -n kagent
   ```
   Example output.
   ```console
   NAME                                 READY   STATUS    RESTARTS   AGE
   kagent-controller-659b58768b-2k6h4   1/1     Running   3          2m
   kagent-default-864fdc4c94-xbsl9      1/1     Running   0          2m
   kagent-postgresql-65cc684b78-9qbh2   1/1     Running   0          2m
   ```
2. Confirm that the WorkerPool reports a ready replica. Agents cannot start until the pool is ready.
   ```bash
   kubectl get workerpools -n kagent
   ```
   Example output.
   ```console
   NAMESPACE   NAME             DESIRED   REPLICAS   READY   AGE
   kagent      kagent-default   1         1          1       2m
   ```
3. Note how you reach the kagent gRPC API, which serves the AgentInstance lifecycle and conversation calls. [Your first agent]({{< link path="get-started/your-first-agent" >}}) assumes the port-forward.
   {{< tabs >}}
   {{% tab name="Cloud Provider LoadBalancer" %}}
   Read the external address of the controller service. The gRPC API listens on port `8084`.
   ```bash
   kubectl get svc -n kagent kagent-controller \
     -o jsonpath="{.status.loadBalancer.ingress[0]['hostname','ip']}"
   ```
   {{% /tab %}}
   {{% tab name="Port-forward for local testing" %}}
   Forward the gRPC port and leave the command running. The API is then available at `localhost:8084`.
   ```bash
   kubectl port-forward -n kagent svc/kagent-controller 8084:8084
   ```
   On a local kind cluster, use the port-forward. kind assigns a LoadBalancer address that is routable from inside the cluster, but not from your workstation.
   {{% /tab %}}
   {{< /tabs >}}

## Next steps

{{< cards >}}
  {{< card link=`{{< link path="get-started/your-first-agent" >}}` title="Your first agent" subtitle="Apply a Harness and AgentTemplate, and talk to the AgentInstance they produce." >}}
  {{< card link=`{{< link path="setup/configure-model-providers" >}}` title="Configure model providers" subtitle="Point kagent at OpenAI, Anthropic, Gemini, or a provider of your own." >}}
{{< /cards >}}
