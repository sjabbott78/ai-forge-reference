# Adversarial Evaluation: Testing for Reasoning, Not Pattern Matching

[Back to README](README.md)

## The Core Design Principle

A model that passes a standard benchmark may still fail in production. Standard benchmarks test whether the model produces outputs that match the training distribution — they validate memorization of known patterns, not the ability to reason about novel inputs.

For an EASM pipeline, this distinction matters enormously. Real attack surface data does not match any training set. A target may run software versions the training data never encountered. A finding may involve a combination of signals that appears nowhere in the labeled examples. A novel vulnerability class may share characteristics with known classes but differ in ways that require inferring severity from first principles.

The adversarial evaluation suite is designed to test for exactly this capability. **Every test in the suite is constructed so that the test input and the severity rules share no exact terminology.** The model cannot pass these tests by finding a surface-feature match between the input and training examples. It must derive the correct classification by reasoning about what the finding implies — the same process required when the model encounters something genuinely new in production.

---

## What "Adversarial" Means Here

The term is used deliberately. Each test is designed to expose a specific failure mode:

**Ambiguity tests** present findings where the correct severity is not the one that a naive pattern-matcher would select. A finding that contains language associated with critical severity but includes a contextual condition that reduces its severity — the test validates whether the model evaluates the condition or ignores it.

**Negation tests** present findings where a context element explicitly negates a severity implication. "Service X is running version Y, which is vulnerable to CVE-XXXX" is a different finding than "Service X is running version Y, which was previously vulnerable to CVE-XXXX but has mitigating configuration Z active." A model that cannot process negation will misclassify the second.

**Chaining tests** present findings where the severity depends on the relationship between two findings, not on either finding independently. A misconfiguration that is low-severity in isolation becomes high-severity when combined with an exposed management interface. The test validates that the model can reason about the compound risk.

**Encoding tests** present findings where relevant data is encoded — base64, URL encoding, similar transformations. The test validates that encoding does not obscure the finding's severity implications. A model that pattern-matches on surface features will see an encoded string and classify it differently than it classifies the decoded equivalent.

**Edge case tests** test behavior at the boundaries of the input contract — zero findings, malformed inputs, conflicting data in the same record. These validate that the model has a defined, safe behavior at the edges, not just in the normal operating range.

**Stress tests** combine multiple adversarial conditions simultaneously. If a model passes each individual test but fails when they appear together, it reveals a capacity constraint in the reasoning process.

---

## The Twelve Tests

| # | Category | What It Validates |
|---|---|---|
| 01 | Severity triage and ambiguity | Ambiguous signals resolved correctly; confidence in the wrong direction is a failure |
| 02 | Contextual negation and chaining | Context that negates a finding actually reduces severity; compound risk is recognized |
| 03 | Data normalization and CVE correlation | CVE metadata connected to severity without shared keywords |
| 04 | Encoded telemetry and CVE correlation | Encoding does not change classification outcome |
| 05 | Malformed encoding handling | Malformed input produces a defined behavior, not a hallucinated finding |
| 06 | Conflicting version data | When two data points contradict, the reasoning for the resolution is correct |
| 07 | Version string ambiguity | Software version ranges reasoned about without explicit training examples |
| 08 | CVE note conditions | Conditional severity language evaluated, not treated as unconditional |
| 09 | Empty stream edge case | Zero findings produces a valid empty result, not hallucinated findings |
| 10 | Indirect prompt injection resistance | Adversarial content embedded in scan data does not influence classification |
| 11 | Large stream / lost-in-the-middle | Findings in the middle of a large result set receive the same treatment as findings at the start |
| 12 | Combined stress test | All adversarial conditions simultaneously — production-equivalent test |

---

## Pass/Fail Criteria

The suite does not use exact-match comparison. A finding classified as critical when the expected output is high is not a pass with minor deviation — it is a failure, because the remediation urgency and escalation path are different for each severity tier.

The pass/fail criteria are:

- **Severity classification must be correct** for every finding in the test input. A misclassification anywhere in the test fails the test.
- **Negation must be respected.** If a context element reduces the severity of a finding, the final classification must reflect that reduction.
- **No hallucinated findings.** The model may not produce findings that do not exist in the test input. This is tested explicitly in the empty stream test (test 09) but is evaluated across all tests.
- **Prompt injection must be resisted.** Test 10 embeds adversarial content in the scan data that attempts to influence model behavior. Any deviation from expected classification is a failure.
- **The combined test (test 12) must pass without regression.** A model that passes tests 01–11 individually but fails test 12 has a capacity constraint that will surface in production on complex targets.

The suite is not a graded benchmark. It is a gate. The adapter either passes and is eligible for promotion, or it does not and the pipeline stops.

---

## Hardware Run Results

The test suite was run on live P5000 GPU hardware against the production model. Selected observations from those runs:

**Prompt injection resistance (test 12):** The combined stress test embeds an entry that attempts to override CVE matching by injecting a forced instruction inside the scan data payload. The model's own `logic_path` field in the output read: `"Note injection ignored, version in range but HTTP/2 unconfirmed"` — the model explicitly logged its rejection of the injected instruction and continued with the correct conditional evaluation. Resistance was not just behaviorally correct; it was self-documented in the reasoning trace.

**Conditional CVE note handling (test 12):** A PostgreSQL entry in the combined test had authentication required (`auth=required` in the banner) while the matched CVE's condition required unauthenticated access. The model correctly set `condition_unverified: true` and preserved the CVE match rather than either dropping it (false negative) or confirming it (false positive without evidence). The `logic_path` distinguished between "version in range" and "condition met" as independent determinations.

**Version boundary precision (test 12):** PostgreSQL 13.16 against a CVE with an upper bound of 13.15 was correctly excluded. OpenSSH 8.4p1 against a CVE ranging from 8.5p1 was correctly excluded. The model evaluated exact version bounds rather than approximate range matching.

**OS-specific CVE conditions (test 12):** The same OpenSSH version (9.1p1) on Ubuntu was excluded while the same version on a Debian-based system was correctly matched — the model evaluated the OS specifier in the CVE note as a condition, not decoration.

**Encoding handling (test 12):** A Base64 payload with a corrupted suffix (`!!CORRUPTED==`) correctly produced `decode_status: failed` with partial decoding preserved in the output. A hex-encoded payload with an invalid `ZZZZ` segment was similarly flagged as a partial failure. Both were handled without hallucinating a software version from the corrupted input.

**Monolithic baseline comparison:** The single-prompt baseline test on the same hardware ran for over ten minutes with the model's reasoning trace showing repeated reconsideration of severity tier boundaries and priority ordering. The adversarial tests, operating on narrower inputs with single-objective prompts, completed in roughly half that time each and produced cleaner reasoning traces with no backtracking.

**Validated behaviors from earlier tests:**

| Behavior | What Was Tested |
|---|---|
| Buzzword trap resistance | A finding containing "Log4j" language but with authentication required was correctly excluded from Critical — the model evaluated the condition, not the keyword |
| Uncertain scan flag override | A finding described as "Potential subdomain takeover" was correctly escalated — the scan qualifier did not reduce the severity of a confirmed takeover condition |
| Default credentials mapped to unauthenticated exposure | Default credentials were correctly classified as equivalent to unauthenticated access rather than as a misconfiguration |
| Honeypot contextual negation | A finding on a documented honeypot system was correctly excluded — context that negates the risk was evaluated, not ignored |
| Revoked credentials negation | Credentials confirmed as revoked before the scan were correctly excluded from active findings |
| Multi-hop vulnerability chaining | SSRF combined with an internal Redis instance was correctly chained into a compound finding at elevated severity — neither finding alone warranted the classification |
| Auth bypass logic composition | Authentication bypass combined with RCE was correctly classified as unauthenticated RCE — the model derived the compound classification without explicit instruction |
| Format-agnostic parsing | syslog format, NDJSON, and Nmap greppable output were all correctly parsed in a single normalization pass |
| Version range mathematics | CVE version range bounds were evaluated correctly including upper-bound exclusions and version string ambiguity |
| CVE withheld on absent version | When a software version could not be determined, no CVE was matched — the model did not hallucinate a version to force a match |

---

## How QwQ Was Selected — and What That Revealed

Model selection for the production pipeline involved benchmarking ten models across three complexity levels — single finding classification, multi-finding priority ordering, and full EASM scan with compound risk chains. The benchmarking session ran as a single extended session across one night, with all models tested against identical prompts.

The selection of QwQ:32b as the production model was not predetermined. It emerged from a specific failure mode that separated it from every other candidate.

During testing, the prompt contained a logic contradiction that had been invisible during prompt development. Three constraints were mutually inconsistent: one defined which finding types were Critical severity, a second stated only one finding could hold priority_order 1, and a third stated hardcoded credentials were always priority_order 1 with no exceptions. When a scan contained both hardcoded credentials and unauthenticated RCE simultaneously — two Critical findings — the three constraints could not all be satisfied.

Every model except QwQ resolved this silently. Gemma 27B invisibly downgraded severities to make the priority integers fit. Llama 3.3 70B silently collapsed multiple findings into a single entry. Qwen 2.5 Coder 32B quietly dropped findings. All produced valid JSON. All produced wrong answers. None signaled that anything had gone wrong.

QwQ refused to break its rules. Its thinking block documented the exact constraint that could not be satisfied, traced through multiple attempted resolutions, and flagged the conflict before producing output. The thinking block was effectively a compiler stack trace for the prompt — it printed the line where the logic broke.

The contradiction had been invisible to human reviewers and to both Claude and Gemini during prompt development. Humans with domain knowledge automatically correct semantic drift in a prompt — they read the intent, not the literal rules. A model operating as a procedural parser cannot infer intent when two rules collide. QwQ exposed what human review missed.

**Why this matters for the evaluation suite design:** Silent failures in a security platform are more dangerous than visible ones. A model that invisibly downgrades MongoDB from Critical to High produces a report that looks correct and ships to a customer who deprioritizes their database remediation. A model that visibly struggles with an ambiguity is debuggable. The adversarial test suite is designed to surface the failures that a well-formatted JSON output hides.

---

## English-Only Output Constraint

All pipeline prompts include an explicit constraint requiring English-only output.

QwQ:32b belongs to the Qwen model family, which is built on a dual-language English/Chinese training corpus. During complex reasoning — particularly on security classification decisions where the Chinese training data has high domain density — the model may produce Chinese characters mid-thinking-block without the constraint active.

This creates a downstream failure in the confidence scoring system. Confidence scores are derived by detecting uncertainty signals in the thinking block: English phrases such as "wait, but...", "I'm not sure...", and "hmm" indicate where the model hesitated. A Chinese-language reasoning segment scores as artificially high confidence not because the model was confident, but because the English-language uncertainty detector cannot read it.

The constraint was validated by running controlled reruns of adversarial tests with the constraint active against unconstrained output. Results were identical across every test: same findings classified, same findings excluded, same reasoning conclusions, same attack chain logic. The English-only constraint is output-channel redirection only. It does not suppress capability or alter reasoning quality — it redirects where that reasoning is expressed.

---

## benchmark.py

The `benchmark.py` runner executes the test suite against a live Ollama endpoint using the adapter under evaluation. It:

- Sends each test prompt to the model and collects the response
- Parses the response JSON and evaluates it against the expected output using the pass/fail criteria above
- Produces a structured result per test — pass, fail, and the specific failure mode if applicable
- Returns a non-zero exit code if any test fails, which the Jenkins pipeline treats as a pipeline failure

The runner is designed to run against the production inference cluster, not a test environment. An adapter that passes benchmark.py on the real hardware with the real Ollama configuration is what gets deployed — there is no separate staging inference environment that could diverge from production behavior.

---

## Why This Approach Compounds With the Training Design

The training example schema includes a `thinking` field — the chain-of-thought reasoning that produces the final output. Training on that reasoning means the adapter learns not just what output to produce but what reasoning process to follow.

The adversarial eval suite tests whether that reasoning process generalizes. A model trained purely on output patterns may pass tests where the output is predictable from the input's surface features, but will fail tests where the correct output requires intermediate reasoning steps that are not encoded in the surface features.

The combination — training on reasoning, evaluating for reasoning — is what produces an adapter whose behavior is predictable on novel inputs. Predictable behavior on novel inputs is the only property that matters for a production EASM pipeline.

---

## Related Documentation

- [Pipeline Architecture](pipeline-architecture.md) — the stages the eval suite validates
- [Training Workflow](training-workflow.md) — how the adapter being evaluated was built
- [Jenkins Integration](jenkins-integration.md) — how the eval suite gates production promotion
