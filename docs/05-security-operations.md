# Security Operations

## 1. Purpose and Scope

Part I established a secured OpenClaw lab baseline. It built the environment,
constrained agent authority, tested selected boundaries, and documented the
resulting evidence states.

Part II asks a different question:

```text
After the agent has been bounded and tested, how can it be used for real
security work without forgetting those boundaries?
```

The purpose of this document is to describe the operating model used when the
lab moved from infrastructure and control validation into SOC and
security-engineering work.

The work did not treat the secured agent as an autonomous security authority.
It treated the agent as an evidence-reading and reasoning aid inside a
human-directed workflow.

The Part II progression became:

```text
Build
→ Harden
→ Validate
      ↓
Use the secured agent
      ↓
Investigation
→ Detection Engineering
```

## 2. Operating Model

The operating model for Part II was:

```text
Case evidence
→ local Qwen extraction/correlation
→ hosted-model reasoning and challenge
→ human analyst decision
→ detection engineering / SOC action
```

The important separation from Part I remained intact:

```text
Model reasoning
≠
tool authority
```

The model providing reasoning did not determine what authority was exposed
through OpenClaw or MCP tools. Authority still came from configured MCP
servers, credential scope, container controls, network controls, and operator
authorization procedures.

## 3. Environment Used for Operations

The Part II work used the already-secured OpenClaw environment:

- Ubuntu Server 24.04.4 LTS VM
- OpenClaw 2026.9.1
- Docker sandboxing for MCP services
- loopback-only OpenClaw Gateway
- controlled SSH-tunnel UI access
- constrained MCP capabilities
- read-only case-evidence exposure through `cases-ro`
- no network access from the agent sandbox

The lab also used a dedicated Splunk VM for detection-engineering work:

- SOC/lab interface: `10.10.10.100/24`
- management/NAT interface: `192.168.93.131/24`
- default route remained on the SOC/lab interface
- Splunk index: `security`

Splunk metadata host was `splunk`. The original endpoint hostname from the
JSON events was available as `extracted_host`. Correlation logic used endpoint
identity from the event data, not Splunk ingestion host metadata.

These environment details are included because they affected how evidence was
accessed, correlated, and validated in the lab. They should not be treated as
production architecture guidance.

## 4. Case Evidence Exposure

Part II introduced a read-only case evidence capability:

```text
cases-ro
```

The intent was to let the agent inspect investigation evidence without giving
it authority to modify that evidence. This preserved the Part I pattern of
separating useful capability from unnecessary authority.

The evidence path for Mission 01 was:

```text
~/labs/openclaw/cases/mission-01-powershell/evidence/
```

The read-only case evidence exposure supported extraction, summarization, and
correlation. It did not make the agent the source of truth. Security-sensitive
state still had to be verified through raw evidence, fresh command output, or
independent system state where practical.

## 5. Local and Hosted Model Roles

Part II compared two model paths:

| Model path          | Role in the workflow                                                         |
| ------------------- | ---------------------------------------------------------------------------- |
| Ollama `qwen3.5:9b` | Local extraction, summarization, and basic evidence reading                  |
| GPT-5.6 Sol         | Deeper reasoning, causality challenge, ATT&CK reasoning, and design critique |

Qwen was useful for reading evidence, extracting artifacts, and producing
basic summaries. It was not dismissed as unusable simply because a stronger
hosted model performed better on harder reasoning tasks.

Sol was stronger when the work required causal reasoning, adversarial
challenge, ATT&CK framing, detection-design critique, or distinguishing
behavioral detection from disposition.

The lesson was not:

```text
local model bad / hosted model good
```

The lesson was:

```text
Different model outputs have different strengths, and each must be evaluated
against evidence.
```

Qwen sometimes selected tools inconsistently and could overclaim beyond the
available evidence. That made raw tool output and independent verification
especially important for security-sensitive conclusions.

## 6. Human Authority

Human authority remained final.

The analyst decided:

- what question was being investigated
- which evidence was in scope
- which actions were authorized
- whether a conclusion exceeded the evidence
- whether a detection result required SOC action
- whether an observed behavior was authorized, required further investigation,
  or warranted escalation

The operating loop remained:

```text
AI proposes
→ human inspects
→ system/test executes
→ evidence returns
→ AI and human challenge result
→ revise
→ freeze
```

This matters because an agent can produce a plausible narrative that is still
wrong, incomplete, or stronger than the underlying evidence permits.

## 7. UI, CLI, and Administrative Verification

The OpenClaw Web UI was useful for interactive investigation, model comparison,
and model-assisted reasoning.

CLI and administrative checks remained important for verification:

- inspecting raw evidence files
- running detection scripts
- checking Splunk searches and saved reports
- confirming event counts and field behavior
- validating Python/Splunk parity
- checking that system state matched agent statements

The UI was not treated as the only source of truth. Agent narrative and model
summaries were useful working material, but security-sensitive state was
verified through raw evidence or system output when it mattered.

The project carried forward the Part I evidence distinction:

```text
CLAIMED
OBSERVED
VALIDATED
UNKNOWN
```

A model statement was not upgraded to `VALIDATED` merely because it sounded
confident. A detection result was considered validated only for the tested
dataset and implementation paths that actually produced evidence.

## 8. Evidence-First Workflow

Part II used an evidence-first workflow for both investigation and detection
engineering.

For investigation:

```text
Evidence
→ extraction
→ correlation
→ hypothesis
→ challenge
→ analyst decision
```

For detection engineering:

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

The goal was not to make the agent produce an answer quickly. The goal was to
use the agent to accelerate reading, correlation, and challenge while
preserving the analyst's obligation to verify the evidence and limit the
claim.

## 9. Operational Boundaries

Part II did not make the lab production-ready.

The work remained a controlled lab exercise using synthetic or lab-managed
evidence. The useful result was a documented operating pattern:

- expose case evidence read-only
- separate model reasoning from tool authority
- use local and hosted models according to observed strengths
- challenge model conclusions against raw evidence
- keep human disposition authority explicit
- validate detection behavior across implementations before calling it
  validated for a dataset
- document failures and revisions rather than rewriting the chronology

The secured baseline made security operations work possible. It did not remove
the need for analyst judgment, fresh evidence, or careful scope limits.
