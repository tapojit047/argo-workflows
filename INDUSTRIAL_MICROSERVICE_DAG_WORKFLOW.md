# Simple Industrial Microservice DAG Workflow

Prepared on 2026-05-22.

This is a short, runnable industrial-style inference workflow. It is **synthetic/generated**, not copied from an existing industrial repository.

## Idea

A factory camera captures one part. The workflow checks whether the image is usable, inspects the part, makes a decision, and prints a final report.

```text
camera event
  -> quality check
  -> visual inspection
  -> decision
  -> report
```

That is the whole workflow.

## Artifact

Runnable Argo Workflow:

- `artifacts/industrial-visual-inspection-workflow.yaml`

Run it:

```bash
argo submit artifacts/industrial-visual-inspection-workflow.yaml -n argo --watch
```

View output:

```bash
argo logs @latest -n argo
```

## Input

The workflow takes two parameters:

| Parameter | Example | Meaning |
|---|---|---|
| `image-id` | `line-17-frame-000184` | Camera frame or image ID |
| `part-type` | `pump-housing` | Expected part type |

Override them like this:

```bash
argo submit artifacts/industrial-visual-inspection-workflow.yaml \
  -n argo \
  -p image-id=line-4-frame-0091 \
  -p part-type=gearbox-cover \
  --watch
```

## Output

The final task prints one JSON report:

```json
{
  "image_id": "line-17-frame-000184",
  "part_type": "pump-housing",
  "quality": "good",
  "defect": "none",
  "decision": "pass"
}
```

The predictions are mocked but deterministic. Changing `image-id` changes the mock scores.

## DAG Nodes

| Node | Meaning |
|---|---|
| `camera-event` | Creates the input event JSON |
| `quality-check` | Simulates image quality inference |
| `visual-inspection` | Simulates part/defect inspection inference |
| `decision` | Chooses `pass`, `rework`, or `manual-review` |
| `report` | Prints the final output |

## Real-Service Version

In a real microservice workflow, the mock stages would become HTTP calls:

```text
camera event
  -> curl http://quality-service/predict
  -> curl http://inspection-service/predict
  -> decision service
  -> audit/report sink
```

So this file is best understood as a small runnable Argo DAG skeleton for an industrial inference workflow.
