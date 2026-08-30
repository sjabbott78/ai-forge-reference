# Jenkins Integration: CI/CD for the Model Lifecycle

[Back to README](README.md)

## The Problem

ML model deployment is typically a manual process: train a model, evaluate it manually, copy weights to a server, restart the inference service. This approach has no auditability, no rollback path, and no enforcement of the quality gate between evaluation and production.

The forge pipeline treats model promotion the same way the platform treats application deployment: as a CI/CD problem. Every adapter version that reaches production passed through the same pipeline, in the same order, with the same quality gate. The history of what was deployed, when, to which hosts, and with what eval results is recorded automatically.

---

## Pipeline Stages

The Jenkins pipeline runs six stages in sequence. Stages cannot be reordered and cannot be skipped except via the `SKIP_TRAINING` parameter, which is not a bypass — it controls which entry point the pipeline uses, not whether the quality gate runs.

### Stage 1 — Validate Dataset

Lints the JSONL training files and verifies schema compliance before any compute is spent. A malformed training example that reaches the cloud forge wastes cloud GPU time and may produce a silently incorrect adapter. Catching format errors locally, before the training run, costs nothing.

The validator checks: JSON parsability, presence of required fields (`input`, `thinking`, `output`, `stage`, `source`), valid `stage` values against the known stage list, and `output` conformance against the per-stage JSON schema.

### Stage 2 — Cloud Forge

Provisions a cloud GPU instance, uploads the validated dataset, and runs the training script. The resulting adapter (`.safetensors` files) is pulled back to the build node immediately on completion. The cloud instance is destroyed before the pipeline continues.

**This stage is skipped when `SKIP_TRAINING=true`.** In that case, the pipeline proceeds directly to Stage 3 with the adapter version specified by `ADAPTER_VERSION`.

### Stage 3 — Pull Adapter

Fetches the specified adapter version from MinIO. When `SKIP_TRAINING=false`, this is the adapter just produced by Stage 2 and uploaded automatically. When `SKIP_TRAINING=true`, this is whatever version is named by `ADAPTER_VERSION` — allowing re-promotion of any previously stored adapter without retraining.

The pull step verifies the adapter's checksum against the value recorded when it was first stored. A checksum mismatch fails the pipeline.

### Stage 4 — Eval Suite

Runs `benchmark.py` against the live LLM VM cluster using the adapter under evaluation. The eval runs against the real inference hardware, not a test environment. An adapter that passes on the actual cluster is what gets promoted.

If any test in the adversarial suite fails, the pipeline stops here. The existing adapter in production is not touched. The failure details are available in the build log.

### Stage 5 — Promote

Deploys the adapter to each target VM in sequence. For each VM:

1. SSH to the VM using the platform's managed SSH credential
2. Copy the adapter files to the Ollama model directory
3. Reload Ollama so the new adapter is active

The deployment is sequential, not parallel. Rolling one VM at a time means a deployment failure on one VM leaves the remaining VMs on the previous adapter version — a defined state — rather than leaving the cluster partially updated in an undefined state.

### Stage 6 — Dead Drop Record

Writes a deployment manifest to MinIO:

```json
{
  "adapter_version": "v1.2.0",
  "model": "easm",
  "deployed_at": "2026-08-21T14:32:00Z",
  "target_vms": ["llm-vm-01", "llm-vm-02"],
  "eval_results": { "passed": 12, "failed": 0 },
  "base_model": "qwq:32b",
  "promoted_by": "jenkins"
}
```

This record is the audit trail for the current production configuration. At any point, the platform can determine what adapter version is running, when it was deployed, what eval results authorized promotion, and what VMs are running it.

---

## Parameters

| Parameter | Default | Description |
|---|---|---|
| `MODEL` | `easm` | Model directory to target — determines which dataset, training config, and prompts are used |
| `SKIP_TRAINING` | `true` | When true, skip Stage 2 and pull the named adapter version from MinIO; when false, run the full training cycle |
| `ADAPTER_VERSION` | `v1.0.0` | Version tag used for MinIO artifact path — must exist in MinIO when `SKIP_TRAINING=true` |
| `TARGET_VMS` | (configured per environment) | Comma-separated list of LLM VM hostnames to promote to |

**Why `SKIP_TRAINING` defaults to true:** Training runs are expensive and deliberate. A pipeline that defaults to triggering a cloud training run on every invocation would make accidental training runs easy and cheap training re-runs awkward. The correct default is to run the eval-and-promote path only, with training as an explicit opt-in. This also means the eval suite and promotion mechanism can be tested and verified independently of the training infrastructure.

---

## Integration With the Platform Shared Library

The forge pipeline is a standalone Jenkins scripted pipeline, not a consumer of the platform's shared library. The shared library handles application deployment — provisioning namespaces, managing Kubernetes secrets, running Helm upgrades. The forge pipeline handles a different lifecycle: ML artifacts moving from a training run to a set of VMs via SSH and a MinIO artifact store.

The two pipelines share the Jenkins infrastructure and the credential store, but the forge pipeline's stages are purpose-built for the model lifecycle and would not benefit from the shared library's application-deployment abstractions.

---

## Rollback

Because every promoted adapter is stored in MinIO with a versioned path and a dead drop record, rollback is a one-parameter pipeline invocation:

1. Set `SKIP_TRAINING=true`
2. Set `ADAPTER_VERSION` to the previous version tag
3. Run the pipeline

The eval suite runs against the previous adapter before it is re-deployed. If the previous adapter also fails the eval suite, the pipeline stops. Rolling back to a known-bad adapter is not possible without investigation.

---

## Related Documentation

- [Training Workflow](training-workflow.md) — the adapter lifecycle before it reaches Jenkins
- [Adversarial Evaluation](adversarial-evaluation.md) — the eval suite that runs in Stage 4
- [Pipeline Architecture](pipeline-architecture.md) — what the deployed adapter is serving
