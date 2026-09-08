# Build and Deployment

## Purpose and Scope

This document describes how to reproduce a comparable build and deployment
baseline for the OpenClaw Secure Agent Lab.

The lab evolved iteratively. The initial work was getting OpenClaw installed
and connected to a model. The later work added scoped MCP capabilities,
containerized services, network controls, GitHub integration, and functional
smoke tests. This guide preserves that progression where it matters, but it is
primarily written as reproduction guidance for the final baseline.

Detailed security rationale belongs in
[02-security-hardening.md](02-security-hardening.md). Detailed adversarial
validation belongs in [03-security-validation.md](03-security-validation.md).
The checks in this document are deployment and functionality checks unless
explicitly stated otherwise.

This was an AI-assisted, human-directed build. AI helped generate and refine
commands and configuration, while the human operator approved sensitive
actions, executed work in the lab, and evaluated results against observed
system behavior.

## Final Lab Architecture

The final baseline used a Windows host running VMware Workstation, an Ubuntu
Server VM named `openclaw-lab`, OpenClaw inside the VM, two model paths, and
five OpenClaw-managed MCP servers.

```text
Windows host
├── VMware Workstation
├── Ollama
│   └── qwen3.5:9b
└── Ubuntu Server VM: openclaw-lab
    ├── OpenClaw 2026.9.1
    ├── OpenAI Responses API model path
    │   └── openai/gpt-5.6-sol
    └── MCP servers
        ├── fetch-net
        ├── files-ro
        ├── files-rw-lab
        ├── github-ro
        └── github-rw-lab
```

All final MCP servers used stdio transport through Dockerized MCP servers.
The filesystem servers were isolated from the network. The network and GitHub
servers used the `openclaw-mcp-egress` Docker network and the
`openclaw-egress-proxy` proxy path.

V1 later added a bounded GitHub-write workflow that keeps `github-rw-lab`
disabled at rest and enables it temporarily only for an operator-launched
dedicated `github-write-job` session. The build details below preserve the MCP
foundation; the bounded-write architecture and validation are documented in
[02-security-hardening.md](02-security-hardening.md) and
[03-security-validation.md](03-security-validation.md).

## Prerequisites

The core prerequisite sequence was:

```text
Host computer
→ VMware Workstation
→ Ubuntu Server 24.04 LTS
→ Internet connectivity
→ OpenClaw installation
```

For the final baseline, the reader should have:

- A Windows host capable of running VMware Workstation.
- VMware Workstation installed.
- Ubuntu Server 24.04 LTS installation media.
- Administrative command-line access inside the Ubuntu VM.
- Internet connectivity from the VM.
- Docker CE available inside the VM for the final MCP server baseline.
- Ollama installed on the Windows host if reproducing the local model path.
- An OpenAI API key if reproducing the hosted model path.
- GitHub credentials with appropriate scope if reproducing the GitHub MCP
  paths.

Git was present in the lab environment, but it was not a core prerequisite for
the initial OpenClaw installation. Git becomes relevant when reproducing
repository-oriented workflows and GitHub MCP operations.

VMware shared folders, shared clipboard, drag-and-drop, and USB auto-connect
were disabled as isolation choices. Those choices are mentioned here only as
baseline context; the security rationale is covered in
[02-security-hardening.md](02-security-hardening.md).

## 1. Create the Ubuntu VM

Create an Ubuntu Server VM in VMware Workstation with the following baseline:

| Setting | Value |
| --- | --- |
| Guest OS | Ubuntu Server 24.04.4 LTS |
| Hostname | `openclaw-lab` |
| vCPU | 4 |
| Memory | 8 GB |
| Disk | 80 GB thin-provisioned |
| Network | VMware NAT |
| Addressing | DHCP |
| Access | Command-line administrative access |

Do not copy a lab-specific private IP address. Use the address assigned to
your own VM by VMware NAT/DHCP.

After installing Ubuntu, confirm the system version and connectivity:

```bash
hostnamectl
ip addr
curl -I https://openclaw.ai/
```

Conclusion: these checks establish the VM identity, assigned network address,
and Internet reachability needed before installing OpenClaw. They do not
validate any security boundary.

## 2. Install Docker Runtime

Docker is required for the final MCP baseline. The lab used Docker CE from
Docker's official Ubuntu repository:

```text
https://download.docker.com/linux/ubuntu noble stable
```

The exact historical Docker apt installation commands were not recovered. For
reproduction, install Docker using Docker's supported Ubuntu installation
procedure for Ubuntu 24.04, then verify the runtime state.

The verified lab versions were:

| Component | Observed lab version |
| --- | --- |
| Docker CE | 29.8.0, build 88096ef |
| `docker-ce` | `5:29.8.0-1~ubuntu.24.04~noble` |
| `docker-ce-cli` | `5:29.8.0-1~ubuntu.24.04~noble` |
| `containerd.io` | `2.3.4-2~ubuntu.24.04~noble` |
| `docker-buildx-plugin` | `0.37.0` |
| `docker-compose-plugin` | `5.5.1` |

These are the versions observed in this lab, not universal requirements for
all reproductions.

Verify Docker after installation:

```bash
docker --version
docker compose version
systemctl status docker --no-pager
docker info --format 'Server Version={{.ServerVersion}} Storage Driver={{.Driver}} Cgroup Driver={{.CgroupDriver}}'
```

Observed final runtime state:

- Docker service enabled and active
- storage driver `overlayfs`
- cgroup driver `systemd`

Conclusion: these checks observe the Docker runtime needed for Dockerized MCP
servers. They do not prove that later container or network security boundaries
cannot be bypassed.

## 3. Install OpenClaw

The lab installed OpenClaw with onboarding disabled, then ran onboarding
separately:

```bash
curl -fsSL --proto '=https' --tlsv1.2 https://openclaw.ai/install.sh | bash -s -- --no-onboard
openclaw onboard --install-daemon
```

During this lab installation, the installer:

- selected npm installation
- installed required build dependencies
- installed Node.js through NodeSource because Node was initially absent
- installed Node.js v24.20.0
- installed npm 11.19.0
- detected that the system npm prefix `/usr` was not writable
- configured the user-local npm prefix `~/.npm-global`
- wrote `~/.npmrc`
- installed OpenClaw 2026.9.1

OpenClaw 2026.9.1 is the version used by this lab. It is not a claim that
readers must always use that exact version.

The exact historical post-onboarding gateway verification command was not
recovered. For current reproduction with OpenClaw 2026.9.1, verify the gateway
and health path with:

```bash
openclaw --version
openclaw gateway status
openclaw health --json
```

Observed current final state:

- service type: systemd user
- service enabled
- service file: `~/.config/systemd/user/openclaw-gateway.service`
- gateway runtime active/running
- OpenClaw CLI version 2026.9.1
- Gateway version 2026.9.1
- bind: loopback `127.0.0.1`
- port: `18789`
- connectivity probe: ok
- health JSON included `"ok": true`

Do not publish process IDs, session IDs, cron IDs, or health output containing
internal session metadata.

Conclusion: gateway status/runtime was observed and the tested gateway health
path succeeded. These reproduction checks do not prove that any MCP server is
correctly configured yet.

## 4. Configure Model Providers

The lab supports a local Ollama model path and a hosted OpenAI API model path.
The model providing reasoning is separate from the MCP tools and credentials
that determine what the agent can do.

### 4.1 Ollama on the Windows Host

The local model path was:

```text
Windows host
→ Ollama
→ qwen3.5:9b
→ VMware NAT / VMnet8
→ Ubuntu VM
→ OpenClaw
```

On the Windows host, discover the VMnet8 address with `ipconfig`. Do not copy
a lab-specific address. The OpenClaw endpoint should use this form:

```text
http://<WINDOWS_HOST_VMNET8_IP>:11434
```

The OpenClaw provider structure used for the local path corresponds to:

```yaml
providers:
  ollama:
    baseUrl: http://<WINDOWS_HOST_VMNET8_IP>:11434
    api: ollama
```

OpenClaw 2026.9.1 accepted the following provider command shape during
independent schema validation with `--dry-run`. For reproduction, run the
command without `--dry-run` after replacing the VMnet8 placeholder:

```bash
openclaw config set models.providers.ollama \
  '{"baseUrl":"http://<WINDOWS_HOST_VMNET8_IP>:11434","apiKey":"ollama-local","api":"ollama","models":[]}' \
  --strict-json
```

The default local model used by the lab was:

```text
ollama/qwen3.5:9b
```

Set the Ollama model as the default with the installed supported syntax:

```bash
openclaw models set ollama/qwen3.5:9b
```

Verify the model configuration:

```bash
openclaw config get models
openclaw models status
```

A models-status diagnostic reported Ollama auth readiness as `indeterminate`,
while the direct functional inference test below succeeded. Treat that as a
diagnostic observation, not as a failed functional test.

A directly verified functional smoke test used:

```bash
openclaw agent exec "Reply exactly: OLLAMA-SMOKE-OK" \
  --model ollama/qwen3.5:9b \
  --local-model-lean \
  --json
```

Observed result:

- `ok: true`
- `final: OLLAMA-SMOKE-OK`
- `provider: ollama`
- `model: qwen3.5:9b`
- no tool calls

Conclusion: this validated the tested functional inference path from OpenClaw
in the Ubuntu VM to Ollama on the Windows host for `qwen3.5:9b`. It did not
test MCP capability boundaries.

### 4.2 OpenAI API

The hosted model path was:

```text
OpenClaw
→ OpenAI Responses API
→ openai/gpt-5.6-sol
```

The OpenAI profile used API-key authentication. Do not publish the key or any
partially masked key identifier. Use a placeholder such as `<YOUR_API_KEY>` in
documentation and store the real value only in an appropriate local secret
location.

The installed OpenClaw CLI supports interactive API-key entry:

```bash
openclaw models auth paste-api-key \
  --provider openai \
  --profile-id openai:manual
```

This syntax was verified through installed CLI help during reconstruction, but
the command was not rerun because doing so would modify the working credential
state.

The important runtime configuration was model-specific:

```text
agents.defaults.models["openai/gpt-5.6-sol"].agentRuntime.id = "openclaw"
```

The model-specific runtime command was independently schema-validated with
`--dry-run`. For reproduction, run the command without `--dry-run`:

```bash
openclaw config set \
  'agents.defaults.models["openai/gpt-5.6-sol"].agentRuntime' \
  '{"id":"openclaw"}' \
  --strict-json
```

Do not document this as a top-level `agentRuntime` setting or as a provider-wide
OpenAI runtime setting. The provider-level setting
`models.providers.openai.agentRuntime` was valid but unset in the lab.

A directly verified functional smoke test used:

```bash
openclaw agent exec "Reply exactly: OPENAI-SMOKE-OK" \
  --model openai/gpt-5.6-sol \
  --json
```

Observed result:

- HTTP 200 from `https://api.openai.com/v1/responses`
- `ok: true`
- `final: OPENAI-SMOKE-OK`
- `provider: openai`
- `model: gpt-5.6-sol`

Hosted API inference can incur usage charges. Any observed smoke-test cost in a
specific lab run should not be treated as a promised or representative price.

Conclusion: this validated the tested functional inference path from OpenClaw
to the OpenAI Responses API for `openai/gpt-5.6-sol`. It did not test tool
authority or data-access boundaries.

## 5. Deploy the MCP Foundation

The final OpenClaw-managed MCP inventory was:

```text
OpenClaw
├── fetch-net
├── files-ro
├── files-rw-lab
├── github-ro
└── github-rw-lab
```

The installed OpenClaw CLI supports this configuration pattern:

```bash
openclaw mcp set <name> '<JSON object>'
```

Tool filtering uses this syntax:

```bash
openclaw mcp tools <name> --include <comma-separated-tools>
```

Useful inspection commands are:

```bash
openclaw mcp list
openclaw mcp status
openclaw mcp show <name>
openclaw mcp probe
```

`openclaw mcp probe` connects to configured MCP servers and enumerates their
exposed capabilities. Treat probe results as connectivity and capability
discovery evidence, not as security-boundary validation.

### 5.1 MCP Configuration Pattern

The examples below show the final configuration shape with public placeholders.
Replace placeholders before running commands:

- `<YOUR_LINUX_USER>`: your Linux account name
- `<READ_ONLY_REPO>`: the repository directory mounted read-only
- `<WRITE_LAB>`: the designated write-lab directory
- `<YOUR_REPOSITORY>`: the target repository used for reproduction

For commands that embed JSON, preserve argument order and quoting as required
by your shell. The examples below are written for a Bash shell on the Ubuntu
VM.

### 5.2 GitHub Credential File

Before configuring the GitHub MCP containers, create the credential directory
and env file:

```bash
mkdir -p ~/.openclaw/secrets
chmod 700 ~/.openclaw/secrets
```

Create `~/.openclaw/secrets/github-mcp.env` with this public shape:

```text
GITHUB_PERSONAL_ACCESS_TOKEN=<YOUR_GITHUB_TOKEN>
```

Then restrict file permissions:

```bash
chmod 600 ~/.openclaw/secrets/github-mcp.env
```

Verified current state:

- secrets directory permissions: `700`
- `github-mcp.env` permissions: `600`
- file owned by the Linux user
- variable name: `GITHUB_PERSONAL_ACCESS_TOKEN`

Optional safe verification:

```bash
stat -c 'Permissions=%a Owner=%U:%G Bytes=%s' \
  ~/.openclaw/secrets/github-mcp.env

sed -E 's/^([^#][^=]*)=.*/\1=<REDACTED>/' \
  ~/.openclaw/secrets/github-mcp.env
```

Do not publish the real token. Credential least-privilege rationale belongs in
[02-security-hardening.md](02-security-hardening.md).

### 5.3 MCP Egress Network and Proxy

The network and GitHub MCP services depend on a controlled egress path:

```text
MCP container
→ internal Docker network openclaw-mcp-egress
→ Squid proxy openclaw-egress-proxy:3128
→ proxy also attached to ordinary Docker bridge
→ external network
```

The verified Docker network state was:

- name: `openclaw-mcp-egress`
- driver: `bridge`
- `Internal=true`
- IPv6 disabled
- observed subnet `172.18.0.0/16`

The following command is a reconstruction of the observed final configuration,
not necessarily the exact historical command:

```bash
docker network create --internal openclaw-mcp-egress
```

Create the proxy configuration directory:

```bash
mkdir -p ~/labs/openclaw/egress-proxy
```

Create `~/labs/openclaw/egress-proxy/squid.conf`:

```text
http_port 3128
pid_filename /run/squid/squid.pid

acl localnet src 172.18.0.0/16

acl blocked_dst dst 0.0.0.0/8
acl blocked_dst dst 10.0.0.0/8
acl blocked_dst dst 100.64.0.0/10
acl blocked_dst dst 127.0.0.0/8
acl blocked_dst dst 169.254.0.0/16
acl blocked_dst dst 172.16.0.0/12
acl blocked_dst dst 192.168.0.0/16
acl blocked_dst dst 224.0.0.0/4
acl blocked_dst dst 240.0.0.0/4

http_access deny blocked_dst
http_access allow localnet
http_access deny all

access_log stdio:/dev/stdout
cache_log /dev/null
logfile_rotate 0
```

Verified final proxy characteristics:

- image: `ubuntu/squid`
- container: `openclaw-egress-proxy`
- Squid user: `13:13`
- read-only root filesystem
- `--cap-drop ALL`
- `no-new-privileges`
- `squid.conf` bind mounted read-only
- `/run/squid` tmpfs:
  `rw,noexec,nosuid,size=16m,uid=13,gid=13,mode=0755`
- restart policy `unless-stopped`
- port `3128` exposed internally but not published to the host
- connected to `openclaw-mcp-egress`
- also attached to the default Docker bridge for external connectivity

The following Docker creation commands are reconstructions of the observed
final configuration:

```bash
docker run -d \
  --name openclaw-egress-proxy \
  --restart unless-stopped \
  --network openclaw-mcp-egress \
  --user 13:13 \
  --read-only \
  --cap-drop ALL \
  --security-opt no-new-privileges \
  --tmpfs /run/squid:rw,noexec,nosuid,size=16m,uid=13,gid=13,mode=0755 \
  --mount type=bind,src="$HOME/labs/openclaw/egress-proxy/squid.conf",dst=/etc/squid/squid.conf,ro \
  ubuntu/squid

docker network connect bridge openclaw-egress-proxy
```

Verify the observed state:

```bash
docker network inspect openclaw-mcp-egress \
  --format 'Name={{.Name}} Driver={{.Driver}} Internal={{.Internal}}'

docker ps --filter name=openclaw-egress-proxy
```

Conclusion: the network and proxy configuration can be observed from Docker
state. A successful fetch through this path validates the specific fetch
operation. Broader egress enforcement remains a separate security-validation
question.

### 5.4 Read-Only Filesystem

The `<READ_ONLY_REPO>` source directory must already exist before configuring
`files-ro`. This guide does not prescribe how the historical read-only
repository was obtained.

Create the final `files-ro` configuration:

```bash
openclaw mcp set files-ro '{
  "command": "docker",
  "args": [
    "run",
    "--rm",
    "-i",
    "--network",
    "none",
    "--read-only",
    "--cap-drop",
    "ALL",
    "--security-opt",
    "no-new-privileges",
    "--mount",
    "type=bind,src=/home/<YOUR_LINUX_USER>/labs/openclaw/repos/<READ_ONLY_REPO>,dst=/projects/<READ_ONLY_REPO>,ro",
    "mcp/filesystem",
    "/projects"
  ]
}'
```

Apply the include filter:

```bash
openclaw mcp tools files-ro --include read_file,read_multiple_files,list_directory,directory_tree,search_files,get_file_info,list_allowed_directories
```

Expected exposed tools:

- `read_file`
- `read_multiple_files`
- `list_directory`
- `directory_tree`
- `search_files`
- `get_file_info`
- `list_allowed_directories`

### 5.5 Write-Lab Filesystem

Create the designated write-lab source directory before configuring
`files-rw-lab`:

```bash
mkdir -p /home/<YOUR_LINUX_USER>/labs/openclaw/<WRITE_LAB>
```

Create the final `files-rw-lab` configuration:

```bash
openclaw mcp set files-rw-lab '{
  "command": "docker",
  "args": [
    "run",
    "--rm",
    "-i",
    "--user",
    "1000:1000",
    "--network",
    "none",
    "--read-only",
    "--cap-drop",
    "ALL",
    "--security-opt",
    "no-new-privileges",
    "--mount",
    "type=bind,src=/home/<YOUR_LINUX_USER>/labs/openclaw/<WRITE_LAB>,dst=/projects/<WRITE_LAB>",
    "mcp/filesystem",
    "/projects"
  ]
}'
```

Apply the include filter:

```bash
openclaw mcp tools files-rw-lab --include read_file,read_multiple_files,list_directory,directory_tree,search_files,get_file_info,list_allowed_directories,write_file,edit_file,create_directory,move_file
```

Expected exposed tools:

- `read_file`
- `read_multiple_files`
- `list_directory`
- `directory_tree`
- `search_files`
- `get_file_info`
- `list_allowed_directories`
- `write_file`
- `edit_file`
- `create_directory`
- `move_file`

### 5.6 Controlled Network Fetch

Create the final `fetch-net` configuration after the egress network and proxy
exist:

```bash
openclaw mcp set fetch-net '{
  "command": "docker",
  "args": [
    "run",
    "--rm",
    "-i",
    "--network",
    "openclaw-mcp-egress",
    "--read-only",
    "--cap-drop",
    "ALL",
    "--security-opt",
    "no-new-privileges",
    "mcp/fetch",
    "--proxy-url",
    "http://openclaw-egress-proxy:3128"
  ]
}'
```

Apply the include filter:

```bash
openclaw mcp tools fetch-net --include fetch
```

Expected exposed tool:

- `fetch`

### 5.7 GitHub Read-Only

Create the final `github-ro` configuration after the credential file, egress
network, and proxy exist:

```bash
openclaw mcp set github-ro '{
  "command": "docker",
  "args": [
    "run",
    "--rm",
    "-i",
    "--network",
    "openclaw-mcp-egress",
    "--read-only",
    "--cap-drop",
    "ALL",
    "--security-opt",
    "no-new-privileges",
    "--env-file",
    "/home/<YOUR_LINUX_USER>/.openclaw/secrets/github-mcp.env",
    "-e",
    "HTTPS_PROXY=http://openclaw-egress-proxy:3128",
    "-e",
    "HTTP_PROXY=http://openclaw-egress-proxy:3128",
    "ghcr.io/github/github-mcp-server:latest",
    "stdio",
    "--toolsets=repos",
    "--read-only"
  ]
}'
```

The `:latest` image tag reflects the lab configuration. Pinning a known image
version or digest improves reproducibility.

Do not publish the contents of `~/.openclaw/secrets/github-mcp.env`. For
reproduction, store only the required GitHub credential value in that file,
using a scoped credential appropriate for your repository and intended
operations.

### 5.8 GitHub Write-Lab

The final `github-rw-lab` configuration used the same container, network, and
security structure as `github-ro`, except the GitHub MCP server `--read-only`
flag was omitted.

Create the final `github-rw-lab` configuration:

```bash
openclaw mcp set github-rw-lab '{
  "command": "docker",
  "args": [
    "run",
    "--rm",
    "-i",
    "--network",
    "openclaw-mcp-egress",
    "--read-only",
    "--cap-drop",
    "ALL",
    "--security-opt",
    "no-new-privileges",
    "--env-file",
    "/home/<YOUR_LINUX_USER>/.openclaw/secrets/github-mcp.env",
    "-e",
    "HTTPS_PROXY=http://openclaw-egress-proxy:3128",
    "-e",
    "HTTP_PROXY=http://openclaw-egress-proxy:3128",
    "ghcr.io/github/github-mcp-server:latest",
    "stdio",
    "--toolsets=repos"
  ]
}'
```

Apply the include filter:

```bash
openclaw mcp tools github-rw-lab --include create_or_update_file
```

Expected exposed tool:

- `create_or_update_file`

The detailed credential-scope rationale belongs in
[02-security-hardening.md](02-security-hardening.md).

## 6. Verify MCP Connectivity

After configuring the five MCP servers, inspect the inventory:

```bash
openclaw mcp list
openclaw mcp status
openclaw mcp show fetch-net
openclaw mcp show files-ro
openclaw mcp show files-rw-lab
openclaw mcp show github-ro
openclaw mcp show github-rw-lab
openclaw mcp probe
```

The verified final probe showed:

| MCP server | Probe result |
| --- | --- |
| `fetch-net` | 1 tool |
| `files-ro` | 7 tools |
| `files-rw-lab` | 11 tools |
| `github-ro` | 13 tools plus resources/prompts |
| `github-rw-lab` | 1 tool plus resources/prompts |

OpenClaw also reported that some MCP tools lacked safety annotations and
therefore required approval under prompting-session postures. Treat that as an
operational observation. It does not prove that security boundaries are
complete or bypass-proof.

Conclusion: the final probe validated MCP connectivity and capability
discovery for the configured servers. It did not validate repository-scope
isolation, prompt-injection resistance, credential least privilege, or network
egress enforcement.

## 7. Run Functional Smoke Tests

These tests check deployment functionality. They are not adversarial security
validation. The repository-scope and prompt-injection validation work is
documented in [03-security-validation.md](03-security-validation.md).

### 7.1 `files-ro`

A functional smoke test used:

```bash
openclaw agent exec "Use the files-ro MCP server to list the allowed directories, then list the top-level contents of the allowed directory. Do not use any other tools." \
  --json
```

Functional smoke behavior:

- OpenClaw invoked `files-ro__list_allowed_directories`.
- OpenClaw invoked `files-ro__list_directory`.
- 2 tool calls were made.
- 0 failures were reported.
- the expected directory was returned.

Conclusion: the tested read-only filesystem MCP path could list the allowed
directory. This did not prove that every filesystem boundary was enforced.

### 7.2 `files-rw-lab`

A functional smoke test used:

```bash
openclaw agent exec "Using files-rw-lab only, create deployment-smoke-test.txt with exactly FILES-RW-SMOKE-OK, then read it back and report whether the final content matches." \
  --json
```

Functional smoke behavior:

- created `deployment-smoke-test.txt`
- wrote `FILES-RW-SMOKE-OK`
- read the file back successfully
- final content matched `FILES-RW-SMOKE-OK`
- tool summary reported 5 calls and 1 intermediate failure
- the exact failed call could not be recovered from audit records
- the overall write/read functional objective succeeded

Conclusion: the tested write-lab filesystem MCP path could write and read a
file in the designated lab resource. The unrecovered intermediate failure
should remain visible in the record rather than being treated as nonexistent.
This did not prove that writes outside the designated scope were impossible.

### 7.3 `fetch-net`

A functional smoke test used:

```bash
openclaw agent exec "Use fetch-net to fetch https://example.com and report the page title." \
  --json
```

Functional smoke behavior:

- fetched `https://example.com`
- used exactly `fetch-net__fetch`
- 1 tool call was made
- 0 failures were reported
- returned title: `Example Domain`

Conclusion: the tested network-fetch MCP path could retrieve the expected
external page through the configured fetch server. This did not prove that all
network egress restrictions were enforced.

### 7.4 `github-ro`

A functional smoke test used:

```bash
openclaw agent exec "Using github-ro only, read the top-level contents of <YOUR_REPOSITORY>." \
  --json
```

Functional smoke behavior:

- read top-level repository contents from `<YOUR_REPOSITORY>`
- used `github-ro__get_file_contents`
- 1 tool call was made
- 0 failures were reported

Conclusion: the tested GitHub read-only MCP path could read repository
contents from the selected in-scope repository. This did not prove that every
repository boundary was isolated.

### 7.5 `github-rw-lab`

A functional smoke test used:

```bash
openclaw agent exec "Using github-rw-lab only, create or update deployment-smoke-test.txt in <YOUR_REPOSITORY> with exactly GITHUB-RW-SMOKE-OK." \
  --json
```

Functional smoke behavior:

- created or updated `deployment-smoke-test.txt`
- wrote `GITHUB-RW-SMOKE-OK`
- used exactly `github-rw-lab__create_or_update_file`
- 1 tool call was made
- 0 failures were reported
- produced a GitHub commit

The result was then independently checked through `github-ro`:

```bash
openclaw agent exec "Using github-ro only, read deployment-smoke-test.txt from <YOUR_REPOSITORY> and reply with its exact contents." \
  --json
```

- used `github-ro__get_file_contents`
- 1 tool call was made
- 0 failures were reported
- returned exactly `GITHUB-RW-SMOKE-OK`

Conclusion: the tested GitHub write-lab MCP path could create or update the
expected file in the selected in-scope repository, and the read-only GitHub
path could independently read the resulting content. This did not prove that
other GitHub write paths, repositories, branches, or credentials were
constrained.

## 8. Final Deployment Baseline

At the end of the build phase, the lab baseline contained:

- Ubuntu Server 24.04.4 LTS running in VMware Workstation as `openclaw-lab`
- OpenClaw 2026.9.1 installed and onboarded
- Docker CE from Docker's official Ubuntu repository
- Docker service enabled and active with `overlayfs` and `systemd` cgroups
- local model path to Ollama on the Windows host
- hosted model path to the OpenAI Responses API
- GitHub credential env file stored under `~/.openclaw/secrets`
- egress network `openclaw-mcp-egress`
- proxy container `openclaw-egress-proxy`
- five OpenClaw-managed MCP servers using Docker stdio transport
- read-only filesystem access through `files-ro`
- designated write-lab filesystem access through `files-rw-lab`
- controlled network fetch through `fetch-net`
- read-only GitHub access through `github-ro`
- separately constrained GitHub write-lab access through `github-rw-lab`
- functional smoke-test evidence for model inference and selected MCP
  operations

Using the evidence language applied throughout this repository:

| Evidence state | Meaning |
| --- | --- |
| CLAIMED | Documentation or a marker says a control exists. |
| OBSERVED | A concrete artifact or configuration demonstrates something exists or occurred, but enforcement was not directly tested. |
| VALIDATED | An appropriate test directly exercised the specific claim and produced the expected result. |
| UNKNOWN | Available evidence is insufficient. |

For this build document:

- model smoke tests validate the tested functional inference paths
- MCP probe validates connectivity and capability discovery
- capability smoke tests validate their specific functional operations
- none of these alone proves that security boundaries cannot be bypassed

## Reproducibility Notes

Use placeholders in public documentation:

- `<YOUR_LINUX_USER>`
- `<YOUR_API_KEY>`
- `<YOUR_GITHUB_TOKEN>`
- `<YOUR_REPOSITORY>`
- `<READ_ONLY_REPO>`
- `<WRITE_LAB>`
- `<WINDOWS_HOST_VMNET8_IP>`

Do not publish credential-file contents, token fragments, private repository
names, private IP addresses, or unnecessary personal paths.

The lab used `ghcr.io/github/github-mcp-server:latest` for GitHub MCP services.
Pinning a known image tag or digest is recommended when reproducibility matters
more than tracking the latest container image.

The exact historical post-onboarding gateway verification command was not
recovered. Any current verification command in this document should therefore
be treated as reproduction guidance, not as a claim about the exact command
used during the first deployment.

The Docker network and proxy creation commands are also reconstructions of the
observed final configuration, not recovered historical commands.

The final hardened architecture did not exist from the beginning. It was built
through iterative installation, model setup, MCP integration, smoke testing,
and later hardening work.

## Next Steps

After reproducing the build baseline:

1. Review the hardening rationale and controls in
   [02-security-hardening.md](02-security-hardening.md).
2. Review adversarial validation procedures, evidence, and limitations in
   [03-security-validation.md](03-security-validation.md).
3. Add new evidence only after sanitizing credentials, token material, private
   repository names where unnecessary, and personal paths.
