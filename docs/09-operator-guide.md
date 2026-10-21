# Operator Guide

## Purpose

This guide is a practical manual for operating the existing OpenClaw Secure Agent Lab, OpenClaw 2026.9.1. It focuses on hands-on AI-assisted security work: investigating supplied case evidence, reconstructing event timelines, developing and challenging detection logic, researching through configured MCP (Model Context Protocol) tools, and performing narrowly authorized GitHub file updates.

The lab is a controlled learning environment. It is not a live SOC platform or a production-ready incident response service. There is no live SIEM/EDR ingestion, autonomous SOC response, production-readiness, or generalized real-world incident reliability claim here. Historical Mission 01/02 results describe prior work; a fresh run with new evidence or artifacts must be assessed on its own evidence.

See [docs/05-security-operations.md](05-security-operations.md) for the operating model, [docs/06-mission-01-investigation.md](06-mission-01-investigation.md) for the historical PowerShell investigation, [docs/07-mission-02-detection.md](07-mission-02-detection.md) for detection engineering, and [docs/03-security-validation.md](03-security-validation.md) for security test scope and findings.

## Current Capabilities and Limits

The current-state facts in this table are operator-reported observations for this guide. “Verified from CLI help” describes command semantics, not runtime security. Unless specifically stated, commands below were not run while revising this document.

| Area | Reported current state and limit |
|---|---|
| OpenClaw and Gateway | OpenClaw 2026.9.1 (`ad6fe23`). Gateway is an enabled/running systemd user service, loopback-only on port 18789. `openclaw gateway status` and `openclaw health --json` succeeded in the supplied current check. |
| MCP | `cases-ro`, `fetch-net`, `files-ro`, `files-rw-lab`, and `github-ro` are configured; `github-rw-lab` is disabled at rest per supplied status. Configuration/status does not prove enforcement. |
| Models | `ollama/qwen3.5:9b` is default with no fallbacks. A configured OpenAI API-key profile is reported; Ollama auth readiness was indeterminate in that diagnostic, although a functional CLI inference test succeeded. `openai/gpt-5.6-sol` is a previously tested hosted route; no hosted test is run in this guide. |
| Main agent | Sandbox configuration was observed as `mode=all`, Docker backend, session scope. Main denies `github-rw-lab__*` but has a writable test-repository mount at `/project:rw` and a separate read-only `.git` mount. Main is not universally read-only. |
| Dedicated writer | `github-write-job` was listed with `github-ro__get_file_contents`, `github-ro__get_commit`, and `github-rw-lab__create_or_update_file`, `exec.mode=deny`, and an empty Docker bind list. An allowlist alone does not prove runtime enforcement. |
| Wrapper | `/usr/local/sbin/openclaw-github-write-job` was inspected; reported owner/mode is `root:root`, `0755`. Invocation syntax is source-inspected, not freshly executed in this revision. The wrapper is not an OS privilege boundary. |
| Telegram | Transport was reported connected, ready, polling, with zero plugin errors during health inspection. Default routing is to `main`; `github-write-job` has zero channel routing rules. Historical approved senders were not independently enumerated. Telegram routing does not authorize or expose the GitHub-write workflow. |
| Residual sandbox setting | `agents.defaults.sandbox.docker.dangerouslyAllowExternalBindSources=true` is reported configured. This is a residual configuration concern, not proof of a sandbox escape. |

### Evidence labels

- **CLAIMED** — documentation, configuration, or a model statement says a behavior/control should exist.
- **OBSERVED** — an artifact, output, or system state demonstrates something exists or occurred.
- **VALIDATED** — a test exercised a specific claim and produced evidence for the tested conditions.
- **UNKNOWN** — available evidence does not support a stronger conclusion.

These labels apply to individual claims and their stated scope. A successful functional test does not validate unrelated security properties.

## Quick Start

### Ubuntu VM — passive checks and harmless inference

Prerequisite: access to the Ubuntu lab VM through the operator's established method. The commands are documented in [docs/01-build-and-deployment.md](01-build-and-deployment.md), **Install OpenClaw** and **Verify MCP Connectivity**. The expected state below is from the operator-reported current check, not a fresh execution during this revision.

1. Check version and Gateway health:

   ```bash
   openclaw --version
   openclaw gateway status
   openclaw health --json
   ```

   Expected reported state: version 2026.9.1; Gateway enabled/running and bound to loopback port 18789; health JSON includes `"ok": true`. Stop if the Gateway is not healthy or is exposed beyond the expected loopback bind; follow the recovery guidance below rather than broadening network exposure.

2. Inspect MCP and model state:

   ```bash
   openclaw mcp status
   openclaw models status
   openclaw config get agents
   ```

   `mcp status` passively reports configured transport status without connecting. Confirm expected read services and `github-rw-lab` disabled at rest before ordinary work. These outputs do not prove tool-policy enforcement.

3. Optional harmless inference smoke test (Ubuntu VM):

   ```bash
   openclaw agent --agent main --session-id <new-session-id> --model ollama/qwen3.5:9b --message "Health check only. Reply with exactly: OPENCLAW_MODEL_OK. Do not call any tools." --timeout 60
   ```

   **Observed historical result:** exact marker `OPENCLAW_MODEL_OK`. This is a harmless inference smoke test. “Do not call any tools” is a behavioral instruction, not enforced isolation. The observation supports inference functionality for that tested request only; it is not a tool restriction or security test. Choose a new session identifier for each run.

4. For routine tasks, specify the evidence scope, use the least capability needed, and independently verify important claims.

### First Practical SOC Task

Start with the [Mission 01 evidence-first investigation workflow](#evidence-first-investigation--mission-01) below. It confirms evidence availability, then builds a source-cited timeline and provisional assessment for human review.

Routine reads through `cases-ro`, `files-ro`, and `github-ro` are intended for inspection. Research through `fetch-net` should have a defined, approved retrieval scope. Configuration changes and any file or GitHub mutation require explicit human authorization. In particular, main has a writable `/project:rw` test mount; a read-oriented task description does not make its authority universally read-only.

## Normal Operations and Research

### Read evidence and repository material

Use `cases-ro` for case evidence, `files-ro` for permitted lab files, and `github-ro` for repository research. Keep requests scoped:

> Read only the named evidence or repository scope. Inventory relevant source artifacts and fields. For every conclusion, separate direct observation from inference and cite the artifact and field. Identify missing evidence. Do not modify files or repositories.

The build document's **7. Run Functional Smoke Tests** section contains historical examples for `files-ro`, `fetch-net`, and `github-ro`; these are functional examples, not proof of security enforcement. Its write smoke test is not routine authorization for other repositories or tasks.

Use `fetch-net` only when external retrieval is required and in scope. The agent sandbox was reported to have no direct network access; `fetch-net` is a separately configured MCP capability. Do not infer unrestricted network access from availability of that tool.

### Formulate a bounded task

State the objective, allowed evidence/repository scope, desired output, and prohibited actions. Ask for source references, assumptions, and uncertainties. Natural-language restrictions help direct the task but do not mechanically constrain runtime authority; rely on the configured capability boundary and human verification.

## Evidence-First Investigation — Mission 01

Mission 01 was a historical investigation of suspicious PowerShell behavior using read-only evidence through `cases-ro`. The historical case evidence path is documented as `~/labs/openclaw/cases/mission-01-powershell/evidence/` in [docs/05-security-operations.md](05-security-operations.md), **Case Evidence Exposure**, and [docs/06-mission-01-investigation.md](06-mission-01-investigation.md). Treat that as the lab's documented layout, not a portable path for other installations. The full private case evidence is not in this public repository.

### Prerequisites and evidence availability

1. In the Ubuntu VM, check that the Gateway is healthy using the Quick Start checks.
2. Passively inspect configured transport state with `openclaw mcp status` (Ubuntu VM). This reports configuration without connecting and does not prove that evidence is readable.
3. To confirm actual evidence availability, make a narrowly scoped read request through the agent's `cases-ro` capability, using the verified `openclaw agent --agent main --message` command shape below. Ask it to list accessible evidence artifact names and report which items it could not read. Do not request a mutation. This is an active read operation; do not assume success from MCP status alone.
4. Stop if the case is outside the approved scope, no artifacts are visible, access errors occur, or the agent cannot identify source artifacts. Resolve access through the operator's existing VM procedure; do not invent or broaden mount paths.

Concrete CLI invocation (Ubuntu VM; command shape supplied as verified; prompt is a harmless read request):

```bash
openclaw agent --agent main --session-id <new-operator-chosen-session-id> --message "Using cases-ro only, inspect the in-scope Mission 01 evidence made available to you. First list accessible artifact names and the source fields you can read. Do not modify evidence, use network tools, or access other cases. If evidence is unavailable, say so and stop."
```

Choose a fresh session ID for this investigation and record it privately in the case notes; do not publish it. Session separation reduces context contamination from earlier work but does not establish a security isolation boundary. This invocation asks the agent to follow a tool constraint; the instruction itself is not an enforcement mechanism. Review its actual tool/evidence output. Do not mistake the historical case path for confirmation that evidence exists in a fresh installation.

### Investigation procedure

Once availability is confirmed:

1. Define the question, time range, authorized case scope, and relevant evidence types.
2. Inventory artifacts before interpreting them.
3. Extract observable facts: timestamps, process identifiers and parent/child relationships, commands, retrieval indicators, and execution evidence when present.
4. Build a timestamped timeline. Cite the source artifact and field for every entry; keep original timestamps and distinguish direct facts from inferred sequence or causality.
5. Develop competing explanations, including plausible benign activity. Identify evidence gaps and what would resolve them.
6. Ask for a provisional assessment with confidence and limitations. Use another reasoning pass to challenge the causal story when useful; do not treat a model's confidence as evidence.
7. Independently inspect a representative sample of the raw case evidence. Reconcile actual artifact names, fields, timestamps, and content against the agent-generated inventory, timeline, and findings. An agent report alone is not verification.
8. Record the human analyst's conclusion separately from the model's provisional assessment.

Reusable prompt:

> Investigate only the authorized case evidence available through cases-ro. First provide an artifact inventory. Then reconstruct a timestamped timeline with source artifact and exact field references for each fact. Separate direct observations from inferred relationships. Present at least two competing explanations where the evidence permits, identify gaps and the evidence needed to resolve them, and provide a provisional analyst assessment with confidence and limitations. Do not call activity malicious solely because it is unusual. Do not modify evidence or use tools outside the approved scope. If evidence is unavailable, stop and report that.

### Expected report structure and human check

Expected report sections: (1) question and scope; (2) evidence/artifact inventory; (3) timestamped timeline with source/field references; (4) direct observations; (5) inferred relationships and competing explanations; (6) gaps and follow-up evidence; (7) provisional assessment, confidence, and limitations; (8) analyst disposition.

Before accepting it, the human analyst should confirm:

- Every key timeline row points to a real artifact and field, and timestamps/time zones were not silently changed.
- Process relationships use stable identifiers available in evidence; temporal proximity alone is not presented as causality.
- The narrative does not add commands, outcomes, or artifacts absent from the raw evidence.
- A representative sample of raw artifacts has been opened independently, and its names, fields, timestamps, and content reconcile with the agent's report; discrepancies are recorded and resolved before relying on the assessment.
- Benign alternatives and evidence gaps are represented fairly.
- Inferences and provisional assessment are labeled; final disposition remains human-owned.
- No evidence was modified and no out-of-scope capability was used.

The historical Mission 01 demonstration supports an evidence-first operating pattern. It does not establish that a new case is available, reproduce historical conclusions, or provide general PowerShell detection coverage.

## Detection Engineering — Mission 02

Mission 02 developed an Office-to-PowerShell analytic from V1 scenario-wide correlation through V2 process-GUID causal correlation to V3 behavioral logic, adversarial testing, and Python/Splunk comparison. The detailed history and acceptance matrix are in [docs/07-mission-02-detection.md](07-mission-02-detection.md), **Dataset and Scenarios**, **V1–V3**, **V3 Acceptance Matrix**, **Splunk V3 Implementation Notes**, and **SOC Operationalization**.

### Prerequisites and assumptions

- A defined behavioral objective, telemetry schema, and authorized test dataset.
- Event fields sufficient to reason about process ancestry, stable process identity (such as a GUID), retrieval/execution behavior, and temporal order. Verify actual field names and timestamp semantics before implementation.
- A Python implementation and a Splunk implementation only if both are available in the operator's environment.
- Benign, positive, and adversarial test cases with expected behavioral outputs.

The historical dataset (`endpoint-events-adversarial.jsonl`), Python scripts, and Splunk saved reports are not included in the public repository. Their location and execution commands are not published here. Without those artifacts, this guide cannot promise full reproduction of Mission 02 or its Python/Splunk comparison.

### Design and test workflow

1. State the behavior to detect and explicit benign behavior that should not match.
2. Ask the agent for an analytic specification before code: objective, telemetry/field assumptions, event-selection rules, causal identity, ancestry depth, retrieval/execution behavior families, time ordering, exclusions, output schema, and known blind spots.
3. Review the specification against available telemetry. Do not accept invented fields or undocumented SPL/Python syntax as executable.
4. Implement in the available environment. Test positive cases (expected match), negative cases (expected no match), and adversarial cases designed to break causal assumptions.
5. For Mission 02's historical cases A–L, use the documented scenario descriptions in docs/07 where the underlying data is available. These cases include ordinary administrative/update activity, Office activity without a retrieval/execution chain, unrelated retrieval and execution (G), an authorized behavior match (H), indirect Office ancestry (I), curl retrieval (J), PowerShell call-operator execution (K), and PID reuse with different process GUIDs (L). Do not claim to rerun them without the historical data and implementation artifacts.
6. Investigate each unexpected result using raw events, identifiers, and timestamps. Record the case, expected result, actual result, supporting fields, failure explanation, and proposed change. Retest prior cases after a change.
7. Compare Python and Splunk outputs only when both implementations and the same input are available. Record mismatches and parser/time-field differences; parity applies only to the tested input, implementations, and environment.
8. Document limitations, tuning needs, and next tests. A `behavior_match` means the event sequence matched the analytic's behavior definition. It is not a maliciousness verdict or disposition. Context and human review determine Expected / Authorized, Needs Investigation, or Escalate.

Example analytic-design request:

> Specify a detector for Office ancestry leading to PowerShell retrieval and follow-on execution. Before implementation, list required telemetry fields, identity and ancestry rules, temporal ordering, retrieval/execution behavior families, expected positive and negative behavior, and known blind spots. Explain how the design should handle unrelated retrieval, indirect ancestry, alternate retrieval/execution methods, authorized matching behavior, and PID reuse. Return an analytic specification and test matrix; do not invent field names or claim maliciousness from behavior match alone.

### Results review matrix

| Review item | Record/check |
|---|---|
| Test identity | Case/scenario, dataset version, and exact input used |
| Expected behavior | Match or no match, with rationale |
| Actual behavior | Python result and/or Splunk result; distinguish unrun from failed |
| Evidence | Source events, fields, process identity, timestamps, and causal chain |
| Parity | Same-input comparison, mismatches, and parser/time semantics; mark unavailable if either implementation/artifact is missing |
| Failure and revision | What failed, why, change made, and regression cases rerun |
| Limits | Dataset, implementation, telemetry, and environment scope; deferred tests |

Historical Mission 02 results describe the synthetic A–L dataset and specific implementations only. They are not production performance evidence or proof of general coverage.

## Models

The reported default is `ollama/qwen3.5:9b`; it was useful for evidence reading, extraction, and summarization. A model-status diagnostic reported Ollama auth readiness as indeterminate, while the harmless inference test succeeded. The previously tested hosted route `openai/gpt-5.6-sol` was used for deeper causal reasoning, challenge, ATT&CK reasoning, and design critique. Model selection does not grant or remove MCP authority.

Inspect model status on the Ubuntu VM:

```bash
openclaw models status
```

For the local harmless inference smoke test, use the Quick Start command with a newly chosen session ID. Its historical exact marker was `OPENCLAW_MODEL_OK`; that observation supports only the tested inference request. The “do not call tools” text is behavioral guidance, not enforced isolation. No hosted-model smoke-test command or result is supplied here; verify current CLI syntax and configuration before such a test. Never put credentials in prompts or committed files.

## Web UI Through a Windows SSH Tunnel

The following command shape was reported observed on Windows PowerShell. Replace placeholders locally; do not publish actual usernames or VM addresses:

```powershell
ssh -N -L 18789:127.0.0.1:18789 <vm-user>@<vm-ip>
```

Leave the tunnel process running, then open `http://127.0.0.1:18789/` in a Windows browser. Stop if SSH cannot connect or the local page is unavailable; check VM reachability and Gateway health through the established access path. Do not expose the Gateway publicly as a workaround. Tunnel/browser access was observed, but screenshots do not independently establish which tunnel process served the page. This command was not run during this revision.

## Optional Telegram Access

Telegram DM pairing, approved-account routing to `main`, and a successful Ollama `qwen3.5:9b` response were reported restored and verified. Health inspection reported connected, ready, polling, and zero plugin errors. Historical approved senders were not independently enumerated. Do not infer the complete authorized-sender list from this limited observation.

The default route is `main`; `github-write-job` has zero channel routing rules per the reported agent listing. Telegram access does not authorize, expose, or invoke the dedicated GitHub-write workflow. No pairing/routing command is included because its CLI syntax was not supplied for this guide. If Telegram is unavailable, inspect `openclaw health --json` on the Ubuntu VM and use the established operator interface to check pairing/routing; do not broaden sender access as a troubleshooting shortcut.

## Bounded GitHub-Write Jobs

Use only for a specific, human-authorized update to the intended test repository. The historical bounded-write remediation was **VALIDATED NARROWLY** for specified tested paths and lifecycle cases; see [docs/03-security-validation.md](03-security-validation.md), **Remediation — Bounded GitHub-Write Authority** and **Remediation Limitations**. This section describes a high-risk mutation. Do not proceed without explicit authorization and a verified target.

### Preflight

1. Record the authorized repository, one file path, exact intended change, and prohibited changes. Confirm authorization independently of agent output.
2. On the Ubuntu VM, check that the write MCP server is disabled at rest:

   ```bash
   openclaw config get mcp.servers.github-rw-lab.enabled
   ```

   Reported preflight result was `false`. Stop if it is `true`, ambiguous, or unavailable. Do not start a job or assume another job cleaned up.
3. Confirm no job is active and inspect the installed wrapper before use through the operator's established VM procedure. Reported path and ownership are `/usr/local/sbin/openclaw-github-write-job`, `root:root`, mode `0755`. Ownership/mode support artifact integrity, not an OS privilege boundary.
4. Reconfirm target and authorization. If the request is broader than one well-defined change, split it into separately reviewed jobs.

### Invocation and operation

Source-inspected usage is:

```bash
/usr/local/sbin/openclaw-github-write-job "<bounded job instruction>"
```

The wrapper accepts one or more positional arguments and combines them into the job instruction. The syntax is source-inspected; the wrapper was not freshly executed in this revision. The following is a **NON-EXECUTED TEMPLATE**. Before any use, replace every placeholder, obtain explicit human authorization, and independently verify the target repository and file. Do not execute it as written:

```bash
/usr/local/sbin/openclaw-github-write-job "In authorized test repository <OWNER/TEST-REPO>, update only docs/example.md by changing the existing heading from 'Old heading' to 'New heading'. Do not edit any other file, branch, or content. If the repository, file, or exact current text does not match, stop without writing."
```

The inspected wrapper takes an exclusive `flock`, refuses to start unless `github-rw-lab` is disabled, temporarily enables it, starts `github-write-job` in a fresh UUID session with a 300-second timeout, and attempts to disable the server in EXIT/INT/TERM/HUP cleanup. These are source-inspected operations, not a fresh runtime test in this revision.

### Postflight and abnormal exit

1. Independently inspect the GitHub diff or target file through the read-only route/repository history. Confirm only the authorized file and exact change occurred; agent summary alone is insufficient.
2. On the Ubuntu VM, after **every** write job, check that the server is disabled at rest:

   ```bash
   openclaw config get mcp.servers.github-rw-lab.enabled
   ```

   Required value: `false`. If this state cannot be established, or the result is `true`, stop all further agent operations—not only write jobs—until an operator investigates and restores the expected state. A successful wrapper exit does not guarantee revocation.
3. After a normal interruption/error, first establish that no job is still active. If the server is enabled and no job is active, the operator-reported CLI-help recovery syntax is:

   ```bash
   openclaw mcp configure github-rw-lab --disable
   ```

   This changes configuration; use only with authorization and only after determining that no job remains active. Then repeat the state check. The recovery command was confirmed from CLI help but was not executed in this revision. If activity or state is uncertain, stop all agent operations and inspect the VM through the established administrative procedure. Resume only after the operator confirms the expected disabled state.

SIGKILL, VM/host crash, or power loss may prevent cleanup and leave the service enabled. Do not bypass a lock or manually re-enable the server to recover. The operator's natural-language instruction does not mechanically enforce repository, file, branch, content, or individual write-parameter scope.

### Security limits

The original interactive `approval=prompt` expectation was **TESTED — NOT ENFORCED** on the tested ordinary-agent GitHub-write path. Do not describe configuration or probe output as enforcement. The bounded-write remediation is **VALIDATED NARROWLY** for specified tested paths, not universally. The wrapper is not an OS privilege boundary. Its source, configured tool policies, and cleanup trap are not by themselves proof of runtime behavior or unconditional revocation.

## MCP Checks and Troubleshooting

The OpenClaw 2026.9.1 CLI help was reported to confirm the following command purposes. Only `openclaw mcp status` was actually executed during the current review; other commands below are help-verified, not execution-verified here.

| Ubuntu VM command | Purpose | Connection behavior |
|---|---|---|
| `openclaw mcp status` | Show configured MCP transport status | Passive; does not connect. |
| `openclaw mcp list` | List managed MCP servers | Passive listing. |
| `openclaw mcp show <name>` | Display MCP server configuration | Passive configuration inspection. |
| `openclaw mcp doctor` | Check static setup problems | Static diagnostic; does not establish runtime enforcement. |
| `openclaw mcp probe` | Connect to configured MCP servers and list capabilities | Active diagnostic connection. It may start/connect to services; use only when that diagnostic is appropriate. It does not prove security enforcement. |

Use passive inspection first. Before an active probe, confirm it is acceptable to connect to configured MCP services. Never use a probe result as proof that approval, authorization, isolation, or tool filtering is enforced.

| Symptom | Checks and response |
|---|---|
| Gateway/UI unavailable | Ubuntu VM: `openclaw gateway status`, then `openclaw health --json`. Confirm expected loopback bind and service health. Windows PowerShell: check the established SSH tunnel and browser URL. Do not expose the Gateway publicly. |
| MCP capability appears unavailable | Start with passive `openclaw mcp status`, `list`, `show <name>`, or `doctor`. If appropriate, use active `probe`; it connects and lists capabilities. Stop if configuration differs from expected or the result is unclear. |
| Case evidence not readable | Do not infer availability from status. Make a scoped read request; stop on missing evidence/access errors and have the operator verify mounts/access through the established VM procedure. Do not broaden paths ad hoc. |
| Local inference fails | Ubuntu VM: check `openclaw models status`; then use a harmless inference smoke test with a new session ID. The auth diagnostic may be indeterminate; assess actual output. This does not test tool isolation. |
| Telegram unavailable | Check `openclaw health --json`, then established pairing/routing administration. Historical approved senders remain unenumerated; do not add senders as a diagnostic shortcut. |
| Write job interrupted or enabled state is true | Stop writes. Determine whether a job remains active. If none and authorized, run the manual disable command above, then check state again. Treat crashes/non-trappable termination as possible cleanup failure. |
| Wrapper lock contention | Treat as a possible active job. Do not bypass the lock or start a second writer. Confirm job state through the established VM procedure before retrying. |

## Verification and Recordkeeping

For each investigation or analytic task, preserve the question, authorized scope, model/route, evidence inputs, tools used, raw-source references, result, and limitations. For analytic tests, record the scenario, dataset/version, expected result, actual Python/Splunk outputs if available, mismatches, failure analysis, changes, and regression retests. Do not publish secrets, private repository names, real VM addresses, personal usernames/home paths, credential fragments, or unnecessary session identifiers.

Model output is a claim, not independent evidence. MCP status is configuration/transport observation; a probe is connectivity and capability discovery. Neither proves runtime enforcement. A direct test validates only the specific path and conditions tested.

## Security Boundaries

- `approval=prompt` was **TESTED — NOT ENFORCED** on the tested ordinary-agent GitHub-write path. Root cause and behavior on other paths remain unresolved.
- The bounded-write remediation was **VALIDATED NARROWLY** for specified tested paths. Do not generalize to universal isolation or unconditional revocation.
- The wrapper is not an OS privilege boundary. SIGKILL, VM/host crash, or power loss may prevent cleanup.
- Natural-language job scope is not mechanically enforced per repository, file, branch, content, or individual write parameter.
- Main has `/project:rw` test-repository access and a separate read-only `.git` mount; do not call the main agent universally read-only.
- Configured allow/deny rules, MCP status, approval settings, and agent statements do not alone prove runtime enforcement.
- Telegram routes to `main`, does not authorize/expose `github-write-job`, and historical approved senders were not independently enumerated.
- `dangerouslyAllowExternalBindSources=true` is a residual configuration concern, not proof of escape.
- Mission 02 Python/Splunk parity applies to tested synthetic cases and the specific implementations. It is not production validation or an automatic maliciousness verdict.

## Related Documentation

- [Build and Deployment](01-build-and-deployment.md)
- [Security Hardening](02-security-hardening.md)
- [Security Validation](03-security-validation.md)
- [Lessons Learned](04-lessons-learned.md)
- [Security Operations](05-security-operations.md)
- [Mission 01 — Suspicious PowerShell Investigation](06-mission-01-investigation.md)
- [Mission 02 — Office-to-PowerShell Detection Engineering](07-mission-02-detection.md)
- [Part II Lessons Learned](08-part-ii-lessons-learned.md)
