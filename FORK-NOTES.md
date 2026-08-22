# About this fork

This is a maintained fork of [clong/DetectionLab](https://github.com/clong/DetectionLab), which has been unmaintained since 2023. As dependencies, package repositories, and vendor download endpoints have drifted since then, the original Vagrant-based provisioning no longer completes cleanly on current environments.

This fork fixes provisioning for **VMware Workstation Pro on Ubuntu 24.04**, verified working as of 2026. Fixes include:

- Installing VMware Tools on boxes that don't ship with it preinstalled
- Fixing the `C:\vagrant` shared folder mount on Windows guests
- Working around a Chocolatey/.NET 4.8 compatibility issue that silently broke osquery installation on Windows Server 2016 boxes
- Pinning Splunk to a version that predates a password-complexity policy that otherwise hangs provisioning indefinitely
- Working around Zeek's upstream package repository dropping Ubuntu 20.04 support

For the full technical details of each issue and fix, see [Vagrant/CLAUDE.md](Vagrant/CLAUDE.md).

## Credit

DetectionLab was created by [Chris Long](https://github.com/clong). A sizable portion of the original box-building code was adapted from [Stefan Scherer](https://twitter.com/stefscherer)'s [packer-windows](https://github.com/StefanScherer/packer-windows) and [adfs2](https://github.com/StefanScherer/adfs2) repositories. This fork only addresses provisioning compatibility on a newer host setup (VMware Workstation Pro, Ubuntu 24.04) — the lab design and tooling choices are entirely the original authors' work.

Original project: <https://github.com/clong/DetectionLab>

## License

DetectionLab is [MIT-licensed](LICENSE). Forking, modifying, and republishing it — exactly what this repo does — is explicitly permitted under that license; the only real condition is preserving the original copyright and license notice. The original [LICENSE](LICENSE) file (Copyright (c) 2017 Chris Long) is unmodified in this repo.

## Safety note

This lab is intentionally insecure. It has not been hardened in any way and runs with default Vagrant credentials. Do not connect or bridge it to any network you care about — its purpose is visibility and introspection into each host, not a secure deployment.
