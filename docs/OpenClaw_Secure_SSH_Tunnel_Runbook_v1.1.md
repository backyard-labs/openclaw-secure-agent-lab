# OpenClaw Secure SSH Access — Four-Tier Deployment, Validation & Rollback Runbook

**Version:** 1.1
**Date:** 2026-10-22
**Status:** Tier 3 tested in the recorded Windows/Ubuntu lab; Tiers 0–2 are reproducible guidance, not separately validated end-to-end in this session.
**Audience:** Lab operators, security engineers, and authorized users of managed Windows workstations.

> **Scope and safety:** This guide provides four *alternatives*, not four mandatory phases. Choose the least complex tier that meets your operational needs. Use only on systems and networks where SSH and port forwarding are authorized. Never weaken organizational endpoint policy, SSH host-key checking, or the Ubuntu Gateway's loopback binding to make a tier work.

## 1. Quick decision guide

| Tier | How it works | Windows administrator? | Persistence / recovery | Best fit |
|---|---|---|---|---|
| **0 — Manual password SSH** | Interactive username/password, terminal stays open | Normally no | None | Quick authorized lab access; password authentication already permitted |
| **1 — Manual key SSH** | Dedicated SSH key, terminal stays open | Normally no | None | **Default for recurring access** |
| **2 — User-launched key SSH** | Key-based connection via user-owned SSH config and shortcut/script | Normally no | User starts/stops; no watchdog | Convenience without task/service changes |
| **3 — Unattended S4U + watchdog** | Two Windows Scheduled Tasks, SSH key, periodic recovery | Often yes, subject to policy | Tested recovery from SSH process termination | Authorized unattended workstation operation |

**Security note:** Password-based SSH is encrypted and can be appropriate with a strong unique password and hardened server. It is not automatically insecure, nor is every key deployment secure. Tier 0 has the *least setup friction*, not a universal security rating. Tier 3 introduces persistent credentials and scheduled execution, which require additional endpoint controls.

### Tested lab values (replace for another environment)

| Parameter | Recorded value |
|---|---|
| Windows account | `Cyb3r` on `DESKTOP-FCMPT46` |
| Ubuntu SSH user | `ali` |
| Ubuntu VM | `192.168.93.130` (VMware VMnet8) |
| Ubuntu Gateway listener | `127.0.0.1:18789` |
| Windows local tunnel listener | `127.0.0.1:18789` |
| Dedicated Windows key | `%USERPROFILE%\.ssh\id_openclaw_lab` |
| Permanent tasks (Tier 3) | `OpenClaw-SSH-Tunnel`, `OpenClaw-Tunnel-Watchdog` |
| Runbook script directory (Tier 3) | `%LOCALAPPDATA%\OpenClawLab` |

## 2. Shared prerequisites — all tiers

**Ubuntu terminal:** Verify that the OpenClaw Gateway listens only on loopback. A `127.0.0.1:18789` or `[::1]:18789` listener is expected; **do not** expose `0.0.0.0:18789` merely for remote access.

```bash
ss -lnt | grep ':18789'
```

**Windows PowerShell:** Verify OpenSSH availability, reachability, and local port availability:

```powershell
ssh -V
Test-NetConnection 192.168.93.130 -Port 22
Get-NetTCPConnection -LocalPort 18789 -State Listen -ErrorAction SilentlyContinue
```

Expected: OpenSSH client installed; `TcpTestSucceeded : True`; no existing Windows listener on `18789` before launching a new tunnel. A listener from an already-working tunnel is not an error; do not launch a competing instance.

**Host identity:** On first SSH connection, compare the server's presented host-key fingerprint against a trusted fingerprint obtained from the Ubuntu administrator or console. Do not use `StrictHostKeyChecking=no` or automatically trust a changed key. If a host key changes unexpectedly, stop and investigate.

**Gateway authentication:** The SSH tunnel encrypts transport and restricts the forwarded port to local loopback. It does **not** replace OpenClaw's own authentication, authorization, or browser-session security.

## 3. Tier 0 — Manual username/password SSH (lowest friction)

**Prerequisite:** The Ubuntu SSH server already allows password authentication for this account. Do not enable it on a hardened or managed host solely for convenience without approval. The Windows client does not need administrator rights for a normal user-level SSH connection.

**Windows PowerShell:**

```powershell
ssh -o PreferredAuthentications=password `
    -o PubkeyAuthentication=no `
    -o ExitOnForwardFailure=yes `
    -o ServerAliveInterval=15 -o ServerAliveCountMax=3 `
    -N -L 127.0.0.1:18789:127.0.0.1:18789 `
    ali@192.168.93.130
```

1. Verify the server's host key if prompted.
2. Enter the **Ubuntu SSH account password** interactively; do not put passwords in command lines, scripts, shortcuts, or logs.
3. Keep this PowerShell terminal open. The command normally shows no output after successful authentication.
4. Open `http://127.0.0.1:18789` in the Windows browser and verify OpenClaw login/access.
5. In a second PowerShell window, optionally verify `Get-NetTCPConnection -LocalAddress 127.0.0.1 -LocalPort 18789 -State Listen`.

**Validation:** SSH remains connected; browser reaches Gateway; Windows listener is loopback only. **Rollback / shutdown:** press `Ctrl+C` in the SSH terminal, verify the listener disappears. No Scheduled Tasks, services, or SSH keys were created by this tier. If the password is rejected, do not change `sshd_config` blindly; check server policy with the administrator.

## 4. Tier 1 — Manual dedicated SSH key (recommended default)

### 4.1 Generate the key on Windows

```powershell
$key = "$env:USERPROFILE\.ssh\id_openclaw_lab"
if (Test-Path $key) { throw "Existing key found; do not overwrite: $key" }
ssh-keygen -t ed25519 -f $key -C 'openclaw-lab-tunnel'
Get-Content "$key.pub"
```

Prefer a strong key passphrase for interactive use, consistent with organization policy. **Only copy the `.pub` contents to Ubuntu.** Never copy or email the private key. Key generation itself does not normally require Windows elevation.

### 4.2 Install the public key on Ubuntu

Log in using an already-authorized method (password or console). As the target Ubuntu user:

```bash
mkdir -p ~/.ssh
chmod 700 ~/.ssh
touch ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys
```

Append the exact public-key line displayed on Windows to `~/.ssh/authorized_keys` **once**, using a text editor or another approved method. Do not replace existing keys or paste the private key. Record the key comment/fingerprint for targeted rollback. On a managed server, key enrollment may require administrator approval.

### 4.3 Validate key authentication and forwarding

```powershell
ssh -i "$env:USERPROFILE\.ssh\id_openclaw_lab" `
    -o IdentitiesOnly=yes -o PasswordAuthentication=no `
    ali@192.168.93.130 'echo OPENCLAW_KEY_OK'
```

Expected: `OPENCLAW_KEY_OK` (a private-key **passphrase** prompt may still appear). Then start the foreground tunnel:

```powershell
ssh -i "$env:USERPROFILE\.ssh\id_openclaw_lab" `
    -o IdentitiesOnly=yes -o PasswordAuthentication=no `
    -o ExitOnForwardFailure=yes `
    -o ServerAliveInterval=15 -o ServerAliveCountMax=3 `
    -N -L 127.0.0.1:18789:127.0.0.1:18789 `
    ali@192.168.93.130
```

Verify `http://127.0.0.1:18789` in the browser. Keep the terminal open. **Rollback / shutdown:** `Ctrl+C`; for complete Tier 1 decommission, remove only the exact matching public-key line from Ubuntu `authorized_keys` and delete the dedicated local key pair only after checking that no other workflow uses it. Preserve other authorized keys.

## 5. Tier 2 — User-managed key SSH (no Scheduled Tasks)

**Prerequisite:** Tier 1 works. Tier 2 is a usability wrapper around the same key-based tunnel, not an additional authentication layer.

### 5.1 Optional per-user SSH configuration

Edit `%USERPROFILE%\.ssh\config` with a text editor. Preserve any existing entries and add:

```sshconfig
Host openclaw-lab
    HostName 192.168.93.130
    User ali
    IdentityFile ~/.ssh/id_openclaw_lab
    IdentitiesOnly yes
    PasswordAuthentication no
    ExitOnForwardFailure yes
    ServerAliveInterval 15
    ServerAliveCountMax 3
    LocalForward 127.0.0.1:18789 127.0.0.1:18789
```

Start in a normal PowerShell window:

```powershell
ssh -N openclaw-lab
```

Alternatively, save a user-owned `Start-OpenClaw-Tunnel.ps1` containing the explicit Tier 1 `ssh` command, and launch it from a terminal or approved shortcut. **Do not store a password** or embed an elevated command. Keep the window available so you can see errors and stop the connection with `Ctrl+C`. Avoid hidden launchers unless you also provide a reliable stop/diagnostic path.

**Validation:** Confirm the listener, browser connectivity, and `ssh -G openclaw-lab` effective configuration if needed. **Rollback:** stop the foreground SSH process with `Ctrl+C`, remove only the `Host openclaw-lab` block and user-created shortcut/script. Retain or revoke the Tier 1 key separately as appropriate. No Windows services or Scheduled Tasks are needed.

## 6. Tier 3 — Unattended S4U tunnel with watchdog

**Prerequisite:** Tier 1's dedicated key works, and the operator has explicit authorization to create Scheduled Tasks and use S4U. A managed workstation may prohibit these actions even if the user is technically able to run them. A passphrase-protected key and Windows `ssh-agent` may behave differently in noninteractive S4U sessions; test actual access and do not assume the interactive agent carries over. Protect any noninteractively usable private key with appropriate file ACLs and endpoint controls.

### 6.1 Architecture

```text
Windows browser → 127.0.0.1:18789
                       |
           ssh.exe, S4U Scheduled Task
                       | SSH TCP/22
                       v
              Ubuntu 192.168.93.130
                       |
            127.0.0.1:18789 Gateway

Every 2 minutes: separate S4U PowerShell watchdog
  └─ If tunnel task is Ready and local listener absent, start tunnel task
```

**Recorded permanent artifacts:** `OpenClaw-SSH-Tunnel` and `OpenClaw-Tunnel-Watchdog`, with `OpenClaw-Tunnel-Watchdog.ps1` under `%LOCALAPPDATA%\OpenClawLab`. Preserve task XML backups before replacing an existing task. The complete tested task-creation, watchdog script, validation, and event diagnostics are retained in **Appendix A** below (original v1.0 implementation). Use those procedures for a fresh Tier 3 build; substitute your account, key, host, and path.

### 6.2 Acceptance checks

1. Tunnel task uses `S4U`, `Limited` run level, direct `ssh.exe`, a sign-in trigger, `IgnoreNew`, and unlimited execution time.
2. Tunnel starts invisibly, listens on **Windows loopback only**, and reaches the Gateway.
3. Watchdog runs under an authorized S4U principal every **2 minutes**, without an unwanted console window.
4. With SSH deliberately terminated, the watchdog starts a **new** tunnel process; browser access returns. Record task history and new PID.
5. A healthy running tunnel is not duplicated by the watchdog.

**Observed limitation:** Task Scheduler's built-in restart policy (`RestartCount=3`, `RestartInterval=PT1M`) did **not** restore the tunnel after a forced SSH termination. The separate watchdog **did**. The tested recovery scenario was forced SSH process termination, not Windows reboot, VM reboot, prolonged outage, or credential expiry.

### 6.3 Tier 3 rollback (maintenance vs removal)

**Intentional maintenance — disable recovery first:**

```powershell
Disable-ScheduledTask -TaskName 'OpenClaw-Tunnel-Watchdog'
Stop-ScheduledTask -TaskName 'OpenClaw-SSH-Tunnel'
```

For maintenance across sign-in/reboot, also **disable the tunnel task** to prevent its sign-in trigger from starting it. Verify that port `18789` has no listener. When ready to resume:

```powershell
Enable-ScheduledTask -TaskName 'OpenClaw-SSH-Tunnel'
Enable-ScheduledTask -TaskName 'OpenClaw-Tunnel-Watchdog'
Start-ScheduledTask -TaskName 'OpenClaw-SSH-Tunnel'
```

**Remove automation, keep manual Tier 1 access:** Stop/disable the watchdog first, stop/disable the tunnel, then unregister only the two task definitions (after backing them up if required):

```powershell
Unregister-ScheduledTask -TaskName 'OpenClaw-Tunnel-Watchdog' -Confirm:$false
Unregister-ScheduledTask -TaskName 'OpenClaw-SSH-Tunnel' -Confirm:$false
```

The dedicated SSH key and Ubuntu public-key authorization can remain for manual Tier 1 access. Do not delete shared SSH services or alter Gateway binding. For restoration of a prior task definition, see Appendix A, section **R3**; confirm the XML backup is valid and note that the recorded backup used Interactive logon and displayed a console window.

**Full decommission:** Remove automation as above; verify no listener; archive/remove only dedicated watchdog files and task backups subject to retention policy; remove the exact matching public key from Ubuntu; unload/delete the dedicated private key only after confirming no other dependencies. Preserve unrelated credentials, SSH configurations, and tasks.

## 7. Cross-tier troubleshooting

| Symptom | Likely checks | Safe next action |
|---|---|---|
| `ssh` command not found | `ssh -V`, Windows OpenSSH Client availability | Request approved client installation; may require IT/admin |
| Password login rejected (Tier 0) | Account status, SSH server auth policy, logs | Confirm credentials and policy; do not weaken server configuration |
| Key login rejected (Tiers 1–3) | Correct `.pub` installed, `~/.ssh` permissions, client `-i` path, host/user | Fix only the matching key entry or permissions |
| Host-key warning | Trusted host-key fingerprint, VM rebuild or possible interception | Stop; verify out-of-band before updating `known_hosts` |
| Port already in use | `Get-NetTCPConnection -LocalPort 18789 -State Listen` | Identify owner; do not kill unknown process |
| Tunnel listens but Web UI fails | Ubuntu `ss -lnt`, Gateway health, OpenClaw auth | Check Gateway separately; tunnel alone doesn't guarantee app health |
| S4U task exits `255` | Task history, SSH reachability, key availability, Ubuntu SSH logs | Diagnose cause; do not assume task restart settings will recover |
| Watchdog not running | Trigger, `Get-ScheduledTaskInfo`, Task Scheduler Operational log | Verify schedule and permissions; inspect history |
| Need planned shutdown | Tier 3 watchdog may restart tunnel | Disable watchdog **before** stopping tunnel |

### Useful Tier 3 diagnostics

```powershell
Get-ScheduledTask -TaskName 'OpenClaw*' | Select-Object TaskName,State
Get-ScheduledTaskInfo -TaskName 'OpenClaw-Tunnel-Watchdog' | Format-List LastRunTime,LastTaskResult,NextRunTime
Get-NetTCPConnection -LocalAddress 127.0.0.1 -LocalPort 18789 -State Listen -ErrorAction SilentlyContinue
Get-WinEvent -ListLog 'Microsoft-Windows-TaskScheduler/Operational' | Select-Object LogName,IsEnabled,RecordCount
```

If authorized and necessary, enable Task Scheduler Operational logging from an elevated PowerShell session:

```powershell
wevtutil sl Microsoft-Windows-TaskScheduler/Operational /e:true
```

## 8. Validation record and remaining tests

**Verified on October 22, 2026:** S4U key authentication; localhost-only listener; browser access; no console window; clean task stop; automatic watchdog execution; successful watchdog recovery after forced SSH process termination; cleanup of temporary test tasks. The last recorded tunnel PID was `59748` and the watchdog's last recorded result was `0` at 2:18:54 PM; these are historical evidence, **not** current-state guarantees.

**Not yet validated:** A fresh installation following this consolidated v1.1 guide; Windows sign-in/reboot recovery; Ubuntu VM restart; extended outage; network address change; key rotation; full rollback/restoration exercise. Record these as future acceptance tests rather than claiming they passed.

## 9. Change log

| Version | Date | Change |
|---|---|---|
| 1.0 | 2026-10-22 | Retrospective automated S4U tunnel and watchdog runbook |
| 1.1 | 2026-10-22 | Four deployment tiers, password and manual-key options, managed-workstation decision guide, per-tier rollback, preserved Tier 3 implementation |

---

# Appendix A — Original v1.0 Tier 3 implementation and detailed rollback

> The following is the original advanced deployment runbook retained for implementation details and audit continuity. Its Phase A–F procedures apply to **Tier 3 only**; they are **not prerequisites** for Tiers 0–2. If the two sections differ in scope or safety advice, follow the v1.1 main guide above. Some procedures are reconstructed guidance rather than fresh-install validated.

# OpenClaw Secure SSH Tunnel — Deployment, Recovery & Rollback Runbook

**Version:** 1.0
**Document date:** 2026-10-22
**Environment:** Windows host → Ubuntu VMware VM → OpenClaw Gateway
**Status:** Core deployment and watchdog recovery tested; reboot/network-outage recovery not yet tested.

> **Scope:** This runbook reconstructs the *successful final design* from a 95-step interactive troubleshooting session. It is a streamlined deployment guide, **not** a verbatim transcript. Commands that were not executed end-to-end in the original session are labeled as reproducible guidance and must be validated in a fresh installation. Run commands in the specified shell.

## 1. Outcome and architecture

A Windows workstation connects securely to an OpenClaw Gateway bound only to Ubuntu loopback. Windows OpenSSH forwards `127.0.0.1:18789` to Ubuntu `127.0.0.1:18789` over SSH. A Windows Scheduled Task launches `ssh.exe` invisibly using S4U; a second Scheduled Task runs a PowerShell watchdog every two minutes to restore a stopped tunnel.

```text
Windows browser
  http://127.0.0.1:18789
           |
Windows localhost TCP 18789
           |
ssh.exe (Scheduled Task: OpenClaw-SSH-Tunnel, S4U)
           |  encrypted SSH, key authentication, TCP/22
           v
Ubuntu VM 192.168.93.130
           |
Ubuntu 127.0.0.1:18789 → OpenClaw Gateway

Windows Scheduled Task: OpenClaw-Tunnel-Watchdog
  every 2 min → if tunnel task is Ready AND localhost port is free,
  start OpenClaw-SSH-Tunnel
```

### Recorded environment

| Item | Verified value |
|---|---|
| Windows user | `Cyb3r` on `DESKTOP-FCMPT46` |
| Ubuntu SSH account | `ali` |
| Ubuntu VM address | `192.168.93.130` (VMware VMnet8) |
| Windows SSH executable | `C:\Windows\System32\OpenSSH\ssh.exe` |
| Dedicated key | `C:\Users\Cyb3r\.ssh\id_openclaw_lab` |
| Key fingerprint | `SHA256:S/C8IqtiQwZe/o6TYrGRlBnAEmiPjmVAMCGs6/mJpVA` |
| Local and remote Gateway port | `18789` |
| Scripts directory | `C:\Users\Cyb3r\AppData\Local\OpenClawLab` |
| Permanent tunnel task | `OpenClaw-SSH-Tunnel` |
| Permanent watchdog task | `OpenClaw-Tunnel-Watchdog` |
| Watchdog script | `OpenClaw-Tunnel-Watchdog.ps1` |
| Existing original-task backup | `OpenClaw-SSH-Tunnel-Backup.xml` |

**Do not copy hostnames, usernames, VM addresses or key paths blindly to another machine.** Substitute the target environment's values.

## 2. Prerequisites and change control

1. Confirm the Ubuntu VM is running, its SSH service is reachable, and OpenClaw Gateway is healthy on Ubuntu `127.0.0.1:18789`.
2. Confirm Windows OpenSSH Client is installed (`ssh -V`) and PowerShell supports `Get-ScheduledTask` / `Register-ScheduledTask`.
3. Confirm Windows can reach Ubuntu TCP/22 (`Test-NetConnection 192.168.93.130 -Port 22`).
4. Confirm Windows port 18789 is unused before deployment (`Get-NetTCPConnection -LocalPort 18789 -State Listen -ErrorAction SilentlyContinue`).
5. Use a Windows account with permission to register Scheduled Tasks. An elevated PowerShell window may be necessary for registration or task changes.
6. Establish a maintenance window; opening and stopping the tunnel interrupts Windows browser access, **not** the Ubuntu Gateway itself.
7. Record any pre-existing Scheduled Tasks, SSH keys, `authorized_keys` entries, or listener configuration before changing them. Back up any existing task definitions:

```powershell
Export-ScheduledTask -TaskName 'OpenClaw-SSH-Tunnel' |
    Set-Content "$env:LOCALAPPDATA\OpenClawLab\OpenClaw-SSH-Tunnel-Backup.xml" -Encoding Unicode
```

Run the backup only **if that task already exists**. Confirm the file exists and is nonempty. The original session had an existing backup; no backup of the watchdog task was required because it was newly created.

## 3. Deployment procedure — clean installation

### Phase A — Gateway and network baseline

**A1 — Ubuntu (SSH terminal): verify Gateway loopback binding.**

```bash
ss -lnt | grep ':18789'
```

Expected: a listener on `127.0.0.1:18789` and/or `[::1]:18789`, **not** a public `0.0.0.0:18789` binding. Do not expose the Gateway directly to the LAN to make this procedure work.

**A2 — Windows PowerShell: verify VM SSH reachability.**

```powershell
Test-NetConnection 192.168.93.130 -Port 22
```

Expected: `TcpTestSucceeded : True`.

### Phase B — Dedicated SSH authentication

**B1 — Windows PowerShell: generate a dedicated ED25519 key if absent.**

```powershell
$key = "$env:USERPROFILE\.ssh\id_openclaw_lab"
if (Test-Path $key) { throw "Key already exists; do not overwrite: $key" }
ssh-keygen -t ed25519 -f $key -C 'openclaw-lab-tunnel'
```

Use the organization's key passphrase policy. The **tested** S4U configuration relied on a key usable noninteractively; Windows `ssh-agent` was enabled and the key loaded. Key availability to S4U should be tested explicitly; do not assume interactive agent access always carries over to S4U. Never copy the private key to Ubuntu or put it in scripts.

**B2 — Windows PowerShell: display the public key.**

```powershell
Get-Content "$env:USERPROFILE\.ssh\id_openclaw_lab.pub"
```

**B3 — Ubuntu: install only the public key** as a new line in `~/.ssh/authorized_keys` for `ali`, after checking for duplicates. Protect the directory and file:

```bash
mkdir -p ~/.ssh
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys
```

**B4 — Windows PowerShell: validate key-only authentication.**

```powershell
ssh -i "$env:USERPROFILE\.ssh\id_openclaw_lab" `
  -o IdentitiesOnly=yes -o BatchMode=yes -o PasswordAuthentication=no `
  ali@192.168.93.130 'echo OPENCLAW_AGENT_OK'
```

Expected: `OPENCLAW_AGENT_OK` without password prompts. Confirm the SSH host key fingerprint through a trusted channel at first connection; **do not disable host-key checking**.

**B5 — Windows: verify ssh-agent if needed.**

```powershell
Get-Service ssh-agent
ssh-add -l
```

The original deployment used `ssh-agent` set to Automatic and Running, with the dedicated key loaded. If a key has a passphrase, validate S4U access rather than assuming an unlocked interactive agent is sufficient.

### Phase C — Prove the port forward manually

**C1 — Windows PowerShell: start a foreground test** (this command occupies the terminal; stop it with Ctrl+C when finished).

```powershell
ssh -i "$env:USERPROFILE\.ssh\id_openclaw_lab" `
  -o IdentitiesOnly=yes -o BatchMode=yes `
  -o ExitOnForwardFailure=yes `
  -o ConnectTimeout=10 `
  -o ServerAliveInterval=15 -o ServerAliveCountMax=3 `
  -N -L 127.0.0.1:18789:127.0.0.1:18789 `
  ali@192.168.93.130
```

**C2 — Separate Windows terminal:**

```powershell
Get-NetTCPConnection -LocalAddress 127.0.0.1 -LocalPort 18789 -State Listen
```

Open `http://127.0.0.1:18789` in a Windows browser. Confirm the Web UI works. Stop the foreground SSH command before scheduling the task, to avoid a port conflict.

### Phase D — Create the permanent hidden S4U tunnel task

**D1 — Windows PowerShell (elevate if necessary):**

```powershell
$sshExe = 'C:\Windows\System32\OpenSSH\ssh.exe'
$key = "$env:USERPROFILE\.ssh\id_openclaw_lab"
$args = '-i "' + $key + '" -o IdentitiesOnly=yes -o BatchMode=yes ' +
        '-o ExitOnForwardFailure=yes -o ConnectTimeout=10 ' +
        '-o ServerAliveInterval=15 -o ServerAliveCountMax=3 ' +
        '-N -L 127.0.0.1:18789:127.0.0.1:18789 ali@192.168.93.130'
$action = New-ScheduledTaskAction -Execute $sshExe -Argument $args
$trigger = New-ScheduledTaskTrigger -AtLogOn -User $env:USERNAME
$principal = New-ScheduledTaskPrincipal -UserId $env:USERNAME -LogonType S4U -RunLevel Limited
$settings = New-ScheduledTaskSettingsSet -MultipleInstances IgnoreNew `
    -RestartCount 3 -RestartInterval (New-TimeSpan -Minutes 1) `
    -ExecutionTimeLimit ([TimeSpan]::Zero)
Register-ScheduledTask -TaskName 'OpenClaw-SSH-Tunnel' `
    -Action $action -Trigger $trigger -Principal $principal -Settings $settings
```

**If the task already exists, do not run `Register-ScheduledTask` over it.** Export and review it first; use `Set-ScheduledTask` for controlled updates. The recorded task used a user-logon trigger, S4U, Limited run level, IgnoreNew, three restarts at one-minute intervals, and unlimited execution time.

**D2 — Verify and start:**

```powershell
Get-ScheduledTask -TaskName 'OpenClaw-SSH-Tunnel' |
    Select-Object TaskName, State, @{N='LogonType';E={$_.Principal.LogonType}}
Start-ScheduledTask -TaskName 'OpenClaw-SSH-Tunnel'
Start-Sleep -Seconds 5
Get-NetTCPConnection -LocalAddress 127.0.0.1 -LocalPort 18789 -State Listen
```

Expected: S4U, task Running, loopback listener, working Web UI, **no console window**. If S4U authentication fails, stop here and investigate; do not silently fall back to storing a Windows password in a task.

### Phase E — Create and test the watchdog

**E1 — Windows PowerShell: create the watchdog script.**

```powershell
$dir = Join-Path $env:LOCALAPPDATA 'OpenClawLab'
New-Item -ItemType Directory -Force -Path $dir | Out-Null
$scriptPath = Join-Path $dir 'OpenClaw-Tunnel-Watchdog.ps1'
@'
$taskName = 'OpenClaw-SSH-Tunnel'
$task = Get-ScheduledTask -TaskName $taskName -ErrorAction Stop
if ($task.State -eq 'Running') { exit 0 }
if ($task.State -ne 'Ready') { exit 1 }
$listener = Get-NetTCPConnection -LocalAddress '127.0.0.1' `
    -LocalPort 18789 -State Listen -ErrorAction SilentlyContinue
if ($listener) { exit 0 }
Start-ScheduledTask -TaskName $taskName -ErrorAction Stop
'@ | Set-Content -Path $scriptPath -Encoding UTF8
```

**E2 — Syntax check (does not execute):**

```powershell
$errorsFound = $null
$null = [System.Management.Automation.Language.Parser]::ParseFile(
    $scriptPath, [ref]$null, [ref]$errorsFound)
if ($errorsFound.Count -eq 0) { 'Syntax validation: PASS' } else { $errorsFound }
```

**E3 — Manual watchdog test:** Stop the tunnel task with `Stop-ScheduledTask`, confirm the port is free, then run:

```powershell
& $scriptPath
Start-Sleep -Seconds 5
Get-ScheduledTask -TaskName 'OpenClaw-SSH-Tunnel' | Select TaskName, State
Get-NetTCPConnection -LocalAddress 127.0.0.1 -LocalPort 18789 -State Listen
```

Expected: task Running, new listener. The watchdog is **deliberately minimal**: it tests task state and local listener presence, not SSH end-to-end health. A stuck task that reports Running, or a different process occupying port 18789, will not be repaired by this script.

**E4 — Register a recurring S4U watchdog task:**

```powershell
$action = New-ScheduledTaskAction -Execute 'powershell.exe' -Argument (
    '-NoProfile -NonInteractive -ExecutionPolicy RemoteSigned -WindowStyle Hidden -File "' + $scriptPath + '"')
$trigger = New-ScheduledTaskTrigger -Once -At (Get-Date).AddMinutes(2) `
    -RepetitionInterval (New-TimeSpan -Minutes 2)
$principal = New-ScheduledTaskPrincipal -UserId $env:USERNAME -LogonType S4U -RunLevel Limited
$settings = New-ScheduledTaskSettingsSet -MultipleInstances IgnoreNew `
    -ExecutionTimeLimit (New-TimeSpan -Minutes 1)
Register-ScheduledTask -TaskName 'OpenClaw-Tunnel-Watchdog' `
    -Action $action -Trigger $trigger -Principal $principal -Settings $settings
```

**E5 — Verify scheduling:**

```powershell
Get-ScheduledTask -TaskName 'OpenClaw-Tunnel-Watchdog' |
    Select TaskName, State, @{N='LogonType';E={$_.Principal.LogonType}}
Get-ScheduledTaskInfo -TaskName 'OpenClaw-Tunnel-Watchdog' |
    Select LastRunTime, LastTaskResult, NextRunTime
```

Wait for the first scheduled execution. Expected `LastTaskResult = 0`, and `NextRunTime` advances approximately two minutes. An initial `LastTaskResult = 267011` / 1999 last-run placeholder means the task has not yet run; it is not proof of failure.

### Phase F — Verify recovery and clean up

**F1 — Record current listener PID and verify `ssh` ownership:**

```powershell
Get-NetTCPConnection -LocalAddress 127.0.0.1 -LocalPort 18789 -State Listen |
    Select LocalAddress, LocalPort, OwningProcess,
        @{N='ProcessName';E={(Get-Process -Id $_.OwningProcess).ProcessName}}
```

**F2 — Controlled failure test (temporary UI interruption):**

```powershell
$listener = Get-NetTCPConnection -LocalAddress 127.0.0.1 `
    -LocalPort 18789 -State Listen -ErrorAction SilentlyContinue
if ($listener -and
    (Get-Process -Id $listener.OwningProcess -ErrorAction SilentlyContinue).ProcessName -eq 'ssh') {
    Stop-Process -Id $listener.OwningProcess -Force
    'SSH tunnel process terminated for recovery test.'
} else {
    'No verified SSH process; nothing terminated.'
}
```

Wait at least three minutes **without manually restarting either task**. Then verify that a new listener PID exists, the tunnel task is Running, watchdog last result is `0`, and the Web UI loads with no console window. This test passed on 2026-10-22: SSH PID changed from `55156` to `59748`.

**F3 — Remove temporary test tasks only if present and stopped:**

```powershell
foreach ($name in 'OpenClaw-S4U-Test', 'OpenClaw-S4U-Tunnel-Test') {
    $task = Get-ScheduledTask -TaskName $name -ErrorAction SilentlyContinue
    if ($task -and $task.State -eq 'Ready') {
        Unregister-ScheduledTask -TaskName $name -Confirm:$false
    }
}
```

Final expected task list: `OpenClaw-SSH-Tunnel` (Running), `OpenClaw-Tunnel-Watchdog` (Ready between executions).

## 4. Rollback plan

### Rollback principles

- **Do not delete the private key or Ubuntu `authorized_keys` entry as a first response.** Restore service first; revoke credentials only if decommissioning or responding to compromise.
- **Disable the watchdog before stopping the tunnel**, otherwise it may bring the tunnel back within two minutes.
- Separate **temporary maintenance**, **remove automation but keep manual SSH**, and **full decommission**.
- Preserve task XML backups and record the pre-change state. Do not import an old task over a working one without reviewing its action, principal, and trigger.

### R1 — Temporary maintenance shutdown (reversible)

Run Windows PowerShell with appropriate task permissions:

```powershell
Disable-ScheduledTask -TaskName 'OpenClaw-Tunnel-Watchdog'
Stop-ScheduledTask -TaskName 'OpenClaw-SSH-Tunnel'
Get-ScheduledTask -TaskName 'OpenClaw-SSH-Tunnel','OpenClaw-Tunnel-Watchdog' |
    Select TaskName, State
Get-NetTCPConnection -LocalPort 18789 -State Listen -ErrorAction SilentlyContinue
```

Expected: no listener on 18789. The watchdog task may show `Disabled` rather than `Ready`. **The tunnel task's logon trigger remains enabled**: for maintenance across Windows sign-in, also disable `OpenClaw-SSH-Tunnel` until maintenance is complete.

**Resume:**

```powershell
Enable-ScheduledTask -TaskName 'OpenClaw-Tunnel-Watchdog'
Start-ScheduledTask -TaskName 'OpenClaw-SSH-Tunnel'
```

Verify listener, browser, and watchdog next run.

### R2 — Roll back only the watchdog

Use when periodic checks misbehave but the direct S4U tunnel is functional:

```powershell
Disable-ScheduledTask -TaskName 'OpenClaw-Tunnel-Watchdog'
Unregister-ScheduledTask -TaskName 'OpenClaw-Tunnel-Watchdog' -Confirm:$false
```

Leave `OpenClaw-SSH-Tunnel` intact. Note that automatic recovery is then **not assured**; use `Start-ScheduledTask` manually if the tunnel exits. Retain the script file until the change is accepted.

### R3 — Restore the prior tunnel task definition

Use **only if a known-good backup exists**. The recorded backup is:

`C:\Users\Cyb3r\AppData\Local\OpenClawLab\OpenClaw-SSH-Tunnel-Backup.xml`

First disable/stop the watchdog and tunnel as in R1. Then review the backup's contents, especially principal/logon type and action arguments:

```powershell
$backup = "$env:LOCALAPPDATA\OpenClawLab\OpenClaw-SSH-Tunnel-Backup.xml"
if (-not (Test-Path $backup)) { throw 'Backup missing; stop rollback.' }
[xml]$xml = Get-Content -Raw $backup
$xml.Task.Actions.Exec
$xml.Task.Principals.Principal
```

**Controlled restoration** (requires appropriate permissions):

```powershell
Unregister-ScheduledTask -TaskName 'OpenClaw-SSH-Tunnel' -Confirm:$false
Register-ScheduledTask -TaskName 'OpenClaw-SSH-Tunnel' -Xml (Get-Content -Raw $backup)
```

**Warning:** The original backed-up task used **Interactive** logon, which previously produced a blank console window. Restoring it deliberately reverts that behavior. Depending on the XML's account/security configuration, import may require adjustments or fail. Do not remove the current task unless the backup was reviewed and restoration is acceptable. If restoration fails, recreate the known-good S4U task using Phase D.

### R4 — Full decommission (irreversible credential removal)

1. Disable both tasks, then stop the tunnel.
2. Verify port 18789 is released.
3. Unregister `OpenClaw-Tunnel-Watchdog` and `OpenClaw-SSH-Tunnel`.
4. Archive the watchdog script and task XML backups if needed for audit; remove the script only after confirmation.
5. Remove **only the dedicated matching public key line** from Ubuntu `~/.ssh/authorized_keys` (match exact key or verified fingerprint). Preserve all other keys and file permissions.
6. Remove the dedicated key from Windows `ssh-agent` using `ssh-add -d <path>` if loaded. Delete the private/public key pair only after confirming it is not used elsewhere and backups/retention requirements are satisfied.
7. Do **not** disable the system SSH server or change the Ubuntu OpenClaw Gateway binding unless that is a separate approved change.

## 5. Troubleshooting and diagnostics

| Symptom | First checks | Action |
|---|---|---|
| Browser cannot open Web UI | `Get-NetTCPConnection` on Windows; task states; Ubuntu Gateway listener | Separate tunnel failure from Gateway failure |
| Tunnel task `Ready`, no listener | Watchdog last run/result; `Test-NetConnection ... -Port 22` | Manually start tunnel only for diagnosis; capture error first |
| Tunnel task `Running`, no listener | Task action PID; SSH arguments; local port conflict | Watchdog does **not** recover this condition automatically |
| SSH exits `255` | VM reachability, SSH host key, authentication, remote SSH logs | `255` is nonspecific; capture verbose SSH output safely |
| Forced termination event `4294967295` | Task Scheduler Operational log | This is unsigned `-1`; event 102 does not prove SSH success |
| Watchdog result `267011` | Trigger start time, next run | Task may not have run yet |
| Console window appears | Confirm S4U logon type and direct `ssh.exe` action | Interactive task and PowerShell hidden launcher showed windows in testing |
| Task does not restart on failure | Watchdog status and scheduling | Built-in 3 × 1-minute restart settings did **not** recover tested failure |
| VM address changes | `Test-NetConnection` and VM network configuration | Update the SSH target after confirming new IP; consider stable addressing |

**Windows diagnostics:**

```powershell
Get-ScheduledTask -TaskName 'OpenClaw-SSH-Tunnel','OpenClaw-Tunnel-Watchdog' |
    Select TaskName, State
Get-ScheduledTaskInfo -TaskName 'OpenClaw-Tunnel-Watchdog' |
    Select LastRunTime, LastTaskResult, NextRunTime
Get-WinEvent -FilterHashtable @{
    LogName='Microsoft-Windows-TaskScheduler/Operational'
    StartTime=(Get-Date).AddHours(-1)
} -ErrorAction SilentlyContinue |
    Where-Object { $_.Message -like '*OpenClaw*' } |
    Select TimeCreated, Id, Message
```

The Operational event log was originally **disabled**. It was enabled using an elevated terminal:

```powershell
wevtutil sl Microsoft-Windows-TaskScheduler/Operational /e:true
```

Enable it before testing failures; old events cannot be recovered retroactively.

**Ubuntu diagnostics:**

```bash
ss -lnt | grep ':18789'
systemctl status ssh --no-pager
journalctl -u ssh --since '30 minutes ago' --no-pager
```

OpenClaw service management varies by installation; use the lab's actual Gateway service command rather than assuming a service name.

## 6. Security notes and limitations

- The Windows and Ubuntu Gateway listeners are loopback-only; SSH is the transport across VMnet8.
- Use `IdentitiesOnly=yes`, `BatchMode=yes`, and `ExitOnForwardFailure=yes`. Never use `StrictHostKeyChecking=no` for convenience.
- Protect the Windows private key and Ubuntu `authorized_keys`. A noninteractive key deserves strict file ACLs and a scoped SSH account; evaluate authorized_keys restrictions separately before enforcing them.
- S4U is a **Windows task logon mechanism**, not SSH authentication. Key/agent behavior may differ by machine and passphrase policy.
- The watchdog checks task state and local port presence, **not** authenticated HTTP health or the identity of an arbitrary listener. A separate health probe is a potential future enhancement.
- The two-minute watchdog task is a periodic recovery mechanism, not a guaranteed two-minute service-level objective. Scheduler delays, power policies, sleep, network loss, or a hung `Running` task may prevent recovery.
- `DisallowStartIfOnBatteries=True` was observed on the tunnel task; test power behavior if using a laptop.
- The logon-trigger configuration and S4U operation after reboot/sign-in have **not** been verified.
- VMware VM reboot, prolonged network interruption, Windows reboot, sleep/wake, and key rotation are **not yet tested**.

## 7. Validation evidence from 2026-10-22

| Test | Observed outcome |
|---|---|
| Dedicated key-based SSH login | Remote marker `OPENCLAW_AGENT_OK` |
| Manual forwarding | Browser accessed Web UI via `127.0.0.1:18789` |
| S4U background tunnel | Worked without a console window |
| Controlled task stop | SSH process and listener released |
| Original built-in restart policy | **Did not recover** after forced SSH termination |
| Task Scheduler event logging | Enabled; task start events 325, 110, 100, 129, 200 captured |
| Forced SSH exit | Event 201 code `4294967295`; event 102 task finished |
| Manual watchdog invocation | Restored tunnel; SSH PID `55156` |
| Periodic watchdog execution | `LastTaskResult=0` |
| Automatic recovery after forced SSH termination | **Passed**; listener PID changed `55156` → `59748` |
| Recovered Web UI and invisible execution | Both confirmed by user |
| Temporary S4U task cleanup | Both test tasks removed |
| Final health check | Tunnel Running, watchdog Ready, listener PID `59748`, watchdog result `0` |

## 8. Post-deployment checklist

- [x] SSH authentication and forwarding verified
- [x] Gateway bound to loopback
- [x] S4U tunnel task verified invisible
- [x] Watchdog syntax and manual recovery verified
- [x] Watchdog periodic execution verified
- [x] Automatic recovery from forced SSH termination verified
- [x] Temporary test tasks removed
- [ ] Test Windows sign-in/reboot
- [ ] Test Ubuntu VM restart and IP stability
- [ ] Test sleep/wake and extended outage
- [ ] Confirm maintenance rollback works in a planned window
- [ ] Optionally add watchdog event logging / HTTP-level health checks

## 9. Change record

**2026-10-22:** Completed core S4U tunnel deployment and tested two-minute watchdog recovery. Removed two temporary S4U test tasks. Preserved original tunnel task XML backup. Created this runbook as a retrospective, streamlined procedure with tiered rollback options.
