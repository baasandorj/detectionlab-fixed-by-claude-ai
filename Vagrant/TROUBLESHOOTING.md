# Reviving DetectionLab for VMware Workstation Pro: A Complete Debugging & AI-Automation Guide

**A companion technical reference to the LinkedIn article**
**Fork:** https://github.com/baasandorj/detectionlab-fixed-by-claude-ai

---

## Table of Contents

1. [Background & Goal](#background--goal)
2. [Attribution & Licensing](#attribution--licensing)
3. [Environment](#environment)
4. [Phase 1: Getting Vagrant + VMware Working At All](#phase-1-getting-vagrant--vmware-working-at-all)
5. [Phase 2: The VMware License Detection Bug](#phase-2-the-vmware-license-detection-bug)
6. [Phase 3: VMware Tools Missing on Every Windows Box](#phase-3-vmware-tools-missing-on-every-windows-box)
7. [Phase 4: The Shared Folder Mystery](#phase-4-the-shared-folder-mystery)
8. [Phase 5: The Chocolatey / .NET 4.8 Wall](#phase-5-the-chocolatey--net-48-wall)
9. [Phase 6: Handing Off to Claude Code (Autonomous Agent)](#phase-6-handing-off-to-claude-code-autonomous-agent)
10. [Phase 7: Issues the Agent Found On Its Own](#phase-7-issues-the-agent-found-on-its-own)
11. [The Full CLAUDE.md Knowledge Base](#the-full-claudemd-knowledge-base)
12. [Git/GitHub Setup for Agent-Driven Commits](#gitgithub-setup-for-agent-driven-commits)
13. [Key Prompts Used Throughout](#key-prompts-used-throughout)
14. [Lessons Learned](#lessons-learned)
15. [Quick-Reference Fix Sheet](#quick-reference-fix-sheet)
16. [Legal & Safety Notes](#legal--safety-notes)

---

## Background & Goal

[DetectionLab](https://github.com/clong/DetectionLab) is a well-known open-source project that builds a small Active Directory lab wired up for detection engineering practice: a domain controller, a Windows Event Forwarding collector, a Windows 10 workstation, and a Splunk logging server — all provisioned via Vagrant.

**The problem:** the project has been explicitly unmaintained since January 1, 2023. Its official documentation site (`detectionlab.network`) is also down. Meanwhile, the tools it depends on kept moving:
- VMware Workstation Pro changed its licensing model (became free for personal use, November 2024)
- Chocolatey (Windows package manager) moved to requiring .NET Framework 4.8
- Splunk tightened default password policies
- Various upstream Linux packages (Zeek) dropped support for older Ubuntu releases

None of this was anticipated by a codebase frozen in 2023. The goal: get all four VMs (`logger`, `dc`, `wef`, `win10`) fully provisioned on **Ubuntu 24.04 + VMware Workstation Pro**, and document every fix so nobody has to repeat the archaeology.

---

## Attribution & Licensing

**Original author:** [Chris Long](https://github.com/clong) (GitHub: `clong`) created and maintained DetectionLab. Per the original README, the project's primary purpose is to let a user quickly build a Windows domain preloaded with security tooling and logging best practices, and Chris personally funded its infrastructure, building, and testing.

**Additional credit:** the original project's README notes that a sizable portion of its code was borrowed and adapted from [Stefan Scherer's](https://github.com/StefanScherer) `packer-windows` and `adfs2` repositories — the foundation much of the box-building process is built on.

**License:** DetectionLab is released under an **MIT-style permissive license** — one of the most open licenses available. In plain terms:
- You are explicitly permitted to fork, clone, modify, and redistribute the project, including publishing your own modified fork publicly (as this guide does).
- The only real obligation is to **preserve the original copyright and license notice** in redistributed copies.
- The license includes the standard "as-is" disclaimer — no warranty, no liability on the original author for anything that breaks (and by extension, none expected of anyone redistributing it either).

**In short:** everything described in this guide — forking the repo, modifying scripts, publishing a fixed version publicly — is fully permitted under the original license. No permission was needed from the original author to do any of this, though credit is both a courtesy and, in spirit, the one condition the license actually asks for.

If you use or extend this fork, please keep the original `LICENSE` file intact and retain credit to Chris Long and Stefan Scherer alongside whatever additional attribution you add for your own changes.

---

## Environment

- **Host OS:** Ubuntu 24.04 (Dell Precision 7750)
- **Hypervisor:** VMware Workstation Pro (free personal-use license, installed fresh)
- **Provisioning:** Vagrant 2.4.9 + `vagrant-vmware-desktop` plugin 3.0.5 + `vagrant-vmware-utility` 1.0.24
- **Target VMs:** `logger` (Ubuntu 20.04), `dc` (Windows Server 2016), `wef` (Windows Server 2016), `win10` (Windows 10 Enterprise Evaluation)

---

## Phase 1: Getting Vagrant + VMware Working At All

### Installing Vagrant
Standard HashiCorp apt repo install. Hit an unrelated but blocking issue immediately:

```
Setting up linux-modules-nvidia-580-7.0.0-30-generic ...
dpkg: error processing package linux-modules-nvidia-580-7.0.0-30-generic
```

**Root cause:** stale/incomplete kernel headers for the currently-installed kernel, unrelated to Vagrant — it just surfaced because `apt install vagrant` triggered a `dpkg --configure -a` pass that exposed pre-existing broken packages.

**Fix:**
```bash
sudo apt install --reinstall linux-headers-7.0.0-30-generic
sudo apt --fix-broken install
```

This in turn surfaced a **second** broken package: `virtualbox-dkms` failing to build against the current kernel. Since VMware (not VirtualBox) was the target provider, the pragmatic fix was to remove VirtualBox entirely:

```bash
sudo apt remove virtualbox-dkms virtualbox
sudo apt --fix-broken install
```

### Installing the VMware provider plugin

```bash
vagrant plugin install vagrant-vmware-desktop
```

This plugin requires a separate binary utility, not installed via the plugin system:

```bash
sudo apt install vagrant-vmware-utility
```

**Gotcha:** the download filename convention changed between versions — `vagrant-vmware-utility_1.0.24_linux_amd64.deb` (guessed/older pattern) 404'd. Using the apt repo (already configured for Vagrant itself) sidestepped needing to guess the exact filename.

### Installing VMware Workstation Pro itself

As of November 2024, VMware Workstation Pro is **free for personal use** — no license key required, just select "Personal Use" during the license-key prompt on first launch. Downloaded from Broadcom's support portal (requires a free Broadcom account).

---

## Phase 2: The VMware License Detection Bug

### Symptom
```
vagrant-vmware-utility[...]: WARN vagrant-vmware-utility.api: failed to determine license edition
vagrant-vmware-utility[...]: WARN vagrant-vmware-utility.api.vmrest: standard vmware license detected, using fallback
```

Then, on first `vagrant up --provider=vmware_desktop`:
```
Command: ["-T", "player", "snapshot", ...]
Stdout: Error: The operation is not supported
```

### Root cause
Since VMware Workstation Pro's licensing changed to free personal-use, `vagrant-vmware-utility` can no longer correctly detect it as the "professional/ws" edition. It falls back to treating it as VMware Player, and **VMware Player mode doesn't support snapshot operations** — which `vagrant-vmware-desktop` relies on internally for cloning boxes.

### Failed approach
Wrapping the `vmrun` binary itself to force `-T ws` on every call. This actually broke things worse — `vmrun` (and many VMware CLI tools) use their own invocation path/name internally to locate sibling binaries like `vmware-vmx`; renaming/wrapping it broke that lookup, producing a new error: `Fail to open executable: No such file or directory`.

### Actual fix
Force the license type explicitly in the Vagrantfile, per VM provider block:

```ruby
cfg.vm.provider "vmware_desktop" do |v|
  v.force_vmware_license = "professional"
  # ... rest of config
end
```

This must be added to **every** `vmware_desktop` provider block in the Vagrantfile (all four VMs), not just one.

---

## Phase 3: VMware Tools Missing on Every Windows Box

### Symptom
```
[fix-second-network.ps1] VMware Tools not found, no need to continue. Exiting.
```
Followed later by shell provisioners failing with:
```
The term 'c:\vagrant\scripts\fix-windows-expiration.ps1' is not recognized...
```

### Root cause
The prebuilt Vagrant boxes (`detectionlab/win2016`, `detectionlab/win10`) — built years ago and never updated — do not ship with VMware Tools installed. Without Tools, there's no HGFS (shared folder) driver, so `C:\vagrant` never mounts, and every script that lives there is invisible to the guest.

### Fix sequence (repeated per VM)

**1. Trigger the Tools install:**
```bash
vmrun -T ws installTools /path/to/machine.vmx
```
(Often returns immediately with no visible output — it's working in the background. A second call while it's running returns `Error: A VMware Tools installation is already in progress`, which is actually confirmation it's working, not a failure.)

**2. Confirm the ISO mounted and find the installer:**
```bash
vagrant winrm <vm> -c 'Get-ChildItem D:\'
```
Look for `setup.exe` (not always `setup64.exe` — verify the actual filename each time).

**3. Run the silent install directly** (autorun often doesn't trigger automatically in a headless WinRM session):
```bash
vagrant winrm <vm> -c 'Start-Process "D:\setup.exe" -ArgumentList "/S /v /qn REBOOT=R" -Wait'
```

**4. Verify:**
```bash
vagrant winrm <vm> -c 'Get-Service | Where-Object {$_.DisplayName -like "*VMware*"}'
```
Look for `VMTools` service in `Running` state.

---

## Phase 4: The Shared Folder Mystery

Even after VMware Tools installed successfully, `C:\vagrant` still didn't exist on some VMs.

### Diagnostic path
```bash
vagrant winrm <vm> -c 'Test-Path "\\vmware-host\Shared Folders\vagrant"'
```
This returned `True` — the underlying share was reachable — but `C:\vagrant` itself didn't exist as a path.

### First attempted fix (partially wrong)
```bash
vagrant winrm <vm> -c 'net use Z: "\\vmware-host\Shared Folders\vagrant" /persistent:no'
vagrant winrm <vm> -c 'cmd /c mklink /d C:\vagrant Z:\'
```
This worked *within that one WinRM session* — but **drive letter mappings created via `net use` do not persist across separate WinRM sessions**. Each `vagrant winrm` invocation is effectively a new session, so the symlink pointing at `Z:\` became a dangling link the moment the mapping session ended.

### Correct, durable fix
Point the symlink directly at the UNC path — no drive letter, no session-dependent state:
```bash
vagrant winrm <vm> -c 'cmd /c mklink /d C:\vagrant "\\vmware-host\Shared Folders\vagrant"'
```

**Note on removing a broken symlink:** PowerShell's `Remove-Item` sometimes fails on a symlink with a mismatched reparse point (`There is a mismatch between the tag specified in the request and the tag present in the reparse point`). Use `cmd`'s `rmdir` instead, which handles it cleanly:
```bash
vagrant winrm <vm> -c 'cmd /c rmdir C:\vagrant'
```

### A second, separate root cause found later
On one VM (`wef`), even after the UNC-symlink fix, the share was still unreachable. Direct inspection of the `.vmx` file revealed the actual folder definition had a malformed guest name:

```
sharedFolder0.guestName = "-vagrant"     # WRONG — leading dash, invalid
```

This was invisible in Windows Explorer and in any log output — only visible by grepping the raw VMX config:
```bash
grep -i "sharedfolder" /path/to/machine.vmx
```

**Fix:** add a corrected shared folder definition via `vmrun` (rather than hand-editing the VMX, which risks syntax errors):
```bash
vmrun -T ws addSharedFolder /path/to/machine.vmx vagrant /host/path/to/DetectionLab
```
This appends a second, correctly-named `sharedFolder1` entry alongside the broken one — the broken one can be left in place, it's simply ignored once a working entry exists.

---

## Phase 5: The Chocolatey / .NET 4.8 Wall

This was the single most time-consuming issue.

### Symptom
On the Windows Server 2016 boxes, `install-utilities.ps1` (which bootstraps Chocolatey) got stuck in an infinite retry loop:
```
WARNING: Try #1 of .NET framework install failed with exit code '87'. Trying again.
The registry key for .Net 4.8 was not found or this is forced
Installing 'C:\Users\vagrant\AppData\Local\Temp\ndp48-x86-x64-allos-enu.exe' ...
[repeats indefinitely]
```

### Root cause investigation
1. Confirmed the box only has **.NET Framework 4.6.1** installed (`Release 394802`), not 4.8.
2. Downloaded and ran the .NET 4.8 installer manually (bypassing Chocolatey's wrapper) to isolate the real error. Needed to force TLS 1.2 first, since the old .NET stack doesn't default to it:
   ```powershell
   [Net.ServicePointManager]::SecurityProtocol = [Net.SecurityProtocolType]::Tls12
   Invoke-WebRequest -Uri "<installer-url>" -OutFile "C:\ndp48.exe"
   ```
3. Ran with logging enabled, exit code was consistently **87** (invalid parameter).
4. Read the installer's own HTML result log (`%TEMP%\Microsoft .NET Framework 4.8 Setup_*.html`) — found the real failure deep inside:
   ```
   Launching CreateProcess with command line = dism.exe /quiet /norestart /online 
   /add-package /packagepath:"...x64-Windows10.0-KB4486129-x64.cab"
   Exe failed with 0x57 - The parameter is incorrect.
   ```
5. Ran `DISM /Online /Cleanup-Image /RestoreHealth` to rule out servicing-store corruption — completed cleanly with no corruption found, and the .NET installer **still failed identically**.

**Conclusion:** this specific KB cab package is genuinely incompatible with this exact build of Windows Server 2016 (`10.0.14393.0`, RTM base, no updates applied) — not a fixable local corruption issue.

### The actual fix
Researched what Chocolatey itself actually requires. Found that **Chocolatey CLI 2.0.0+ hard-requires .NET Framework 4.8** — it's a real gate, not a bypassable check. But **Chocolatey 1.x does not have this requirement** and works fine on .NET 4.5+.

Pinned Chocolatey to version 1.4.0, installed manually *before* the provisioning scripts run, so they detect it as already present and skip their own bootstrap attempt:

```powershell
$env:chocolateyVersion = "1.4.0"
[Net.ServicePointManager]::SecurityProtocol = [Net.SecurityProtocolType]::Tls12
iex ((New-Object System.Net.WebClient).DownloadString("https://community.chocolatey.org/install.ps1"))
```

This completely avoids the .NET 4.8 dependency chain. Once done manually on the first affected box, the fix was later baked into `install-utilities.ps1` / `install-choco-extras.ps1` directly so future runs don't need manual intervention.

---

## Phase 6: Handing Off to Claude Code (Autonomous Agent)

Once the recurring pattern was clear — the same handful of issues would resurface on every Windows box — it made sense to stop manually relaying commands and let an agent run the loop itself.

### Setup
1. Opened the DetectionLab `Vagrant/` folder as a workspace in VS Code with Claude Code.
2. Created a `CLAUDE.md` file in the project root — Claude Code reads this automatically at the start of a session. This is the single highest-leverage step: it meant the agent didn't have to rediscover any of the above from scratch.
3. Gave a clear top-level goal: get all four VMs to print their "Provisioning Complete!" message.

### Why this mattered
Without `CLAUDE.md`, an agent re-running `vagrant up` from scratch would very likely hit the exact same walls (VMware Tools missing, shared folder bug, Chocolatey/.NET wall) and have to independently diagnose each one — potentially taking just as long as the manual session did. With it, the agent could apply known fixes immediately and spend its effort on genuinely new problems instead.

---

## Phase 7: Issues the Agent Found On Its Own

This is the most interesting part of the exercise: once equipped with the known-issues context, the agent went on to independently diagnose **two new problems** that the manual session never encountered.

### 1. The silent Splunk password-policy hang

**Symptom:** `logger` (the Splunk indexer VM) sat silent for over 5 hours, with `vmware-vmx` still consuming CPU but no new provisioning log output — easy to mistake for "just a slow install."

**Root cause:** `logger_bootstrap.sh` auto-resolves and installs the *latest* Splunk Enterprise release (10.4.2 at the time), then seeds it with:
```bash
splunk start --accept-license --answer-yes --no-prompt --seed-passwd changeme
```
Splunk 9.x+ enforces mandatory admin password complexity and **rejects "changeme"** as too weak. The seed step fails validation, and Splunk falls back to an *interactive* password re-prompt — which has no terminal to answer over a Vagrant/SSH provisioning session. It just hangs forever, silently.

**Fix:** pinned Splunk to version 8.2.12 (predates the stricter policy) in `logger_bootstrap.sh`'s `install_splunk()` function, rather than auto-resolving "latest." (Note: the separate Windows Universal Forwarder script already pinned an older version and was unaffected.)

### 2. Zeek packaging dropped for Ubuntu 20.04

**Symptom:** a silent failure partway through `logger`'s bootstrap script.

**Root cause:** Zeek's official upstream packaging no longer publishes builds for Ubuntu 20.04 — the `apt install zeek` step fails with a package-not-found error.

**Fix:** made the Zeek install step non-fatal (continue past the failure) rather than aborting the whole bootstrap script, since Zeek isn't required for the core DetectionLab detection pipeline (Sysmon/osquery/Splunk/WEF all function independently of it).

---

## The Full CLAUDE.md Knowledge Base

This is the actual, final content of the project's `CLAUDE.md` — reproduced in full since it's the most reusable artifact from this whole exercise.

```markdown
# DetectionLab on VMware Workstation Pro (Ubuntu 24.04) — Known Issues

This repo (clong/DetectionLab) is unmaintained since 2023. When running
`vagrant up`/`vagrant provision` with the vmware_desktop provider, expect:

1. VMware Tools are NOT preinstalled on these boxes. After a VM boots,
   check `Get-Service | Where DisplayName -like "*VMware*"` via
   `vagrant winrm <name> -c '...'`. If missing, run:
   `vmrun -T ws installTools <path-to-vmx>`, then find the mounted CD
   drive letter and run `D:\setup.exe /S /v /qn REBOOT=R` inside the guest.

2. Shared folders often fail to mount at C:\vagrant even after enabling.
   Fix: `vmrun -T ws enableSharedFolders <vmx>` then
   `vmrun -T ws addSharedFolder <vmx> vagrant <host-path>`, then inside
   guest: `cmd /c mklink /d C:\vagrant "\\vmware-host\Shared Folders\vagrant"`
   (use the UNC path directly, NOT a mapped drive letter — drive mappings
   don't persist across separate WinRM sessions).

3. Windows Server 2016 boxes only have .NET 4.6.1. Chocolatey 2.x+ REQUIRES
   .NET 4.8, and the auto-installer for .NET 4.8 fails with exit code 87
   on these boxes (DISM package error, unresolved even after
   DISM /RestoreHealth). WORKAROUND: pre-install Chocolatey 1.4.0
   manually before running install-utilities.ps1:
   `$env:chocolateyVersion = "1.4.0"; iex ((New-Object
   System.Net.WebClient).DownloadString("https://community.chocolatey.org/install.ps1"))`

4. Wireshark's silent installer sometimes fails with exit code 2 on
   Server 2016 boxes (Npcap driver consent). Non-blocking; skip/ignore.

5. VMware license detection bug: `vagrant-vmware-utility` may misreport
   the free "Personal Use" license as "standard/player" mode, breaking
   snapshot operations. Fix in Vagrantfile per-provider block:
   `v.force_vmware_license = "professional"`

6. logger_bootstrap.sh auto-resolves and installs the "latest" Splunk
   Enterprise (currently 10.4.2), then runs
   `splunk start --accept-license --answer-yes --no-prompt --seed-passwd changeme`.
   Splunk 9.x+ enforces a mandatory admin password complexity policy that
   rejects weak/common passwords, including "changeme" — the seed fails
   validation and Splunk falls back to an interactive re-prompt that has
   no TTY to answer it over vagrant/SSH provisioning, hanging forever
   (observed: 5.5+ hours with vmware-vmx still consuming CPU but the log
   silent). FIX APPLIED: pinned to Splunk 8.2.12 (predates the policy)
   in logger_bootstrap.sh's install_splunk() instead of auto-resolving
   latest. If this repo is updated to properly generate a complex seed
   password instead, the pin can be reverted. The Windows Universal
   Forwarder script (scripts/install-splunkuf.ps1) already pins 8.2.2.1
   and is not affected.

7. Network Location popup ("Do you want to allow your PC to be discoverable...")
   blocks headless/WinRM sessions since it needs GUI interaction. Fix:
   `New-Item -Path "HKLM:\SYSTEM\CurrentControlSet\Control\Network\NewNetworkWindowOff" -Force`
   to suppress it going forward, and
   `Get-NetConnectionProfile | Set-NetConnectionProfile -NetworkCategory Private`
   to resolve one already stuck.

8. General pattern: if a provisioning step goes silent for an extended
   period (10+ min) with vmware-vmx CPU still active but no new log
   output, suspect an interactive prompt with no TTY to answer it
   (password policy rejection, EULA dialog, Windows Update restart
   confirmation, etc.) rather than a slow-but-normal install. Check
   `Get-Process` in the guest for a process waiting idle, not actively
   computing, before assuming it's just "taking a while."

9. logger_bootstrap.sh attempts to install Zeek via apt, but Zeek's
   official packaging for Ubuntu 20.04 has been dropped upstream —
   the install step fails with a package-not-found error. FIX APPLIED:
   made the Zeek install step non-fatal (continue past failure) in
   logger_bootstrap.sh, since Zeek isn't a hard requirement for the
   core DetectionLab pipeline (Sysmon/osquery/Splunk/WEF still function
   without it). If Zeek support matters to you, it would need to be
   installed via source build or a different package source.

## Tool installation failure policy

When any tool/package install fails (Chocolatey, MSI, silent installer, etc.):

1. Retry once with the SAME version — transient network/download issues are common.
2. If it fails again, try an OLDER version of that specific tool
   (e.g., Chocolatey 1.4.0 instead of 2.7.4 — see .NET 4.8 issue above for
   the known example). Check the tool's package repo for available older
   versions before giving up.
3. If an older version also fails, or no older version is a viable fix,
   SKIP that tool and continue provisioning. Do not block the rest of the
   VM's setup on one non-critical utility.
4. When skipping, log clearly what was skipped and why (e.g., in the
   commit message or a running NOTES.md) so it's not silently lost.
5. This applies to convenience/utility tools (Notepad++, WinRAR, Wireshark,
   Process Hacker, etc.) — NOT to core DetectionLab components (Sysmon,
   osquery, Splunk UF, Velociraptor, WEF subscriptions, AD DS). If a core
   component fails after steps 1-2, STOP and report the failure — don't
   silently skip something the lab depends on.

Goal: get all 4 VMs (logger, dc, wef, win10) to print their
"Provisioning Complete!" messages.

## Git/GitHub workflow

- This repo's changes should be committed and pushed to:
  https://github.com/baasandorj/detectionlab-fixed-by-claude-ai
- Commit meaningful, working checkpoints only — not mid-fix broken state
- Never commit .vagrant/ or any VM state directories
- Write clear commit messages describing what was fixed and why
- Ask before force-pushing or rewriting history
```

---

## Git/GitHub Setup for Agent-Driven Commits

To let Claude Code commit and push on your behalf:

**1. Git identity** (one-time):
```bash
git config --global user.name "Your Name"
git config --global user.email "you@example.com"
```

**2. Authentication** — SSH key (recommended if you already have one set up):
```bash
ssh-add -l                      # confirm key is loaded
ssh -T git@github.com           # confirm GitHub recognizes it
```
Expected success response: `Hi <username>! You've successfully authenticated, but GitHub does not provide shell access.` (exit code 1 here is normal SSH behavior, not an error.)

**3. Repo setup** — new repo (not a GitHub-native fork), since the goal was a clean home for the fixes rather than contributing back to an abandoned upstream:
```bash
git remote rename origin upstream
git remote add origin git@github.com:<you>/<new-repo-name>.git
```
This preserves access to the original project as `upstream` while making your own repo the default push target.

**4. CLAUDE.md workflow instructions** — tell the agent explicitly what "done" looks like:
```markdown
## Git/GitHub workflow
- Commit meaningful, working checkpoints only — not mid-fix broken state
- Never commit .vagrant/ or any VM state directories
- Write clear commit messages describing what was fixed and why
- Ask before force-pushing or rewriting history
```

---

## Key Prompts Used Throughout

For anyone wanting to replicate this workflow, here are the actual prompts that drove the agentic phase:

**Kicking off the autonomous run:**
> "Read CLAUDE.md for known issues. Run `vagrant destroy -f && vagrant up --provider=vmware_desktop`, then work through provisioning each VM (logger, dc, wef, win10) using the documented fixes whenever those errors appear. Keep retrying until each shows its 'Provisioning Complete!' message. Ask me before running anything destructive I haven't already approved."

**Adding a newly-discovered fix to the knowledge base:**
> "Please add the Zeek non-fatal install fix as a new numbered item in CLAUDE.md, following the same format as the other entries, before you commit."

**Setting up git safely (separate terminal, non-disruptive):**
> "In a separate terminal, without interrupting the current vagrant provisioning, set up the git remote to point at github.com/\<you\>/\<repo\> and confirm the empty repo exists first."

**Final commit with review step:**
> "Show me git status and git diff --stat first, then commit all the changes ([file list]) with a clear commit message describing all the fixes, and push to origin."

**README update:**
> "Edit the main README.md to add two things near the top: (1) a link to FORK-NOTES.md, (2) a note that the original documentation site is down, with a Wayback Machine archive link for historical deployment guidance (Azure/AWS/VirtualBox instructions, etc.). Show me the diff before committing, then commit and push."

---

## Lessons Learned

1. **"Unmaintained" compounds.** A frozen codebase doesn't fail all at once — it fails at every point where an external dependency (a license model, a package manager's minimum requirements, an application's default security policy) moved on without it. Each fix here was independently necessary; skipping any one of them blocks the whole pipeline downstream.

2. **Silence isn't always progress.** The single most expensive lesson: `vmware-vmx` burning CPU with no new log output looks identical whether a process is doing real work or stuck on an interactive prompt with no way to answer it. The fix isn't "wait longer" — it's checking actual guest process/service state.

3. **The fastest fix isn't always the "correct" one.** Rather than solving the .NET 4.8 / DISM incompatibility directly (which may not be solvable at all on this specific OS build), the actual fix was sidestepping the dependency entirely by using an older tool version. Recognizing when to route around a problem versus solve it head-on saved significant time.

4. **Context transfer between a diagnostic conversation and an autonomous agent is the real force-multiplier.** The manual debugging session was valuable on its own, but its output — a structured `CLAUDE.md` — is what let the agent skip re-deriving known fixes and spend its effort on genuinely new problems (the Splunk and Zeek issues) instead.

5. **Documentation-as-you-go beats documentation-after.** Every fix was added to `CLAUDE.md` in the same commit as the code change that implemented it, in the same consistent format (symptom → root cause → why the obvious fix failed → actual fix). This is what makes the resulting fork genuinely reusable rather than just "working, once, for me."

---

## Quick-Reference Fix Sheet

| Issue | Fix |
|---|---|
| VMware snapshot ops fail with "operation not supported" | `v.force_vmware_license = "professional"` in Vagrantfile per-provider block |
| VMware Tools missing on guest | `vmrun -T ws installTools <vmx>`, then manually run `D:\setup.exe /S /v /qn REBOOT=R` inside guest |
| `C:\vagrant` won't mount | `vmrun -T ws enableSharedFolders/addSharedFolder <vmx>`, then `mklink /d C:\vagrant "\\vmware-host\Shared Folders\vagrant"` (UNC path, not a drive letter) |
| Shared folder unreachable despite config existing | Check for malformed `guestName` in the raw `.vmx` file (e.g., a stray leading dash) |
| Chocolatey/.NET 4.8 install loop (exit code 87) | Pin Chocolatey to 1.4.0: `$env:chocolateyVersion = "1.4.0"` before install |
| Wireshark silent install fails (exit code 2) | Non-blocking — skip, it's a convenience tool not a core dependency |
| Splunk install hangs for hours, no error | Pin Splunk to an older version (e.g., 8.2.12) that predates mandatory password-complexity enforcement |
| Zeek install fails on Ubuntu 20.04 | Make the install step non-fatal; Zeek isn't required for core pipeline |
| Network Location popup blocks headless provisioning | `New-Item -Path "HKLM:\SYSTEM\CurrentControlSet\Control\Network\NewNetworkWindowOff" -Force` |

---

## Legal & Safety Notes

- **Licensing:** DetectionLab is MIT-licensed (see [Attribution & Licensing](#attribution--licensing) above). Forking, modifying, and republishing it — exactly what this guide walks through — is explicitly permitted, with the sole real condition being preservation of the original copyright/license notice.
- **This is a deliberately insecure lab environment.** The original project's README is explicit about this: it runs with default Vagrant credentials and has not been hardened in any way. Its entire purpose is visibility and introspection into each host for detection-engineering practice — not production-grade security. **Do not connect or bridge this lab to any network you care about**, and treat every credential in it as public knowledge.
- **Third-party tool licensing still applies independently.** This guide covers *DetectionLab's* license, not the licenses of the tools it installs (VMware Workstation Pro, Splunk, Windows Server/10 evaluation editions, Chocolatey, etc.). Each of those has its own terms — for example, the Windows evaluation editions used here are time-limited and intended for evaluation/lab use, not production deployment.
- **No warranty, from anyone, at any layer.** The original project disclaims warranty under MIT terms; this fork and this guide extend that same disclaimer. Everything here is provided as-is, based on one specific point-in-time environment.

---

*This guide and the referenced fork are provided as-is, based on one specific environment (Ubuntu 24.04, VMware Workstation Pro, August 2026). Your mileage may vary depending on exact package versions available at the time you read this — that's precisely the nature of the problem this whole exercise was about.*
