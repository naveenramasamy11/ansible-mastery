# 🔧 Core Modules — File, Service, URI & delegate_to — Ansible Mastery

> **Ansible's built-in module library eliminates thousands of lines of shell scripting — but only engineers who know the full parameter surface and idempotency semantics of each module use them to their true potential.**

## 📖 Concept

Ansible ships with over 3,000 modules, but the day-to-day work of infrastructure automation relies on roughly two dozen. These fall into four categories: **file system modules** (file, copy, template, fetch, lineinfile, blockinfile), **service and package modules** (service, systemd, package, yum, apt, dnf), **network modules** (uri, get_url), and **delegation patterns** (delegate_to, run_once, local_action).

The **file system modules** are the backbone of configuration management. The `template` module is particularly powerful — it renders Jinja2 templates on the control node and pushes the result to managed hosts, making it the correct tool for any config file that differs per host or environment. The `copy` module is for static files. `lineinfile` and `blockinfile` are surgical tools for editing existing files without replacing them entirely — ideal for making targeted changes to config files managed by other tools.

The **service and package modules** are where idempotency matters most. The `service` module's `state` parameter accepts `started`, `stopped`, `restarted`, and `reloaded` — and only `started`/`stopped` are truly idempotent (they check current state before acting). `restarted` always restarts, which is why it belongs in handlers rather than regular tasks. The `package` module is OS-agnostic — it delegates to `yum`, `apt`, `dnf`, or `zypper` based on the target OS.

The **uri module** has become essential as infrastructure APIs proliferate. It handles Kubernetes API calls, HashiCorp Vault token renewal, PagerDuty incident management, Slack notifications, and AWS API calls that don't have dedicated modules. Understanding its authentication, retry, and pagination patterns is a core skill for modern Ansible work.

**delegate_to** fundamentally changes where a task runs — instead of on the target host, it runs on a different host (often `localhost`). This enables a critical pattern: running AWS CLI or Terraform commands on your control node while iterating over your managed host inventory.

---

## 💡 Real-World Use Cases

- Use `template` to generate per-host Prometheus scrape configs, nginx virtual host files, and JVM options based on instance type — eliminating all manual config management.
- Use `uri` to call the AWS EC2 metadata API from within a play, or to trigger a GitHub Actions workflow dispatch after a deployment completes.
- Use `delegate_to: localhost` with `run_once: true` to deregister all hosts from an ALB target group before a rolling update, then re-register them after.

---

## 🔧 Commands & Examples

### File Module — Manage Files, Directories, Symlinks

```yaml
- name: Create application directory structure
  ansible.builtin.file:
    path: "{{ item.path }}"
    state: directory
    owner: "{{ item.owner | default('appuser') }}"
    group: "{{ item.group | default('appuser') }}"
    mode: "{{ item.mode | default('0755') }}"
  loop:
    - { path: /opt/myapp }
    - { path: /opt/myapp/releases }
    - { path: /opt/myapp/shared/config }
    - { path: /opt/myapp/shared/logs, mode: '0770' }
    - { path: /etc/myapp, owner: root, group: appuser, mode: '0750' }

- name: Create symlink for current release
  ansible.builtin.file:
    src: "/opt/myapp/releases/{{ app_version }}"
    dest: /opt/myapp/current
    state: link
    force: true   # replace existing symlink

- name: Remove old release directories (keep last 5)
  ansible.builtin.file:
    path: "{{ item }}"
    state: absent
  loop: "{{ old_releases }}"   # computed elsewhere via find + set_fact

- name: Ensure a file exists with correct permissions (touch)
  ansible.builtin.file:
    path: /var/log/myapp/app.log
    state: touch
    owner: appuser
    mode: "0644"
    modification_time: preserve   # don't update mtime if file exists
    access_time: preserve
```

---

### Copy Module — Push Static Files

```yaml
- name: Copy SSL certificate to all web hosts
  ansible.builtin.copy:
    src: files/ssl/wildcard.example.com.crt
    dest: /etc/ssl/certs/wildcard.example.com.crt
    owner: root
    group: root
    mode: "0644"
  notify: Reload nginx

- name: Copy binary with checksum validation
  ansible.builtin.copy:
    src: files/myapp-{{ app_version }}-linux-amd64
    dest: /usr/local/bin/myapp
    owner: root
    mode: "0755"
    checksum: "sha256:{{ app_binary_checksum }}"  # validates after copy

- name: Write content directly to a file (no local source needed)
  ansible.builtin.copy:
    content: |
      [Unit]
      Description=MyApp Service
      After=network.target

      [Service]
      User=appuser
      ExecStart=/usr/local/bin/myapp serve
      Restart=always

      [Install]
      WantedBy=multi-user.target
    dest: /etc/systemd/system/myapp.service
    owner: root
    mode: "0644"
  notify:
    - Reload systemd
    - Restart myapp

- name: Copy file only if destination does not exist
  ansible.builtin.copy:
    src: files/app.conf.defaults
    dest: /etc/myapp/app.conf
    force: false   # idempotent — never overwrites existing file
```

---

### Template Module — Jinja2-Rendered Configs

```yaml
# templates/nginx.conf.j2
# upstream {{ app_name }}_backend {
#   {% for host in groups['app'] %}
#   server {{ hostvars[host]['ansible_host'] }}:{{ app_port }};
#   {% endfor %}
# }
#
# server {
#   listen 80;
#   server_name {{ nginx_server_name }};
#   location / {
#     proxy_pass http://{{ app_name }}_backend;
#   }
# }

- name: Deploy nginx config from template
  ansible.builtin.template:
    src: templates/nginx.conf.j2
    dest: /etc/nginx/nginx.conf
    owner: root
    group: root
    mode: "0644"
    validate: /usr/sbin/nginx -t -c %s    # validate BEFORE writing
    backup: true   # keep a backup of the previous version
  notify: Reload nginx

- name: Deploy per-host JVM options
  ansible.builtin.template:
    src: templates/jvm.options.j2
    dest: /etc/myapp/jvm.options
  vars:
    # Instance-type-specific heap sizes
    heap_size: "{{ ansible_memtotal_mb | int // 2 }}m"

- name: Generate Prometheus node-exporter scrape config
  ansible.builtin.template:
    src: templates/prometheus_targets.yml.j2
    dest: /etc/prometheus/targets/{{ inventory_hostname }}.yml
  delegate_to: "{{ prometheus_host }}"
```

---

### Fetch Module — Pull Files from Managed Hosts

```yaml
- name: Fetch SSL certificate from each host for auditing
  ansible.builtin.fetch:
    src: /etc/ssl/certs/site.crt
    dest: /tmp/certs/{{ inventory_hostname }}/
    flat: false   # preserves source path under dest/hostname/

- name: Fetch app logs for debugging (flat mode)
  ansible.builtin.fetch:
    src: /var/log/myapp/error.log
    dest: /tmp/logs/{{ inventory_hostname }}-error.log
    flat: true    # single file, no directory structure

- name: Fetch and validate /etc/hosts
  ansible.builtin.fetch:
    src: /etc/hosts
    dest: /tmp/hosts_audit/
    validate_checksum: true
```

---

### Service & Systemd Module

```yaml
- name: Ensure nginx is running and enabled
  ansible.builtin.service:
    name: nginx
    state: started
    enabled: true   # enable at boot

- name: Stop and disable a service
  ansible.builtin.service:
    name: telnet
    state: stopped
    enabled: false

# Use systemd module for fine-grained systemd control
- name: Reload systemd after unit file change
  ansible.builtin.systemd:
    daemon_reload: true

- name: Start service with timeout and force
  ansible.builtin.systemd:
    name: myapp
    state: started
    enabled: true
    force: true

- name: Mask a service to prevent starting
  ansible.builtin.systemd:
    name: bluetooth
    masked: true

- name: Get service status
  ansible.builtin.systemd:
    name: nginx
  register: nginx_status

- name: Debug nginx state
  ansible.builtin.debug:
    msg: "nginx is {{ nginx_status.status.ActiveState }}"
```

---

### Package Management

```yaml
# OS-agnostic (uses yum/apt/dnf/zypper automatically)
- name: Install required packages
  ansible.builtin.package:
    name:
      - curl
      - jq
      - unzip
      - python3-boto3
    state: present

- name: Remove unwanted packages
  ansible.builtin.package:
    name:
      - telnet
      - rsh-server
    state: absent

# yum-specific (RHEL/CentOS/Amazon Linux)
- name: Install specific version of Java
  ansible.builtin.yum:
    name: java-17-openjdk-17.0.5.0.8-2.el8
    state: present
    update_cache: true

- name: Install from local RPM
  ansible.builtin.yum:
    name: /tmp/myapp-2.4.1-1.el8.x86_64.rpm
    state: present

- name: Upgrade all packages (security patches only)
  ansible.builtin.yum:
    name: "*"
    state: latest
    security: true   # only apply security updates

# apt-specific (Ubuntu/Debian)
- name: Add repository and install package
  block:
    - name: Add Docker GPG key
      ansible.builtin.apt_key:
        url: https://download.docker.com/linux/ubuntu/gpg
        state: present

    - name: Add Docker repository
      ansible.builtin.apt_repository:
        repo: "deb [arch=amd64] https://download.docker.com/linux/ubuntu {{ ansible_distribution_release }} stable"
        state: present

    - name: Install Docker CE
      ansible.builtin.apt:
        name: docker-ce
        state: present
        update_cache: true
```

---

### URI Module — REST API Calls

```yaml
# Basic GET request
- name: Check application health endpoint
  ansible.builtin.uri:
    url: "http://{{ ansible_host }}:8080/health"
    method: GET
    status_code: 200
    timeout: 10
  register: health_check
  retries: 5
  delay: 10
  until: health_check.status == 200

# POST with JSON body and auth
- name: Trigger GitHub Actions workflow dispatch
  ansible.builtin.uri:
    url: "https://api.github.com/repos/{{ github_org }}/{{ github_repo }}/actions/workflows/deploy.yml/dispatches"
    method: POST
    headers:
      Authorization: "Bearer {{ github_token }}"
      Accept: application/vnd.github+json
    body_format: json
    body:
      ref: main
      inputs:
        environment: "{{ env }}"
        version: "{{ app_version }}"
    status_code: 204
  delegate_to: localhost

# Send Slack notification
- name: Notify Slack of deployment
  ansible.builtin.uri:
    url: "{{ slack_webhook_url }}"
    method: POST
    body_format: json
    body:
      channel: "#deployments"
      text: ":rocket: Deployed {{ app_version }} to {{ env }} on {{ ansible_play_hosts | length }} hosts"
    status_code: 200
  delegate_to: localhost
  run_once: true

# Interact with HashiCorp Vault
- name: Get Vault token
  ansible.builtin.uri:
    url: "{{ vault_addr }}/v1/auth/aws/login"
    method: POST
    body_format: json
    body:
      role: "{{ vault_role }}"
      pkcs7: "{{ lookup('url', 'http://169.254.169.254/latest/dynamic/instance-identity/pkcs7') | replace('\n', '') }}"
    return_content: true
  register: vault_auth
  delegate_to: localhost

- name: Read secret from Vault
  ansible.builtin.uri:
    url: "{{ vault_addr }}/v1/secret/data/myapp/{{ env }}"
    method: GET
    headers:
      X-Vault-Token: "{{ vault_auth.json.auth.client_token }}"
    return_content: true
  register: vault_secret
  delegate_to: localhost

# Paginated API calls
- name: Get all EC2 instances via AWS API (paginated)
  ansible.builtin.uri:
    url: "https://ec2.{{ aws_region }}.amazonaws.com/"
    method: GET
    aws_access_key: "{{ aws_access_key_id }}"
    # ... use amazon.aws collection modules for AWS instead
```

---

### delegate_to & run_once Patterns

```yaml
# Pattern 1: De/register from load balancer during rolling update
- name: Deregister host from ALB target group
  amazon.aws.elb_target:
    target_group_arn: "{{ alb_target_group_arn }}"
    target_id: "{{ ansible_host }}"
    target_port: 8080
    state: absent
    region: "{{ aws_region }}"
  delegate_to: localhost

- name: Perform upgrade
  ansible.builtin.package:
    name: myapp
    state: latest
  become: true
  notify: Restart myapp

- name: Wait for service to be healthy
  ansible.builtin.uri:
    url: "http://{{ ansible_host }}:8080/health"
    status_code: 200
  retries: 20
  delay: 5

- name: Re-register host with ALB target group
  amazon.aws.elb_target:
    target_group_arn: "{{ alb_target_group_arn }}"
    target_id: "{{ ansible_host }}"
    target_port: 8080
    state: present
    region: "{{ aws_region }}"
  delegate_to: localhost

# Pattern 2: Run a task once across all hosts (e.g. DB migration)
- name: Run database migrations (once, on first app host)
  ansible.builtin.command:
    cmd: /opt/myapp/bin/migrate --db-url {{ db_url }}
    chdir: /opt/myapp
  run_once: true
  delegate_to: "{{ groups['app'][0] }}"

# Pattern 3: Gather facts from a host you're not targeting
- name: Gather facts from bastion host
  ansible.builtin.setup:
  delegate_to: bastion.example.com
  delegate_facts: true   # store facts as bastion.example.com's facts

# Pattern 4: Write results to control node
- name: Save deployment manifest to control node
  ansible.builtin.copy:
    content: "{{ deployment_manifest | to_nice_yaml }}"
    dest: "/tmp/deployments/{{ env }}-{{ app_version }}-{{ ansible_date_time.epoch }}.yml"
  delegate_to: localhost
  run_once: true
```

---

### lineinfile & blockinfile — Surgical File Editing

```yaml
- name: Ensure sysctl setting for Kubernetes
  ansible.builtin.lineinfile:
    path: /etc/sysctl.d/99-kubernetes.conf
    regexp: '^net.bridge.bridge-nf-call-iptables'
    line: 'net.bridge.bridge-nf-call-iptables = 1'
    create: true
    owner: root
    mode: "0644"

- name: Add environment variable to /etc/environment
  ansible.builtin.lineinfile:
    path: /etc/environment
    regexp: '^JAVA_HOME='
    line: 'JAVA_HOME=/usr/lib/jvm/java-17'
    state: present

- name: Insert a block of config into nginx.conf
  ansible.builtin.blockinfile:
    path: /etc/nginx/nginx.conf
    marker: "# {mark} ANSIBLE MANAGED BLOCK — rate limiting"
    insertafter: "http {"
    block: |
      limit_req_zone $binary_remote_addr zone=api:10m rate=10r/s;
      limit_req_status 429;
```

---

## ⚠️ Gotchas & Pro Tips

- **`template` validate runs BEFORE the file is written:** The `%s` placeholder in `validate` points to a temp file, not the destination. Make sure the validator command accepts a file path argument and doesn't have side effects. For nginx, `nginx -t -c %s` works perfectly. For JSON configs, use `python3 -m json.tool %s > /dev/null`.

- **`copy` with `force: false` vs `lineinfile`:** `copy` with `force: false` will never update a file once it exists — even if the source changes. This is useful for config files that admins may have modified locally. Use `lineinfile` or `blockinfile` when you need to manage specific lines in a file that may have other hand-managed content.

- **`service` vs `systemd` module:** The `service` module is OS-agnostic (works with SysV, systemd, OpenRC). The `systemd` module gives you systemd-specific features: `daemon_reload`, `masked`, `scope`, and access to the full unit status object. Always use `systemd` when you need `daemon_reload: true` — using `command: systemctl daemon-reload` is less idempotent.

- **`delegate_to: localhost` and inventory variables:** When a task is delegated to localhost, variable resolution still happens against the **current inventory host** (the one being iterated). So `{{ ansible_host }}` refers to the managed host, not localhost. This is usually what you want — but `ansible_user`, `ansible_become`, and connection variables may also resolve from the current host context, which can cause authentication issues if localhost needs different credentials.

- **`uri` return_content performance:** `return_content: true` stores the full HTTP body in memory on the control node. For large API responses (AWS describe-instances on 1000 instances), this can be significant. Use `return_content: false` unless you need to parse the body, and use dedicated AWS collection modules (amazon.aws) for AWS API calls rather than raw `uri`.

---

*Part of the [Ansible Mastery](https://github.com/naveenramasamy11/ansible-mastery) series.*
