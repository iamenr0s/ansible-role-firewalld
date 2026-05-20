# Fix Code Review Issues — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Fix all issues identified in code review without breaking the role's functionality.

**Architecture:** Each fix is self-contained and touches one file at a time. Tests (verify.yml + linting) are written/added first, then each bug fix is applied. Linting is run after every change.

**Tech Stack:** Ansible, Molecule (Podman driver), yamllint, ansible-lint

---

## File Map

| File | Action | Reason |
|------|--------|--------|
| `molecule/default/verify.yml` | **Create** | Add missing test assertions |
| `molecule/default/prepare.yml` | **Modify** | Fix `become`, duplicate name, `/tmp` mode, remove redundant install |
| `meta/main.yml` | **Modify** | Replace EOL Ubuntu `kinetic` with `noble` |
| `README.md` | **Modify** | Fix dependency source from "Git" to Galaxy |
| `handlers/main.yml` | **Modify** | Remove dead `Restart firewalld` handler |
| `.ansible-lint` | **Modify** | Remove template placeholder entries |

---

### Task 1: Add molecule verify step (new test)

**Files:**
- Create: `molecule/default/verify.yml`

- [ ] **Step 1: Write the verify playbook**

```yaml
# molecule/default/verify.yml
---

- name: Verify
  hosts: all
  gather_facts: true
  become: true

  tasks:
    - name: Gather service facts
      ansible.builtin.service_facts:

    - name: Assert firewalld service is running
      ansible.builtin.assert:
        that:
          - >-
            ('firewalld' in ansible_facts.services and
             ansible_facts.services['firewalld'].state == 'running') or
            ('firewalld.service' in ansible_facts.services and
             ansible_facts.services['firewalld.service'].state == 'running')
        fail_msg: "firewalld service is not running"
        success_msg: "firewalld service is running"

    - name: Gather firewalld zone info
      ansible.posix.firewalld_info:
        zones:
          - public
      register: fw_info

    - name: Assert ssh service is enabled in public zone
      ansible.builtin.assert:
        that:
          - "'ssh' in fw_info.firewalld_info.zones.public.services"
        fail_msg: "ssh service not found in public zone"
        success_msg: "ssh service is present in public zone"

    - name: Assert 8080/tcp port is open in public zone
      ansible.builtin.assert:
        that:
          - "'8080/tcp' in fw_info.firewalld_info.zones.public.ports"
        fail_msg: "port 8080/tcp not found in public zone"
        success_msg: "port 8080/tcp is open in public zone"

    - name: Assert rich rule is present in public zone
      ansible.builtin.assert:
        that:
          - fw_info.firewalld_info.zones.public.rich_rules | length > 0
        fail_msg: "No rich rules found in public zone"
        success_msg: "Rich rules are present in public zone"
```

- [ ] **Step 2: Lint the new file**

```bash
cd /mnt/c/Users/enros/Development/ansible-role-firewalld
yamllint molecule/default/verify.yml
ansible-lint molecule/default/verify.yml
```

Expected: no errors

- [ ] **Step 3: Commit**

```bash
git add molecule/default/verify.yml
git commit -m "test(molecule): add verify step for firewalld service and zone state"
```

---

### Task 2: Fix `prepare.yml` — `become: false` (HIGH)

**Files:**
- Modify: `molecule/default/prepare.yml:6`

- [ ] **Step 1: Change `become: false` to `become: true`**

In `molecule/default/prepare.yml`, change line 6:

```yaml
# Before
  become: false

# After
  become: true
```

- [ ] **Step 2: Lint**

```bash
yamllint molecule/default/prepare.yml
ansible-lint molecule/default/prepare.yml
```

Expected: no errors

- [ ] **Step 3: Commit**

```bash
git add molecule/default/prepare.yml
git commit -m "fix(molecule): run prepare tasks with become to allow package installation"
```

---

### Task 3: Fix `prepare.yml` — duplicate task name (HIGH)

**Files:**
- Modify: `molecule/default/prepare.yml:41`

- [ ] **Step 1: Rename the misnamed task**

In `molecule/default/prepare.yml`, change the task at line ~41 from:

```yaml
# Before
    - name: Install procps-ng package (RHEL/Fedora/AlmaLinux)
      ansible.builtin.package:
        name: python3-rpm
        state: present
```

to:

```yaml
# After
    - name: Install python3-rpm package (RHEL/Fedora/AlmaLinux)
      ansible.builtin.package:
        name: python3-rpm
        state: present
```

- [ ] **Step 2: Lint**

```bash
yamllint molecule/default/prepare.yml
ansible-lint molecule/default/prepare.yml
```

Expected: no errors

- [ ] **Step 3: Commit**

```bash
git add molecule/default/prepare.yml
git commit -m "fix(molecule): correct task name for python3-rpm installation step"
```

---

### Task 4: Fix `prepare.yml` — `/tmp` mode and redundant install (LOW + INFO)

**Files:**
- Modify: `molecule/default/prepare.yml:10,47-51`

- [ ] **Step 1: Fix `/tmp` mode and remove redundant firewalld install**

In `molecule/default/prepare.yml`:

Change the `/tmp` task mode:
```yaml
# Before
    - name: Ensure /tmp exists
      ansible.builtin.file:
        path: /tmp
        state: directory
        mode: '0777'

# After
    - name: Ensure /tmp exists
      ansible.builtin.file:
        path: /tmp
        state: directory
        mode: '01777'
```

Remove the redundant firewalld pre-install block entirely (the role installs it):
```yaml
# Remove this entire task (firewalld is installed by tasks/main.yml):
    - name: Ensure firewalld package is present when package manager available
      ansible.builtin.package:
        name: firewalld
        state: present
      when: (ansible_pkg_mgr is defined) and (ansible_pkg_mgr | default('unknown')) != 'unknown'
```

- [ ] **Step 2: Lint**

```bash
yamllint molecule/default/prepare.yml
ansible-lint molecule/default/prepare.yml
```

Expected: no errors

- [ ] **Step 3: Commit**

```bash
git add molecule/default/prepare.yml
git commit -m "fix(molecule): set sticky bit on /tmp and remove redundant firewalld pre-install"
```

---

### Task 5: Fix `meta/main.yml` — EOL Ubuntu platform (MEDIUM)

**Files:**
- Modify: `meta/main.yml:26`

- [ ] **Step 1: Replace `kinetic` with `noble`**

In `meta/main.yml`, under the Ubuntu platforms block:

```yaml
# Before
    - name: "Ubuntu"
      versions:
        - "focal"
        - "jammy"
        - "kinetic"

# After
    - name: "Ubuntu"
      versions:
        - "focal"
        - "jammy"
        - "noble"
```

- [ ] **Step 2: Lint**

```bash
yamllint meta/main.yml
ansible-lint meta/main.yml
```

Expected: no errors

- [ ] **Step 3: Commit**

```bash
git add meta/main.yml
git commit -m "fix(meta): replace EOL Ubuntu kinetic with noble (24.04 LTS)"
```

---

### Task 6: Fix `README.md` — dependency source documentation (MEDIUM)

**Files:**
- Modify: `README.md:100-101`

- [ ] **Step 1: Update dependency source from Git to Galaxy**

In `README.md`, find the Requirements section and update:

```markdown
<!-- Before -->
- External roles (installed via `requirements.yml`):
  - `ansible-role-pkg-management` (Git)
  - `ansible-role-kernel-configuration` (Git)

<!-- After -->
- External roles (installed via `requirements.yml`):
  - `iamenr0s.ansible_role_pkg_management` (Ansible Galaxy)
  - `iamenr0s.ansible_role_kernel_configuration` (Ansible Galaxy)
```

Also update the Dependencies section (~line 98-101):

```markdown
<!-- Before -->
Defined in `requirements.yml`:

- `ansible-role-pkg-management` from Git: `git@github.com:iamenr0s/ansible-role-pkg-management.git`
- `ansible-role-kernel-configuration` from Git: `git@github.com:iamenr0s/ansible-role-kernel-configuration.git`

<!-- After -->
Defined in `requirements.yml`:

- `iamenr0s.ansible_role_pkg_management` from Ansible Galaxy
- `iamenr0s.ansible_role_kernel_configuration` from Ansible Galaxy
```

- [ ] **Step 2: Verify yamllint still passes (README is not YAML but run full lint)**

```bash
yamllint .
```

Expected: no errors

- [ ] **Step 3: Commit**

```bash
git add README.md
git commit -m "docs: update dependency source from Git to Ansible Galaxy"
```

---

### Task 7: Remove dead `Restart firewalld` handler (LOW)

**Files:**
- Modify: `handlers/main.yml`

- [ ] **Step 1: Remove the unused handler**

Replace the entire contents of `handlers/main.yml` with an empty handler file:

```yaml
---

# Role - Handlers
# (no handlers currently needed; service management is done inline in tasks/main.yml)
```

- [ ] **Step 2: Lint**

```bash
yamllint handlers/main.yml
ansible-lint handlers/main.yml
```

Expected: no errors

- [ ] **Step 3: Commit**

```bash
git add handlers/main.yml
git commit -m "refactor: remove dead Restart firewalld handler (never notified)"
```

---

### Task 8: Clean up `.ansible-lint` template placeholders (LOW)

**Files:**
- Modify: `.ansible-lint`

- [ ] **Step 1: Remove placeholder entries**

In `.ansible-lint`:

1. Remove `skip_this_tag` from `skip_list` (leave the list empty or remove it):
```yaml
# Before
skip_list:
  - skip_this_tag

# After — remove the skip_list entirely or leave empty
# (no tags to skip)
```

2. Remove the boilerplate mock_roles entries:
```yaml
# Before
mock_roles:
  - mocked_role
  - author.role_name # old standalone galaxy role
  - fake_namespace.fake_collection.fake_role # role within a collection

# After — remove mock_roles block entirely (no roles to mock)
```

3. Remove `skip_this_tag` from `warn_list`:
```yaml
# Before
warn_list:
  - skip_this_tag
  - experimental

# After
warn_list:
  - experimental
```

- [ ] **Step 2: Lint the ansible-lint config itself**

```bash
yamllint .ansible-lint
ansible-lint --version  # confirm ansible-lint can still parse its config
```

Expected: no errors

- [ ] **Step 3: Run full lint to confirm nothing regressed**

```bash
yamllint .
ansible-lint
```

Expected: no errors

- [ ] **Step 4: Commit**

```bash
git add .ansible-lint
git commit -m "chore: remove template placeholder entries from .ansible-lint config"
```

---

## Final Verification

- [ ] Run full yamllint: `yamllint .`
- [ ] Run full ansible-lint: `ansible-lint`
- [ ] If Docker/Podman available: `molecule test` (or `MOLECULE_DISTRO=almalinux9 molecule test`)

---

## Review

| # | Issue | Severity | Task | Status |
|---|-------|----------|------|--------|
| 1 | `prepare.yml` `become: false` blocks package installs | HIGH | Task 2 | [ ] |
| 2 | Duplicate task name hides `python3-rpm` install | HIGH | Task 3 | [ ] |
| 3 | README documents Git deps, requirements.yml uses Galaxy | MEDIUM | Task 6 | [ ] |
| 4 | No molecule verify step | MEDIUM | Task 1 | [ ] |
| 5 | Ubuntu `kinetic` (EOL) in meta/main.yml | MEDIUM | Task 5 | [ ] |
| 6 | Dead `Restart firewalld` handler | LOW | Task 7 | [ ] |
| 7 | `.ansible-lint` template placeholders | LOW | Task 8 | [ ] |
| 8 | `/tmp` mode `0777` missing sticky bit | LOW | Task 4 | [ ] |
| 9 | Redundant firewalld install in prepare.yml | INFO | Task 4 | [ ] |
