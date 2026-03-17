# 🔧 Role Structure & Ansible Galaxy — Ansible Mastery

> **Roles are the unit of reuse in Ansible — engineers who structure them well build a library of battle-tested automation that accelerates every future project.**

## 📖 Concept

An Ansible **role** is a self-contained unit of automation with a defined directory structure that Ansible understands natively. Unlike a flat playbook that mixes tasks, variables, and templates in one file, a role enforces separation of concerns: tasks live in `tasks/`, configuration defaults in `defaults/`, sensitive values in `vars/`, templates in `templates/`, static files in `files/`, and test infrastructure in `molecule/`. This structure makes roles shareable, testable, and composable.

The single most important design decision in any role is **what goes in `defaults/` vs `vars/`**. `defaults/main.yml` is the role's public API — variables here are designed to be overridden by callers (via `group_vars`, `host_vars`, or playbook `vars`). `vars/main.yml` holds implementation details that should rarely change — paths, internal names, computed values. Putting user-facing configuration in `vars/` instead of `defaults/` is the most common role design mistake, as it prevents callers from customising behaviour without hacking the role itself.

**Role dependencies** defined in `meta/main.yml` allow roles to declare prerequisite roles. When you include a role that depends on others, Ansible automatically runs the dependencies first. This is how complex automation stacks are composed: an `app_deploy` role might depend on `java`, `nginx`, and `base_hardening` — each of which has its own dependency chain. Dependencies are resolved once per play, preventing duplicate execution.

**Ansible Galaxy** is both the public role/collection registry and the `ansible-galaxy` CLI tool for managing them. For production environments, using `requirements.yml` to pin Galaxy roles and collections to specific versions is essential — unpinned Galaxy installs are the infrastructure equivalent of `npm install` without a lock file.

**Molecule** is the de-facto testing framework for Ansible roles. It creates test instances (Docker, EC2, Vagrant), runs the role, verifies idempotency (runs twice, checks for changes on second run), and runs Testinfra or Goss assertions. Integrating Molecule into CI/CD ensures roles don't regress.

---

## 💡 Real-World Use Cases

- Build a `java` role with `defaults/` exposing `java_version` and `java_heap_size` — reuse it across 15 different app roles with environment-specific overrides in `group_vars`.
- Use role dependencies so that any role that includes `myapp_deploy` automatically gets `base_hardening` and `cloudwatch_agent` applied first, without callers needing to remember the full dependency chain.
- Run Molecule in GitHub Actions to test every role on every pull request against a Docker container, catching idempotency regressions before merge.

---

## 🔧 Commands & Examples

### Role Directory Structure

```
roles/
└── myapp/
    ├── defaults/
    │   └── main.yml          # ← public API: all user-facing variables
    ├── vars/
    │   └── main.yml          # ← private: rarely-changed implementation vars
    ├── tasks/
    │   ├── main.yml          # ← entry point (imports other task files)
    │   ├── install.yml
    │   ├── configure.yml
    │   └── service.yml
    ├── handlers/
    │   └── main.yml          # ← handlers for this role
    ├── templates/
    │   ├── app.conf.j2
    │   └── systemd.service.j2
    ├── files/
    │   └── logrotate.conf    # ← static files
    ├── meta/
    │   └── main.yml          # ← role metadata and dependencies
    ├── tests/
    │   ├── inventory
    │   └── test.yml
    └── molecule/
        └── default/
            ├── molecule.yml
            ├── converge.yml
            └── verify.yml
```

---

### defaults/main.yml — Public Role API

```yaml
# roles/myapp/defaults/main.yml
---
# Package settings
myapp_version: "2.4.1"
myapp_package_url: "https://releases.example.com/myapp-{{ myapp_version }}.tar.gz"
myapp_checksum: "sha256:abc123..."

# Runtime settings
myapp_port: 8080
myapp_workers: "{{ ansible_processor_vcpus | default(2) }}"
myapp_log_level: info
myapp_java_heap: "{{ (ansible_memtotal_mb | int * 0.5) | int }}m"

# Directory layout
myapp_install_dir: /opt/myapp
myapp_config_dir: /etc/myapp
myapp_log_dir: /var/log/myapp
myapp_data_dir: /var/lib/myapp

# User/group
myapp_user: appuser
myapp_group: appuser
myapp_user_uid: 1500

# Feature flags
myapp_ssl_enabled: false
myapp_metrics_enabled: true
myapp_ha_enabled: false
```

```yaml
# roles/myapp/vars/main.yml
---
# Internal vars — these shouldn't need changing per environment
_myapp_service_name: myapp
_myapp_pid_file: /run/myapp/myapp.pid
_myapp_required_packages:
  - java-17-openjdk
  - curl
  - jq
```

---

### tasks/main.yml — Structured Entry Point

```yaml
# roles/myapp/tasks/main.yml
---
- name: Include OS-specific variables
  ansible.builtin.include_vars: "{{ item }}"
  with_first_found:
    - "{{ ansible_distribution }}-{{ ansible_distribution_major_version }}.yml"
    - "{{ ansible_distribution }}.yml"
    - "{{ ansible_os_family }}.yml"
    - default.yml
  tags: always

- name: Assert required variables are set
  ansible.builtin.assert:
    that:
      - myapp_version is defined
      - myapp_version | length > 0
      - myapp_port | int > 0
      - myapp_port | int < 65536
    fail_msg: "Required role variables are not set correctly"
    success_msg: "Role variable assertions passed"
  tags: always

- name: Install myapp
  ansible.builtin.import_tasks: install.yml
  tags: [install, deploy]

- name: Configure myapp
  ansible.builtin.import_tasks: configure.yml
  tags: [configure, deploy]

- name: Manage myapp service
  ansible.builtin.import_tasks: service.yml
  tags: [service]
```

```yaml
# roles/myapp/tasks/install.yml
---
- name: Create myapp system user
  ansible.builtin.user:
    name: "{{ myapp_user }}"
    uid: "{{ myapp_user_uid }}"
    system: true
    shell: /sbin/nologin
    home: "{{ myapp_install_dir }}"
    create_home: false

- name: Create directory structure
  ansible.builtin.file:
    path: "{{ item }}"
    state: directory
    owner: "{{ myapp_user }}"
    group: "{{ myapp_group }}"
    mode: "0755"
  loop:
    - "{{ myapp_install_dir }}"
    - "{{ myapp_config_dir }}"
    - "{{ myapp_log_dir }}"
    - "{{ myapp_data_dir }}"

- name: Install required packages
  ansible.builtin.package:
    name: "{{ _myapp_required_packages }}"
    state: present

- name: Download and extract myapp
  ansible.builtin.unarchive:
    src: "{{ myapp_package_url }}"
    dest: "{{ myapp_install_dir }}"
    remote_src: true
    checksum: "{{ myapp_checksum }}"
    owner: "{{ myapp_user }}"
    group: "{{ myapp_group }}"
    creates: "{{ myapp_install_dir }}/bin/myapp"  # idempotent guard
```

---

### meta/main.yml — Dependencies

```yaml
# roles/myapp/meta/main.yml
---
galaxy_info:
  role_name: myapp
  author: naveenramasamy11
  description: Deploys and manages the MyApp application
  license: MIT
  min_ansible_version: "2.14"
  platforms:
    - name: EL        # RHEL / CentOS / Amazon Linux
      versions:
        - "8"
        - "9"
    - name: Ubuntu
      versions:
        - focal
        - jammy
  galaxy_tags:
    - web
    - java
    - application

dependencies:
  - role: base_hardening
    vars:
      hardening_level: standard
  - role: java
    vars:
      java_version: "17"
      java_heap_size: "{{ myapp_java_heap }}"
  - role: cloudwatch_agent
    when: aws_environment | default(false)
```

---

### requirements.yml — Pinned Galaxy Dependencies

```yaml
# requirements.yml — at project root
---
roles:
  - name: geerlingguy.docker
    version: "7.1.0"
  - name: geerlingguy.nginx
    version: "3.2.0"
  - src: https://github.com/myorg/internal-role.git
    scm: git
    version: v1.4.2
    name: myorg.internal_role

collections:
  - name: amazon.aws
    version: ">=7.0.0,<8.0.0"
  - name: community.kubernetes
    version: "3.0.0"
  - name: community.vmware
    version: "4.0.0"
  - name: ansible.posix
    version: ">=1.5.0"
```

```bash
# Install all roles and collections from requirements.yml
ansible-galaxy install -r requirements.yml

# Install to project-local directory (keeps versions isolated)
ansible-galaxy install -r requirements.yml -p ./roles/

# Install collections only
ansible-galaxy collection install -r requirements.yml

# Show installed collections and versions
ansible-galaxy collection list

# Search Galaxy for roles
ansible-galaxy search nginx --author geerlingguy

# Get info on a role
ansible-galaxy role info geerlingguy.nginx
```

---

### include_role vs import_role — Critical Distinction

```yaml
# import_role — STATIC: processed at parse time
# - Tags applied to import_role cascade to all tasks inside
# - Can't use variables for role name (resolved before runtime)
# - Handlers from imported roles are available to the full play
# - Best for: roles that always run, role composition in plays

- name: Harden all servers
  ansible.builtin.import_role:
    name: base_hardening
  tags: [hardening]   # ALL tasks in base_hardening get this tag

# include_role — DYNAMIC: processed at runtime
# - Variables for role name are resolved at runtime
# - Tags apply only to the include task itself (not tasks inside)
# - Handlers from included roles are only available after include runs
# - Best for: conditional role inclusion, loop-based role inclusion

- name: Apply environment-specific role
  ansible.builtin.include_role:
    name: "{{ env }}_config"   # variable role name — only possible with include_role
  when: custom_config_required | default(false)

- name: Apply roles from a list
  ansible.builtin.include_role:
    name: "{{ item }}"
  loop:
    - monitoring_agent
    - log_forwarder
    - backup_agent
```

---

### Molecule Testing

```yaml
# molecule/default/molecule.yml
---
dependency:
  name: galaxy
  options:
    requirements-file: requirements.yml

driver:
  name: docker

platforms:
  - name: instance-el9
    image: "geerlingguy/docker-rockylinux9-ansible:latest"
    command: /usr/lib/systemd/systemd
    capabilities:
      - SYS_ADMIN
    volumes:
      - /sys/fs/cgroup:/sys/fs/cgroup:rw
    cgroupns_mode: host
    privileged: true

  - name: instance-ubuntu22
    image: "geerlingguy/docker-ubuntu2204-ansible:latest"
    command: /lib/systemd/systemd
    privileged: true

provisioner:
  name: ansible
  inventory:
    group_vars:
      all:
        myapp_version: "2.4.1"
        myapp_log_level: debug

verifier:
  name: ansible
```

```yaml
# molecule/default/converge.yml
---
- name: Converge
  hosts: all
  tasks:
    - name: Include the role
      ansible.builtin.include_role:
        name: myapp
      vars:
        myapp_port: 8080
```

```yaml
# molecule/default/verify.yml
---
- name: Verify
  hosts: all
  tasks:
    - name: Check that myapp service is running
      ansible.builtin.service_facts:

    - name: Assert myapp service is active
      ansible.builtin.assert:
        that:
          - "'myapp.service' in ansible_facts.services"
          - "ansible_facts.services['myapp.service'].state == 'running'"

    - name: Assert health endpoint responds
      ansible.builtin.uri:
        url: "http://localhost:8080/health"
        status_code: 200
```

```bash
# Full Molecule test cycle
molecule test

# Individual stages
molecule create        # create test instances
molecule converge      # apply the role
molecule idempotence   # run again, assert no changes
molecule verify        # run assertions
molecule destroy       # destroy instances

# Molecule with a specific scenario
molecule test -s production

# Run against only one platform
molecule converge -- -l instance-el9
```

---

## ⚠️ Gotchas & Pro Tips

- **Role dependencies run once per play, regardless of how many times the role is called:** If role A depends on role B, and you call role A three times in a play, role B only runs once. This is correct for idempotent setup tasks but can surprise you if you expect dependency behaviour on each call. Use `allow_duplicates: true` in `meta/main.yml` to override this.

- **`include_vars` with `with_first_found` is OS-aware variable loading:** The pattern of loading `RedHat.yml`, `Ubuntu.yml`, and falling back to `default.yml` is the professional way to handle OS-specific variable differences in a role. It keeps OS branches out of your tasks and in your variable files where they belong.

- **Molecule idempotence is the most valuable test:** After the first role run, Molecule runs the role again and asserts zero tasks are `changed`. This catches the most common role authoring mistakes: missing `creates:` guards on unarchive, missing `changed_when: false` on read-only commands, and non-idempotent shell commands. Make this test part of your CI gate.

- **Galaxy version pinning protects production:** An unpinned `ansible-galaxy install geerlingguy.docker` fetches the latest version, which may have breaking changes. Always specify `version:` in `requirements.yml` and commit the file. Treat it like `package.json` with a lock file.

- **`assert` at the top of `main.yml` is self-documenting:** Adding an `assert` task that validates required variables and documents their constraints (type, range, allowed values) serves as both a runtime check and documentation for role callers. It prevents cryptic failures deep in role execution from bad input.

---

*Part of the [Ansible Mastery](https://github.com/naveenramasamy11/ansible-mastery) series.*
