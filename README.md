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
