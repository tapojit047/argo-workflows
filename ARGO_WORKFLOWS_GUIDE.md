# Argo Workflows Beginner Guide

This guide explains Argo Workflows in simple words.

If you are new to Kubernetes, pipelines, or workflow tools, that is okay. The goal of this file is to help you understand the basic idea first, then slowly build up to real examples.

## 1. What Is Argo Workflows?

Argo Workflows is a tool for running multi-step jobs on Kubernetes.

In simple words:

> Argo Workflows lets you describe a job as a list of steps, and Kubernetes runs those steps for you.

For example, imagine you want to do this:

1. Download some data.
2. Clean the data.
3. Run a Python script.
4. Save the result.
5. Send a notification.

You could put all of that in one big script. But that can become hard to manage.

With Argo Workflows, you can describe each step separately. Argo then runs the steps in the right order.

## 2. Why Would You Use It?

Argo Workflows is useful when your work has many steps.

Common examples:

- Data processing
- Machine learning jobs
- Batch jobs
- Report generation
- Testing and build pipelines
- ETL jobs
- Automation tasks

You might use Argo Workflows when you need to say:

- Run step A, then step B.
- Run step A, then run B and C at the same time.
- If a step fails, retry it.
- Pass a value from one step to another.
- Pass a file from one step to another.
- Run this workflow every day.

## 3. The Big Idea

Argo Workflows runs tasks as containers.

That means each step usually runs inside a Docker image, such as:

- `alpine`
- `python`
- `node`
- your own custom application image

Argo does not directly run your code on your laptop. It creates Kubernetes pods, and those pods run your containers.

Simple mental model:

```text
Workflow YAML file
        |
        v
Argo Workflow Controller
        |
        v
Kubernetes Pods
        |
        v
Your containers run
```

## 4. Important Words

Before writing YAML, learn these basic terms.

### Workflow

A workflow is one complete run of a job.

Example:

```text
daily-report-abc123
```

This could be one run of a daily report workflow.

### Template

A template describes one piece of work.

A template might say:

- Run this container.
- Run this script.
- Run these steps.
- Run this DAG.

Think of a template like a recipe for one task.

### Entrypoint

The entrypoint tells Argo where to start.

Example:

```yaml
spec:
  entrypoint: main
```

This means:

> Start by running the template named `main`.

### Step

A step is one task in a workflow.

Example:

```text
download-data
clean-data
train-model
```

### DAG

DAG means "Directed Acyclic Graph".

That sounds scary, but the idea is simple:

> A DAG lets you say which tasks depend on other tasks.

Example:

```text
       download
       /      \
   clean     validate
       \      /
        report
```

Here:

- `download` runs first.
- `clean` and `validate` can run at the same time.
- `report` waits until both are finished.

### Parameter

A parameter is a small value passed into a workflow or step.

Examples:

- `environment=dev`
- `name=Alice`
- `imageTag=v1.2.3`

### Artifact

An artifact is a file passed between steps.

Examples:

- CSV file
- JSON file
- Machine learning model
- Report PDF
- Log file

Simple rule:

- Use **parameters** for small text values.
- Use **artifacts** for files.

## 5. Installing Argo Workflows

You need a Kubernetes cluster first.

For learning, you can use:

- Docker Desktop Kubernetes
- Minikube
- Kind
- A test cloud Kubernetes cluster

Create a namespace:

```bash
kubectl create namespace argo
```

Install Argo Workflows:

```bash
export ARGO_WORKFLOWS_VERSION="v4.0.5"

kubectl apply --server-side -n argo \
  -f "https://github.com/argoproj/argo-workflows/releases/download/${ARGO_WORKFLOWS_VERSION}/quick-start-minimal.yaml"
```

Check the official release page before installing in a real environment:

<https://github.com/argoproj/argo-workflows/releases>

## 6. Installing the Argo CLI

The Argo CLI is a command-line tool.

You use it to:

- submit workflows
- list workflows
- see workflow status
- view logs
- retry failed workflows

Install it from:

<https://github.com/argoproj/argo-workflows/releases>

Check that it works:

```bash
argo version
```

## 7. Your First Workflow

Create a file called `hello-world.yaml`.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
  generateName: hello-world-
spec:
  entrypoint: say-hello
  templates:
    - name: say-hello
      container:
        image: alpine:3.20
        command: [sh, -c]
        args: ["echo hello from Argo Workflows"]
```

Now submit it:

```bash
argo submit hello-world.yaml -n argo --watch
```

What happens?

1. Argo creates a new workflow.
2. Kubernetes starts a pod.
3. The pod runs the `alpine` container.
4. The container prints `hello from Argo Workflows`.
5. The workflow finishes.

View logs:

```bash
argo logs <workflow-name> -n argo
```

List workflows:

```bash
argo list -n argo
```

## 8. Understanding the YAML

Here is the same workflow again:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
  generateName: hello-world-
spec:
  entrypoint: say-hello
  templates:
    - name: say-hello
      container:
        image: alpine:3.20
        command: [sh, -c]
        args: ["echo hello from Argo Workflows"]
```

Line by line:

- `apiVersion`: tells Kubernetes this is an Argo Workflows object.
- `kind: Workflow`: says this file defines a workflow.
- `generateName`: tells Argo to create a name that starts with `hello-world-`.
- `entrypoint`: tells Argo which template to run first.
- `templates`: contains the tasks in the workflow.
- `container`: says this template runs a container.
- `image`: the container image to use.
- `command` and `args`: what the container runs.

## 9. Running Steps in Order

Now let us run two steps.

Create `two-steps.yaml`:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
  generateName: two-steps-
spec:
  entrypoint: main
  templates:
    - name: main
      steps:
        - - name: first
            template: print-message
            arguments:
              parameters:
                - name: message
                  value: "this is step one"

        - - name: second
            template: print-message
            arguments:
              parameters:
                - name: message
                  value: "this is step two"

    - name: print-message
      inputs:
        parameters:
          - name: message
      container:
        image: alpine:3.20
        command: [sh, -c]
        args: ["echo {{inputs.parameters.message}}"]
```

Submit it:

```bash
argo submit two-steps.yaml -n argo --watch
```

This workflow does:

1. Run `first`.
2. Wait for `first` to finish.
3. Run `second`.

The `print-message` template is reused by both steps.

## 10. Running Steps in Parallel

Steps inside the same list run at the same time.

Example:

```yaml
steps:
  - - name: task-a
      template: print-message
    - name: task-b
      template: print-message
```

Here, `task-a` and `task-b` run in parallel.

Full example:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
  generateName: parallel-
spec:
  entrypoint: main
  templates:
    - name: main
      steps:
        - - name: task-a
            template: print-message
            arguments:
              parameters:
                - name: message
                  value: "running task A"
          - name: task-b
            template: print-message
            arguments:
              parameters:
                - name: message
                  value: "running task B"

    - name: print-message
      inputs:
        parameters:
          - name: message
      container:
        image: alpine:3.20
        command: [sh, -c]
        args: ["echo {{inputs.parameters.message}}"]
```

## 11. Using Parameters

Parameters let you change values without changing the workflow logic.

Example:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
  generateName: parameter-
spec:
  entrypoint: main
  arguments:
    parameters:
      - name: username
        value: beginner
  templates:
    - name: main
      container:
        image: alpine:3.20
        command: [sh, -c]
        args: ["echo hello {{workflow.parameters.username}}"]
```

Submit with the default value:

```bash
argo submit parameter.yaml -n argo --watch
```

Override the value:

```bash
argo submit parameter.yaml -n argo -p username=Sam --watch
```

Now the workflow prints:

```text
hello Sam
```

## 12. Passing a Value from One Step to Another

Sometimes one step creates a value, and another step uses it.

Example:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
  generateName: pass-value-
spec:
  entrypoint: main
  templates:
    - name: main
      steps:
        - - name: create-message
            template: create-message

        - - name: print-message
            template: print-message
            arguments:
              parameters:
                - name: message
                  value: "{{steps.create-message.outputs.parameters.message}}"

    - name: create-message
      script:
        image: alpine:3.20
        command: [sh]
        source: |
          echo "hello from the first step" > /tmp/message.txt
      outputs:
        parameters:
          - name: message
            valueFrom:
              path: /tmp/message.txt

    - name: print-message
      inputs:
        parameters:
          - name: message
      container:
        image: alpine:3.20
        command: [sh, -c]
        args: ["echo {{inputs.parameters.message}}"]
```

Important part:

```yaml
value: "{{steps.create-message.outputs.parameters.message}}"
```

This means:

> Take the output parameter named `message` from the `create-message` step.

## 13. Passing Files with Artifacts

Use artifacts when you want to pass files between steps.

Example:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
  generateName: artifact-
spec:
  entrypoint: main
  templates:
    - name: main
      steps:
        - - name: create-file
            template: create-file

        - - name: read-file
            template: read-file
            arguments:
              artifacts:
                - name: message-file
                  from: "{{steps.create-file.outputs.artifacts.message-file}}"

    - name: create-file
      container:
        image: alpine:3.20
        command: [sh, -c]
        args: ["echo hello artifact > /tmp/message.txt"]
      outputs:
        artifacts:
          - name: message-file
            path: /tmp/message.txt

    - name: read-file
      inputs:
        artifacts:
          - name: message-file
            path: /tmp/input/message.txt
      container:
        image: alpine:3.20
        command: [sh, -c]
        args: ["cat /tmp/input/message.txt"]
```

This workflow:

1. Creates a file.
2. Saves it as an artifact.
3. Sends it to the next step.
4. Reads the file.

## 14. DAG Example

A DAG is useful when some tasks can run at the same time, but others must wait.

Example:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
  generateName: simple-dag-
spec:
  entrypoint: pipeline
  templates:
    - name: pipeline
      dag:
        tasks:
          - name: download
            template: echo
            arguments:
              parameters:
                - name: message
                  value: "download data"

          - name: clean
            dependencies: [download]
            template: echo
            arguments:
              parameters:
                - name: message
                  value: "clean data"

          - name: validate
            dependencies: [download]
            template: echo
            arguments:
              parameters:
                - name: message
                  value: "validate data"

          - name: report
            dependencies: [clean, validate]
            template: echo
            arguments:
              parameters:
                - name: message
                  value: "create report"

    - name: echo
      inputs:
        parameters:
          - name: message
      container:
        image: alpine:3.20
        command: [sh, -c]
        args: ["echo {{inputs.parameters.message}}"]
```

This runs in this order:

1. `download`
2. `clean` and `validate` at the same time
3. `report`

## 15. Retrying Failed Steps

Sometimes a task fails because of a temporary problem.

For example:

- network issue
- temporary API failure
- database not ready

You can tell Argo to retry.

```yaml
retryStrategy:
  limit: 3
```

Example:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
  generateName: retry-
spec:
  entrypoint: main
  templates:
    - name: main
      retryStrategy:
        limit: 3
      container:
        image: alpine:3.20
        command: [sh, -c]
        args: ["echo trying this task"]
```

This allows the task to try again if it fails.

## 16. Running a Workflow on a Schedule

Use `CronWorkflow` when you want to run a workflow regularly.

Example:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: CronWorkflow
metadata:
  name: daily-hello
  namespace: argo
spec:
  schedule: "0 9 * * *"
  timezone: "UTC"
  workflowSpec:
    entrypoint: main
    templates:
      - name: main
        container:
          image: alpine:3.20
          command: [sh, -c]
          args: ["echo hello from the daily workflow"]
```

This runs every day at 09:00 UTC.

Apply it:

```bash
kubectl apply -f daily-hello.yaml
```

## 17. Reusing Workflows with WorkflowTemplate

A `WorkflowTemplate` is a reusable workflow.

You create it once, then submit workflows from it many times.

Example:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: WorkflowTemplate
metadata:
  name: reusable-hello
  namespace: argo
spec:
  entrypoint: main
  arguments:
    parameters:
      - name: message
        value: hello
  templates:
    - name: main
      container:
        image: alpine:3.20
        command: [sh, -c]
        args: ["echo {{workflow.parameters.message}}"]
```

Create it:

```bash
argo template create reusable-hello.yaml -n argo
```

Run it:

```bash
argo submit --from workflowtemplate/reusable-hello -n argo -p message="hello again"
```

## 18. Useful Argo Commands

Submit a workflow:

```bash
argo submit hello-world.yaml -n argo
```

Submit and watch it:

```bash
argo submit hello-world.yaml -n argo --watch
```

List workflows:

```bash
argo list -n argo
```

Show one workflow:

```bash
argo get <workflow-name> -n argo
```

Show logs:

```bash
argo logs <workflow-name> -n argo
```

Retry a failed workflow:

```bash
argo retry <workflow-name> -n argo
```

Delete a workflow:

```bash
argo delete <workflow-name> -n argo
```

## 19. Debugging Problems

When something fails, start with:

```bash
argo get <workflow-name> -n argo
```

Then check logs:

```bash
argo logs <workflow-name> -n argo
```

If you need more detail, use Kubernetes commands:

```bash
kubectl get pods -n argo
kubectl describe pod <pod-name> -n argo
kubectl logs <pod-name> -n argo
```

Common problems:

- The container image name is wrong.
- Kubernetes cannot pull the image.
- The command inside the container fails.
- The workflow YAML has incorrect indentation.
- A parameter name is wrong.
- An artifact path is wrong.
- The workflow does not have permission to do something.
- The cluster does not have enough CPU or memory.

## 20. Beginner Learning Path

Learn Argo Workflows in this order:

1. Understand what a workflow is.
2. Run the hello-world workflow.
3. Learn `argo submit`, `argo list`, `argo get`, and `argo logs`.
4. Write a workflow with two steps.
5. Run two steps in parallel.
6. Use parameters.
7. Pass a value from one step to another.
8. Pass a file using artifacts.
9. Write a simple DAG.
10. Add retries.
11. Create a scheduled workflow.
12. Create a reusable workflow template.

## 21. Simple Practice Project

Try building this workflow:

1. Create a text file.
2. Count the number of words in the file.
3. Print the result.

After that, try a slightly bigger one:

1. Create a small CSV file.
2. Validate that the CSV has a header.
3. Count the rows.
4. Print a summary.

This will teach you:

- steps
- parameters
- artifacts
- logs
- debugging

## 22. Best Practices for Beginners

- Start with very small workflows.
- Use `alpine` for simple shell examples.
- Use `python` images for Python examples.
- Give steps clear names.
- Keep each step focused on one job.
- Use parameters for small values.
- Use artifacts for files.
- Check logs when something fails.
- Do not put passwords directly in YAML.
- Avoid using `latest` image tags in production.
- Add CPU and memory limits for real workloads.

## 23. Final Mental Model

Remember this:

> Argo Workflows is a way to run container-based steps on Kubernetes.

The YAML file says:

- what should run
- which container image to use
- what order steps should run in
- what values or files should move between steps

Argo reads that YAML and asks Kubernetes to run the work.

Once you understand that, the rest of Argo Workflows becomes much easier.
