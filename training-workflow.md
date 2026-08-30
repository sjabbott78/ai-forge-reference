# Training Workflow: Rent the Forge, Own the Weights

[Back to README](README.md)

## The Sovereignty Constraint

EASM scan data contains real infrastructure details — software versions, exposed service endpoints, configuration signals from live systems. That data cannot leave the platform. It cannot be sent to a third-party training service or ingested by a managed ML provider. The sovereignty constraint is not a preference about cost or vendor lock-in; it is a hard requirement of the domain.

This constraint shapes the entire training architecture. Every stage of the workflow — data collection, sanitization, training, artifact storage, deployment — is designed to keep data on the platform unless there is a specific, bounded reason to leave it. The only exception is compute for production-scale training: the 7B validation adapter runs on sovereign hardware — the same P5000 GPUs used for production inference. Scaling to a production-grade 32B model requires A100-class compute that is not available on-premises. Cloud GPU instances are used as temporary forges for that stage only. The compute is rented. Everything else stays sovereign.

---

## The Local Data Forge

Before any model training, the training dataset must be built. The local data forge is the controlled pipeline for doing that without exposing real target data.

The forge mirrors the production scanning infrastructure locally — the same scanner container images that run as Kubernetes jobs in production are run locally against approved targets. An orchestrator loops through a target list and fires ephemeral scanner containers, one per target per scanner type. A single-shot webhook receiver catches each scanner's JSON payload, saves it to disk, and terminates — one receiver instance per scan, with no shared state between captures.

This design is deliberate. A persistent receiver that accumulates payloads creates race conditions and makes it harder to attribute a given payload to a specific scan invocation. The single-shot pattern means each payload is cleanly isolated and immediately auditable.

**Sanitization:** Raw payloads contain data that is appropriate for a security scan but not for a training dataset — internal hostnames, IP addresses, organization-specific identifiers that would make the training data traceable. A sanitization pass strips this data before any JSONL files are built. The sanitized data is what enters the training pipeline; raw payloads remain in a gitignored local directory.

**JSONL construction:** Sanitized scan data is combined with labeled classification outcomes to build the training examples. Each example follows the schema described in [Pipeline Architecture](pipeline-architecture.md) — input prompt, expected chain-of-thought reasoning, expected JSON output, stage label, and source label. The source label distinguishes authorized-scan examples from controlled-environment examples and adversarial test cases, which is used to weight the training mix.

---

## Two-Stage Training Design

The training workflow is deliberately two-staged. The first stage validates everything — data collection, prompt engineering, training schema, and evaluation criteria — at low cost on sovereign hardware before any money is spent on cloud compute. The second stage scales the validated approach to a larger model on a cloud GPU instance once confidence in the methodology is established.

This sequencing matters. Cloud GPU instances are not cheap. Renting an A100 to discover that the training data has a labeling inconsistency, or that the prompt schema doesn't produce the reasoning structure expected, is an expensive way to find a problem that could have been found locally. The 7B stage exists specifically to surface those problems before they have a cloud price tag attached.

---

## Stage 1 — Sovereign Hardware (7B Validation)

The Qwen 7B LoRA adapter was trained on the platform's own P5000 GPUs — the same LLM VMs that run production inference. QLoRA (Quantized Low-Rank Adaptation) makes this practical: training on quantized weights dramatically reduces VRAM requirements compared to full fine-tuning, putting 7B adapter training within reach of the P5000's available VRAM.

Stage 1 validates:
- **Data collection pipeline** — does the local forge produce correctly structured, sanitized JSONL that the training script accepts cleanly?
- **Prompt engineering** — do the per-stage system prompts produce the reasoning structure and output schema the training examples target?
- **Training schema** — does the `{input, thinking, output, stage, source}` structure produce a trainable signal? Does loss converge correctly?
- **Evaluation criteria** — does the adversarial test suite surface the failure modes it was designed to catch, and does a trained adapter actually pass it?

The complete training workflow for Stage 1 — dataset, training run, and resulting adapter — never left the platform. The sovereignty constraint applied to the compute as well as the data.

**Why adapters instead of custom base models:** An adapter is a small set of weight deltas applied on top of an existing base model. The base model does not need to be retrained — only the delta is stored, versioned, and promoted. Artifact sizes stay manageable, and rollback is trivial.

---

## Stage 2 — Cloud Forge (Production Scale)

Once the data pipeline, prompt engineering, and evaluation criteria are validated against the 7B, the same methodology scales to a production-grade model on a cloud GPU instance. The training script and hyperparameter configuration are identical — only the base model and the compute change.

The workflow:

1. Dataset validated in Stage 1 is used directly — no changes
2. A RunPod A100 instance is provisioned on demand
3. The training script runs with Unsloth QLoRA — same script, same hyperparameters
4. The resulting adapter is pulled back to the platform immediately
5. The cloud instance is destroyed
6. The adapter is pushed to MinIO and promoted through the eval suite

The cloud instance touches the sanitized training data and produces the adapter. It does not touch raw scan data, production credentials, or the EASM application. The cloud exposure surface is bounded to: sanitized JSONL files in, adapter weights out.

The cloud instance is temporary by design. The adapter comes home before the instance is destroyed.

---

## Artifact Lifecycle

Every trained adapter is versioned and stored in MinIO before any deployment step runs.

**MinIO artifact path:** `platform-artifacts/ai-forge/{model}/adapters/{version}/`

The version tag follows semantic versioning. The minor version increments on dataset additions or hyperparameter changes that are expected to improve behavior. The major version increments on base model changes or pipeline architecture changes that require re-evaluation from scratch.

A dead drop record is written to MinIO at the end of every successful deployment — a manifest recording the adapter version, the target VMs it was deployed to, the eval suite results that authorized promotion, and the timestamp. This record is the audit trail for the production inference configuration.

**Why MinIO instead of a managed artifact registry:** The same sovereignty constraint that applies to training data applies to the artifacts produced from it. An adapter trained on sanitized EASM data is an artifact that encodes domain knowledge about the platform's attack surface data. It stays on the platform.

---

## Adapter Promotion

The adapter is not deployed directly from the training step. Promotion requires the eval suite to pass first.

The Jenkins pipeline enforces this gate: the eval stage runs `benchmark.py` against the adversarial test suite on a live LLM VM using the new adapter. If the suite passes, the promote stage deploys the adapter to all target VMs and reloads Ollama. If the suite fails, the pipeline stops and the existing adapter remains in production.

The `SKIP_TRAINING` parameter separates the training path from the eval-and-promote path. When set to true, the pipeline pulls the specified adapter version from MinIO and runs eval + promote without triggering a new training run. This allows re-promotion of a previous adapter version, re-running the eval suite after a model configuration change, or deploying to a new VM that was added to the cluster — all without incurring cloud training cost.

---

## Related Documentation

- [Pipeline Architecture](pipeline-architecture.md) — the 5-stage sieve design the adapter is trained to serve
- [Adversarial Evaluation](adversarial-evaluation.md) — the eval suite that gates adapter promotion
- [Jenkins Integration](jenkins-integration.md) — the CI/CD pipeline that orchestrates the full lifecycle
