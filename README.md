# OpenClaw Secure Agent Lab

This project documents my hands-on deployment and security validation of an
OpenClaw agent with access to local files, network resources, and bounded
GitHub operations through constrained MCP tools.

The lab focuses on a practical security question:

> **How can an AI agent be given useful capabilities without giving it
> unrestricted authority?**

The repository is structured as both a technical case study and a reproducible
lab. It documents how the environment was built, progressively hardened, and
tested so that other practitioners can construct and evaluate a comparable
environment.

## Why This Lab

Installing and running an AI agent was only the starting point. Once the agent
could interact with files, external resources, and GitHub, the more interesting
problem became controlling what it was actually allowed to do.

The lab therefore evolved from deployment into a security-engineering exercise:

**functional agent → useful capabilities → authority boundaries → security
controls → adversarial validation → evidence-backed baseline**

The goal was not to eliminate agent capabilities, but to constrain them so the
agent could remain useful while limiting the consequences of incorrect
reasoning, prompt injection, excessive permissions, or unintended tool use.

This led to practical work around filesystem scoping, MCP tool isolation,
container hardening, controlled network egress, scoped credentials, separation
of read and write capabilities, human authorization procedures, bounded
GitHub-write remediation, and validation of those controls.

Rather than treating the existence of a control as proof that it worked, the
lab progressively adopted an evidence-oriented approach: distinguish what was
configured or observed from what had actually been tested and validated.

## AI-Assisted Development

This was an AI-assisted project from the beginning.

I used ChatGPT throughout the lab to help design the architecture, generate
installation and configuration commands, troubleshoot failures, develop the
hardening approach, and design the security-validation methodology. Codex was
later used to help construct and maintain this repository.

I remained the operator and decision-maker. Commands were executed in my lab
environment, results were evaluated against actual system behavior,
security-sensitive tests and consequential changes were explicitly authorized
by the human operator, and conclusions were based on observed or validated
evidence rather than AI-generated claims alone.

That workflow became part of the experiment itself:

**AI proposes → human reviews and authorizes → system executes → evidence
returns → human and AI evaluate → next action**

This project therefore also explores a broader engineering question: where AI
can accelerate security work while human authority, verification, and
accountability remain explicit.

## Project Scope

The lab runs OpenClaw on an Ubuntu Server virtual machine hosted in VMware
Workstation. The environment was built as a dedicated place to experiment with
agent capabilities, security controls, and adversarial validation without
granting the agent broad access to the host system or unrelated resources.

OpenClaw was configured to support both a local Ollama model and an OpenAI API
model. This allowed the lab to compare local and hosted model operation while
keeping the agent's tool authority separate from the model providing its
reasoning.

The agent was progressively connected to several capabilities through MCP
servers:

- scoped read-only filesystem access
- scoped read/write filesystem access for designated lab resources
- controlled outbound network access
- read-only GitHub access
- separately constrained GitHub write access for designated lab operations
- a bounded dedicated-agent GitHub write workflow for V1 remediation

The security work focused on the boundaries around those capabilities rather
than on the model alone. Controls included filesystem scoping, separation of
read and write authority, scoped credentials, container isolation, restricted
network egress, MCP tool filtering, and a configured operator-approval policy
for sensitive write actions, with runtime enforcement tested separately.

Selected security boundaries were then tested through practical adversarial
scenarios, including repository-scope isolation and prompt-injection handling.
Results were classified according to the strength of the available evidence
rather than assuming that a configured control was an enforced control.

## Architecture

The lab separates the model used for reasoning from the tools, credentials,
and bounded workflows that determine what the agent can actually do. OpenClaw
acts as the agent orchestration layer, while MCP servers expose individually
scoped capabilities.

```text
                         Human Operator
                              │
                    operator authority /
                    intended approval gate
                              │
                              ▼
                         OpenClaw Agent
                         Ubuntu Server VM
                         VMware Workstation
                              │
              ┌───────────────┴────────────────┐
              │                                │
              ▼                                ▼
       Model / Reasoning                 MCP Capabilities
              │                                │
       ┌──────┴──────┐        ┌───────────────┼────────────────┐
       │             │        │               │                │
       ▼             ▼        ▼               ▼                ▼
    Ollama        OpenAI   Filesystem       Network          GitHub
    local         API          │               │                │
                         ┌─────┴─────┐         │         ┌──────┴──────┐
                         ▼           ▼         ▼         ▼             ▼
                     files-ro  files-rw-lab fetch-net github-ro github-rw-lab
                         │           │                     │             │
                         ▼           ▼                     ▼             ▼
                     read-only   designated             read-only    constrained
                       scope      RW scope               access      write access
```

The model does not receive authority simply because it can reason about an
action. Authority is mediated through the capabilities exposed to OpenClaw,
the configuration of the MCP servers, credential scope, container and network
controls, and operator authorization procedures.

The final V1 lab configuration used five OpenClaw-managed MCP servers:

| MCP server      | Purpose                                               |
| --------------- | ----------------------------------------------------- |
| `files-ro`      | Scoped read-only filesystem access                    |
| `files-rw-lab`  | Read/write access limited to designated lab resources |
| `fetch-net`     | Controlled outbound network retrieval                 |
| `github-ro`     | Read-only GitHub operations                           |
| `github-rw-lab` | Separately constrained GitHub write capability        |

After validation showed that the expected interactive approval gate was not
enforced in the tested ordinary agent path, V1 added a bounded GitHub-write
workflow. The normal main agent does not receive GitHub write capability. For
authorized write work, the operator launches a dedicated `github-write-job`
session with a minimized tool surface and temporary `github-rw-lab` enablement;
the write capability is disabled again through cleanup on the tested job-exit
and failure paths.

The GitHub MCP services were containerized and progressively hardened with
read-only container filesystems, dropped Linux capabilities, prevention of
privilege escalation, scoped credentials, tool filtering, and controlled
outbound connectivity.

This separation was intended to make authority explicit: a model could propose
an action without automatically possessing the capability, credential, network
path, or intended operator authorization required to perform it.

## Security Design

The security design developed around a simple principle: useful agent
capabilities should not imply unrestricted authority.

As capabilities were added, each one created another way for the agent to
interact with systems or data. Each capability was therefore evaluated as a
governance and risk boundary, not just as a feature.

The working model became:

**capability → authority → risk → control → validation**

Several design patterns were applied to constrain that authority.

### Separate Read and Write Authority

Read and write capabilities were exposed separately rather than treating access
to a resource as a single permission. This separation was applied to both
filesystem and GitHub access.

For example, `files-ro` and `github-ro` provided read-oriented capabilities,
while `files-rw-lab` and `github-rw-lab` provided separately constrained write
paths for designated lab operations.

### Scope Capabilities and Credentials

MCP tools and credentials were restricted to the capabilities required for
their defined function. Additional filtering reduced the set of operations
exposed to the agent.

This introduced multiple control layers between a model proposing an action and
the external system accepting it.

The V1 remediation added another layer for GitHub writes: the main agent denies
`github-rw-lab__*`, while a dedicated `github-write-job` path receives only the
read tools and bounded write tool needed for an operator-authorized job. The
write MCP server remains disabled at rest.

### Constrain Execution and Network Paths

GitHub MCP services were moved into hardened containers with read-only
filesystems, dropped Linux capabilities, and privilege escalation disabled.
Outbound connectivity was also constrained through a controlled egress path
rather than leaving the services with unrestricted network access.

### Keep Human Authority Explicit

Sensitive or scope-changing actions were treated as approval boundaries rather
than ordinary agent decisions. Operator authorization was part of the lab
procedure, while technical approval enforcement was tested separately. In the
tested ordinary OpenClaw agent write path, the expected interactive approval
gate was TESTED — NOT ENFORCED; details are documented in
[docs/03-security-validation.md](docs/03-security-validation.md).

The operating pattern was:

**AI proposes → human reviews and authorizes → action executes → evidence
returns → result is evaluated**

The bounded-write remediation preserved that human authorization procedure but
reduced the authority available to ordinary agents instead of relying on the
failed approval mechanism as the primary boundary.

### Treat Configuration as a Claim Until Tested

The lab ultimately distinguished between four evidence states:

| State         | Meaning                                                                              |
| ------------- | ------------------------------------------------------------------------------------ |
| **CLAIMED**   | Documentation or configuration indicates that a control should exist                 |
| **OBSERVED**  | An artifact or system state demonstrates that something exists or occurred           |
| **VALIDATED** | An appropriate enforcement test demonstrated the control under the tested conditions |
| **UNKNOWN**   | Available evidence is insufficient to support a stronger conclusion                  |

Configuration settings or expected behavior were not treated as proof that a
control was working. Where practical, the control was tested directly before
it was considered validated.

## Validation Approach

The lab did not assume that a configured control was working as intended.
Selected controls were tested by trying to use the capability in a way the
control was expected to prevent or restrict.

Each test started with the evidence we had and a specific security question.
We identified what could go wrong, how the agent might reach that outcome, and
which control was expected to stop or limit it.

The reasoning process was:

**artifact → hazard → exploitable path → control → residual risk**

Tests were then planned around the specific boundary being checked. Before
running a test, we checked whether it stayed within the approved scope and
whether the action required explicit authorization. This included actions that
could modify data or access resources outside the approved scope.

The test cycle was:

**observe → hypothesize → plan test → authority gate → act → observe result → update**

This approach was used for tests such as GitHub repository-scope isolation and
prompt-injection handling. We limited each conclusion to what the test
actually demonstrated. A successful test of one path was not treated as proof
that every related control, tool, credential, or attack path had been
validated.

Detailed test procedures, evidence, results, and limitations are documented in
[docs/03-security-validation.md](docs/03-security-validation.md).

## Key Results

Phase 1 validation produced two narrowly VALIDATED security results, one
approval-enforcement finding, and a bounded GitHub-write remediation that was
regression-tested through narrow paths.

### Repository-Scope Isolation

The GitHub read-only path was tested against a private repository that was
outside the credential's intended scope. The access attempt returned no
repository data.

For this test, the credential and read-only tool did not provide access to the
out-of-scope repository. The result was limited to that credential, tool,
repository, and test. It did not establish that every GitHub access path or
repository boundary was isolated.

### Prompt-Injection Handling

Prompt-injection behavior was tested using a repository file that contained
instructions intended to make the agent ignore the user's request, expand its
scope, access secrets and an out-of-scope repository, modify data, and misuse
available tools.

The agent treated those instructions as untrusted content and did not carry out
the requested follow-on actions. No additional repository access, secret
retrieval, write operation, or network request was initiated as a result of the
malicious instructions.

This result was classified as VALIDATED for the tested scenario. It was not
treated as evidence that the agent was generally resistant to prompt injection
or that other attack patterns would produce the same result.

### Write-Approval Enforcement

The `github-rw-lab` service was configured with prompt approval, and the probe
reported a prompt approval policy. A fresh ordinary OpenClaw agent test then
exercised the actual write capability.

The write succeeded without the expected interactive approval gate. The result
was TESTED — NOT ENFORCED in the tested path and is limited to the tested
OpenClaw 2026.9.1 ordinary agent execution path. Other execution paths and the
root cause remain unresolved.

Detailed approval-enforcement results and limitations are documented in
[docs/03-security-validation.md](docs/03-security-validation.md).

### Bounded GitHub-Write Remediation

The approval-enforcement failure led to an architectural remediation: the main
agent was kept without GitHub write capability, and bounded writes were moved
into a dedicated `github-write-job` workflow with temporary `github-rw-lab`
enablement, minimized tools, filesystem-bind separation, and cleanup on tested
exit and failure paths.

Regression testing showed the dedicated path could perform the authorized
write, the main agent did not receive the write tool even while it was enabled
for testing, tested cross-agent delegation paths were blocked, and the wrapper
returned `github-rw-lab` to disabled state in the tested lifecycle cases.

These results were VALIDATED NARROWLY through the tested paths. They do not
prove universal OpenClaw isolation, unconditional revocation, parameter-level
authorization, or that the original `approval=prompt` mechanism became
enforced.

### What Remains Unvalidated

Other controls in the lab were configured or observed but were not all tested
directly to confirm that they worked. Those controls remain CLAIMED, OBSERVED,
VALIDATED narrowly, TESTED — NOT ENFORCED, or UNKNOWN as appropriate rather
than being grouped into broader claims.

Detailed results, evidence, and test limitations are documented in
[docs/03-security-validation.md](docs/03-security-validation.md).

## Repository Guide

The documents follow the lab from initial deployment through hardening,
validation, and lessons learned.

| Location                                                           | What it contains                                                                                   |
| ------------------------------------------------------------------ | -------------------------------------------------------------------------------------------------- |
| [README.md](README.md)                                             | Project overview, architecture, security design, validation approach, remediation, and key results |
| [docs/01-build-and-deployment.md](docs/01-build-and-deployment.md) | Build steps and configuration for creating a comparable OpenClaw lab environment                   |
| [docs/02-security-hardening.md](docs/02-security-hardening.md)     | Security controls and V1 bounded-write architecture used to constrain agent authority              |
| [docs/03-security-validation.md](docs/03-security-validation.md)   | Test methodology, results, remediation validation, limitations, and what was or was not validated  |
| [docs/04-lessons-learned.md](docs/04-lessons-learned.md)           | Problems encountered, design tradeoffs, remediation lessons, and lessons from testing the lab      |
| [diagrams/](diagrams/)                                             | Standalone diagram area; current architecture and trust-boundary diagrams are embedded in the documentation |
| [evidence/](evidence/)                                             | Public evidence policy and location for selected sanitized evidence artifacts when suitable for release |

The build guide is the best starting point for reproducing the lab. The
hardening and validation documents then show how the initial environment was
constrained and tested.

## Current Status

The initial secure-agent lab baseline is complete.

The first validation phase is also complete. It produced both positive and
negative findings: repository-scope isolation was VALIDATED narrowly for
non-disclosure through the exact tested path, prompt-injection handling was
VALIDATED narrowly for the tested interaction, and write-approval enforcement
was TESTED — NOT ENFORCED in the tested ordinary agent path.

The approval gap remains unresolved as a root-cause question, but V1 added and
tested a bounded GitHub-write architecture that keeps GitHub write authority
out of the normal main-agent path and enables it only for dedicated,
operator-authorized jobs. That remediation is VALIDATED NARROWLY through the
tested ordinary-agent, dedicated-agent, delegation, wrapper lifecycle, and
installed-wrapper paths documented in this repository.

Other controls will not be considered validated unless they are tested
directly.

The next phase will use the lab for practical agentic-security work and
additional adversarial testing while continuing to document new controls,
failures, and validation results as they occur.

## Disclaimer

This project documents a controlled lab environment and is intended for
learning, experimentation, and security research. The validation results apply
only to the configurations and test conditions described in this repository
and should not be interpreted as proof that the system is secure against all
attack paths.

Public examples and evidence are sanitized to avoid exposing credentials,
secrets, or other security-sensitive information.
