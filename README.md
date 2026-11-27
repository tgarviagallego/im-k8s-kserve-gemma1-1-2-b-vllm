# Author

- **Name:** Juan Vicente Herrera Ruiz de Alejo
- **GitHub:** [@juanviz](https://github.com/juanviz)
- **Role:** Author and Maintainer
- **Affiliation:** ICAI - Universidad Pontificia Comillas

# License

This project is licensed under the MIT License. See the [MIT License](https://opensource.org/licenses/MIT).


# Deploying Gemma 2 on GKE Autopilot with KServe and vLLM

This repository demonstrates how to deploy a production-oriented generative AI inference service on Google Kubernetes Engine (GKE) Autopilot. It combines Kubernetes security practices with Knative, Istio, Cert-Manager, NVIDIA GPU provisioning, Hugging Face credentials, KServe, and an OpenAI-compatible API.

The deployed model is `gemma-1.1-2b-it`, served by vLLM through KServe. The example requests one NVIDIA L4 GPU in GKE Autopilot and exposes Swagger UI and OpenAI-compatible endpoints.

## Learning objectives

- Deploy and configure GKE Autopilot for GPU-backed inference workloads.
- Install the serverless and networking components required by KServe.
- Configure TLS with Cert-Manager and HTTP routing with Istio.
- Store and consume a Hugging Face access token through Kubernetes Secrets.
- Request NVIDIA L4 hardware with `nodeSelector` and GPU resource limits.
- Deploy a model with a KServe `InferenceService`.
- Validate text-generation APIs in Swagger UI and with `curl`.
- Remove cloud resources and stop services to control operational costs.

## Architecture

```text
GitHub Actions
      |
      | authenticate and apply manifests
      v
GKE Autopilot cluster
      |
      +-- kserve-test namespace
      |     +-- hf-secret                 # Hugging Face token
      |     +-- huggingface-gemma2        # KServe InferenceService
      |           |
      |           +-- Knative serving revision
      |           +-- vLLM/Hugging Face serving runtime
      |           +-- 1 x NVIDIA L4 GPU
      |
      +-- Istio ingress and routing
      +-- Cert-Manager TLS certificates
      +-- Knative serverless lifecycle
```

The deployment flow is:

1. GitHub Actions authenticates to Google Cloud.
2. `gcloud` retrieves credentials for the target GKE cluster.
3. Kubernetes creates or selects the `kserve-test` namespace.
4. A Kubernetes `Secret` stores the Hugging Face token.
5. KServe creates the `InferenceService`.
6. Knative provisions a serverless serving revision.
7. The pod is scheduled onto an Autopilot node with an NVIDIA L4 GPU.
8. Istio routes external traffic to the model, while Cert-Manager manages certificates.
9. Clients call the model through its OpenAI-compatible API or Swagger UI.

## Repository layout

| Path | Purpose |
|---|---|
| [`k8s/InferenceService.yaml`](k8s/InferenceService.yaml) | Defines the Hugging Face model server and CPU requirements. |
| [`k8s/InferenceServiceVLLM.yaml`](k8s/InferenceService.yaml) | Defines the Hugging Face model server and GPU requirements. |
| [`.github/workflows/deploy.yml`](.github/workflows/deploy.yml) | Authenticates to GCP and deploys the namespace, secret, and `InferenceService` from the `main` branch. |
| [`README.md`](README.md) | Project documentation and operational guide. |

## Infrastructure components

### GKE Autopilot

GKE Autopilot manages node provisioning and operational details for the cluster. The manifest requests GPU capacity using the GKE accelerator labels and the Kubernetes GPU resource:

- Accelerator: `nvidia-l4`
- Accelerator count: `1`
- Node label: `cloud.google.com/gke-accelerator=nvidia-l4`
- Kubernetes resource: `nvidia.com/gpu: "1"`

GPU availability and accelerator quotas vary by Google Cloud project and region. The selected cluster region must support NVIDIA L4 capacity.

### KServe

KServe manages the model-serving lifecycle through the `serving.kserve.io/v1beta1` `InferenceService` custom resource. The resource is named `huggingface-gemma2` and is created in the `kserve-test` namespace.

### Knative

Knative Serving provides Kubernetes-native serverless revisions, autoscaling, revisions, and request routing for the inference workload. KServe uses Knative to expose the model through a predictable serving endpoint.

### Istio

Istio provides service ingress and traffic routing for the inference endpoint. DNS, Gateway, VirtualService, and ingress configuration must match the cluster setup and desired public or internal endpoint.

### Cert-Manager

Cert-Manager issues and renews TLS certificates for HTTPS serving. Configure a supported issuer, such as Let's Encrypt or Google-managed infrastructure, before relying on a publicly reachable HTTPS endpoint.

## The InferenceService

The manifest [`k8s/InferenceService.yaml`](k8s/InferenceService.yaml) contains:

- Model format: `huggingface`
- Model: `google/gemma-1.1-2b-it`
- KServe model name: `gemma2`
- NVIDIA L4 node selection
- One GPU request and limit
- CPU request and limit: 6 cores
- Memory request and limit: 24 GiB
- Hugging Face token loaded from `hf-secret/hf_api_token`
- Swagger/OpenAPI documentation enabled with `--enable_docs_url=True`

Example CPU and memory values consume the configured Autopilot quota while the pod is active. GPU workloads should be monitored closely because they are substantially more expensive to run than CPU workloads.

## Prerequisites

You need:

- A Google Cloud project with billing enabled.
- A GKE Autopilot cluster with GPU support.
- Quota for NVIDIA L4 GPUs in the cluster region.
- `gcloud`, `kubectl`, and a GitHub account.
- KServe, Knative Serving, Istio, and Cert-Manager installed in the cluster.
- A Hugging Face account and access token for the selected model.
- Permission to create cluster and namespaced resources.

Typical tools are:

```bash
gcloud --version
kubectl version --client
git --version
```

The Hugging Face model can require accepting model terms or possessing the relevant account permissions. Verify that `google/gemma-1.1-2b-it` is accessible with the token before deployment.

## Cluster setup

Create and connect to a GKE Autopilot cluster. Replace the example project, region, and cluster name.

```bash
export PROJECT_ID="<gcp-project-id>"
export REGION="<gcp-region>"
export CLUSTER_NAME="<gke-cluster-name>"

gcloud config set project "$PROJECT_ID"

gcloud container clusters create-auto "$CLUSTER_NAME" \
  --region "$REGION"

gcloud container clusters get-credentials "$CLUSTER_NAME" \
  --region "$REGION"
```

The exact GKE GPU configuration, KServe version, Knative configuration, Istio ingress setup, and Cert-Manager issuer depend on the Kubernetes platform versions and chosen architecture. Install compatible versions and confirm all required custom resource definitions and pods are healthy before deploying the model.

For a basic installation check:

```bash
kubectl get nodes
kubectl get pods -A
kubectl get crd | grep -E 'inferenceservice|knative|istio|cert-manager'
```

## GitHub Actions deployment

Every push to `main` runs [`.github/workflows/deploy.yml`](.github/workflows/deploy.yml). The workflow:

1. Checks out the repository.
2. Authenticates with a Google Cloud service account key.
3. Installs `gke-gcloud-auth-plugin`.
4. Retrieves cluster credentials.
5. Creates the `kserve-test` namespace.
6. Creates the `hf-secret` Kubernetes Secret.
7. Applies the `InferenceService`.

Configure these GitHub Actions repository secrets:

| Secret | Description |
|---|---|
| `GCP_SA_KEY` | JSON credentials for a Google Cloud service account with permission to access the GKE cluster. |
| `GKE_CLUSTER` | GKE cluster name. |
| `GKE_REGION` | GKE cluster region. |
| `HF_TOKEN` | Hugging Face access token used to download the model. |

For stronger security, prefer Workload Identity Federation or another short-lived credential mechanism instead of storing a long-lived JSON key. The deployed Kubernetes token should be handled as sensitive configuration and not committed to Git.

## Manual deployment

The workflow can also be executed manually:

```bash
kubectl create namespace kserve-test

kubectl -n kserve-test create secret generic hf-secret \
  --from-literal=hf_api_token='<hugging-face-token>'

kubectl apply -f k8s/InferenceService.yaml
```

Inspect deployment state:

```bash
kubectl -n kserve-test get inferenceservice
kubectl -n kserve-test get pods
kubectl -n kserve-test describe pod -l serving.kserve.io/inferenceservice=huggingface-gemma2
kubectl -n kserve-test get inferenceservice huggingface-gemma2 \
  -o yaml
```

Watch pod startup because model download and GPU initialization can take several minutes:

```bash
kubectl -n kserve-test get pods -w
```

## GPU validation

Check that the pod received an NVIDIA L4 GPU and that the NVIDIA device plugin is visible:

```bash
kubectl -n kserve-test get pod \
  -l serving.kserve.io/inferenceservice=huggingface-gemma2 \
  -o jsonpath='{range .items[*]}{.metadata.name}{"\n"}{end}'

kubectl -n kserve-test describe pod \
  -l serving.kserve.io/inferenceservice=huggingface-gemma2 |
  grep -E 'nvidia-l4|nvidia.com/gpu|Accelerator'
```

For a pod named from the command above:

```bash
export POD="<pod-name>"
kubectl -n kserve-test exec "$POD" -- nvidia-smi
```

## Endpoint validation

Retrieve the service URL advertised by KServe:

```bash
export INFERENCE_URL="$(
  kubectl -n kserve-test get inferenceservice huggingface-gemma2 \
    -o jsonpath='{.status.url}'
)"

echo "$INFERENCE_URL"
```

The runtime enables Swagger/OpenAPI documentation. Open the following in a browser after DNS and TLS ingress are configured:

- Swagger UI: `"$INFERENCE_URL/docs"`
- OpenAPI schema: `"$INFERENCE_URL/openapi.json"`

In Swagger UI, expand the chat-completion or text-generation operation and select **Try it out**. Include a short prompt to validate inference.

### OpenAI-compatible chat request

The Hugging Face/vLLM KServe runtime exposes an OpenAI-style chat endpoint. Depending on the runtime version and ingress path, the standard local endpoint is usually:

```bash
curl -sS "$INFERENCE_URL/v1/chat/completions" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "gemma2",
    "messages": [
      {"role": "system", "content": "You are a concise assistant."},
      {"role": "user", "content": "Explain Kubernetes in one sentence."}
    ],
    "temperature": 0.2,
    "max_tokens": 128
  }'
```

List models:

```bash
curl -sS "$INFERENCE_URL/v1/models"
```

If the service is published under a KServe or Istio hostname rather than directly at the root path, add the hostname and API path defined by the ingress configuration. Use the runtime's `/openapi.json` or `/docs` response to confirm the exact routes for the installed serving runtime.

## Credential security

- Never add `HF_TOKEN`, `GCP_SA_KEY`, or `hf_api_token` to source control.
- Use Kubernetes Secrets for workload credentials.
- Restrict GitHub Actions and Google Cloud IAM permissions to the resources required for deployment.
- Use separate credentials and projects for development and production.
- Rotate Hugging Face and Google Cloud credentials periodically.
- Consider the Kubernetes Secrets data-plane encryption and Google Secret Manager integrations available in the target cluster.
- Review pod and namespace RBAC so only authorized identities can read `hf-secret`.

The repository does not contain a static credential. The workflow obtains `HF_TOKEN` only from GitHub Actions Secrets and injects it into a Kubernetes Secret.

## Troubleshooting

### Model or pod not starting

```bash
kubectl -n kserve-test get events --sort-by=.lastTimestamp
kubectl -n kserve-test logs -l serving.kserve.io/inferenceservice=huggingface-gemma2
kubectl -n kserve-test describe inferenceservice huggingface-gemma2
```

Common causes include insufficient GPU quota, unavailable L4 capacity in the region, Hugging Face access restrictions, an invalid token, or insufficient CPU/memory resources.

### Secret not found

```bash
kubectl -n kserve-test get secret hf-secret
kubectl -n kserve-test describe secret hf-secret
```

Never print a Secret's decoded value into CI logs or terminal output unless absolutely necessary.

### Endpoint returns 404 or is unreachable

Verify the KServe status URL, DNS record, Istio ingress, TLS issuer, and Cert-Manager certificates:

```bash
kubectl -n kserve-test get inferenceservice huggingface-gemma2
kubectl get gateway -A
kubectl get certificates -A
kubectl get pods -n istio-system
kubectl get pods -n cert-manager
```

### Hugging Face authentication failure

Confirm that the token has access to Gemma 2, that the model terms are accepted, and that the Secret key name is exactly `hf_api_token`.

## Cost management and cleanup

GPU inference nodes and cloud load-balancing resources can incur significant costs. Stop or remove the workload as soon as it is no longer needed.

Delete the model and credentials:

```bash
kubectl -n kserve-test delete inferenceservice huggingface-gemma2
kubectl -n kserve-test delete secret hf-secret
kubectl delete namespace kserve-test
```

Delete optional infrastructure components individually when they are no longer needed:

```bash
# Helm-based installations vary; use the original release and namespace names.
helm uninstall knative-serving -n knative-serving
helm uninstall kserve -n kserve
helm uninstall cert-manager -n cert-manager
```

Delete the entire GKE cluster when it is not required:

```bash
gcloud container clusters delete "$CLUSTER_NAME" \
  --region "$REGION"
```

Before cleanup, verify that no external IPs, load balancers, persistent disks, or GPU nodes remain in the project:

```bash
gcloud compute forwarding-rules list
gcloud compute addresses list
gcloud compute disks list
gcloud container clusters list
```

