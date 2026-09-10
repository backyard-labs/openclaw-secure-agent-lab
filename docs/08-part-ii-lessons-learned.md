# Part II Lessons Learned

## 1. Purpose of This Retrospective

Part I focused on building, constraining, and validating the OpenClaw lab.
Part II used that secured baseline for practical security work:

- Mission 01: suspicious PowerShell investigation
- Mission 02: Office-to-PowerShell detection engineering

This retrospective does not repeat the full mission chronology. It extracts
the engineering lessons from using the secured agent for investigation,
detection design, adversarial testing, and SOC disposition.

## 2. Detection Work Requires Precise Questions

A recurring problem in Mission 02 was that similar-looking questions had
different meanings.

```text
Suspicious behavior
≠
detection criteria
```

```text
Detection
≠
investigation
```

```text
Behavior match
≠
malicious
```

The detector's job was to identify Office-to-PowerShell behavior associated
with retrieval or follow-on execution. It was not the detector's job to decide
whether every matching workflow was malicious.

That distinction kept Scenario H honest. Scenario H structurally matched the
behavior and therefore remained `behavior_match = TRUE`. Disposition could
later classify it as `Expected / Authorized` if context supported that result.

The principle is:

```text
Disposition does not rewrite detector truth.
```

## 3. Process Identity Matters

V1 treated scenario-wide facts as enough:

```text
Office spawned PowerShell
+ some retrieval existed
+ some execution existed
```

Scenario G broke that assumption because the facts existed in the same
scenario but not in the same causal process chain.

The lesson was:

```text
Temporal proximity
≠
causality
```

V3 also preserved the related identity lesson:

```text
PID correlation
≠
process identity
```

Scenario L demonstrated why stable `process_guid` and `parent_process_guid`
correlation mattered. PID reuse can make unrelated activity appear connected
if the detector treats PID alone as identity.

## 4. Initial Success Is Not Robustness

V1 appeared correct across scenarios A through F. That did not make it robust.
Scenario G exposed a false positive. V2 fixed that causal failure, then
scenarios H through K exposed new assumptions about authorization, ancestry,
retrieval coverage, and execution coverage.

The lesson was:

```text
Passing initial tests
≠
robust detection
```

Adversarial cases G through L were intentionally designed to challenge
assumptions rather than merely increase test count.

A useful detection test set should include:

- clear positives
- clear negatives
- near-miss negatives
- authorized behavior that structurally matches
- indirect ancestry
- alternate but equivalent behavior
- missing or reused identifiers
- implementation-specific failure modes

The goal is to make hidden assumptions visible while revision is still
relatively cheap.

## 5. Detection and Investigation Need Different Outputs

Mission 02 became clearer once output names matched responsibility.

The detector produced:

```text
behavior_match
```

It did not produce:

```text
malicious = TRUE
```

Investigation then asked:

1. Was the Office activity expected?
2. Was PowerShell use expected?
3. Was the retrieval expected?
4. Was the executed content expected?
5. What happened next?

The minimal disposition model was:

1. Expected / Authorized
2. Needs Investigation
3. Escalate

This separation made the detection logic easier to test and the SOC workflow
more honest. A behavior detector can be correct and still require human
context before action.

## 6. AI Narrative Is Not Evidence

Mission 01 showed the usefulness and risk of AI-assisted investigation.

Qwen was useful for extraction, summarization, and basic evidence reading. Sol
was stronger for causality, adversarial challenge, ATT&CK reasoning, and
detection design critique.

Both remained reasoning aids.

The lesson was:

```text
AI narrative
≠
validated evidence
```

For security-sensitive conclusions, raw evidence and system output were
preferred over agent narrative. If a model claimed that a command represented
retrieval, execution, or malicious behavior, the supporting command, event
relationship, or artifact still had to be checked.

The useful operating stance was:

```text
Model output proposes a path through the evidence.
Evidence determines whether the path holds.
```

## 7. Configured Controls Still Require Validation

Part II depended on Part I's secured baseline, but it did not forget the most
important Part I lesson:

```text
Configured control
≠
enforced control
```

The existence of `cases-ro`, Docker sandboxing, loopback gateway exposure, a
Splunk saved report, or an MCP configuration was not treated as proof of every
security property someone might infer from it.

The evidence language remained:

| Evidence state | Meaning                                                                        |
| -------------- | ------------------------------------------------------------------------------ |
| CLAIMED        | Documentation, configuration, or a model statement says something exists.      |
| OBSERVED       | A concrete artifact or system state demonstrates something exists or occurred. |
| VALIDATED      | A relevant test demonstrated the claim under the tested conditions.            |
| UNKNOWN        | Available evidence is insufficient.                                            |

Python V3 and Splunk V3 returning the intended A through L results can be
described as `VALIDATED` for the synthetic A through L dataset and those
tested implementations.

They should not be described as production validated.

## 8. Detection-Engineering Loop

The working detection-engineering loop was:

```text
security objective
→ behavior hypothesis
→ telemetry requirements
→ detection logic
→ implementation
→ adversarial testing
→ failure analysis
→ revision
→ cross-platform validation
→ SOC operationalization
```

Mission 02 followed that loop across V1, V2, and V3.

V1 established a simple hypothesis. Scenario G showed the hypothesis confused
scenario proximity with causality.

V2 introduced stable process identity and causal scope. Scenarios H through K
showed that causality alone was not enough: H exposed the need to separate
authorization and disposition from behavior matching, while I through K exposed
gaps in ancestry, retrieval, and execution coverage.

V3 broadened the behavioral logic while keeping causal identity, bounded
ancestry, temporal ordering, and fail-closed behavior for missing identity.

The final cross-platform step mattered. Python and Splunk V3 achieved semantic
parity across A through L, increasing confidence that the intended behavioral
logic was implemented consistently in both environments.

## 9. AI-Assisted Engineering Loop

The AI-assisted engineering loop was:

```text
AI proposes
→ human inspects
→ system/test executes
→ evidence returns
→ AI and human challenge result
→ revise
→ freeze
```

AI helped accelerate:

- evidence reading
- hypothesis generation
- detection design
- adversarial case design
- ATT&CK framing
- critique of false positives and false negatives
- documentation

Human review remained necessary for:

- scope
- authorization
- evidence interpretation
- disposition
- deciding whether a claim exceeded evidence
- deciding when an implementation was ready to freeze

The strongest results came when AI output was challenged by tests rather than
accepted as an answer.

## 10. Operational Lessons

Part II produced several compact lessons that should carry into future work:

```text
Suspicious behavior ≠ detection criteria
Detection ≠ investigation
Behavior match ≠ malicious
PID correlation ≠ process identity
Temporal proximity ≠ causality
Passing initial tests ≠ robust detection
AI narrative ≠ validated evidence
Configured control ≠ enforced control
```

These lessons are not slogans detached from the work. Each one came from a
specific failure mode or design distinction encountered during Mission 01 or
Mission 02.

## 11. Future Work

Future work can build on the Part II dataset and V3 analytic.

A dedicated Splunk Detection Engineering / Shuhari learning project would be a
natural continuation. That future project could focus on production-style SPL
optimization, broader telemetry, structured tuning, playbook development, and
repeatable detection-engineering exercises.

That is not an active deliverable in this repository.

For this repository, the Part II deliverable is the documented transition from
secured agent baseline to practical investigation and detection engineering,
including the failures that forced the detection logic to improve.

## 12. Closing Perspective

Part II showed that a secured agent lab becomes more valuable when it is used
against concrete security work.

The important result was not that the agent solved investigation or detection
engineering. The important result was that constrained tool authority,
evidence-first reasoning, adversarial testing, and human approval made AI
assistance usable without treating it as proof.

The recurring operating model is:

```text
Use the agent to accelerate the work.
Use evidence to decide what is true.
Use the human analyst to decide what should happen next.
```
