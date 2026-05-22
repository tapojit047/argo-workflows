# Runnable Container-Based ML Prediction Workflow Recommendation

Prepared on 2026-05-22.

This document updates the earlier broad survey according to `prompt.txt`.

The prompt asks for one workflow for now with these constraints:

- Ignore serverless functions for now.
- Prefer a ready-made or existing workflow.
- The workflow should be an ML prediction/inference workflow, not mainly a training pipeline.
- It should have multiple DAG nodes/tasks.
- Each task should be container based.
- It should use web-service/HTTP style operation where possible.
- It can run on Argo Workflows, but another platform is acceptable if it is a better fit.
- It should be runnable.

## Selected Workflow

Use **KServe Image Processing Inference Pipeline with InferenceGraph**.

Main resource:

- <https://kserve.github.io/website/docs/model-serving/inferencegraph/image-pipeline>

Supporting resources:

- KServe InferenceGraph concept: <https://kserve.github.io/website/docs/concepts/resources/inferencegraph>
- KServe quickstart installation: <https://kserve.github.io/website/docs/getting-started/quickstart-guide>
- KServe GitHub repository: <https://github.com/kserve/kserve>

## Why This Is the Best Fit for Now

This is the best immediate candidate because it is:

- **Prediction-focused**: it performs image classification inference, not ML training orchestration.
- **Container based**: KServe deploys each model as a Kubernetes `InferenceService`, backed by model-serving containers/pods.
- **HTTP/web-service based**: requests are sent through an HTTP endpoint, and the graph router forwards requests to downstream inference services.
- **DAG-like**: the `InferenceGraph` defines ordered and conditional stages.
- **Runnable**: the official KServe tutorial provides model storage URIs, manifests, and test requests.
- **Kubernetes-native**: it runs on Kubernetes, which matches the Argo/Kubernetes environment.
- **Industrial direction**: KServe is a production-oriented model-serving platform, not a toy training script.

The first caveat is scale: the ready-made example has two ML model services plus the graph router, so it is a runnable starting point rather than the final large benchmark DAG. The extension section below shows how to grow it into a larger video analytics workflow with more nodes.

The second caveat is runtime choice: this workflow is better suited to **KServe** than raw **Argo Workflows**. Argo is excellent for batch workflows. KServe is a better option for online prediction workflows made of HTTP inference services.

Argo can still be used around it for batch testing, profiling, traffic generation, or scheduled experiments.

## Workflow DAG

The runnable KServe example has this prediction flow:

```text
Input image request
  -> InferenceGraph router
  -> cat-dog-classifier InferenceService
  -> if prediction is dog:
       dog-breed-classifier InferenceService
     else:
       return cat prediction
  -> final HTTP response
```

Expanded as a DAG:

```text
image JSON
  -> graph-router
  -> cat-dog-classifier
  -> condition-check
  -> dog-breed-classifier
  -> prediction-response
```

This is not a training pipeline. It is a multi-stage prediction pipeline.

## Workflow Nodes

| Node | Type | Container based? | HTTP/service based? | Purpose |
|---|---|---:|---:|---|
| `dog-breed-pipeline` | KServe `InferenceGraph` router | Yes | Yes | Receives request and controls routing |
| `cat-dog-classifier` | KServe `InferenceService` | Yes | Yes | Classifies image as cat or dog |
| condition check | InferenceGraph routing logic | Router container | Internal graph logic | Decides whether to run breed classifier |
| `dog-breed-classifier` | KServe `InferenceService` | Yes | Yes | Runs dog breed classification if needed |
| client/test driver | `curl` or Argo container task | Optional | Yes | Sends image request and records latency/output |

## Official KServe Manifest

I also added a runnable manifest here:

- `artifacts/kserve-dog-breed-inferencegraph.yaml`

It contains:

- `cat-dog-classifier` InferenceService
- `dog-breed-classifier` InferenceService
- `dog-breed-pipeline` InferenceGraph

The model artifacts are referenced from the public KServe example model storage URIs. The cluster must be able to pull KServe runtime images and download the model artifacts.

## How to Run It

### 1. Prepare Kubernetes

You need a Kubernetes cluster with `kubectl` configured.

For local testing, use one of:

- Kind
- Minikube
- Docker Desktop Kubernetes

### 2. Install KServe in Standard Mode

The prompt says to ignore serverless functions, so use KServe **standard mode**, not Knative/serverless mode.

From the official KServe quickstart:

```bash
curl -s "https://raw.githubusercontent.com/kserve/kserve/refs/tags/v0.17.0/install/v0.17.0/kserve-standard-mode-full-install-with-manifests.sh" | bash
```

Check KServe pods:

```bash
kubectl get pods -A | rg 'kserve|istio|cert-manager|envoy|gateway'
```

### 3. Deploy the Inference Services and Graph

```bash
kubectl apply -f artifacts/kserve-dog-breed-inferencegraph.yaml
```

Check the services:

```bash
kubectl get inferenceservices
kubectl get inferencegraph
kubectl get pods
```

Wait until the services are ready:

```bash
kubectl wait --for=condition=Ready inferenceservice/cat-dog-classifier --timeout=600s
kubectl wait --for=condition=Ready inferenceservice/dog-breed-classifier --timeout=600s
kubectl wait --for=condition=Ready inferencegraph/dog-breed-pipeline --timeout=600s
```

### 4. Send a Test Prediction Request

Follow the KServe tutorial section "Test the InferenceGraph" for the exact ingress setup and sample `cat.json` / `dog.json` payloads:

- <https://kserve.github.io/website/docs/model-serving/inferencegraph/image-pipeline/#test-the-inferencegraph>

General request shape:

```bash
SERVICE_HOSTNAME=$(kubectl get inferencegraph dog-breed-pipeline -o jsonpath='{.status.url}' | cut -d '/' -f 3)

curl -v \
  -H "Host: ${SERVICE_HOSTNAME}" \
  -H "Content-Type: application/json" \
  "http://${INGRESS_HOST}:${INGRESS_PORT}" \
  -d @./dog.json
```

Expected behavior:

- If the image is a cat, the pipeline returns the cat/dog classifier result.
- If the image is a dog, the pipeline sends the request to the dog breed classifier and returns breed probabilities.

## Why Not Argo Workflows as the Primary Runner?

Argo Workflows is still useful, but it is not the best primary tool for this specific workflow.

Reason:

- This is an **online prediction service graph**.
- Nodes are long-running HTTP inference services.
- The workflow is triggered by HTTP requests.
- Each model service should scale independently.

That is exactly what KServe `InferenceGraph` is designed for.

Argo is better when each node is a short-lived batch task, such as:

```text
preprocess dataset -> run batch inference -> aggregate results -> write report
```

KServe is better when the workflow is:

```text
HTTP request -> model service -> routing decision -> another model service -> HTTP response
```

## How Argo Can Still Be Used

Use Argo Workflows as a **driver** around the KServe graph.

For example:

```text
Argo task 1: prepare test images
Argo task 2: send N HTTP requests to KServe InferenceGraph
Argo task 3: collect latency and accuracy results
Argo task 4: summarize profiling results
```

This gives you:

- container-based profiling tasks
- repeatable experiments
- latency measurement
- batch request generation
- integration with scheduler experiments

An Argo wrapper DAG could look like:

```text
prepare-request-payloads
  -> warm-up-kserve-endpoint
  -> run-cat-requests
  -> run-dog-requests
  -> collect-latency-metrics
  -> summarize-results
```

## How to Make It Larger Later

The ready-made KServe example is a good runnable starting point, but the final project may need a larger DAG. It can be extended without changing the basic architecture.

Possible larger prediction DAG:

```text
input image/video frame
  -> image validation service
  -> resize/preprocess service
  -> object type classifier
  -> switch condition
  -> dog breed classifier
  -> cat breed classifier
  -> confidence scorer
  -> optional high-accuracy model
  -> result formatter
  -> storage/API/logging service
```

For project alignment, this can be turned into a fire/smoke or edge video analytics workflow:

```text
camera frame
  -> frame sampler
  -> resize/preprocess
  -> lightweight smoke/fire detector
  -> confidence gate
  -> high-accuracy detector if confidence is low
  -> event aggregator
  -> alert service
  -> result storage
```

This matches the project plan better than a training pipeline because it is prediction-oriented and can expose accuracy-latency tradeoffs.

## Why Other Candidates Were Not Picked for Now

| Candidate | Reason not selected for this prompt |
|---|---|
| vSwarm | Strong benchmark source, but prompt says ignore serverless functions for now. |
| SeBS / SeBS-Flow | Good FaaS workflow benchmark, but also serverless-focused. |
| TFX / Kubeflow Taxi pipeline | Industrial and containerized, but mainly a training/ML lifecycle pipeline. |
| AWS SageMaker Pipelines | Industrial, but also mostly training/evaluation/registration pipelines and requires AWS setup. |
| Azure ML pipelines | Useful, but many examples are train/score/eval pipelines rather than online prediction DAGs. |
| DeathStarBench | Strong microservice benchmark, but not an ML prediction workflow. |
| IoT Cloud Data Pipeline | Runnable and containerized, but not ML prediction unless we add a model node. |
| VLSC / ECO | Strong project alignment, but they are research designs rather than immediately runnable public workflows. |

## Final Recommendation

For now, use **KServe Image Processing Inference Pipeline with InferenceGraph**.

It is the best fit for the updated prompt because it is:

- runnable
- prediction-focused
- Kubernetes-native
- container-backed
- HTTP-based
- not primarily a training pipeline
- not a serverless-function workflow
- easy to extend into a larger video analytics DAG

Use KServe as the primary runtime for the online prediction workflow. Use Argo Workflows later as a profiling or batch-experiment driver around the KServe endpoint.

## Source Links

- KServe image processing InferenceGraph tutorial: <https://kserve.github.io/website/docs/model-serving/inferencegraph/image-pipeline>
- KServe InferenceGraph concept: <https://kserve.github.io/website/docs/concepts/resources/inferencegraph>
- KServe quickstart guide: <https://kserve.github.io/website/docs/getting-started/quickstart-guide>
- KServe GitHub repository: <https://github.com/kserve/kserve>
- Argo Workflows: <https://argoproj.github.io/workflows/>
- Project plan: `resources/Summer_Work_Plan.pptx`
