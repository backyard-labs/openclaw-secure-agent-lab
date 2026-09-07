# Security Hardening

## 1. Security Objective and Threat Model

The goal of this lab was not to prove that an AI agent was safe in an absolute
sense. The goal was to reduce unnecessary authority, make trust boundaries
explicit, and test selected boundaries before treating them as enforced.

OpenClaw was deployed as an agent capable of interacting with local files,
network resources, GitHub repositories, credentials, and external services.
Each added capability created useful functionality, but it also created a new
path by which model behavior, prompt injection, operator error, or tool misuse
could affect a real system.

The working security model was:

```text
Capability → creates authority → creates attack surface → requires controls → controls require validation
```

The practical threat model included:

- excessive filesystem access
- unnecessary write authority
- unrestricted network egress
- credential exposure
- excessive MCP tool surface
- cross-repository access
- model misuse of legitimate tools
- prompt injection or untrusted content influencing tool behavior
- assuming a configured policy is enforced without testing it

For individual controls, the lab used this reasoning sequence:

```text
ARTIFACT → HAZARD → EXPLOITABLE PATH → CONTROL → RESIDUAL RISK
```

That framing kept the work tied to concrete system behavior. A configuration
setting was not treated as a security conclusion until a relevant enforcement
attempt supported that conclusion.

## 2. Hardening Strategy

The hardening strategy combined defense in depth with capability minimization.
Instead of relying on model instructions alone, controls were applied at
multiple layers:

- VM boundary
- gateway exposure
- container runtime
- filesystem mounts
- container networking
- egress proxy
- MCP server capabilities
- OpenClaw tool filtering
- GitHub credential scope
- operator authorization policy

The operating principle was:

```text
Capability → authority → attack surface → control
```

The project evolved approximately as:

```text
Get OpenClaw working
→ connect useful capabilities
→ recognize trust boundaries
→ constrain capabilities
→ separate read/write authority
→ introduce operator-control mechanisms
→ test the boundaries
→ distinguish configured controls from demonstrated enforcement
→ establish a more defensible secure-agent baseline
```

No single layer was treated as complete isolation. Each layer reduced or
clarified authority, and the remaining risk depended on how that control
behaved when exercised.

## 3. Host and VM Attack-Surface Reduction

The lab ran OpenClaw inside an Ubuntu Server VM named `openclaw-lab` on a
Windows host using VMware Workstation.

Verified VMware configuration included:

- Shared Folders disabled
- drag-and-drop disabled
- copy/paste disabled
- USB Controller removed
- NAT networking used

These choices reduced routine host/guest integration paths. Disabling shared
folders removed an automatic file-sharing bridge. Disabling clipboard and
drag-and-drop reduced accidental transfer paths. Removing USB passthrough
reduced device exposure from the host into the VM. NAT networking avoided
directly bridging the VM onto the external network while still permitting
outbound connectivity through VMware virtual networking.

These are attack-surface reduction measures, not proof of complete VM
isolation. NAT alone is not a comprehensive security boundary, and this lab did
not attempt to prove that all host/guest compromise paths were eliminated.

Evidence state: OBSERVED configuration state.

## 4. Gateway Exposure

The OpenClaw Gateway ran as a user systemd service:

```text
openclaw-gateway.service
```

Verified gateway state included:

- service type: systemd user
- service enabled
- service file: `~/.config/systemd/user/openclaw-gateway.service`
- gateway runtime active/running
- OpenClaw CLI version 2026.9.1
- Gateway version 2026.9.1
- IPv4 loopback listener: `127.0.0.1:18789`
- IPv6 loopback listener: `[::1]`
- no non-loopback listener observed during verification
- health JSON included `"ok": true`

Loopback binding reduces network exposure because the service is not directly
listening on an externally reachable VM interface. In the verified state, the
gateway was reachable locally and was not observed listening on a non-loopback
address.

This conclusion is intentionally narrow. Configuration/listener state was
observed, and loopback functionality was tested. Remote resistance was not
comprehensively adversarially validated in this document, and loopback binding
should not be described as complete network isolation.

Evidence state: OBSERVED listener and service state; functional gateway health
path confirmed for the tested local path.

## 5. MCP Container Hardening

The following MCP capabilities were containerized:

- `fetch-net`
- `files-ro`
- `files-rw-lab`
- `github-ro`
- `github-rw-lab`

Verified common container controls included:

- read-only container root filesystem
- `--cap-drop ALL`
- `no-new-privileges`

The `files-rw-lab` container additionally ran as UID/GID `1000:1000`.

The controls served different purposes. A read-only container root filesystem
reduced opportunities to persist changes inside the container image runtime.
Dropped Linux capabilities reduced privileged kernel-facing operations
available to the container process. `no-new-privileges` prevented the process
from gaining additional privileges through setuid/setgid or similar execution
paths. Running the write-lab filesystem service as UID/GID `1000:1000` avoided
running that process as container root and supported access to the deliberately
writable mounted path.

Important distinction:

```text
read-only container root filesystem ≠ read-only MCP capability
```

A container can have a read-only root filesystem while still having a
deliberately writable bind mount. That was the intended pattern for
`files-rw-lab`: the container root remained read-only, while a designated lab
path was mounted writable.

These container controls reduce attack surface, but they do not prove complete
container containment.

Evidence state: OBSERVED configuration state.

## 6. Filesystem Authority Separation

The lab separated filesystem read authority from filesystem write authority.
That separation was preferable to giving one filesystem MCP server broad mixed
authority over all relevant paths.

`files-ro` was configured with:

- designated filesystem content mounted read-only
- network disabled
- read-oriented tool surface

`files-rw-lab` was configured with:

- designated lab path mounted writable
- read-only container root filesystem
- network disabled
- process running as UID/GID `1000:1000`
- filesystem write capabilities exposed for the intended mounted area

This design allowed routine inspection to use `files-ro` without exposing
write authority. Write operations were routed through a separate MCP server
whose purpose and mounted scope were narrower.

The writable scope remained intentionally constrained, but writable capability
is inherently higher risk than read-only capability. Out-of-scope filesystem
write resistance was not comprehensively adversarially validated in this
document.

Evidence state: OBSERVED configuration and functional smoke-test evidence for
the exercised read/write operations; broader filesystem enforcement remains
outside the evidence in this document.

## 7. Network Egress Control

The verified network-capable MCP architecture included an internal Docker
network:

```text
openclaw-mcp-egress
```

The observed subnet was:

```text
172.18.0.0/16
```

Conceptually, network-capable MCP services used this path:

```text
Network-capable MCP
→ internal Docker network
→ Squid proxy
→ Docker bridge / external network
→ Internet
```

The proxy container was `openclaw-egress-proxy`. Verified proxy hardening
included:

- read-only root filesystem
- dropped Linux capabilities
- `no-new-privileges`
- no host-facing proxy port

The Squid configuration included private and reserved destination restrictions,
with network-capable MCP services routed through the internal network and proxy
for policy-controlled external access.

The objective was to avoid giving network-capable MCP services unrestricted
direct egress merely because they needed external connectivity. `fetch-net`,
`github-ro`, and `github-rw-lab` depended on the proxy-mediated path rather
than direct unrestricted container networking.

The evidence supports a narrow conclusion. Intended proxy-mediated
connectivity was demonstrated for the exercised fetch path. Complete egress
bypass resistance was not demonstrated, and SSRF/bypass testing was not
comprehensive.

Evidence state: OBSERVED network/proxy configuration; functional validation
for the tested fetch operation; broader egress enforcement remains a separate
security-validation question.

## 8. GitHub Read/Write Authority Separation

The lab used separate GitHub MCP instances for read-oriented and write-capable
operations.

`github-ro` used:

- GitHub MCP server
- repository toolset
- server-side `--read-only` mode
- live probe evidence showing read-oriented tools

`github-ro` should not be described as having an OpenClaw tool filter. Its
primary read-only control was the GitHub MCP server's server-side read-only
mode.

`github-rw-lab` used:

- separate GitHub MCP instance
- no server-side `--read-only` flag
- OpenClaw tool filter exposing exactly `create_or_update_file`
- live probe evidence showing one exposed tool and 18 filtered tools
- intended use with a designated lab repository

The write server itself was not read-only. Its risk reduction came from being
separate from routine read operations, using scoped credentials, and exposing a
narrow model-visible write tool surface.

This separation meant that routine GitHub reads did not require write-capable
tooling. Write actions were intentionally routed through a separately named
capability whose purpose was easier to inspect, restrict, and reason about.

Evidence state: OBSERVED configuration and live probe surface; functional
smoke-test evidence for the exercised read and write operations.

## 9. Credential Scope and Protection

Verified local credential protections included:

- secrets directory mode `700`
- credential/environment file mode `600`
- credential file owned by the intended Linux user
- credentials injected into containers using an environment file rather than
  embedded in public configuration examples

The GitHub credential was a fine-grained PAT with:

- access to one selected repository
- Metadata: Read
- Contents/Code: Read and Write
- no user-level permissions observed
- short expiration during the lab

The PAT itself should not be described as read-only. It had write authority for
Contents/Code in the selected repository. The read/write separation came from
server configuration, tool filtering, credential scope, and repository
selection, not from the token being read-only.

The defense-in-depth posture was:

```text
credential scope
+ filesystem permissions
+ container isolation
+ tool filtering
+ repository separation
```

Together, those controls reduced authority more effectively than any single
control would have. The token value must never be published.

Evidence state: OBSERVED local credential-file protections and credential
scope; validation of specific repository boundaries belongs in
[03-security-validation.md](03-security-validation.md).

## 10. Tool-Surface Minimization

Tool-surface minimization reduced what the model could invoke through the
runtime interface.

Verified live probe results included:

| MCP server | Exposed surface |
| --- | --- |
| `github-ro` | 13 exposed tools, no write tools observed, GitHub MCP server operating read-only |
| `github-rw-lab` | 1 exposed tool, 18 filtered, exposed tool `create_or_update_file` |
| `files-ro` | 7 exposed tools, 4 filtered |
| `files-rw-lab` | 11 exposed tools |
| `fetch-net` | one fetch capability |

Minimizing model-visible tools matters because a model cannot invoke a tool
that is not exposed to it through that runtime surface. This is not equivalent
to complete sandboxing. Tool visibility does not prove that all underlying
server, credential, repository, filesystem, or network boundaries behave as
intended.

Live probe evidence demonstrates the projected/exposed tool surface. It does
not by itself validate every enforcement boundary.

Evidence state: OBSERVED live probe surface.

## 11. Operator Authorization and Approval Behavior

Operator authorization was a central part of the intended design. Sensitive or
scope-changing actions should require explicit human authority rather than
being treated as ordinary model decisions.

`github-rw-lab` was explicitly configured with:

```bash
openclaw mcp configure github-rw-lab --approval prompt --probe
```

The saved configuration contained:

```json
{
  "codex": {
    "defaultToolsApprovalMode": "prompt"
  }
}
```

The MCP probe reported:

```json
{
  "codexApprovalMode": "prompt"
}
```

These facts support two observed states:

- configured approval policy: OBSERVED
- probe-reported prompt policy: OBSERVED

They do not, by themselves, prove mandatory interactive approval enforcement.
That had to be tested separately.

A fresh explicit ordinary OpenClaw agent session was used to test runtime
enforcement. The exported trajectory demonstrated:

- `github-rw-lab__create_or_update_file` was exposed
- the model invoked that exact tool
- the requested GitHub file was successfully created
- no interactive operator approval was surfaced before execution
- no approval event appeared in the trajectory

The successful write produced file SHA:

```text
4f9283f60b5db477bb76a692a8503afcc75d1559
```

and commit:

```text
8e940f330870be3fd50fcb458fa73fe01b3824bd
```

No private repository identity is needed to understand the finding.

Conclusion: OpenClaw 2026.9.1 accepted and reported the prompt approval policy,
but the policy did not enforce a mandatory interactive human gate in the
ordinary OpenClaw agent execution path tested in this lab.

Source inspection found approval-enforcement logic associated with the Codex
MCP harness/projection implementation. Bundled OpenClaw documentation describes
`plugin-sdk/codex-mcp-projection` as a private-local bundled Codex helper for
projecting user MCP configuration into Codex thread/app-server thread
configuration.

This does not prove that `approval=prompt` universally works only with Codex.
It does not prove that all OpenClaw execution paths bypass approval. It does
not establish a product-wide vulnerability or a definitive root cause.
Behavior outside the tested OpenClaw 2026.9.1 ordinary agent execution path
remains outside the demonstrated evidence.

The intended `--approval prompt` policy should remain configured. The lesson is
that configured policy and reported policy are not the same as demonstrated
runtime enforcement:

```text
Configured policy → reported policy ≠ demonstrated runtime enforcement
```

A separate trajectory observation showed that, even when the model was
explicitly instructed to use only `github-rw-lab`, it first invoked `github-ro`
tools before performing the write. That is distinct from approval enforcement,
but it supports a related lesson: natural-language tool-use instructions should
not be treated as a security boundary.

Evidence state: approval configuration OBSERVED; probe-reported policy
OBSERVED; runtime enforcement TESTED — expected mandatory interactive approval
was NOT ENFORCED in the tested ordinary agent path.

## 12. Defense-in-Depth Summary

| Control | Security objective | Evidence state | Important limitation |
| --- | --- | --- | --- |
| VM integration reduction | Reduce host/guest transfer paths | OBSERVED | Does not prove complete VM isolation |
| Loopback Gateway | Reduce external network exposure of OpenClaw Gateway | OBSERVED plus local health path checked | Remote resistance was not comprehensively adversarially validated |
| Container hardening | Reduce container process and filesystem authority | OBSERVED | Does not prove complete container containment |
| Filesystem RO/RW separation | Keep routine reads separate from writable lab scope | OBSERVED plus functional smoke tests | Out-of-scope write resistance was not exhaustively tested here |
| Proxy-mediated egress | Route network-capable MCP services through a controlled path | OBSERVED plus functional fetch test | Complete egress-bypass resistance was not demonstrated |
| GitHub RO/RW separation | Keep routine reads separate from write-capable GitHub operations | OBSERVED plus functional smoke tests | Does not prove all GitHub boundaries or paths are isolated |
| Credential scoping | Limit token authority to selected repository permissions | OBSERVED | The PAT still had Contents/Code write authority for the selected repository |
| MCP tool filtering | Reduce model-visible tool surface | OBSERVED live probe surface | Tool filtering is not complete sandboxing |
| `approval=prompt` configuration | Require prompt-mode approval policy for sensitive write tooling | OBSERVED | Configuration state does not prove runtime enforcement |
| Mandatory interactive approval enforcement | Confirm human approval gate appears before write execution | TESTED — NOT ENFORCED in tested path | Applies to the tested OpenClaw 2026.9.1 ordinary agent execution path only |

The table intentionally distinguishes observed configuration from demonstrated
enforcement. Where a control was configured but not enforced in the tested path,
the result is stated directly rather than forced into a misleading success
state.

## 13. Residual Risk

The final baseline reduced authority and attack surface, but it did not
eliminate risk.

Remaining risks include:

- the model can misuse legitimate exposed capabilities
- prompt injection remains relevant when untrusted content reaches the model
- writable capabilities remain inherently higher risk than read-only
  capabilities
- credentials remain sensitive even when narrowly scoped
- egress proxy bypass was not comprehensively adversarially tested
- filesystem scope enforcement was not exhaustively tested
- approval enforcement was not demonstrated for the tested ordinary agent path
- tool filtering reduces exposed capability but is not equivalent to complete
  sandboxing
- future OpenClaw versions may change behavior

The useful security outcome is not a claim of perfect safety. The outcome is a
clearer, smaller authority surface with evidence labels that separate observed
state from tested enforcement.

## 14. From Hardening to Validation

Hardening establishes intended controls. Validation asks whether those controls
actually constrain behavior.

The handoff from this document to the validation document follows this
sequence:

```text
Configured control
→ observable implementation
→ enforcement attempt
→ evidence
→ evidence state
```

The next document,
[03-security-validation.md](03-security-validation.md), examines
adversarial/control-validation tests including repository-scope isolation,
prompt injection handling, approval enforcement, and other tested boundaries.
