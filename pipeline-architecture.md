# Pipeline Architecture: 5-Stage AI Sieve

[Back to README](README.md)

## The Problem With a Single Prompt

An EASM scan against a target can produce hundreds of findings across dozens of scanner types — Nmap service enumeration, Nuclei template matches, TLS configuration issues, header analysis, DNS enumeration. Feeding all of that into a single AI prompt has two failure modes.

**Context collapse:** A 32B reasoning model has a large context window, but the quality of its attention degrades over long inputs. Findings buried in the middle of a large result set receive less reliable treatment than findings at the beginning or end. A single-prompt approach means the reliability of a finding's severity classification depends partly on where it happened to appear in the scanner output.

**Precision loss:** A single prompt that asks the model to simultaneously inventory findings, classify severity across multiple categories, and produce a final reduced summary requires the model to hold competing objectives in tension. Performance on each objective degrades when they share a single inference pass.

This was validated empirically. A monolithic prompt test — all findings in a single inference pass — produced a correct output but the model's reasoning trace showed it reconsidering the same priority ordering decisions multiple times: re-examining where an open relay ranked relative to unauthenticated database exposure, re-evaluating compound severity rules mid-reasoning, backtracking on remediation phase assignments. The model arrived at correct answers after that deliberation, but the cognitive overhead was visible and the inference time reflected it. At the density of a real attack surface scan — dozens of findings across multiple scanner types — that deliberation compounds into misclassifications and unpredictable inference time.

The 5-stage sieve architecture addresses both problems by decomposing the task. Each stage has a single, narrow objective. Each stage operates only on the output of the previous stage, progressively reducing the volume and increasing the precision of the findings set.

---

## Pipeline Contract: consumed_ids

Each sieve stage returns not just its classified findings but a `consumed_ids` list — the source IDs of every finding classified at that tier. The orchestration layer removes those IDs from the working payload before passing it to the next stage. This ensures each finding is classified exactly once across the full pipeline, makes each stage independently retryable without reprocessing completed stages, and gives the orchestrator a complete accounting of which findings were consumed at which severity tier.

The inventory stage does not produce `consumed_ids` — it normalizes, it does not classify. The contract applies to stages 2 through 4, where classification decisions are made.

---

## CVE Enrichment

Between Stage 1 and the sieve stages, the orchestration layer enriches the normalized finding payload with CVE data. For each finding that carries a software version, the platform's local NVD mirror is queried and matching CVE records are injected into the payload alongside the finding.

The model does not perform CVE lookups. By the time a finding reaches a sieve stage, the relevant CVE data is already present in the prompt payload. The sieve reasons against that enriched data — evaluating version boundaries, CVE note conditions, OS specifiers, and severity implications — without needing to retrieve anything independently.

This separation keeps the model focused on reasoning rather than retrieval, and it means CVE data accuracy is determined by the NVD mirror, not by the model's training data.

---

## The Five Stages

### Stage 1 — Inventory

**Input:** Raw scanner output (structured JSON from each scanner type)
**Output:** Normalized finding list — one record per finding, schema-conformant, no severity classification

The inventory stage has one job: extract every finding from the raw scanner output and normalize it into a consistent schema. It does not classify. It does not filter. A scanner that reports the same finding in three different formats should produce one normalized record here.

This stage also handles the structural complexity that raw scanner output introduces — nested JSON, variable field names, findings that require joining data from multiple scanner records to construct a complete picture. The model's role is extraction and normalization, not analysis.

**Why a dedicated stage:** Mixing extraction with classification creates a compound failure mode. An extraction error in a combined stage silently drops a finding from all downstream analysis. With a dedicated inventory stage, extraction failures are isolated and auditable before any classification happens.

---

### Stage 2 — Critical Sieve

**Input:** Normalized finding list from Stage 1
**Output:** Critical-severity findings only, with classification rationale

The critical sieve reviews every finding from the inventory and applies a single question: is this critical severity? Findings that are not critical are passed through without classification — they are not discarded, they are queued for the next stage.

The rationale for each critical classification is preserved in the chain-of-thought output, not just the final verdict. This provides the audit trail required for a security finding — why was this rated critical, and what reasoning produced that verdict.

---

### Stage 3 — High Sieve

**Input:** Findings not classified as critical in Stage 2
**Output:** High-severity findings, with rationale; remaining findings queued

Same pattern as Stage 2, applied to the high-severity threshold. The model reviews the remaining pool with a single question: is this high severity? Findings that do not meet the threshold continue to Stage 4.

**Why separate critical and high sieves instead of a single classification stage:**
The severity thresholds have genuinely different characteristics. Critical findings typically involve active exploitation risk, unauthenticated remote access, or confirmed credential exposure. High findings involve significant risk with additional preconditions. Combining the two into a single stage means the model must hold both threshold definitions in tension simultaneously — a source of misclassification at the boundary. Separate stages let each threshold be evaluated without interference from the other.

---

### Stage 4 — Medium/Low Sieve

**Input:** Findings not classified as critical or high
**Output:** Medium and low severity findings, classified and labeled; informational findings identified

Medium and low severity are handled in a single stage because the engineering judgment at this threshold is different in character. Critical and high findings require precise individual assessment. Medium and low findings often involve patterns — the same misconfiguration appearing across multiple endpoints, a class of header findings that share a root cause. The model at this stage is identifying patterns as much as classifying individual findings.

Informational findings — findings that describe the attack surface without carrying a direct severity implication — are also identified here. They are not discarded; they are preserved as context for the reduce stage.

---

### Stage 5 — Reduce

**Input:** Classified findings from all sieve stages, informational findings
**Output:** Final structured report — deduplicated, prioritized, with summary

The reduce stage collapses the output of all four preceding stages into the final report delivered to the EASM platform. Its responsibilities:

- **Deduplication:** A finding that appeared in multiple scanner outputs (Nmap and Nuclei both detecting an exposed service, for example) should appear once in the final report with its evidence consolidated.
- **Prioritization:** Within each severity tier, findings are ordered by exploitability and remediation impact.
- **Summary:** A brief characterization of the overall attack surface posture — not a finding-by-finding list, but a human-readable synthesis of what the scan found.

The reduce stage also has access to the informational findings that were not severity-classified earlier. These provide context for the summary — an informational finding about infrastructure fingerprinting, for example, is relevant to the posture summary even without a direct severity classification.

---

## Chain-of-Thought as a Training Target

Each stage's chain-of-thought reasoning is preserved alongside the final output. In production, this reasoning is stored as the audit trail for every finding classification — a human reviewer can inspect not just the verdict but the reasoning that produced it.

A thinking block record is written at every pipeline stage — normalization, each sieve pass, and the reduce pass — not just the final report. This matters because classification reasoning happens upstream. If a Critical finding is later challenged, the reasoning that produced that classification is in the Critical sieve pass's thinking block. The final report's thinking block cannot reconstruct what happened at an earlier stage. Thinking block length per stage is also a quality signal in its own right: an unusually long thinking block on a specific finding at a specific stage indicates classification ambiguity at exactly that decision point.

In the training pipeline, this reasoning becomes a first-class training target. The training example schema includes a `thinking` field:

```json
{
  "input": "full prompt fed to the model",
  "thinking": "expected chain-of-thought reasoning block",
  "output": "expected final JSON output matching stage schema",
  "stage": "inventory | sieve-critical | sieve-high | sieve-medium-low | reduce",
  "source": "adversarial-test | authorized-scan | controlled-environment"
}
```

Training on the `thinking` field means the adapter learns not just what output to produce but what reasoning process to follow. This is what makes the adversarial eval suite meaningful — a model trained on reasoning patterns rather than output patterns is more likely to generalize correctly to novel inputs.

---

## App Integration

The EASM application has no knowledge of this pipeline. It calls the Ollama API with a prompt and receives JSON back. The pipeline stages, the adapter version in use, the base model — none of that is visible to the application layer.

This decoupling is intentional. The pipeline can evolve — stages can be added, the base model can change, adapters can be retrained — without any change to the application. The interface contract is Ollama JSON in, structured finding JSON out.

---

## Related Documentation

- [Training Workflow](training-workflow.md) — how adapters are built and promoted into production
- [Adversarial Evaluation](adversarial-evaluation.md) — how the pipeline is validated before promotion
- [Jenkins Integration](jenkins-integration.md) — the CI/CD lifecycle that connects training to production
