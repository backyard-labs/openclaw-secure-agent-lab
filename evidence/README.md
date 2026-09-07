# Evidence

This directory is reserved for selected, sanitized evidence artifacts that support the public findings documented in this repository.

The OpenClaw Secure Agent Lab uses the following evidence states:

- CLAIMED — documentation, configuration, a marker, or an agent statement indicates that a control or event exists.
- OBSERVED — a concrete artifact or system state demonstrates that something exists or occurred, but enforcement has not been tested.
- VALIDATED — an appropriate enforcement attempt demonstrated the expected control or boundary.
- UNKNOWN — available evidence is insufficient to support a stronger conclusion.

A test can also demonstrate that an expected control was not enforced. In that case, the result should be stated directly, such as:

TESTED — NOT ENFORCED

The private lab evidence set can include items such as:

- command output
- OpenClaw status and probe output
- agent trajectories
- repository artifacts
- validation notes
- system and configuration observations

Raw evidence is not automatically published.

Use this publication model:

private evidence
→ select
→ sanitize
→ review
→ public documentation

Sensitive material such as credentials, tokens, private repository names, private URLs, raw trajectory bundles, internal session identifiers, and unnecessary personal paths is excluded from the public repository.

Detailed validation results and evidence interpretation are documented in:

[Security Validation](../docs/03-security-validation.md)

Additional sanitized evidence artifacts may be added to this directory when they provide useful public support for a documented finding without exposing sensitive lab information.
