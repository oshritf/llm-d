# Well-lit Path: Intelligent Inference Scheduling for Multi Model Serving

This guide covers advanced scenarios where multiple Large Language Models (LLMs) and their respective Low-Rank Adaptations (LoRAs) are served within the same cluster. 

By following this well-lit path, you will learn how to configure the **Kubernetes Gateway API Inference Extension** to intelligently route requests across multiple `InferencePools` using **Body-Based Routing (BBR)**.

---

## 🌟 The Scenario

A modern AI platform often needs to deploy multiple base LLMs simultaneously. For example:
* A **Qwen3-32b** model offers complex reasoning and fast, general conversation.
* A **DeepSeek** model handling recommendation engines or code generation.

Furthermore, each of these base models may host multiple fine-tuned **LoRA adapters** (e.g., `ski-resorts`, `movie-critique`). LoRAs associated with the same base model must be routed to the specific backend inference server hosting that base model. 

Since clients interact with a single unified endpoint and specify the requested model/LoRA in the JSON payload (e.g., `"model": "deepseek/vllm-deepseek-r1", `"model": "ski-resorts"`), the routing layer must intelligently inspect the request body and schedule the inference accordingly.

---

## 📋 Prerequisites

* Have the [proper client tools installed on your local system](../prereq/client-setup/README.md) to use this guide.
* Ensure your cluster infrastructure is sufficient to [deploy high scale inference](../prereq/infrastructure)
* Configure and deploy your [Gateway control plane](../prereq/gateway-provider/README.md).
* Have the [Monitoring stack](../../docs/monitoring/README.md) installed on your system.
* Create a namespace for installation.

  ```bash
  export NAMESPACE=llm-d-multi-model # or any other namespace (shorter names recommended)
  kubectl create namespace ${NAMESPACE}
  ```

* [Create the `llm-d-hf-token` secret in your target namespace with the key `HF_TOKEN` matching a valid HuggingFace token](../prereq/client-setup/README.md#huggingface-token) to pull models.
* [Choose an llm-d version](../prereq/client-setup/README.md#llm-d-version)
* Set the gateway extensions version

```bash
  export IGW_CHART_VERSION=v1.3.1 

## Installation

Use the helmfile to compose and install the stack. The Namespace in which the stack will be deployed will be derived from the `${NAMESPACE}` environment variable. If you have not set this, it will default to `llm-d-multi-model` in this example.

### Deploy

```bash
cd guides/multi-node-serving

<!-- TABS:START -->

<!-- TAB:GPU deployment:default -->

#### GPU deployment

```bash
helmfile apply -n ${NAMESPACE}
```

<!-- TAB:CPU deployment -->

#### CPU-only deployment

```bash
helmfile apply -e cpu -n ${NAMESPACE}
```

<!-- TABS:END -->

**_NOTE:_** This uses Istio as the default provider, see [Gateway Options](./README.md#gateway-options) for installing with a specific provider.

Check Deployments and Pods in the ${NAMESPACE} and make sure all is up and running.
```bash
kubectl get pods
```
Confirm that the Gateway was assigned an IP address and reports a Programmed=True status:
```bash
kubectl get gateway infra-multi-model-inference-gateway
```

---

## Deploy the Body-Based Routing (BBR) Extension

The Body-Based Router (BBR) acts as the intelligent dispatcher. It extracts the model name from the incoming HTTP request body, looks up the corresponding base model via a ConfigMap, and injects this information into the `X-Gateway-Base-Model-Name` header. The Gateway then uses this header to route traffic to the correct `InferencePool` using the matching `HttpRoute`.

> **Note:** All base model and LoRA names must be globally unique across your deployment.

Choose your Gateway provider and deploy the BBR via Helm:

```bash
export CHART_VERSION=v0
export GATEWAY_PROVIDER=istio # Options: gke, istio, or none

helm install body-based-router \
  --set provider.name=$GATEWAY_PROVIDER \
  --version $CHART_VERSION \
  oci://us-central1-docker.pkg.dev/k8s-staging-images/gateway-api-inference-extension/charts/body-based-routing
```

*(Note for Kgateway users: Kgateway implements BBR natively via an `AgentgatewayPolicy` and does not require this Helm chart).*

---

## Upgrade the first model

**1. Create a ConfigMap for the first model's:**
Body-based-routing looks for a model ConfigMap with the model and it's LoRAs

```bash
kubectl apply -f vllm-qwen3-32b-adapters-allowlist
```

**2. Upgrade the Helm release for the first model to enable experimental HTTP routes and generate an HTTPRoute backed by the inferencepool:**
```bash
helm upgrade gaie-inference-scheduling oci://us-central1-docker.pkg.dev/k8s-staging-images/gateway-api-inference-extension/charts/inferencepool \
  --version $IGW_CHART_VERSION \
  --dependency-update \
  --set "inferencePool.modelServers.matchLabels.llm-d\.ai/model=Qwen3-32B"
  --set provider.name=$GATEWAY_PROVIDER \
  --set experimentalHttpRoute.enabled=true \
  --set experimentalHttpRoute.baseModel=Qwen/Qwen3-32B \
  --set experimentalHttpRoute.inferenceGatewayName=infra-multi-model-inference-gateway
```
Make sure you can select the model node (used by the InferencePool):
```bash
kubectl get pods -l llm-d.ai/model=Qwen/Qwen3-32B
```

Verify all pods are up and running, and that you now have one InferencePool and one HttpRoute configured
```bash
kubectl get inferencepools
kubectl get httproute
kubectl get pods
```

Examine httpexpriements values were set correctly:
```bash
helm get values vllm-deepseek-r1 -n ${NAMESPACE}
helm get values gaie-multi-model -n ${NAMESPACE}
```

Check that the HTTPRoute and InferencePool were successfully configured and references were resolved:
```bash
kubectl get httproute ${INFERENCE_POOL_NAME} -o yaml
kubectl get inferencepool ${INFERENCE_POOL_NAME} -o yaml
```
The HttpRoute and InferencePool status should include Accepted=True and ResolvedRefs=True
---

## Deploy a Second Model Server

In addition to the primary model deployed, we will introduce a second model server using a vLLM simulator. In this example, we deploy `deepseek/vllm-deepseek-r1` as the base model, serving two distinct LoRAs: `ski-resorts` and `movie-critique`.

Apply the manifest to deploy the second model server and its LoRA-to-Base-Model mapping:

```bash
kubectl apply -f mdeepseek.yaml
```
Make sure you can select the model node (used by the InferencePool):
```bash
kubectl get pods -l app=deepseek/vllm-deepseek-r1
```
---

## Deploy the Second InferencePool and Endpoint Picker (EPP)

Now, use the helm install command below to create and map the new DeepSeek model to an `InferencePool` and an associated Endpoint Picker Extension (EPP). Ensure you use the `experimentalHttpRoute` flags to bind the route directly to the base model.

```bash
helm install vllm-deepseek-r1 \
  oci://us-central1-docker.pkg.dev/k8s-staging-images/gateway-api-inference-extension/charts/inferencepool \
  --version $IGW_CHART_VERSION \
  --dependency-update \
  --set inferencePool.modelServers.matchLabels.app=vllm-deepseek-r1 \
  --set provider.name=$GATEWAY_PROVIDER \
  --set experimentalHttpRoute.enabled=true \
  --set experimentalHttpRoute.baseModel=deepseek/vllm-deepseek-r1 \
  --set experimentalHttpRoute.inferenceGatewayName=infra-multi-model-inference-gateway

```

**Verification:**
Make sure all pods are up and running and verify that you now have two `InferencePools`, two `HttpRoutes` and two Endpoint Picker pods running:

```bash
kubectl get inferencepools
kubectl get httproute
kubectl get pods
```

Examine httpexpriements values were set correctly:
```bash
helm get values vllm-deepseek-r1 -n ${NAMESPACE}
```

Check the second models' HTTPRoute and InferencePool were successfully configured and references were resolved:
```bash
kubectl get httproute ${INFERENCE_POOL_NAME} -o yaml
kubectl get inferencepool ${INFERENCE_POOL_NAME} -o yaml
```
The HttpRoute and InferencePool status should include Accepted=True and ResolvedRefs=True
---

## Test the multi model deployment

Once the Gateway is fully ready, test the routing by dynamically querying different base models and LoRAs through the same unified gateway IP.

Port forward if needed
```bash
# Create port-forward to the gateway service
kubectl port-forward -n ${NAMESPACE} svc/infra-multi-model-inference-gateway-istio 8080:80 &
```

### Test 1: Route to Qwen3-32b Model
```bash
curl -X POST http://localhost:8080/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "Qwen/Qwen3-32B",
    "prompt": "Hello, how are you?",
    "max_tokens": 50
  }'
```

### Test 2: Route to DeepSeek model
```bash
curl -X POST http://localhost:8080/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "deepseek/DeepSeek-r1",
    "prompt": "What is the best ski resort in Austria?",
    "max_tokens": 50,
    "temperature": 0,
  }'
```

### Test 3: Route to DeepSeek LoRA (`ski-resorts`)
```bash
curl -X POST http://localhost:8080/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "ski-resorts",
    "prompt": "Tell me about ski deals in Austria?",
    "max_tokens": 50,
    "temperature": 0,
  }'
```

### Test 4: Route to DeepSeek LoRA (`movie-critique`)
```bash
curl -X POST http://localhost:8080/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "movie-critique",
    "prompt":  "What are the best movies of 2025?",
    "max_tokens": 100,
    "temperature": 0,
  }'
```

By completing this well-lit path, your infrastructure is now capable of zero-friction, payload-aware scheduling. The Gateway API dynamically delegates traffic across multiple base inference pools, gracefully maintaining LoRA affinities behind a single endpoint.