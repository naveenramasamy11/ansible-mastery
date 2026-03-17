# 🔧 Async Tasks, Dynamic Inventory & Custom Modules — Ansible Mastery

> **The engineers who push Ansible to its limits — running against 10,000 nodes, building zero-downtime pipelines, extending Ansible with custom modules — operate at the intersection of these three advanced patterns.**

## 📖 Concept

**Async tasks and polling** solve Ansible's default blocking execution model. By default, every task waits for completion before moving to the next — fine for most operations, but problematic for long-running tasks (OS updates, large package installs, application restarts). Async tasks fire and immediately return a job ID; Ansible can check status later with `async_status`. The fire-and-forget pattern (`poll: 0`) is especially useful for triggering parallel operations across hundreds of hosts without serialising them through the control node.

**Dynamic inventory** is the bridge between Ansible and ephemeral, API-driven infrastructure. Static INI/YAML inventory files don't scale when you have auto-scaling EC2 groups, freshly-provisioned EKS nodes, or Terraform-managed VMs. Dynamic inventory plugins (or scripts) call APIs at playbook start to build the inventory in real time. The `amazon.aws.aws_ec2` inventory plugin is the de-facto standard for AWS environments — it queries EC2 and organises hosts by tags, instance type, VPC, and dozens of other attributes.

**Custom modules** are Python scripts that implement the `AnsibleModule` API. You write them when no existing module does exactly what you need — wrapping an internal API, integrating with a proprietary system, or encapsulating complex logic that would otherwise live in 20 shell tasks. A well-written custom module is idempotent, handles errors properly, returns structured data, and supports `--check` mode. Once written, it's indistinguishable from a built-in module to playbook authors.

**Callback plugins** intercept Ansible events (task start, task result, play complete) and let you customise output, ship metrics to monitoring systems, or trigger external workflows. The built-in `profile_tasks` callback identifies slow tasks; `mail` and `slack` callbacks send notifications; custom callbacks can push results to Splunk, Datadog, or a database.

---

## 💡 Real-World Use Cases

- Use async tasks to parallelize a 30-minute OS patching job across 200 hosts — launch all patches async with `poll: 0`, then poll status in a separate task, cutting total runtime from 100 hours to 30 minutes.
- Use the `aws_ec2` dynamic inventory plugin to automatically target all EC2 instances tagged `Role=web` in `env=production` — no inventory file maintenance, auto-adapts to auto-scaling events.
- Write a custom module to interact with your organisation's internal CMDB API for host registration/deregistration during deployment — making it a first-class Ansible module with proper idempotency and check mode support.

---

## 🔧 Commands & Examples

### Async Tasks & Polling

```yaml
# Pattern 1: Fire-and-forget parallel tasks
- name: Trigger OS patching on all hosts asynchronously
  ansible.builtin.yum:
    name: "*"
    state: latest
    security: true
  async: 1800        # task timeout: 30 minutes
  poll: 0            # don't wait — fire and forget
  register: patch_jobs

- name: Wait for all patching jobs to complete
  ansible.builtin.async_status:
    jid: "{{ item.ansible_job_id }}"
  register: job_status
  loop: "{{ patch_jobs.results }}"
  until: job_status.finished
  retries: 60
  delay: 30
  when: item.ansible_job_id is defined

- name: Report patching results
  ansible.builtin.debug:
    msg: "Host {{ item.item.inventory_hostname }}: {{ 'success' if item.rc == 0 else 'FAILED' }}"
  loop: "{{ job_status.results }}"
  when: item.finished
```

```yaml
# Pattern 2: Long-running task with progressive polling
- name: Run database backup (up to 2 hours)
  ansible.builtin.shell: /opt/myapp/bin/backup.sh --full --compress
  async: 7200
  poll: 60           # check every 60 seconds
  register: backup_result

# poll > 0 blocks until done but shows progress every 60s
# Use this for single slow tasks that must complete before continuing
```

```yaml
# Pattern 3: Async per-host with centralized status check
- name: Start long-running migration on all db hosts
  ansible.builtin.command:
    cmd: /opt/myapp/bin/migrate --target={{ target_schema_version }}
    chdir: /opt/myapp
  async: 3600
  poll: 0
  register: migration_jobs

- name: Check migration status on all hosts
  ansible.builtin.async_status:
    jid: "{{ hostvars[item]['migration_jobs']['ansible_job_id'] }}"
  delegate_to: "{{ item }}"
  register: migration_status
  loop: "{{ groups['db'] }}"
  until: migration_status.finished
  retries: 120
  delay: 30
```

---

### Dynamic Inventory — AWS EC2 Plugin

```yaml
# inventory/aws_ec2.yml — AWS EC2 dynamic inventory plugin
---
plugin: amazon.aws.aws_ec2
regions:
  - us-east-1
  - us-west-2

# Filter: only running instances with specific tags
filters:
  instance-state-name: running
  tag:ManagedBy: ansible

# Group hosts by tag values
keyed_groups:
  - key: tags.Environment
    prefix: env
    separator: "_"
  - key: tags.Role
    prefix: role
    separator: "_"
  - key: instance_type
    prefix: instance_type
  - key: placement.availability_zone
    prefix: az

# Additional groupings
groups:
  web_prod: "'web' in tags.Role and 'production' in tags.Environment"
  app_staging: "'app' in tags.Role and 'staging' in tags.Environment"
  large_instances: "instance_type.startswith('m5.4xlarge') or instance_type.startswith('c5.4xlarge')"

# Use private IP for connections (within VPC)
hostnames:
  - private-ip-address
  - tag:Name

# Variables to include for each host
compose:
  ansible_host: private_ip_address
  ansible_user: "'ec2-user' if 'amzn' in image_id else 'ubuntu'"
  instance_name: tags.Name
  instance_env: tags.Environment

# Cache settings (avoids API calls on every run)
cache: true
cache_plugin: jsonfile
cache_connection: /tmp/aws_inventory_cache
cache_timeout: 300   # 5 minutes

# IAM profile for authentication (no static keys needed)
iam_role_arn: arn:aws:iam::123456789:role/AnsibleInventoryRole
```

```bash
# Test dynamic inventory
ansible-inventory -i inventory/aws_ec2.yml --list | jq '.role_web'
ansible-inventory -i inventory/aws_ec2.yml --graph

# Run against dynamically-discovered web hosts
ansible-playbook site.yml -i inventory/aws_ec2.yml --limit role_web

# Refresh cache manually
ansible-inventory -i inventory/aws_ec2.yml --list --refresh
```

---

### Dynamic Inventory — Terraform State Plugin

```yaml
# inventory/terraform.yml — read inventory from Terraform state
---
plugin: cloud.terraform.terraform_state
project_path: ../terraform/environments/prod
search_child_modules: true
backend_type: s3
backend_config:
  bucket: myorg-terraform-state
  key: prod/terraform.tfstate
  region: us-east-1
```

```python
#!/usr/bin/env python3
# inventory/terraform_json.py — custom script for Terraform output-based inventory
# Usage: ansible-playbook site.yml -i inventory/terraform_json.py

import json
import subprocess
import sys

def get_terraform_output():
    result = subprocess.run(
        ['terraform', 'output', '-json'],
        capture_output=True, text=True,
        cwd='../terraform/environments/prod'
    )
    return json.loads(result.stdout)

def build_inventory():
    tf_output = get_terraform_output()
    inventory = {
        '_meta': {'hostvars': {}},
        'web': {'hosts': [], 'vars': {'app_port': 80}},
        'app': {'hosts': [], 'vars': {'app_port': 8080}},
        'db': {'hosts': [], 'vars': {}},
    }

    for instance in tf_output.get('web_instances', {}).get('value', []):
        hostname = instance['private_ip']
        inventory['web']['hosts'].append(hostname)
        inventory['_meta']['hostvars'][hostname] = {
            'ansible_host': instance['private_ip'],
            'instance_id': instance['id'],
            'instance_type': instance['type'],
        }

    return inventory

if __name__ == '__main__':
    if '--list' in sys.argv:
        print(json.dumps(build_inventory(), indent=2))
    elif '--host' in sys.argv:
        print(json.dumps({}))
```

---

### Custom Python Modules

```python
#!/usr/bin/env python3
# library/cmdb_host.py — custom module for CMDB integration

from ansible.module_utils.basic import AnsibleModule
import requests

DOCUMENTATION = '''
---
module: cmdb_host
short_description: Register or deregister a host in internal CMDB
description:
  - Manages host records in the internal CMDB API
  - Supports check mode
options:
  hostname:
    description: FQDN of the host to manage
    required: true
  state:
    description: Whether the host should be present or absent
    choices: [present, absent]
    default: present
  environment:
    description: Environment tag
    required: true
  role:
    description: Host role
    required: true
  cmdb_url:
    description: CMDB API base URL
    required: true
  cmdb_token:
    description: API auth token
    required: true
    no_log: true
'''

def get_host(cmdb_url, token, hostname):
    resp = requests.get(
        f"{cmdb_url}/api/v1/hosts/{hostname}",
        headers={"Authorization": f"Bearer {token}"},
        timeout=10
    )
    if resp.status_code == 404:
        return None
    resp.raise_for_status()
    return resp.json()

def create_host(cmdb_url, token, hostname, environment, role):
    resp = requests.post(
        f"{cmdb_url}/api/v1/hosts",
        json={"hostname": hostname, "environment": environment, "role": role},
        headers={"Authorization": f"Bearer {token}"},
        timeout=10
    )
    resp.raise_for_status()
    return resp.json()

def delete_host(cmdb_url, token, hostname):
    resp = requests.delete(
        f"{cmdb_url}/api/v1/hosts/{hostname}",
        headers={"Authorization": f"Bearer {token}"},
        timeout=10
    )
    resp.raise_for_status()

def main():
    module = AnsibleModule(
        argument_spec=dict(
            hostname=dict(type='str', required=True),
            state=dict(type='str', default='present', choices=['present', 'absent']),
            environment=dict(type='str', required=True),
            role=dict(type='str', required=True),
            cmdb_url=dict(type='str', required=True),
            cmdb_token=dict(type='str', required=True, no_log=True),
        ),
        supports_check_mode=True
    )

    hostname = module.params['hostname']
    state = module.params['state']
    env = module.params['environment']
    role = module.params['role']
    cmdb_url = module.params['cmdb_url']
    token = module.params['cmdb_token']

    try:
        existing = get_host(cmdb_url, token, hostname)

        if state == 'present':
            if existing:
                module.exit_json(changed=False, host=existing, msg="Host already registered")
            else:
                if module.check_mode:
                    module.exit_json(changed=True, msg="Would register host (check mode)")
                result = create_host(cmdb_url, token, hostname, env, role)
                module.exit_json(changed=True, host=result, msg="Host registered")

        elif state == 'absent':
            if not existing:
                module.exit_json(changed=False, msg="Host not found, nothing to remove")
            else:
                if module.check_mode:
                    module.exit_json(changed=True, msg="Would deregister host (check mode)")
                delete_host(cmdb_url, token, hostname)
                module.exit_json(changed=True, msg="Host deregistered")

    except requests.RequestException as e:
        module.fail_json(msg=f"CMDB API error: {str(e)}")

if __name__ == '__main__':
    main()
```

```yaml
# Using the custom module in a playbook
- name: Register host in CMDB on provisioning
  cmdb_host:
    hostname: "{{ inventory_hostname }}"
    state: present
    environment: "{{ env }}"
    role: "{{ host_role }}"
    cmdb_url: "{{ cmdb_api_url }}"
    cmdb_token: "{{ vault_cmdb_token }}"

# Test in check mode first
# ansible-playbook provision.yml --check
```

---

### Callback Plugins

```ini
# ansible.cfg — enable callback plugins
[defaults]
callback_plugins   = ./callback_plugins
stdout_callback    = yaml        # or: json, minimal, debug
callback_whitelist = profile_tasks, timer, slack_notify
```

```python
# callback_plugins/slack_notify.py — post results to Slack
from ansible.plugins.callback import CallbackBase
import requests, json

class CallbackModule(CallbackBase):
    CALLBACK_VERSION = 2.0
    CALLBACK_TYPE = 'notification'
    CALLBACK_NAME = 'slack_notify'
    CALLBACK_NEEDS_WHITELIST = True

    def __init__(self):
        super().__init__()
        self._webhook_url = None
        self._failed_tasks = []

    def set_options(self, task_keys=None, var_options=None, direct=None):
        super().set_options(task_keys=task_keys, var_options=var_options, direct=direct)
        self._webhook_url = self.get_option('webhook_url')

    def v2_runner_on_failed(self, result, ignore_errors=False):
        if not ignore_errors:
            self._failed_tasks.append({
                'host': result._host.get_name(),
                'task': result._task.get_name(),
                'msg': result._result.get('msg', 'Unknown error')
            })

    def v2_playbook_on_stats(self, stats):
        if self._failed_tasks and self._webhook_url:
            msg = f":red_circle: Playbook completed with {len(self._failed_tasks)} failures:\n"
            for t in self._failed_tasks:
                msg += f"• {t['host']} — {t['task']}: {t['msg']}\n"
            requests.post(self._webhook_url, json={"text": msg}, timeout=10)
```

```bash
# Use the built-in profile_tasks callback to identify slow tasks
ANSIBLE_CALLBACKS_ENABLED=profile_tasks ansible-playbook site.yml

# Use timer callback for total runtime
ANSIBLE_CALLBACKS_ENABLED=timer ansible-playbook site.yml

# JSON output for CI/CD parsing
ANSIBLE_STDOUT_CALLBACK=json ansible-playbook site.yml 2>&1 | jq '.plays[].tasks[].hosts'
```

---

## ⚠️ Gotchas & Pro Tips

- **Async job IDs are per-host, not global:** Each managed host gets its own `ansible_job_id`. When you use `async` in a loop across hosts, `patch_jobs.results` is a list — one result per host. Loop over this list in `async_status` to check each host's job. Mixing up the per-host vs global nature of async job IDs is the most common async bug.

- **Dynamic inventory caching is critical for large fleets:** Without caching, the EC2 dynamic inventory plugin makes API calls on every playbook run. For a 500-instance fleet, this adds 10–30 seconds of startup time. Set `cache: true` and `cache_timeout: 300` in the plugin config. Invalidate manually with `--refresh` when you need fresh data.

- **Custom modules must handle `check_mode`:** If your module doesn't check `module.check_mode` before making changes, `--check` mode will still execute it, potentially breaking check mode for entire playbooks that include your module. Always add `if module.check_mode: module.exit_json(changed=True, msg="Would make change (check mode)")` before any state-changing API calls.

- **`forks` vs `serial` — they serve different purposes:** `forks` (in `ansible.cfg` or `-f`) controls how many hosts Ansible communicates with **simultaneously** at the connection layer. `serial` controls how many hosts receive each **wave of tasks** in a play. On a 200-host fleet with `forks=50` and `serial=20`, Ansible opens up to 50 SSH connections at once but delivers tasks in batches of 20 hosts. Tune `forks` for network/SSH capacity; tune `serial` for deployment safety.

- **Dynamic inventory with `compose` avoids separate `group_vars`:** The `compose` section in `aws_ec2.yml` lets you define host variables directly from EC2 attributes (`ansible_user`, `ansible_host`, custom vars from tags). This eliminates the need for `host_vars` files for dynamically-discovered hosts and keeps all per-host config in the inventory plugin config where it's visible alongside the filter logic.

---

*Part of the [Ansible Mastery](https://github.com/naveenramasamy11/ansible-mastery) series.*
