# AI Forge Reference: Domain-Specific MLOps for EASM

A production MLOps pipeline for training, evaluating, and deploying domain-specific AI adapters against a sovereign LLM inference cluster. This repository documents the architectural decisions behind that pipeline — the pipeline design, the evaluation philosophy, the data sovereignty model, and the CI/CD integration that promotes adapters from training to production.

This is not a general-purpose MLOps framework. It is a purpose-built system for a specific domain — External Attack Surface Management — and the design decisions reflect that specificity at every layer.

---

## Why Adversarial Evaluation Comes First

Most ML pipelines validate models by checking how well they reproduce patterns in training data. That approach has a fundamental limitation: it confirms the model has memorized the training distribution, not that it can reason about the domain.

This pipeline takes a different approach. The adversarial evaluation suite is built around a single constraint: **test inputs and severity rules share no exact terminology**. The model cannot pass these tests by surface-feature matching. It must derive the relationship between a vulnerability description and a severity classification independently — the same reasoning required in production, where novel findings will not match anything in the training set.

The suite covers twelve test scenarios:

| Test | What It Validates |
|---|---|
| Severity triage with ambiguity | Does the model resolve competing signals correctly, or does it pick the highest-confidence surface match? |
| Contextual negation and chaining | Does context that negates a finding actually reduce its severity, or is the finding scored in isolation? |
| CVE correlation | Can the model connect CVE metadata to severity implications without shared keywords? |
| Encoded telemetry | Does encoding obscure a finding enough to change its classification? |
| Malformed input handling | How does the model behave at the edges of its input contract? |
| Conflicting version data | When two pieces of information contradict each other, which wins and why? |
| Version string ambiguity | Can the model reason about software version ranges without explicit comparisons in the training data? |
| CVE note conditions | Conditional severity language in CVE notes — does the model evaluate the condition or ignore it? |
| Empty stream edge case | Is a zero-finding result handled correctly without hallucinating findings? |
| Prompt injection resistance | Does embedded adversarial content in scan data influence classification behavior? |
| Large stream / lost-in-the-middle | Do findings buried in a large result set receive the same treatment as findings at the beginning? |
| Combined stress test | All failure modes simultaneously — the production-equivalent test. |

The test suite exists because the alternative is deploying a model that passes training benchmarks but fails on real attack surface data that looks nothing like the training set.

---

## The Forge Philosophy

The pipeline is built around two distinct sovereignty constraints that apply to different data flows.

**Production inference is fully sovereign.** Real EASM scan data — the actual attack surface findings from systems under assessment — never leaves the platform. It is not sent to an external API, not used to train a third-party model, and not processed anywhere outside the platform's own LLM VM cluster. This is a hard requirement of the domain, not a preference.

**Training compute is rented, not owned.** The training dataset is built from three sources: real scanner output from authorized external targets (bug bounty programs, VDP sites, and other targets where scanning is explicitly permitted), controlled environment scans (deliberately vulnerable server and Docker configurations that produce real findings from real scanners), and adversarial test cases constructed specifically to cover edge cases and failure modes. No customer scan data is used. The training data has no operational sensitivity and can leave the platform boundary. Cloud GPU instances (A100-class) receive the training dataset, execute the training run, and hand back the resulting adapter weights. The cloud instance is destroyed immediately after. The weights are pulled back to the platform and stored in its own MinIO instance.

The operational model this produces: **rent the forge, own the weights**. From the moment the adapter is promoted to production, inference runs entirely on the platform's sovereign LLM VM cluster. The cloud touched the training data. It never touches production scan data.

**Why the implementation isn't here:** The training scripts, `benchmark.py`, and scanner containers described throughout this repo exist in a private codebase — not because the pipeline doesn't work, but because it's the operational core of an active EASM platform. Publishing it would mean publishing the exact detection logic and classification thresholds that an adversary would want to test against before targeting the platform. The same sovereignty principle that keeps scan data off the public internet applies to the code that processes it.

---

## Repository Structure

```
ai-forge-reference/
├── README.md                    ← this file
├── pipeline-architecture.md     ← 5-stage sieve design and why each stage is separate
├── training-workflow.md         ← data sovereignty model, local forge, cloud forge, artifact lifecycle
├── adversarial-evaluation.md    ← evaluation philosophy, test design, pass/fail criteria
└── jenkins-integration.md       ← CI/CD pipeline: validate → forge → eval → promote
```

If you're reading linearly, [pipeline-architecture](pipeline-architecture.md) establishes what the system does, [adversarial-evaluation](adversarial-evaluation.md) covers how it's validated, [training-workflow](training-workflow.md) covers how adapters are built, and [jenkins-integration](jenkins-integration.md) shows how those pieces connect into a deployable CI/CD lifecycle. Each document also stands on its own.

---

## How It Fits

This pipeline sits between two other systems. On one side: the platform's EASM application, which runs scanner jobs against a target's attack surface and receives findings as structured JSON. On the other side: the ZTSRCP sovereign platform, which hosts the LLM VM cluster that runs inference.

The connection to both is intentionally thin. The EASM application calls Ollama and receives JSON back — it has no knowledge of the pipeline architecture, the model version, or the adapter in use. The ZTSRCP platform provides the compute and the secrets infrastructure the pipeline requires, but the pipeline is not coupled to any specific cluster configuration.

The decoupling is deliberate. The forge pipeline can evolve — new training approaches, different base models, updated eval criteria — without touching the EASM app or the platform configuration.

**The work documented in this repository was executed on the sovereign cloud's LLM cluster** — dual Quadro P5000 GPUs across two Proxmox nodes, running Ollama on VMs provisioned on that platform. The initial Qwen 7B LoRA adapter was trained on those GPUs. The adversarial eval suite was run against that same hardware. The benchmark logs are not synthetic outputs; they are actual inference runs on actual sovereign infrastructure.

See the [Sovereign Cloud Reference](https://github.com/sjabbott78/sovereign-cloud-reference) for the platform context and [ZTSRCP App](https://github.com/sjabbott78/ztsrcp-app) for an example of a deployed application on that platform.

---

## AI Attribution

The implementation code throughout this project was developed using AI as an active part of the engineering workflow. I will not claim otherwise. What I contributed: the architecture, the constraints definition, the engineering decisions, the "why" behind every design choice, the validation that the implementation was correct for this specific context, the corrections when it wasn't, and the operational knowledge that comes from running these systems in production. What AI contributed: accelerated implementation of those decisions — generating code that I directed, reviewed, tested, and corrected.

I do not view AI-assisted development as a weakness or a shortcut. I view it as the future of engineering. AI is the encyclopedia of software — but you still have to know what problem you are solving, why a particular approach fits your constraints, whether the output is correct, and how to integrate it into a system that actually works. That judgment is what these reference repositories are designed to demonstrate.

This is also a domain where that judgment is especially load-bearing. Using AI to build an AI pipeline means the decisions about what to test, how to design adversarial cases, what constitutes a meaningful failure, and whether a model is actually reasoning or pattern matching — none of those come from the tool. They come from understanding the problem.
