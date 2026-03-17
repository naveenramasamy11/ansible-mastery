# 🔧 Inventory Formats & Ad-Hoc Commands — Ansible Mastery

> **Mastering inventory design and ad-hoc commands is the foundation that separates engineers who "run playbooks" from those who truly operate Ansible at scale.**

## 📖 Concept

Ansible's inventory is the source of truth for **who** your automation targets. Getting inventory architecture right from day one pays dividends when you're managing thousands of EC2 instances, VMware VMs, or Kubernetes nodes. Ansible supports two native static inventory formats — INI and YAML — plus dynamic inventory scripts and plugins (covered in section 09). Understanding when to choose each format, and how to structure groups and host variables within them, is critical for maintainable infrastructure code.

The **INI format** is compact and familiar to anyone who has worked with configuration files. It is ideal for small, static environments where humans edit the file directly. However, it starts to show its limits when hosts need rich variable structures, nested groups, or when you want to generate inventory programmatically from Terraform state or AWS tags.

The **YAML inventory format** is a first-class citizen in modern Ansible projects. It supports the full variable hierarchy, composes cleanly with `group_vars/` and `host_vars/` directories, and can be version-controlled and linted alongside your playbooks. For any environment larger than a handful of hosts — especially in AWS, EKS, or VMware contexts — YAML inventory is the professional choice.

`ansible.cfg` is often overlooked as "just configuration," but it profoundly affects performance, security, and debugging experience. A well-tuned `ansible.cfg` can cut playbook run times by 30–50% through SSH pipelining and multiplexing alone.

Ad-hoc commands are Ansible's Swiss Army knife for one-off operations: gathering facts before writing a playbook, verifying connectivity, pushing an emergency fix, or checking the state of a service across a fleet. Every Ansible engineer should be fluent in ad-hoc patterns — they are invaluable during incident response.

---

## 💡 Real-World Use Cases

- Build a YAML inventory that mirrors your AWS VPC structure (prod/staging/dev accounts, subnets as groups) and share it across multiple playbooks.
- Use `ansible.cfg` to enable SSH pipelining and ControlMaster multiplexing before running a fleet-wide patching playbook — dramatically reducing runtime on 500+ hosts.
- Run ad-hoc commands during a P1 incident to restart a service across all app-tier hosts without writing a full playbook.

---

## 🔧 Commands & Examples

### INI Inventory Format

```ini
# inventory/hosts.ini

# Ungrouped hosts
jumphost.example.com

# Web tier — all in us-east-1
[web]
web01.prod.example.com ansible_host=10.0.1.10
web02.prod.example.com ansible_host=10.0.1.11
web03.prod.example.com ansible_host=10.0.1.12

# App tier
[app]
app01.prod.example.com ansible_host=10.0.2.10
app02.prod.example.com ansible_host=10.0.2.11

# DB tier — using ranges
[db]
db[01:03].prod.example.com

# Group of groups — all prod hosts
[prod:children]
web
app
db

# Group variables (applied to all [web] hosts)
[web:vars]
http_port=80
nginx_worker_processes=auto

# All-hosts variables
[all:vars]
ansible_user=ec2-user
ansible_ssh_private_key_file=~/.ssh/prod_key.pem
ansible_python_interpreter=/usr/bin/python3
```

```bash
# Verify the inventory is parsed correctly
ansible-inventory -i inventory/hosts.ini --list

# Ping all hosts in [web] group
ansible web -i inventory/hosts.ini -m ping

# Target a specific host
ansible web01.prod.example.com -i inventory/hosts.ini -m ping
```

---

### YAML Inventory Format (Production-Grade)

```yaml
# inventory/hosts.yaml

all:
  vars:
    ansible_user: ec2-user
    ansible_ssh_private_key_file: ~/.ssh/prod_key.pem
    ansible_python_interpreter: /usr/bin/python3

  children:
    prod:
      children:
        web:
          hosts:
            web01.prod.example.com:
              ansible_host: 10.0.1.10
              nginx_worker_processes: 4
            web02.prod.example.com:
              ansible_host: 10.0.1.11
              nginx_worker_processes: 4
            web03.prod.example.com:
              ansible_host: 10.0.1.12
              nginx_worker_processes: 8   # larger instance

        app:
          vars:
            java_heap_size: 4g
            app_port: 8080
          hosts:
            app01.prod.example.com:
              ansible_host: 10.0.2.10
            app02.prod.example.com:
              ansible_host: 10.0.2.11

        db:
          vars:
            postgres_version: "15"
            postgres_max_connections: 500
          hosts:
            db01.prod.example.com:
              ansible_host: 10.0.3.10
              postgres_role: primary
            db02.prod.example.com:
              ansible_host: 10.0.3.11
              postgres_role: replica
            db03.prod.example.com:
              ansible_host: 10.0.3.12
              postgres_role: replica

    staging:
      vars:
        ansible_user: ubuntu
      children:
        web:
          hosts:
            web01.staging.example.com:
              ansible_host: 10.1.1.10
        app:
          hosts:
            app01.staging.example.com:
              ansible_host: 10.1.2.10
```

```bash
# Validate YAML inventory and show host vars for a specific host
ansible-inventory -i inventory/hosts.yaml --host web01.prod.example.com

# List all hosts in prod group as JSON
ansible-inventory -i inventory/hosts.yaml --list | jq '.prod.hosts'

# Show the full inventory graph
ansible-inventory -i inventory/hosts.yaml --graph
```

---

### ansible.cfg — Tuned for Production

```ini
# ansible.cfg  (project root — takes precedence over /etc/ansible/ansible.cfg)

[defaults]
# Inventory source — can be a file or directory
inventory           = inventory/

# Reduce output noise in CI/CD; use 'debug' for troubleshooting
stdout_callback     = yaml

# Number of parallel forks — default is 5, too low for large fleets
forks               = 50

# Timeout for each connection attempt
timeout             = 30

# Disable host key checking for dynamic cloud environments
# WARNING: enable this only with other network controls in place
host_key_checking   = False

# Retry failed hosts — creates a .retry file you can reuse
retry_files_enabled = True
retry_files_save_path = ~/.ansible/retry-files/

# Roles path — colon-separated for multiple locations
roles_path          = roles:~/.ansible/roles

# Collections path
collections_path    = collections:~/.ansible/collections

# Fact caching — avoid re-gathering facts on every run
fact_caching        = jsonfile
fact_caching_connection = /tmp/ansible_facts_cache
fact_caching_timeout = 3600   # 1 hour TTL

# Log everything to a file (useful for audit trails in regulated environments)
log_path            = /var/log/ansible/ansible.log

[ssh_connection]
# SSH pipelining — eliminates temporary file creation per task, huge speedup
pipelining          = True

# ControlMaster multiplexing — reuses SSH connections across tasks
ssh_args            = -o ControlMaster=auto -o ControlPersist=60s -o StrictHostKeyChecking=no

# Preferred connection method
transfer_method     = smart

[privilege_escalation]
become              = True
become_method       = sudo
become_user         = root
become_ask_pass     = False
```

```bash
# Show effective ansible.cfg settings (merged from all config files)
ansible-config dump --only-changed

# Show where Ansible is reading its config from
ansible --version
```

---

### Ad-Hoc Commands — Essential Patterns

#### Connectivity & Fact Gathering

```bash
# Ping all hosts (uses Ansible's ping module, not ICMP)
ansible all -i inventory/ -m ping

# Ping only hosts in [app] group with verbose output
ansible app -i inventory/ -m ping -v

# Gather all facts from a single host and pipe to jq
ansible web01.prod.example.com -i inventory/ -m setup | jq '.ansible_facts.ansible_distribution'

# Gather only a subset of facts (faster)
ansible web -i inventory/ -m setup -a 'filter=ansible_memory_mb'

# Check disk usage across the web tier
ansible web -i inventory/ -m command -a 'df -h /'
```

#### Service & Package Management (Incident Response)

```bash
# Restart nginx on all web hosts — idempotent and safe
ansible web -i inventory/ -m service -a 'name=nginx state=restarted' --become

# Check if a service is running without changing state
ansible app -i inventory/ -m service -a 'name=myapp state=started' --check --become

# Install a security patch across all hosts — useful for emergency CVE response
ansible all -i inventory/ -m package -a 'name=openssl state=latest' --become

# Remove a package
ansible db -i inventory/ -m package -a 'name=telnet state=absent' --become
```

#### File Operations

```bash
# Copy a file to all web hosts
ansible web -i inventory/ -m copy \
  -a 'src=/tmp/nginx.conf dest=/etc/nginx/nginx.conf owner=root mode=0644' \
  --become

# Create a directory on all app hosts
ansible app -i inventory/ -m file \
  -a 'path=/opt/myapp/logs state=directory owner=appuser mode=0755' \
  --become

# Fetch a log file from all db hosts to the control node
ansible db -i inventory/ -m fetch \
  -a 'src=/var/log/postgresql/postgresql.log dest=/tmp/pg_logs/ flat=no'
```

#### Shell & Command Modules

```bash
# Run a raw shell command (use sparingly — not idempotent)
ansible web -i inventory/ -m shell -a 'systemctl list-units --failed'

# Run a command (no shell features like pipes/redirects)
ansible all -i inventory/ -m command -a 'uptime'

# Execute with sudo in a specific working directory
ansible app -i inventory/ -m shell \
  -a 'chdir=/opt/myapp ./health_check.sh' \
  --become --become-user=appuser

# Run a raw command (bypasses Python — useful for bootstrapping)
ansible web -i inventory/ -m raw -a 'rpm -qa | grep kernel'
```

#### Limiting Targets

```bash
# Target a single host from a group
ansible web -i inventory/ -m ping --limit web01.prod.example.com

# Target hosts matching a pattern
ansible all -i inventory/ -m ping --limit 'web*'

# Exclude a host
ansible all -i inventory/ -m ping --limit 'all:!db01.prod.example.com'

# Use a retry file from a previous failed run
ansible all -i inventory/ -m ping --limit @~/.ansible/retry-files/site.retry

# Run against a random 20% of hosts (canary-style)
ansible web -i inventory/ -m ping --limit '20%'
```

#### Parallel Execution & Timing

```bash
# Override forks on the command line (useful for quick jobs)
ansible all -i inventory/ -m ping -f 20

# Time how long an ad-hoc command takes across the fleet
time ansible all -i inventory/ -m command -a 'hostname' -f 50

# Async ad-hoc command (fire and forget, check status separately)
ansible web -i inventory/ -m shell \
  -a 'sleep 30 && echo done' \
  --async 60 --poll 0
```

---

## ⚠️ Gotchas & Pro Tips

- **`ansible.cfg` location precedence:** Ansible reads config in this order: `ANSIBLE_CONFIG` env var → `./ansible.cfg` (current directory) → `~/.ansible.cfg` → `/etc/ansible/ansible.cfg`. Always use a project-local `ansible.cfg` so settings don't bleed across projects. Run `ansible --version` to confirm which config file is active.

- **`command` vs `shell` module:** The `command` module does **not** invoke a shell — no `|`, `&&`, `>`, or variable expansion. Use `shell` when you need those features, but prefer `command` for idempotency and security (avoids shell injection). Never use `raw` in production playbooks — it skips Python entirely and returns no structured output.

- **INI inventory variable types:** All variable values in INI format are **strings**. `ansible_port=22` is the string `"22"`, not the integer `22`. This bites engineers when modules do strict type checking. Use YAML inventory or `group_vars/` YAML files to preserve proper types.

- **`host_key_checking = False` in production:** Only safe when combined with network-level controls (VPC security groups, VPNs). Without those, you're vulnerable to MITM attacks. A better pattern for dynamic cloud environments is to use the `known_hosts` module in a pre-task to populate `~/.ssh/known_hosts` from AWS/VMware APIs.

- **Fact caching reduces runtime significantly:** On a 200-host fleet, fact gathering alone can take 30–60 seconds. Enabling `fact_caching = jsonfile` with a 1-hour TTL means subsequent runs skip fact gathering entirely. Combine with `gather_facts: false` in playbooks and explicit `setup` calls only where needed.

- **`pipelining = True` requires `requiretty` to be disabled:** Many default RHEL/CentOS sudoers configs have `requiretty` enabled, which breaks pipelining. Add `Defaults !requiretty` to `/etc/sudoers` (via a bootstrap playbook) or the performance gains won't materialise.

---

*Part of the [Ansible Mastery](https://github.com/naveenramasamy11/ansible-mastery) series.*
