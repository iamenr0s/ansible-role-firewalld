[![Molecule](https://github.com/iamenr0s/ansible-role-firewalld/actions/workflows/molecule.yml/badge.svg)](https://github.com/iamenr0s/ansible-role-firewalld/actions/workflows/molecule.yml) [![Release](https://github.com/iamenr0s/ansible-role-firewalld/actions/workflows/release.yml/badge.svg)](https://github.com/iamenr0s/ansible-role-firewalld/actions/workflows/release.yml) ![Ansible Role](https://img.shields.io/ansible/role/d/iamenr0s/ansible_role_cri_o) [![CodeFactor](https://www.codefactor.io/repository/github/iamenr0s/ansible-role-firewalld/badge)](https://www.codefactor.io/repository/github/iamenr0s/ansible-role-firewalld)

# Ansible Role: Firewalld

Manage `firewalld` installation and service state, with optional kernel parameters/modules and package management integration.

## Features

- Ensures required packages for `firewalld` are present
- Optionally ensures kernel modules and sysctl settings needed by firewalld
- Starts and enables the `firewalld` service
- Integrates with external roles for package management and kernel configuration
- Manages zones and reloads after zone creation when configured
- Applies services, ports (single/range), and rich rules per zone
- Manages sources and interfaces per zone
- Configures masquerade, ICMP blocks, and port forwarding

## Requirements

- Ansible 2.9 or higher
- External roles (installed via `requirements.yml`):
  - `ansible-role-pkg-management` (Git)
  - `ansible-role-kernel-configuration` (Git)

Install with:

```
ansible-galaxy install -r requirements.yml
```

## Supported Platforms

- Ubuntu (20.04, 22.04+)
- Debian (11, 12+)
- EL/RHEL/Rocky/Alma (8, 9, 10)
- Fedora (39+)

## Role Variables

- `firewalld_manage_kernel` (bool): Whether to manage kernel modules/sysctl needed by firewalld. Default: `true`.
- `firewalld_required_kernel_modules` (list): Modules to ensure loaded. Default: `[nf_conntrack, nf_tables]`.
- `firewalld_kernel_sysctl` (dict): Sysctl to apply (key/value). Default: `{}`.
- `firewalld_service_name` (string): Service name. Default: `firewalld`.
- `firewalld_required_packages_map` (dict): Per OS-family package mapping. Default includes `firewalld` for Debian/Ubuntu/RedHat/Rocky/Fedora.
- `firewalld_required_packages` (list): Resolved packages for the current host using `ansible_os_family`. Default: `['firewalld']` when no mapping exists.
- `firewalld_zones_present` (list): Zones to create permanently. Default: `[]`.
- `firewalld_reload_after_zone_changes` (bool): Reload service after zone creation if changed. Default: `true`.
- `firewalld_services` (list of dict): Services per zone.
  - Keys: `name`, `zone` (default `public`), `state` (`enabled`/`disabled`, default `enabled`), `permanent` (default `true`), `immediate` (default `false`).
- `firewalld_ports` (list of dict): Ports per zone.
  - Keys: `port` (e.g., `8080/tcp` or range `161-162/udp`), `zone` (default `public`), `state`, `permanent`, `immediate`.
- `firewalld_rich_rules` (list of dict): Rich rules per zone.
  - Keys: `rich_rule` (full rule string), `zone` (default `public`), `state`, `permanent`, `immediate`.
- `firewalld_sources` (list of dict): Source CIDRs per zone.
  - Keys: `source`, `zone`, `state` (default `enabled`), `permanent` (default `true`).
- `firewalld_interfaces` (list of dict): Interfaces per zone.
  - Keys: `interface`, `zone`, `state` (default `enabled`), `permanent` (default `true`).
- `firewalld_masquerade` (list of dict): Masquerade per zone.
  - Keys: `zone`, `state` (default `enabled`), `permanent` (default `true`).
- `firewalld_icmp_blocks` (list of dict): ICMP blocks per zone.
  - Keys: `name` (e.g., `echo-request`), `zone`, `state` (default `enabled`), `permanent` (default `true`).
- `firewalld_forward_ports` (list of dict): Port forward rules per zone.
  - Keys: `zone` (default `public`), `state` (default `enabled`), `permanent` (default `true`), `immediate` (default `false`), `port` (source port), `proto` (`tcp`/`udp`), `toport` (dest port), `toaddr` (optional dest address).

Example variables snippet:

```
firewalld_zones_present:
  - public

firewalld_services:
  - { name: ssh, zone: public }

firewalld_ports:
  - { port: "8080/tcp", zone: public }

firewalld_rich_rules:
  - { rich_rule: "rule family=ipv4 source address=10.0.0.0/8 accept", zone: public }

firewalld_masquerade:
  - { zone: public }

firewalld_forward_ports:
  - { zone: public, port: "8080", proto: "tcp", toport: "80", toaddr: "10.0.0.10" }
```

## Dependencies

Defined in `requirements.yml`:

- `ansible-role-pkg-management` from Git: `git@github.com:iamenr0s/ansible-role-pkg-management.git`
- `ansible-role-kernel-configuration` from Git: `git@github.com:iamenr0s/ansible-role-kernel-configuration.git`

Collections:

- `ansible.posix` (used for `ansible.posix.firewalld` and `ansible.posix.firewalld_info`). Install with: `ansible-galaxy collection install ansible.posix` if not present.

## Example Playbook

```
- hosts: all
  become: true
  roles:
    - role: iamenr0s.ansible_role_firewalld
      vars:
        firewalld_manage_kernel: true
        firewalld_required_kernel_modules:
          - nf_conntrack
          - nf_tables
        firewalld_kernel_sysctl:
          net.ipv4.ip_forward: 0

        # Zones to create permanently
        firewalld_zones_present:
          - public
          - dmz

        # Services
        firewalld_services:
          - { name: ssh, zone: public }
          - { name: http, zone: dmz }

        # Ports
        firewalld_ports:
          - { port: "8080/tcp", zone: public }
          - { port: "161-162/udp", zone: dmz, state: enabled }

        # Rich rules
        firewalld_rich_rules:
          - { rich_rule: "rule family=ipv4 source address=10.0.0.0/8 accept", zone: public }

        # Sources and interfaces
        firewalld_sources:
          - { source: "192.0.2.0/24", zone: dmz }
        firewalld_interfaces:
          - { interface: "eth0", zone: public }

        # Masquerade and ICMP
        firewalld_masquerade:
          - { zone: public }
        firewalld_icmp_blocks:
          - { name: echo-request, zone: public, state: disabled }

        # Port forwarding
        firewalld_forward_ports:
          - { zone: public, port: "8080", proto: "tcp", toport: "80", toaddr: "10.0.0.10" }
```

## Troubleshooting

- Ensure external roles are installed via `ansible-galaxy install -r requirements.yml`.
- If the kernel role uses different variable names, map them in `tasks/main.yml` vars.
- When creating zones, permanent operations are required; reloads are handled via `firewalld_reload_after_zone_changes`.
- Port forwarding accepts only one forward entry per operation; this role loops one-by-one.
- Immediate changes on newly created zones require a reload first; the role handles zone reloads when enabled.

## Installation

- Install role dependencies: `ansible-galaxy install -r requirements.yml`
- Ensure `ansible.posix` collection is available: `ansible-galaxy collection install ansible.posix`

## Molecule

- Run locally:
  - `molecule converge` to apply actions
  - `molecule test` for full cycle
  The converge scenario exercises services, ports, rich rules, sources, interfaces, masquerade, ICMP, and forwarding in the `public` zone.

## License

MIT

## Author Information

Author: iamenr0s
Galaxy: `iamenr0s.ansible_role_firewalld`

## Contributing

Contributions are welcome! Please:
1. Fork the repository
2. Create a feature branch
3. Make your changes
4. Add tests if applicable
5. Submit a pull request

## Changelog

See `CHANGELOG.md` for version history and release notes.
