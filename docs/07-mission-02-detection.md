# Mission 02 — Office-to-PowerShell Detection Engineering

## 1. Purpose and Scope

Mission 02 used the secured OpenClaw lab for detection engineering.

The detection objective was:

```text
Detect suspicious Office-to-PowerShell execution associated with payload
retrieval or follow-on execution while avoiding alerts on ordinary
administrative PowerShell and legitimate software-update activity.
```

The mission preserved the same evidence discipline used in Part I. A detection
idea was not treated as validated because it sounded plausible. It had to be
implemented, tested, broken where possible, revised, and compared across
Python and Splunk.

The important distinction was:

```text
Is this suspicious?
≠
Should this specific detection fire?
```

Detection logic determines whether the detection fires. Investigation and
enrichment determine what the detection means.

```text
DETECTION LOGIC
→ determines whether the detection fires

INVESTIGATION / ENRICHMENT
→ determines what the detection means
```

The final output concept was:

```text
behavior_match
```

not:

```text
malicious
alert
```

Implementation artifacts referenced in this document, including the JSONL
datasets, Python detection scripts, and Splunk saved reports, were retained in
the lab environment and are not currently included in this public repository.

## 2. Dataset and Scenarios

The final adversarial dataset was:

```text
endpoint-events-adversarial.jsonl
```

It contained 42 events across scenarios A through L.

| Scenario | Expected behavioral result | Description |
| -------- | -------------------------- | ----------- |
| A | TRUE | Suspicious Word → hidden/encoded PowerShell → Invoke-WebRequest → temp payload → child PowerShell `-File` |
| B | FALSE | Benign Excel → PowerShell → `Get-Date \| Out-File` |
| C | FALSE | SCCM/CcmExec administrative PowerShell inventory activity |
| D | FALSE | Legitimate software updater → PowerShell `ApplyUpdate.ps1` |
| E | FALSE | Office → hidden PowerShell discovery commands, but no retrieval/execution chain |
| F | FALSE | Benign Word → `splwow64.exe` |
| G | FALSE | Office launches benign PowerShell, while unrelated processes separately perform retrieval and execution |
| H | TRUE | Authorized workflow that structurally matches the behavior: Excel → PowerShell retrieval → PowerShell execution |
| I | TRUE | Malicious indirect ancestry: Word → cmd → PowerShell retrieval → child PowerShell `-File` |
| J | TRUE | Malicious alternate retrieval: Excel → PowerShell using curl → child PowerShell `-File` |
| K | TRUE | Same-process execution: Excel → PowerShell retrieval → later 4104 same GUID using PowerShell call operator `&` |
| L | FALSE | PID reuse adversarial negative: Excel → benign PowerShell using one GUID, then unrelated PowerShell reuses the PID with a different GUID and performs retrieval/execution |

Scenario G was the critical adversarial negative for V1. It showed that
scenario-wide or time-proximate facts are not necessarily causal.

Scenario H was intentionally tricky because the behavior detector should
return `TRUE` even when later disposition recognizes the activity as
authorized.

Scenario L tested whether the detection used stable process identity rather
than PID alone.

## 3. V1 — Scenario-Wide Correlation

The first Python implementation was:

```text
detect-v1.py
```

V1 used scenario-wide booleans:

1. Office spawned PowerShell.
2. Some retrieval existed in the scenario.
3. Some execution existed in the scenario.

The alert condition was:

```text
alert = all three
```

V1 initially appeared correct across scenarios A through F.

Adversarial Scenario G then exposed a false positive. In G, Office launched a
benign PowerShell process while unrelated processes separately performed
retrieval and execution. V1 incorrectly correlated those facts because they
existed in the same scenario.

V1 metrics across A through G:

| Metric | Value |
| ------ | ----- |
| True positives | 1 |
| False positives | 1 |
| True negatives | 5 |
| False negatives | 0 |
| Precision | 50% |
| Recall | 100% |

Lesson:

```text
Temporal or scenario proximity is not causality.
```

## 4. V2 — Process-GUID Causal Correlation

The second Python implementation was:

```text
detect-v2.py
```

V2 changed the design from scenario-wide correlation to process identity and
causal scope.

V2 used:

- per-host process graph
- stable `process_guid`
- `parent_process_guid` edges
- direct Office → PowerShell root
- retrieval inside the root or causal subtree
- observed descendant execution preferred
- same-process execution fallback
- no host/time-wide fallback when process identity was missing

Across A through G, only Scenario A matched. Scenario G remained `FALSE`.

The corresponding Splunk saved report was:

```text
Mission 02 - Office PowerShell Causal Detection V2
```

Python V2 and the Splunk V2 saved report achieved semantic parity across the
synthetic A through G dataset.

The Splunk implementation used `join` and was a lab prototype, not a
production-scale optimized analytic.

## 5. Adversarial Testing of V2

V2 fixed the unrelated-chain false positive exposed by Scenario G, but the
next adversarial cases exposed new assumptions.

| Scenario | Result | Exposed assumption |
| --- | --- | --- |
| H | Behavior match TRUE; false positive only if treated automatically as malicious/escalated | Authorization and business context were absent from the detector. |
| I | False negative | Direct Office → PowerShell ancestry was too narrow. |
| J | False negative | Retrieval recognition was too narrow. |
| K | False negative | Execution recognition was too narrow; the PowerShell call operator `&` was not recognized. |

These were not merely coding mistakes. They were detection-design assumptions
made visible by adversarial test cases.

Scenario H was especially important because it separated behavior matching
from disposition:

```text
behavior_match = TRUE
does not mean
malicious = TRUE
```

## 6. V3 — Behavioral Design

The third Python implementation was:

```text
detect-v3.py
```

V3 treated the detector as a behavioral engine.

### Office Ancestry

Office ancestry was bounded to three edges.

Examples:

```text
Office → PowerShell
Office → cmd → PowerShell
Office → cmd → wscript → PowerShell
```

This addressed the direct-parent-only assumption exposed by Scenario I while
still keeping ancestry bounded.

### Retrieval Behavior Families

V3 recognized broader retrieval behavior families:

- `Invoke-WebRequest`
- `Invoke-RestMethod`
- `WebClient`
- `HttpClient`
- `Start-BitsTransfer`
- `curl` / `curl.exe`

This addressed the retrieval-method gap exposed by Scenario J.

### Execution Behavior Families

V3 recognized broader execution behavior families:

- `-File`
- `Invoke-Expression`
- `IEX`
- `ScriptBlock.Create`
- PowerShell call operator `&`
- observed descendant process with recognized execution behavior

This addressed the execution gap exposed by Scenario K.

### Causal Scope

V3 kept causal scope tied to stable identity:

- `process_guid`
- `parent_process_guid`
- observed ancestry

It did not use:

- scenario key as causality
- host/time-only fallback
- PID-only correlation

Missing identity failed closed for high-confidence causal correlation.

### Temporal Ordering

V3 required causal ordering:

```text
Office/root established
→ retrieval
→ execution after retrieval
```

The final output remained:

```text
behavior_match
```

not a maliciousness verdict.

## 7. V3 Acceptance Matrix

The final expected behavioral result was:

| Scenario | Expected result |
| -------- | --------------- |
| A | TRUE |
| B | FALSE |
| C | FALSE |
| D | FALSE |
| E | FALSE |
| F | FALSE |
| G | FALSE |
| H | TRUE |
| I | TRUE |
| J | TRUE |
| K | TRUE |
| L | FALSE |

The final Python V3 regression passed.

The final Splunk V3 saved report was:

```text
Mission 02 - Office PowerShell Behavioral Detection V3
```

Final Splunk V3 returned exactly five behavioral matches:

- A
- H
- I
- J
- K

Scenarios B, C, D, E, F, G, and L were absent from the Splunk V3 result.

Python V3 and Splunk V3 achieved semantic parity across the synthetic A
through L dataset.

Evidence state: VALIDATED for the synthetic A through L dataset and the tested
Python/Splunk implementations. This is not production validation.

## 8. Splunk V3 Implementation Notes

Important implementation notes from the final Splunk V3 work:

- bounded Office ancestry was validated, including Scenario I at depth 2
- root selection excluded nested PowerShell descendants as new roots
- Scenario J validated curl retrieval
- Scenario K validated PowerShell call operator execution
- stable GUID correlation protected against Scenario L PID reuse
- temporal correlation ultimately used the JSON timestamp parsed with
  `strptime`

The Splunk work also preserved an implementation lesson. Intermediate SPL
joins obscured or dropped timing fields during development. Raw timestamp and
`_time` values were checked, source event timing was verified, and the final
SPL used the original JSON timestamp explicitly for causal ordering.

The point is not the debugging minutiae. The point is that detection
engineering requires checking the data path, not only the final narrative.

## 9. ATT&CK Mapping

The primary relevant ATT&CK mappings were:

| Technique                     | Use in this mission                                                   |
| ----------------------------- | --------------------------------------------------------------------- |
| T1059.001 — PowerShell        | PowerShell execution behavior was central to the detection objective. |
| T1105 — Ingress Tool Transfer | Retrieval behavior represented potential payload or tool transfer.    |

The mapping is intentionally narrow. The mission did not try to force every
event into a broader ATT&CK story without evidence.

## 10. SOC Operationalization

Behavior match does not equal malicious.

The minimal disposition model was:

1. Expected / Authorized
2. Needs Investigation
3. Escalate

When authorization was unknown, the initial disposition was:

```text
Needs Investigation
```

Analyst questions:

1. Was the Office activity expected?
2. Was PowerShell use expected?
3. Was the retrieval expected?
4. Was the executed content expected?
5. What happened next?

The disposition flow was:

```text
behavior_match = TRUE
→ context/enrichment
→ authorization / legitimacy assessment
```

An approved workflow can become:

```text
Expected / Authorized
```

Insufficient context remains:

```text
Needs Investigation
```

Unapproved, malicious, or materially risky activity becomes:

```text
Escalate
```

Scenario H must remain `behavior_match = TRUE` even when disposition becomes
`Expected / Authorized`.

Disposition does not rewrite detector truth.

## 11. Known Limitations and Deferred Work

This analytic is not production-ready.

Deferred work includes:

- production-scale SPL optimization
- replacing join-heavy prototype logic
- full authorization/context engine
- enterprise allowlisting and tuning
- broader telemetry coverage
- broader retrieval and execution behavior coverage
- full SOC playbook
- real-world prevalence and performance claims

Scenario H should not be hardcoded as an allowlist exception. It exists to
demonstrate that a behavior detector can be correct while disposition depends
on authorization and context.

## 12. Mission Conclusion

Mission 02 moved from plausible detection logic to adversarially challenged
behavioral detection.

The useful chronology was:

```text
V1
→ scenario-wide correlation
→ initially passed A-F
→ adversarial Scenario G exposed false correlation

V2
→ process-GUID causal correlation
→ fixed unrelated-chain correlation
→ adversarial H-K exposed new assumptions

V3
→ bounded Office ancestry
→ broader retrieval behavior
→ broader execution behavior
→ stable GUID causality
→ temporal ordering
→ Python and Splunk parity across A-L
```

The strongest lesson was that passing initial tests is not enough. Adversarial
cases G through L were used to challenge assumptions, not merely to increase
the test count.
