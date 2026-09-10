# Mission 01 — Suspicious PowerShell Investigation

## 1. Purpose and Scope

Mission 01 used the secured OpenClaw lab for an AI-assisted investigation of
suspicious PowerShell behavior.

The purpose was not to write a generic PowerShell tutorial. The purpose was to
test an operational pattern:

```text
Evidence
→ extraction
→ correlation
→ hypothesis
→ challenge
→ analyst decision
```

The case evidence was exposed through the read-only `cases-ro` MCP capability.
The evidence location was:

```text
~/labs/openclaw/cases/mission-01-powershell/evidence/
```

The agent could read the evidence, but it was not given authority to modify
the case evidence. Human authority remained final for investigation scope,
interpretation, and conclusion.

## 2. Investigation Setup

Mission 01 used the already-secured OpenClaw environment documented in Part I
and summarized in [05-security-operations.md](05-security-operations.md).

The relevant operating controls were:

- OpenClaw 2026.9.1 running in the Ubuntu lab VM
- Docker-sandboxed MCP services
- loopback-only gateway exposure
- controlled SSH-tunnel UI access
- read-only evidence access through `cases-ro`
- no network access from the agent sandbox
- constrained MCP capability exposure

The model paths compared during the investigation were:

| Model path          | Observed operational role                                |
| ------------------- | -------------------------------------------------------- |
| Ollama `qwen3.5:9b` | Evidence reader, extractor, and summarizer               |
| GPT-5.6 Sol         | Causal reasoning, challenge, and security interpretation |

The model comparison was about operational fit, not authority. The model that
reasoned about evidence still did not determine which tools or data were
available.

## 3. Method

The investigation followed a bounded evidence workflow.

First, evidence was read from the case directory through `cases-ro`. The
initial task was to identify relevant artifacts and summarize suspicious
PowerShell behavior without modifying the evidence.

Second, extracted facts were correlated into hypotheses. The investigation
looked for relationships among process behavior, command content, retrieval or
execution indicators, and other available evidence.

Third, the hosted model was used to challenge the interpretation. The
goal was to test whether the initial hypothesis was supported by evidence, too
strong, or missing plausible alternatives.

Finally, the human analyst retained responsibility for the conclusion. Agent
narrative was treated as working analysis, not proof.

The investigation loop was:

```text
Read evidence
→ extract facts
→ correlate events
→ form hypothesis
→ challenge hypothesis
→ verify against raw evidence
→ analyst decision
```

## 4. Model Behavior Observed

Qwen was useful for:

- reading evidence files
- extracting candidate facts
- summarizing visible artifacts
- providing a first-pass description of suspicious PowerShell activity

Qwen was less reliable for:

- strict tool selection
- avoiding overclaims
- distinguishing what evidence showed from what a plausible narrative implied

Sol was stronger for:

- causal reasoning
- identifying unsupported leaps
- adversarial challenge
- security interpretation
- connecting behavior to detection-engineering questions

The useful conclusion was not that one model should always replace the other.
The conclusion was that model outputs have to be evaluated against evidence,
and stronger reasoning does not remove the need for analyst verification.

## 5. Evidence and Claim Strength

The investigation preserved the repository's evidence language:

| Evidence state | Use in this mission                                                               |
| -------------- | --------------------------------------------------------------------------------- |
| CLAIMED        | A model or narrative statement asserted that something occurred.                  |
| OBSERVED       | The case evidence showed an artifact, command, timestamp, or relationship.        |
| VALIDATED      | A conclusion was checked against the relevant raw evidence or independent result. |
| UNKNOWN        | Evidence was insufficient to support a stronger conclusion.                       |

For security-sensitive conclusions, raw tool evidence was preferred over agent
narrative.

An agent statement such as "this indicates payload retrieval" was not enough
by itself. The underlying command, process relationship, artifact, or telemetry
had to support the claim.

## 6. Investigation Conclusions

Mission 01 demonstrated that the secured agent could be useful for practical
investigation work when its authority was constrained and its conclusions were
checked.

Key conclusions:

1. Read-only case evidence exposure was a useful operating pattern.
2. Qwen was useful as an evidence reader and summarizer.
3. Sol produced stronger causal reasoning and security challenge.
4. Qwen sometimes overclaimed beyond the available evidence.
5. Raw tool evidence was preferred over agent narrative for
   security-sensitive conclusions.
6. Human analyst authority remained final.

The investigation did not prove general PowerShell detection coverage. It
demonstrated an evidence-first AI-assisted investigation workflow in a secured
OpenClaw lab.

## 7. Limits

This mission should be read as a technical case study, not as a production SOC
playbook.

The public repository does not include the full private case evidence. The
claims in this document are therefore limited to the documented operating
pattern and the investigation lessons preserved from the lab work.

The mission did not establish:

- general PowerShell maliciousness rules
- enterprise detection coverage
- generic prompt-injection resistance
- production readiness of the lab
- that model narrative alone can serve as evidence

The durable lesson is that a secured agent can help investigate evidence, but
the analyst still owns the conclusion.
