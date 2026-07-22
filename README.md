[![Molecule](https://github.com/iamenr0s/ansible-role-firewalld/actions/workflows/molecule.yml/badge.svg)](https://github.com/iamenr0s/ansible-role-firewalld/actions/workflows/molecule.yml) ![Ansible Role](https://img.shields.io/ansible/role/d/iamenr0s/ansible_role_firewalld) [![CodeFactor](https://www.codefactor.io/repository/github/iamenr0s/ansible-role-firewalld/badge)](https://www.codefactor.io/repository/github/iamenr0s/ansible-role-firewalld)

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
- Optionally manages SELinux state and booleans

## Requirements

- Ansible 2.9 or higher
- External roles (installed via `requirements.yml`):
  - `iamenr0s.ansible_role_pkg_management` (Ansible Galaxy)
  - `iamenr0s.ansible_role_kernel_configuration` (Ansible Galaxy)

- When managing SELinux, ensure SELinux tooling is available on targets:
  - `python3-libsemanage` (or `libsemanage-python` on older distros)

Install with:

```
ansible-galaxy install -r requirements.yml
```

## Supported Platforms

- Ubuntu (22.04, 24.04)
- Debian (12, 13)
- EL/RHEL/Rocky/Alma (8, 9, 10)
- Fedora (42, 43, 44)

## Role Variables

- `firewalld_manage_kernel` (bool): Whether to manage kernel modules/sysctl needed by firewalld. Default: `true`.
- `firewalld_required_kernel_modules` (list): Modules to ensure loaded. Default: `[nf_conntrack, nf_tables]`.
- `firewalld_kernel_sysctl` (dict): Sysctl to apply (key/value). Default: `{}`.
- `firewalld_manage_selinux` (bool): Whether to manage SELinux. Default: `false`.
- `firewalld_selinux_state` (string|null): Desired SELinux state (`enforcing`, `permissive`, `disabled`). Default: `null` (no change).
- `firewalld_selinux_policy` (string|null): SELinux policy name (e.g., `targeted`). Default: `null` (no change).
- `firewalld_selinux_booleans` (list of dict): SELinux booleans to toggle.
  - Keys: `name` (boolean), `state` (`true`/`false`, default `true`), `persistent` (`true`/`false`, default `true`).
- `firewalld_service_name` (string): Service name. Default: `firewalld`.
- `firewalld_required_packages_map` (dict): Per OS-family package mapping. Default includes `firewalld` for Debian/Ubuntu/RedHat/Rocky/Fedora.
- `firewalld_required_packages` (list): Resolved packages for the current host using `ansible_os_family`. Default: `['firewalld']` when no mapping exists.
- `firewalld_zones_present` (list): Zones to create permanently. Default: `[]`.
- `firewalld_reload_after_zone_changes` (bool): Reload firewalld after any permanent configuration change (zones, services, ports, rules, sources, interfaces, masquerade, ICMP blocks, forward ports) so the change takes effect on the running daemon. Default: `true`.
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

- `iamenr0s.ansible_role_pkg_management` from Ansible Galaxy
- `iamenr0s.ansible_role_kernel_configuration` from Ansible Galaxy

Collections:

- `ansible.posix` (used for `ansible.posix.firewalld`, `ansible.posix.selinux`, `ansible.posix.seboolean`, and `ansible.posix.firewalld_info`). Install with: `ansible-galaxy collection install ansible.posix` if not present.

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

### Example: Manage SELinux

```
- hosts: all
  become: true
  roles:
    - role: iamenr0s.ansible_role_firewalld
      vars:
        firewalld_manage_selinux: true
        firewalld_selinux_state: enforcing
        firewalld_selinux_policy: targeted
        firewalld_selinux_booleans:
          - { name: httpd_can_network_connect, state: true, persistent: true }
          - { name: nis_enabled, state: false }
```

## Troubleshooting

- Ensure external roles are installed via `ansible-galaxy install -r requirements.yml`.
- If the kernel role uses different variable names, map them in `tasks/main.yml` vars.
- Zone/service/port/rule changes are permanent by default; the role reloads firewalld once at the end (via a handler) whenever anything changed, controlled by `firewalld_reload_after_zone_changes`.
- Port forwarding accepts only one forward entry per operation; this role loops one-by-one.
- Set `immediate: true` on individual items if you need a specific rule to apply without waiting for the end-of-role reload.

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
