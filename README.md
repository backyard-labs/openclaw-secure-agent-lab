# OpenClaw Secure Agent Lab

This project documents my hands-on deployment and security validation of an
OpenClaw agent with access to local files, network resources, and GitHub through
constrained MCP tools.

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
of read and write capabilities, human approval boundaries, and validation of
those controls.

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
security-sensitive actions required explicit approval, and conclusions were
based on observed or validated evidence rather than AI-generated claims alone.

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

The security work focused on the boundaries around those capabilities rather
than on the model alone. Controls included filesystem scoping, separation of
read and write authority, scoped credentials, container isolation, restricted
network egress, MCP tool filtering, and explicit human approval for sensitive
actions.

Selected security boundaries were then tested through practical adversarial
scenarios, including repository-scope isolation and prompt-injection resistance.
Results were classified according to the strength of the available evidence
rather than assuming that a configured control was an enforced control.

## Architecture

The lab separates the model used for reasoning from the tools and credentials
that determine what the agent can actually do. OpenClaw acts as the agent
orchestration layer, while MCP servers expose individually scoped capabilities.

```text
                         Human Operator
                              │
                    approval / authority
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
controls, and human approval boundaries.

The final lab configuration used five OpenClaw-managed MCP servers:

| MCP server      | Purpose                                               |
| --------------- | ----------------------------------------------------- |
| `files-ro`      | Scoped read-only filesystem access                    |
| `files-rw-lab`  | Read/write access limited to designated lab resources |
| `fetch-net`     | Controlled outbound network retrieval                 |
| `github-ro`     | Read-only GitHub operations                           |
| `github-rw-lab` | Separately constrained GitHub write capability        |

The GitHub MCP services were containerized and progressively hardened with
read-only container filesystems, dropped Linux capabilities, prevention of
privilege escalation, scoped credentials, tool filtering, and controlled
outbound connectivity.

This separation was intended to make authority explicit: a model could propose
an action without automatically possessing the capability, credential, network
path, or approval required to perform it.
