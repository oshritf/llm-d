# Well-lit Path: Intelligent Inference Scheduling for Multi Model Serving

This guide covers a deployment of a stack with multiple Large Language Models (LLMs) and their respective Low-Rank Adaptations (LoRAs) that are served within the same cluster.

By following this well-lit path, you will learn how to configure the **Kubernetes Gateway API Inference Extension** to intelligently route requests across multiple `InferencePools` using **Body-Based Routing (BBR)**.

---

## The Scenario

A modern AI platform often needs to deploy multiple base LLMs simultaneously. For example:
* A **Qwen3-32b** model offers complex reasoning and fast, general conversation.
* A **DeepSeek** model handling recommendation engines or code generation.

Furthermore, each of these base models may host multiple fine-tuned **LoRA adapters** (e.g., `ski-resorts`, `movie-critique`). LoRAs associated with the same base model must be routed to the specific backend inference server hosting that base model. 

Since clients interact with a single unified endpoint and specify the requested model/LoRA in the JSON payload (e.g., `"model": "deepseek/Deepseek-r1", `"model": "ski-resorts"`), the routing layer must intelligently inspect the request body and schedule the inference accordingly.

---

## Prerequisites

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

### Gateway options

To see specify your gateway choice you can use the `-e <gateway option>` flag, ex:

```bash
helmfile apply -e kgateway -n ${NAMESPACE}
```

To see what gateway options are supported refer to our [gateway provider prereq doc](../prereq/gateway-provider/README.md#supported-providers). Gateway configurations per provider are tracked in the [gateway-configurations directory](../prereq/gateway-provider/common-configurations/).

You can also customize your gateway, for more information on how to do that see our [gateway customization docs](../../docs/customizing-your-gateway.md).

---

## Deploy the Body-Based Routing (BBR) Extension

The Body-Based Router (BBR) acts as the intelligent dispatcher. It extracts the model name from the incoming HTTP request body, looks up the corresponding base model via a ConfigMap, and injects this information into the `X-Gateway-Base-Model-Name` header. The Gateway then uses this header to route traffic to the correct `InferencePool` using the matching `HttpRoute`.

> **Note:** All base model and LoRA names must be globally unique across your deployment.

* Set the gateway extensions version
Check for Gateway API Inference Extension version used in the deployment helmfile
```bash
export IGW_CHART_VERSION=$(helm get metadata gaie-multi-model -n $NAMESPACE -o json 2>/dev/null | \
  jq -r '.version // empty')
```
or look for latest available version as a fallback if it does not exists
```bash
  export IGW_CHART_VERSION=$(curl -s https://api.github.com/repos/kubernetes-sigs/gateway-api-inference-extension/releases \
  | jq -r '.[] | select(.prerelease == false) | .tag_name' \
  | sort -V \
  | tail -n1)
```

* Identify your gateway name
```bash
export GATEWAY_NAME=$(kubectl get gateway -n $NAMESPACE -o jsonpath='{.items[0].metadata.name}')
```

* Choose your Gateway provider and deploy the BBR via Helm:

```bash
export GATEWAY_PROVIDER=istio # Options: gke, istio, or none

helm install body-based-router \
  --set provider.name=$GATEWAY_PROVIDER \
  --set inferenceGateway.name=$GATEWAY_NAME \
  --set bbr.plugins=null \
  --version $IGW_CHART_VERSION \
  oci://us-central1-docker.pkg.dev/k8s-staging-images/gateway-api-inference-extension/charts/body-based-routing
```

*(Note for Kgateway users: Kgateway implements BBR natively via an `AgentgatewayPolicy` and does not require this Helm chart).*

---

## Upgrade the first model

**1. Create a ConfigMap for the first model:**
Body-based-routing looks for a model ConfigMap with the model and it's LoRAs

```bash
kubectl apply -f vllm-qwen3-32b-adapters-allowlist
```

**2. Upgrade the Helm release for the first model to enable experimental HTTP routes and generate an HTTPRoute backed by the inferencepool:**
```bash
helm upgrade gaie-multi-model \
 oci://us-central1-docker.pkg.dev/k8s-staging-images/gateway-api-inference-extension/charts/inferencepool \
  --version $IGW_CHART_VERSION \
  --dependency-update \
  --set provider.name=$GATEWAY_PROVIDER \
  --set "inferencePool.modelServers.matchLabels.llm-d\.ai/model=Qwen3-32B" \
  --set experimentalHttpRoute.enabled=true \
  --set experimentalHttpRoute.baseModel=Qwen/Qwen3-32B \
  --set experimentalHttpRoute.inferenceGatewayName=$GATEWAY_NAME
```

Make sure you can select the model node (used by the InferencePool):
```bash
kubectl get pods -l llm-d.ai/model=Qwen3-32B
```

Verify all pods are up and running, and that you now have one InferencePool and one HttpRoute configured
```bash
kubectl get inferencepools
kubectl get httproute
kubectl get pods
```

Examine httpexpriements values were set correctly:
```bash
helm get values gaie-multi-model -n ${NAMESPACE}
```

Check that the HTTPRoute and InferencePool were successfully configured and references were resolved:
```bash
kubectl get httproute gaie-multi-model -o yaml
kubectl get inferencepool gaie-multi-model -o yaml
```
The HttpRoute and InferencePool status should include Accepted=True and ResolvedRefs=True
---

## Deploy a Second Model Server

In addition to the primary model deployed, we will introduce a second model server using a vLLM simulator. In this example, we deploy `deepseek/deepseek-r1` as the base model, serving two distinct LoRAs: `ski-resorts` and `movie-critique`.

Apply the manifest to deploy the second model server and its LoRA-to-Base-Model mapping:

```bash
kubectl apply -f mdeepseek.yaml
```
Make sure you can select the model node (used by the InferencePool):
```bash
kubectl get pods -l app=vllm-deepseek-r1
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
  --set experimentalHttpRoute.baseModel=deepseek/DeepSeek-r1 \
  --set experimentalHttpRoute.inferenceGatewayName=$GATEWAY_NAME
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
kubectl get httproute vllm-deepseek-r1 -o yaml
kubectl get inferencepool vllm-deepseek-r1 -o yaml
```
The HttpRoute and InferencePool status should include Accepted=True and ResolvedRefs=True

By completing this well-lit path, your deployment is now capable of scheduling by dynamically delegating traffic across multiple base inference pools, gracefully maintaining LoRA affinities behind a single endpoint.
---

## Verify the Installation

* Firstly, you should be able to list all helm releases to view the 5 charts got installed into your chosen namespace:

```bash
helm list -n ${NAMESPACE}
NAME            REVISION	STATUS  	CHART                    	APP VERSION
body-based-router	  	1    deployed	body-based-routing-v0    	v0
gaie-multi-model 	  	2    deployed	inferencepool-v1.3.1     	v1.3.1
infra-multi-model	  	1    deployed	llm-d-infra-v1.3.6       	v0.3.0
ms-multi-model   	  	1    deployed	llm-d-modelservice-v0.4.7	v0.4.0
vllm-deepseek-r1 	  	1    deployed	inferencepool-v1.3.1     	v1.3.1
```

* Out of the box with this example you should have the following resources:
```bash
kubectl get all -n ${NAMESPACE}
NAME                                                          READY   STATUS
pod/body-based-router-599577fdc6-h4kq7                         1/1  Running
pod/gaie-multi-model-epp-6ddcb7d564-6kq89                      1/1     Running
pod/infra-multi-model-inference-gateway-istio-55b487c66c-kkf5p 1/1     Running
pod/ms-multi-model-llm-d-modelservice-decode-cff66486-xvnvn    1/1     Running
pod/vllm-deepseek-r1-64d85c5c69-tljgt                          1/1     Running
pod/vllm-deepseek-r1-epp-55f6956758-5fswn                      1/1     Running

NAME                                                TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)
service/body-based-router                           ClusterIP   172.30.100.0    <none>        9004/TCP
service/gaie-multi-model-epp                        ClusterIP   172.30.152.45   <none>        9002/TCP,9090/TCP
service/gaie-multi-model-ip-e79c9b5e                ClusterIP   None            <none>        54321/TCP
service/infra-multi-model-inference-gateway-istio   ClusterIP   172.30.154.46   <none>        15021/TCP,80/TCP
service/vllm-deepseek-r1-epp                        ClusterIP   172.30.169.17   <none>        9002/TCP,9090/TCP
service/vllm-deepseek-r1-ip-452ad6f3                ClusterIP   None            <none>        54321/TCP

NAME                                                        READY   UP-TO-DATE   AVAILABLE
deployment.apps/body-based-router                           1/1     1            1
deployment.apps/gaie-multi-model-epp                        1/1     1            1
deployment.apps/infra-multi-model-inference-gateway-istio   1/1     1            1
deployment.apps/ms-multi-model-llm-d-modelservice-decode    1/1     1            1
deployment.apps/vllm-deepseek-r1                            1/1     1            1
deployment.apps/vllm-deepseek-r1-epp                        1/1     1            1

NAME                                                                   DESIRED   CURRENT   READY
replicaset.apps/body-based-router-599577fdc6                           1         1         1
replicaset.apps/gaie-multi-model-epp-6ddcb7d564                        1         1         1
replicaset.apps/infra-multi-model-inference-gateway-istio-55b487c66c   1         1         1
replicaset.apps/ms-multi-model-llm-d-modelservice-decode-cff66486      1         1         1
replicaset.apps/vllm-deepseek-r1-64d85c5c69                         1         1         1
replicaset.apps/vllm-deepseek-r1-epp-55f6956758                        1         1         1
```
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
curl -X POST http://localhost:8080/v1/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "Qwen/Qwen3-32B",
    "prompt": "Hello, how are you?",
    "max_tokens": 50
  }'
```

### Test 2: Route to DeepSeek model
```bash
curl -X POST http://localhost:8080/v1/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "deepseek/DeepSeek-r1",
    "prompt": "What is the best ski resort in Austria?",
    "max_tokens": 50,
    "temperature": 0
  }'
```

### Test 3: Route to DeepSeek LoRA (`ski-resorts`)
```bash
curl -X POST http://localhost:8080/v1/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "ski-resorts",
    "prompt": "Tell me about ski deals in Austria?",
    "max_tokens": 50,
    "temperature": 0
  }'
```

### Test 4: Route to DeepSeek LoRA (`movie-critique`)
```bash
curl -X POST http://localhost:8080/v1/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "movie-critique",
    "prompt":  "What are the best movies of 2025?",
    "max_tokens": 100,
    "temperature": 0
  }'
```

## Cleanup

To remove the deployment:

```bash
helmfile destroy -n ${NAMESPACE}

# Or uninstall manually
helm uninstall infra-multi-model -n ${NAMESPACE} --ignore-not-found
helm uninstall gaie-multi-model -n ${NAMESPACE}
helm uninstall ms-multi-model -n ${NAMESPACE}
```

**_NOTE:_** If you set the `$RELEASE_NAME_POSTFIX` environment variable, your release names will be different from the command above: `infra-$RELEASE_NAME_POSTFIX`, `gaie-$RELEASE_NAME_POSTFIX` and `ms-$RELEASE_NAME_POSTFIX`.

Cleanup BodyBasedRouter and Second model:
```bash
helm uninstall body-based-router
helm uninstall vllm-deepseek-r1
kubectl delete -f mdeepseek.yaml  -n ${NAMESPACE}
```
