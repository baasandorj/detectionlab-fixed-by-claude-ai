# About this fork

This is a maintained fork of [clong/DetectionLab](https://github.com/clong/DetectionLab), which has been unmaintained since 2023. As dependencies, package repositories, and vendor download endpoints have drifted since then, the original Vagrant-based provisioning no longer completes cleanly on current environments.

This fork fixes provisioning for **VMware Workstation Pro on Ubuntu 24.04**, verified working as of 2026. Fixes include:

- Installing VMware Tools on boxes that don't ship with it preinstalled
- Fixing the `C:\vagrant` shared folder mount on Windows guests
- Working around a Chocolatey/.NET 4.8 compatibility issue that silently broke osquery installation on Windows Server 2016 boxes
- Pinning Splunk to a version that predates a password-complexity policy that otherwise hangs provisioning indefinitely
- Working around Zeek's upstream package repository dropping Ubuntu 20.04 support

For the full technical details of each issue and fix, see [Vagrant/CLAUDE.md](Vagrant/CLAUDE.md).
