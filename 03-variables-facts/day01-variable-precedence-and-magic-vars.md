# 🔧 Variable Precedence & Magic Variables — Ansible Mastery

> **Ansible's 22-level variable precedence system is the engine under every playbook — engineers who understand it write maintainable automation; those who don't spend hours debugging mysterious overrides.**

## 📖 Concept

Ansible resolves variables from **22 different sources**, applied in a strict precedence order from lowest to highest. The most important mental model: **lower-precedence sources provide sensible defaults, higher-precedence sources allow targeted overrides**. The most common production mistake is defining the same variable in multiple places and being surprised by which value wins.

The practical day-to-day precedence order engineers encounter most often (simplified from 22 to the most impactful):

1. **role defaults** (`roles/myrole/defaults/main.yml`) — lowest, easily overridden by anything
2. **inventory group_vars** (`group_vars/all`, `group_vars/web`)
3. **inventory host_vars** (`host_vars/web01.example.com`)
4. **playbook vars** (`vars:` block in play)
5. **vars_files** (files loaded via `vars_files:`)
6. **role vars** (`roles/myrole/vars/main.yml`) — high precedence, harder to override
7. **`set_fact`** — runs at task time, very high precedence
8. **`-e` / `--extra-vars`** — absolute highest, overrides everything including `set_fact`

**group_vars and host_vars** directories are where most variable configuration lives in real projects. They support YAML files and subdirectories, letting you split large variable sets into logical files. The directory structure is co-located with your inventory and is automatically loaded — no `vars_files:` needed.

**Magic variables** are special variables that Ansible populates automatically with runtime context: `inventory_hostname`, `ansible_host`, `hostvars`, `groups`, `group_names`, `ansible_play_hosts`, and more. They are invaluable for cross-host coordination — having one host read a fact from another, building dynamic lists, or making decisions based on group membership.

**Registered variables** (`register:`) capture task output and make it available for subsequent tasks. Combined with `set_fact`, they let you transform raw command output into clean variables that can be used later in the play or exported to other hosts via `hostvars`.

---

## 💡 Real-World Use Cases

- Use `group_vars/prod` vs `group_vars/staging` to define environment-specific AWS regions, RDS endpoints, and app configs — the same playbook runs across all environments without any code changes.
- Use `hostvars` to build a dynamic Nginx upstream block: iterate over `groups['app']` and pull `ansible_host` from each app server's facts to generate a load balancer config from a single template.
- Use `set_fact` to transform the output of `aws_ec2_info` into a clean list of instance IDs that downstream tasks can loop over.

---

## 🔧 Commands & Examples

### group_vars & host_vars Directory Layout

```
inventory/
├── hosts.yaml
├── group_vars/
│   ├── all/
│   │   ├── common.yml        # vars for all hosts
│   │   └── vault.yml         # encrypted vars (ansible-vault)
│   ├── prod/
│   │   ├── main.yml          # prod-specific vars
│   │   └── aws.yml           # AWS-specific prod vars
│   ├── staging/
│   │   └── main.yml
│   └── web/
│       └── nginx.yml         # vars only for [web] group
└── host_vars/
    ├── web01.prod.example.com/
    │   └── main.yml          # host-specific overrides
    └── db01.prod.example.com/
        └── main.yml
```

```yaml
# group_vars/all/common.yml
---
ansible_user: ec2-user
ansible_python_interpreter: /usr/bin/python3
ntp_servers:
  - 169.254.169.123    # AWS NTP
  - 0.pool.ntp.org
log_retention_days: 30
monitoring_enabled: true

# group_vars/prod/main.yml
---
env: production
aws_region: us-east-1
db_endpoint: prod-db.cluster.us-east-1.rds.amazonaws.com
app_replicas: 3
log_level: warning

# group_vars/staging/main.yml
---
env: staging
aws_region: us-west-2
db_endpoint: staging-db.cluster.us-west-2.rds.amazonaws.com
app_replicas: 1
log_level: debug

# host_vars/web01.prod.example.com/main.yml
---
nginx_worker_processes: 8     # this host has more CPUs
primary_node: true            # used for tasks that run on only one node
```

```bash
# Debug what variable value a specific host resolves
ansible web01.prod.example.com -i inventory/ -m debug -a 'var=env'
ansible web01.prod.example.com -i inventory/ -m debug -a 'var=nginx_worker_processes'

# Show all variables for a host (includes all precedence layers)
ansible web01.prod.example.com -i inventory/ -m debug -a 'var=hostvars[inventory_hostname]'

# Show which groups a host belongs to
ansible web01.prod.example.com -i inventory/ -m debug -a 'var=group_names'
```

---

### Variable Precedence — Practical Examples

```yaml
# roles/myapp/defaults/main.yml  (lowest precedence — safe defaults)
---
app_port: 8080
app_workers: 2
app_log_level: info
java_heap_size: 512m

# roles/myapp/vars/main.yml  (high precedence — rarely override these)
---
app_user: appuser
app_install_dir: /opt/myapp
app_config_dir: /etc/myapp
```

```yaml
# Playbook vars override group_vars but not role vars
- name: Deploy app
  hosts: app
  vars:
    app_port: 9090          # overrides group_vars, but NOT roles/myapp/vars/main.yml
    app_log_level: debug    # overrides roles/myapp/defaults/main.yml

  tasks:
    - name: Show resolved vars
      ansible.builtin.debug:
        msg: "port={{ app_port }}, level={{ app_log_level }}, dir={{ app_install_dir }}"
      # Output: port=9090, level=debug, dir=/opt/myapp
```

```bash
# Extra vars — override everything, including role vars
ansible-playbook deploy.yml -e "app_port=9999 app_log_level=trace"

# Pass extra vars as a JSON/YAML file
ansible-playbook deploy.yml -e @extra_vars.yml

# Pass extra vars as inline JSON
ansible-playbook deploy.yml -e '{"app_port": 9999, "java_heap_size": "2g"}'
```

---

### Magic Variables — Essential Reference

```yaml
# inventory_hostname — the name as it appears in inventory (FQDN or alias)
# ansible_hostname — the actual hostname reported by the OS
# ansible_host — the IP or hostname Ansible connects to
# ansible_fqdn — fully qualified domain name from facts

- name: Show host identity
  ansible.builtin.debug:
    msg: |
      inventory_hostname: {{ inventory_hostname }}
      ansible_hostname:   {{ ansible_hostname }}
      ansible_host:       {{ ansible_host }}
      ansible_fqdn:       {{ ansible_fqdn }}
      group_names:        {{ group_names | join(', ') }}
```

```yaml
# groups — dict of all groups and their member lists
# group_names — list of groups the current host belongs to
# ansible_play_hosts — list of hosts active in the current play
# ansible_play_batch — current serial batch of hosts

- name: Print all app servers in the inventory
  ansible.builtin.debug:
    msg: "App servers: {{ groups['app'] | join(', ') }}"

- name: Act only if this host is in the 'db' group
  ansible.builtin.debug:
    msg: "I am a DB host"
  when: "'db' in group_names"

- name: Run only on the first host in the play
  ansible.builtin.debug:
    msg: "I am the first host"
  when: inventory_hostname == ansible_play_hosts[0]

- name: Run only on the last host in the play
  ansible.builtin.debug:
    msg: "I am the last host"
  when: inventory_hostname == ansible_play_hosts[-1]
```

```yaml
# hostvars — access variables and facts from OTHER hosts
# Use case: build nginx upstream config from app server IPs

- name: Generate nginx upstream block
  ansible.builtin.template:
    src: nginx_upstream.conf.j2
    dest: /etc/nginx/conf.d/upstream.conf
  vars:
    app_servers: "{{ groups['app'] | map('extract', hostvars, 'ansible_host') | list }}"
  # Template uses: {% for ip in app_servers %}  server {{ ip }}:8080; {% endfor %}
```

```yaml
# ansible_play_hosts_all — all hosts targeted, even failed ones
# ansible_play_hosts — only currently active (non-failed) hosts

- name: Show progress
  ansible.builtin.debug:
    msg: "Processing {{ inventory_hostname }} ({{ ansible_play_hosts.index(inventory_hostname) + 1 }}/{{ ansible_play_hosts | length }})"
```

---

### Registered Variables

```yaml
- name: Check current Java version
  ansible.builtin.command: java -version
  register: java_version_result
  changed_when: false
  ignore_errors: true

- name: Show Java version output
  ansible.builtin.debug:
    var: java_version_result

# Registered variable structure (typical command result):
# java_version_result:
#   rc: 0                    # return code
#   stdout: ""               # standard output
#   stderr: "openjdk version 17.0.2"  # java -version writes to stderr
#   stdout_lines: []         # stdout split by newlines
#   stderr_lines: [...]      # stderr split by newlines
#   changed: false
#   failed: false

- name: Install Java only if not present or wrong version
  ansible.builtin.package:
    name: java-17-openjdk
    state: present
  when:
    - java_version_result.rc != 0 or
      '17' not in java_version_result.stderr
```

```yaml
# Register loop results
- name: Check multiple services
  ansible.builtin.service_facts:

- name: Start services that are stopped
  ansible.builtin.service:
    name: "{{ item }}"
    state: started
  loop:
    - nginx
    - myapp
    - redis
  when: ansible_facts.services[item + '.service'] is defined and
        ansible_facts.services[item + '.service'].state != 'running'

# Register output from a loop
- name: Create multiple users and capture results
  ansible.builtin.user:
    name: "{{ item }}"
    state: present
    create_home: true
  loop:
    - alice
    - bob
    - carol
  register: user_creation_results

- name: Show which users were created
  ansible.builtin.debug:
    msg: "Created: {{ user_creation_results.results | selectattr('changed') | map(attribute='name') | list }}"
```

---

### set_fact — Dynamic Variable Creation

```yaml
# set_fact runs at task time and has very high precedence
- name: Determine deployment strategy based on environment
  ansible.builtin.set_fact:
    deploy_strategy: "{{ 'rolling' if env == 'production' else 'recreate' }}"
    max_surge: "{{ '25%' if env == 'production' else '100%' }}"
    canary_enabled: "{{ env == 'production' and canary_feature_flag | default(false) }}"

- name: Show resolved strategy
  ansible.builtin.debug:
    msg: "Strategy: {{ deploy_strategy }}, max_surge: {{ max_surge }}"
```

```yaml
# set_fact with cacheable: true — persists across plays in the same run
- name: Discover primary DB node
  ansible.builtin.command: >
    psql -h {{ db_endpoint }} -c "SELECT pg_is_in_recovery()" -t
  register: db_recovery_check
  delegate_to: "{{ groups['db'][0] }}"
  run_once: true
  changed_when: false

- name: Set db_is_primary fact on all hosts
  ansible.builtin.set_fact:
    db_primary_host: "{{ groups['db'] | map('extract', hostvars, 'ansible_host') | first }}"
  cacheable: true    # survives to subsequent plays in the same playbook run
```

```yaml
# Transform AWS CLI output into usable vars
- name: Get RDS instance endpoint
  ansible.builtin.command: >
    aws rds describe-db-instances
    --db-instance-identifier myapp-prod
    --query 'DBInstances[0].Endpoint.Address'
    --output text
    --region us-east-1
  register: rds_endpoint_raw
  delegate_to: localhost
  changed_when: false

- name: Set clean db endpoint fact
  ansible.builtin.set_fact:
    db_host: "{{ rds_endpoint_raw.stdout | trim }}"

- name: Use the fact in a template
  ansible.builtin.template:
    src: app.conf.j2
    dest: /etc/myapp/app.conf
  # Template: database_url = jdbc:postgresql://{{ db_host }}:5432/myapp
```

---

### Custom Facts (Local Facts)

```bash
# Custom facts live in /etc/ansible/facts.d/ on managed hosts
# They must be executable scripts (returning JSON) or static .fact files
```

```ini
# /etc/ansible/facts.d/myapp.fact  (static INI format)
[app]
version=2.4.1
install_date=2026-03-17
install_by=ansible

[database]
schema_version=47
last_migration=2026-03-15
```

```bash
#!/bin/bash
# /etc/ansible/facts.d/aws_metadata.fact  (executable returning JSON)
TOKEN=$(curl -s -X PUT "http://169.254.169.254/latest/api/token" \
  -H "X-aws-ec2-metadata-token-ttl-seconds: 21600")
INSTANCE_ID=$(curl -s -H "X-aws-ec2-metadata-token: $TOKEN" \
  http://169.254.169.254/latest/meta-data/instance-id)
AZ=$(curl -s -H "X-aws-ec2-metadata-token: $TOKEN" \
  http://169.254.169.254/latest/meta-data/placement/availability-zone)

echo "{\"instance_id\": \"${INSTANCE_ID}\", \"availability_zone\": \"${AZ}\"}"
```

```yaml
# Deploy custom facts and use them
- name: Deploy custom fact file
  ansible.builtin.copy:
    src: files/myapp.fact
    dest: /etc/ansible/facts.d/myapp.fact
    mode: "0644"
  notify: Refresh facts

- name: Refresh local facts after deploying
  ansible.builtin.setup:
    filter: ansible_local

- name: Use custom facts
  ansible.builtin.debug:
    msg: "App version: {{ ansible_local.myapp.app.version }}"

- name: Skip migration if schema is up to date
  ansible.builtin.command: /opt/myapp/bin/migrate
  when: ansible_local.myapp.database.schema_version | int < target_schema_version | int
```

---

## ⚠️ Gotchas & Pro Tips

- **`role vars` beat `group_vars` and `host_vars`:** This surprises almost everyone. Variables in `roles/myrole/vars/main.yml` have higher precedence than `host_vars` and `group_vars`. Only `set_fact`, task `vars:`, and `-e` can override them. If you need a role variable to be overridable per host, put it in `defaults/main.yml`, not `vars/main.yml`.

- **YAML type preservation in `group_vars`:** Unlike INI inventory, YAML files in `group_vars/` preserve types. `port: 8080` is the integer `8080`, not the string `"8080"`. This matters when you pass values to modules that do type checking. Be explicit with quotes when you intend strings: `port: "8080"`.

- **`hostvars` only has facts if `gather_facts` ran:** If you access `hostvars['web01']['ansible_host']` from another host, it works because `ansible_host` comes from inventory. But `hostvars['web01']['ansible_default_ipv4']` requires that facts were gathered for `web01` earlier in the run. Use `delegate_to` + `setup` to gather facts from a host mid-play if needed.

- **`set_fact` variables are host-scoped:** A `set_fact` on one host does NOT automatically apply to other hosts. To share a value computed on host A with host B, use `hostvars['hostA']['my_fact']` on host B, or use `delegate_to: localhost` + `run_once: true` to compute the value and set it in play scope.

- **Variable naming collisions with facts:** Ansible facts (`ansible_*`) and magic variables are reserved namespaces. Don't name your variables `ansible_user_groups` or similar — they may silently override or conflict with built-in facts. Prefix custom variables with your project or role name: `myapp_port`, `myapp_db_host`.

---

*Part of the [Ansible Mastery](https://github.com/naveenramasamy11/ansible-mastery) series.*
