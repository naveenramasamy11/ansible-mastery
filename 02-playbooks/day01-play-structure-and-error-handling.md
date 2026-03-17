# 🔧 Play Structure & Error Handling — Ansible Mastery

> **Understanding play anatomy and robust error handling is the difference between playbooks that limp through failures and automation you can trust in production at 2 AM.**

## 📖 Concept

A **play** is the fundamental unit of Ansible orchestration. It binds a set of hosts (from inventory) to a set of tasks, and every design decision you make — handler ordering, tag structure, error recovery — happens at the play level. Most engineers learn the basics of a play quickly but miss the subtle mechanics that matter at scale: when handlers actually fire, how `any_errors_fatal` interacts with `serial`, and why `block/rescue/always` is your single most powerful tool for resilient automation.

**Handlers** are one of Ansible's most misunderstood features. They are tasks that only run when notified by another task, and they run **once** at the end of a play regardless of how many tasks notified them. This is exactly right for service restarts — you don't want nginx restarted five times because five config files changed. But the default behaviour means handlers don't run if a play fails mid-way. `force_handlers: true` and `flush_handlers` exist precisely to solve this in production scenarios where you need cleanup to happen even after failure.

**Tags** allow surgical execution — deploying only the app layer, or running only smoke tests, without touching the full play. In CI/CD pipelines that use Ansible (GitHub Actions + Ansible, Jenkins, GitLab CI), tags are the primary mechanism for separating deployment stages. Engineers who don't use tags find themselves running full 20-minute playbooks to push a config file change.

**block/rescue/always** is Ansible's equivalent of try/catch/finally, and it is indispensable for production automation. `block` groups tasks that might fail. `rescue` runs only when a block fails — perfect for rollback logic. `always` runs regardless — perfect for cleanup, notifications, and closing database connections. For complex AWS provisioning flows, a block/rescue/always pattern ensures you never leave orphaned resources on a failed run.

---

## 💡 Real-World Use Cases

- Use `block/rescue/always` in an EC2 provisioning play: block creates the instance and configures it, rescue terminates it on failure (no zombie instances), always sends a Slack/PagerDuty notification either way.
- Use `tags` to split a 40-task Kubernetes deployment playbook into `bootstrap`, `deploy`, `smoke-test`, and `rollback` stages — each runnable independently from GitHub Actions.
- Use `serial: "20%"` with `max_fail_percentage: 10` for a rolling nginx restart across 100 web hosts — halt the rollout automatically if more than 10% fail.

---

## 🔧 Commands & Examples

### Play Anatomy — Full Reference

```yaml
# site.yml — complete play structure with all major knobs
---
- name: Deploy application to web tier
  hosts: web                          # inventory group, pattern, or all
  order: sorted                       # sorted | reverse_sorted | reverse_inventory | shuffle
  gather_facts: true                  # default true; set false for speed if facts not needed
  gather_subset:                      # limit which facts are gathered
    - network
    - hardware
  gather_timeout: 10                  # seconds before fact gathering times out
  any_errors_fatal: false             # true = abort entire play on any host failure
  max_fail_percentage: 10             # abort play if >10% of hosts fail
  serial: 5                           # process 5 hosts at a time (rolling updates)
  force_handlers: true                # run handlers even if the play fails
  ignore_unreachable: false           # true = treat unreachable hosts as non-fatal
  become: true
  become_user: root
  environment:                        # environment variables set for all tasks in the play
    JAVA_HOME: /usr/lib/jvm/java-17
    APP_ENV: production

  vars:
    app_version: "2.4.1"
    deploy_dir: /opt/myapp

  vars_files:
    - vars/common.yml
    - vars/prod.yml

  pre_tasks:
    - name: Check disk space before deploying
      ansible.builtin.command: df -h /opt
      register: disk_usage
      changed_when: false

    - name: Fail if disk is over 85% full
      ansible.builtin.fail:
        msg: "Disk usage critical: {{ disk_usage.stdout }}"
      when: "'8[5-9]%\|9[0-9]%\|100%' is search(disk_usage.stdout)"

  roles:
    - role: base_hardening
      tags: [hardening]
    - role: app_deploy
      tags: [deploy]

  tasks:
    - name: Place application config
      ansible.builtin.template:
        src: templates/app.conf.j2
        dest: /etc/myapp/app.conf
        owner: appuser
        mode: "0640"
      notify: Restart application

    - name: Ensure application is started
      ansible.builtin.service:
        name: myapp
        state: started
        enabled: true

  post_tasks:
    - name: Run smoke test
      ansible.builtin.uri:
        url: "http://{{ ansible_host }}:8080/health"
        status_code: 200
      register: smoke
      retries: 5
      delay: 10
      until: smoke.status == 200

  handlers:
    - name: Restart application
      ansible.builtin.service:
        name: myapp
        state: restarted
      listen: "Restart application"
```

---

### Handlers — Advanced Patterns

```yaml
# Handlers execute ONCE at end of play, in definition order, not notification order
handlers:
  - name: Reload nginx config
    ansible.builtin.service:
      name: nginx
      state: reloaded

  - name: Restart nginx
    ansible.builtin.service:
      name: nginx
      state: restarted

  # Handler chains — one handler notifying another
  - name: Validate nginx config
    ansible.builtin.command: nginx -t
    notify: Reload nginx config

tasks:
  - name: Update nginx.conf
    ansible.builtin.template:
      src: nginx.conf.j2
      dest: /etc/nginx/nginx.conf
    notify: Validate nginx config    # triggers chain: validate → reload

  # Force handlers to run NOW (mid-play), not at end
  - name: Flush handlers before smoke test
    ansible.builtin.meta: flush_handlers

  - name: Smoke test nginx
    ansible.builtin.uri:
      url: http://localhost/health
      status_code: 200
```

```bash
# Run a playbook and show what would be notified (check mode)
ansible-playbook site.yml --check --diff

# List all handlers defined in the play
ansible-playbook site.yml --list-tasks | grep -i handler
```

---

### Tags — Selective Execution

```yaml
---
- name: Full stack deployment
  hosts: all
  tasks:
    - name: Install base packages
      ansible.builtin.package:
        name: "{{ item }}"
        state: present
      loop:
        - curl
        - jq
        - git
      tags:
        - bootstrap
        - packages

    - name: Deploy application artifact
      ansible.builtin.copy:
        src: dist/myapp-{{ app_version }}.tar.gz
        dest: /opt/myapp/
      tags:
        - deploy
        - always    # 'always' is a special tag — runs even with --skip-tags

    - name: Run database migrations
      ansible.builtin.command: /opt/myapp/bin/migrate
      tags:
        - deploy
        - migrations

    - name: Run integration tests
      ansible.builtin.shell: /opt/myapp/bin/test --suite integration
      tags:
        - test
        - smoke

    - name: Rollback to previous version
      ansible.builtin.shell: /opt/myapp/bin/rollback
      tags:
        - rollback
        - never      # 'never' tag — ONLY runs when explicitly targeted
```

```bash
# Run only bootstrap + package tasks
ansible-playbook site.yml --tags "bootstrap,packages"

# Run everything EXCEPT tests
ansible-playbook site.yml --skip-tags "test"

# Run only rollback (tagged 'never' — requires explicit tag)
ansible-playbook site.yml --tags "rollback"

# List all tasks with their tags before running
ansible-playbook site.yml --list-tasks

# Show only tasks that would run with a given tag set
ansible-playbook site.yml --tags "deploy" --list-tasks
```

---

### block / rescue / always — Robust Error Handling

```yaml
# Pattern 1: Basic try/catch/finally
- name: Deploy with rollback on failure
  hosts: app
  tasks:
    - name: Application deployment block
      block:
        - name: Stop application
          ansible.builtin.service:
            name: myapp
            state: stopped

        - name: Deploy new artifact
          ansible.builtin.unarchive:
            src: dist/myapp-{{ app_version }}.tar.gz
            dest: /opt/myapp/
            remote_src: false

        - name: Run migrations
          ansible.builtin.command:
            cmd: /opt/myapp/bin/migrate
            chdir: /opt/myapp
          register: migration_result

        - name: Start application
          ansible.builtin.service:
            name: myapp
            state: started

        - name: Smoke test
          ansible.builtin.uri:
            url: "http://localhost:8080/health"
            status_code: 200
          retries: 10
          delay: 5

      rescue:
        - name: Log failure
          ansible.builtin.debug:
            msg: "Deployment failed: {{ ansible_failed_result }}"

        - name: Rollback to previous version
          ansible.builtin.shell: |
            cd /opt/myapp
            ln -sfn releases/previous current
            systemctl restart myapp

        - name: Notify Slack on failure
          ansible.builtin.uri:
            url: "{{ slack_webhook_url }}"
            method: POST
            body_format: json
            body:
              text: ":red_circle: Deployment of {{ app_version }} FAILED on {{ inventory_hostname }}. Rolled back."

      always:
        - name: Ensure application is running
          ansible.builtin.service:
            name: myapp
            state: started
          ignore_errors: true

        - name: Record deployment outcome
          ansible.builtin.lineinfile:
            path: /var/log/deployments.log
            line: "{{ ansible_date_time.iso8601 }} | {{ app_version }} | {{ ansible_failed_result is defined | ternary('FAILED', 'SUCCESS') }}"
            create: true
```

```yaml
# Pattern 2: Nested blocks for fine-grained recovery
- name: AWS infrastructure with nested error handling
  hosts: localhost
  tasks:
    - name: Provision infrastructure
      block:
        - name: Create VPC
          amazon.aws.ec2_vpc_net:
            name: myapp-vpc
            cidr_block: 10.0.0.0/16
            region: us-east-1
            state: present
          register: vpc

        - name: Create instances
          block:
            - name: Launch EC2 instances
              amazon.aws.ec2_instance:
                name: myapp-{{ item }}
                instance_type: t3.medium
                image_id: ami-0abcdef1234567890
                vpc_subnet_id: "{{ subnet.subnet.id }}"
                region: us-east-1
                state: running
              loop: [web01, web02, web03]
              register: instances

          rescue:
            - name: Terminate any partially-created instances
              amazon.aws.ec2_instance:
                instance_ids: "{{ instances.results | selectattr('instance_ids', 'defined') | map(attribute='instance_ids') | flatten }}"
                state: absent
                region: us-east-1
              when: instances is defined

      rescue:
        - name: Delete VPC if provisioning failed
          amazon.aws.ec2_vpc_net:
            vpc_id: "{{ vpc.vpc.id }}"
            state: absent
            region: us-east-1
          when: vpc is defined
```

```yaml
# Pattern 3: ignore_errors + failed_when for nuanced control
- name: Check if service exists before managing it
  ansible.builtin.command: systemctl status myapp
  register: svc_check
  ignore_errors: true           # don't abort if command fails
  changed_when: false           # this command never changes state

- name: Install service only if not present
  ansible.builtin.package:
    name: myapp
    state: present
  when: svc_check.rc != 0

- name: Run test suite — allow up to 5 failures
  ansible.builtin.command: /opt/myapp/bin/test
  register: test_result
  failed_when:
    - test_result.rc != 0
    - "'5 tests failed' not in test_result.stdout"   # tolerate up to 5 test failures
```

---

### Serial Execution & Rolling Updates

```yaml
# Rolling deployment across web fleet — 20% at a time
- name: Rolling nginx upgrade
  hosts: web
  serial: "20%"           # Can also be: 1, 5, [1, 5, 10] (increasing batch sizes)
  max_fail_percentage: 0  # Abort immediately if ANY host fails
  any_errors_fatal: true

  tasks:
    - name: Remove host from load balancer
      ansible.builtin.uri:
        url: "http://{{ lb_api }}/deregister/{{ inventory_hostname }}"
        method: POST
      delegate_to: localhost

    - name: Upgrade nginx
      ansible.builtin.package:
        name: nginx
        state: latest
      become: true
      notify: Restart nginx

    - name: Flush handlers (restart nginx now)
      ansible.builtin.meta: flush_handlers

    - name: Wait for nginx to be healthy
      ansible.builtin.uri:
        url: "http://{{ ansible_host }}/health"
        status_code: 200
      retries: 10
      delay: 5

    - name: Re-add host to load balancer
      ansible.builtin.uri:
        url: "http://{{ lb_api }}/register/{{ inventory_hostname }}"
        method: POST
      delegate_to: localhost

  handlers:
    - name: Restart nginx
      ansible.builtin.service:
        name: nginx
        state: restarted
```

```bash
# Simulate a rolling update without making changes
ansible-playbook rolling-upgrade.yml --check

# Override serial from command line (not natively supported, use extra vars)
ansible-playbook rolling-upgrade.yml -e "serial_count=1"

# Run with step-by-step confirmation at each task
ansible-playbook site.yml --step
```

---

### Check Mode & Diff Mode

```bash
# Check mode — show what would change, make no changes
ansible-playbook site.yml --check

# Diff mode — show file diffs for template/copy/lineinfile tasks
ansible-playbook site.yml --diff

# Both together — most useful for PR review
ansible-playbook site.yml --check --diff

# Run in check mode but override for specific tasks
# In the playbook, mark tasks that must run even in check mode:
```

```yaml
- name: Check current app version (safe to run always)
  ansible.builtin.command: /opt/myapp/bin/version
  register: current_version
  check_mode: false     # runs even with --check
  changed_when: false

- name: Deploy new version (skipped in check mode)
  ansible.builtin.copy:
    src: dist/myapp.tar.gz
    dest: /opt/myapp/
  # No check_mode override — will be skipped with --check
```

---

## ⚠️ Gotchas & Pro Tips

- **Handlers only fire once per play, at the very end:** If a task notifies a handler but the play fails before completion, the handler never runs — unless `force_handlers: true` is set. This is critical for service restarts: if your play fails after deploying a config but before the handler fires, the service restarts on the next successful run with stale config. Set `force_handlers: true` for deployment plays.

- **`serial` interacts with `max_fail_percentage` per batch:** With `serial: 5` and `max_fail_percentage: 20`, Ansible checks if >20% of the **current batch of 5** failed, not >20% of the total fleet. On a 100-host fleet with serial: 5, you could have 19 hosts fail across 4 batches without triggering the threshold. Use `any_errors_fatal: true` if you want true abort-on-first-failure semantics.

- **`rescue` does not reset `ansible_failed_result`:** Inside `rescue`, you can access `ansible_failed_result` to inspect what failed. But the play is still considered "failed" from Ansible's perspective unless you add `- meta: clear_host_errors` at the end of `rescue`. Without it, the host is marked failed in the play summary even if rescue succeeded.

- **Tags cascade through `import_*` but NOT `include_*`:** Tags applied to `import_role` or `import_tasks` are inherited by all tasks within — they're resolved at parse time. Tags on `include_role` or `include_tasks` only apply to the include itself, not the included tasks. This is one of the most common sources of "my tag isn't working" bugs.

- **`changed_when: false` is essential for command/shell tasks:** The `command` and `shell` modules always report `changed` unless you tell Ansible otherwise. This pollutes change reports, confuses handlers (they fire on `changed`), and makes idempotency checking useless. Always add `changed_when: false` to read-only commands, and `changed_when: result.rc == 0` or similar logic to write commands.

---

*Part of the [Ansible Mastery](https://github.com/naveenramasamy11/ansible-mastery) series.*
