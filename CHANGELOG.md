# Changelog

All notable changes to this project are documented in this file.

## [1.0.1] - 2025-11-12
- Add SELinux management options and tasks
 - Manage SELinux state and policy via `ansible.posix.selinux`
 - Manage SELinux booleans via `ansible.posix.seboolean`
 - Install SELinux tooling packages when SELinux management is enabled
 - Update README with SELinux usage and requirements

## [1.0.0] - 2025-11-10
- Initial release
 - Firewalld role managing zones, services, ports, rich rules, sources, interfaces
 - Support for masquerade, ICMP blocks, and port forwarding
 - README, examples, and Molecule scenario included

## Format
This changelog follows a simple, readable format. Future versions may adopt the
Keep a Changelog style and semantic versioning.
