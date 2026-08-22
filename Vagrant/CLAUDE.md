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

9. logger_bootstrap.sh's install_zeek() points at the OpenSUSE Build
   Service repo `security:zeek/xUbuntu_20.04`, which no longer exists —
   that OBS project dropped Ubuntu 20.04 packaging entirely and now only
   publishes for xUbuntu_22.04/24.04/25.10/26.04 and Debian 12/13, but
   `logger` runs on the `bento/ubuntu-20.04` box. Retargeting the repo to
   xUbuntu_22.04 doesn't work either: that build of zeek-lts (8.0.10) is
   compiled against glibc 2.35, newer than 20.04's glibc 2.31, so
   `apt-get install zeek-lts` fails with "E: Unable to correct problems,
   you have held broken packages." Left unhandled, this cascades through
   the rest of install_zeek() (zkg, node.cfg, systemd) and hits the
   function's own `exit 1` when the zeek service fails to start, killing
   the whole `vagrant up` run. FIX APPLIED: after the apt-get install
   attempt, check for `/opt/zeek/bin/zeek-config`; if it's not present,
   log a warning and `return` early instead of continuing — Zeek is
   skipped on `logger` but the rest of provisioning completes. This is
   consistent with the tool installation failure policy below (Zeek is
   not one of the protected core components).

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