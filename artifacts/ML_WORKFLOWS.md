# ML Argo Workflow Examples

This folder contains beginner-friendly machine learning workflow examples for Argo Workflows.

These examples use public Python images and install Python packages at runtime. That makes them easy to try, but slower than production workflows. In production, build your own container image with dependencies already installed.

## 1. Single-Step Iris Training

File: `ml-iris-train.yaml`

What it does:

1. Loads the built-in Iris dataset from scikit-learn.
2. Splits the data into train and test sets.
3. Trains a logistic regression model.
4. Prints accuracy and a classification report.

Run it:

```bash
argo submit artifacts/ml-iris-train.yaml -n argo --watch
```

View logs:

```bash
argo logs @latest -n argo
```

## 2. Multi-Step ML Pipeline with a Shared Volume

File: `ml-pipeline-volume.yaml`

What it does:

1. `prepare-data`: creates train and test CSV files.
2. `train-model`: trains a random forest model and saves it.
3. `evaluate-model`: loads the saved model and prints accuracy.

Run it:

```bash
argo submit artifacts/ml-pipeline-volume.yaml -n argo --watch
```

This workflow uses a Kubernetes `volumeClaimTemplates` section so steps can share files through `/mnt/work`.

Your cluster needs a working default storage class for this example. If the workflow stays pending, check the PVC and pod events:

```bash
kubectl get pvc -n argo
kubectl get pods -n argo
kubectl describe pod <pod-name> -n argo
```

## 3. Parallel Hyperparameter Search

File: `ml-hyperparameter-search.yaml`

What it does:

1. Runs four logistic regression training jobs in parallel.
2. Each job uses a different `C` value.
3. Each job writes its accuracy as an output parameter.
4. A final step compares the results and prints the best one.

Run it:

```bash
argo submit artifacts/ml-hyperparameter-search.yaml -n argo --watch
```

This example teaches parallel steps and output parameters.

## Notes

These workflows require the workflow pods to reach PyPI so they can run `pip install`. If your cluster has no internet access, build a custom image that already contains packages like `scikit-learn`, `pandas`, and `joblib`.

A production ML workflow would usually add:

- a custom ML container image
- durable artifact storage
- model registry upload
- experiment tracking
- resource requests and limits
- secrets for private data sources
- pinned package versions
