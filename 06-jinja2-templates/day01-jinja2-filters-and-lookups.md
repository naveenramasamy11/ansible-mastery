# 🔧 Jinja2 Filters & Lookup Plugins — Ansible Mastery

> **Jinja2 is the language of dynamic configuration in Ansible — engineers who master its filters and lookups can generate any config from any data structure without writing a line of Python.**

## 📖 Concept

Ansible uses Jinja2 as its templating engine for two purposes: **in-line expressions** within playbook YAML (variable substitution, conditionals, filters applied to task parameters) and **template files** rendered by the `template` module. Both contexts share the same Jinja2 engine, but template files additionally support multi-line Jinja2 constructs — loops, macros, block-level conditionals — that would be unwieldy inline.

**Filters** are the workhorse of Jinja2 in Ansible. They transform values: `{{ my_list | sort }}`, `{{ my_string | upper }}`, `{{ my_dict | to_json }}`. Ansible ships with dozens of filters beyond the standard Jinja2 set, covering data manipulation (`combine`, `dict2items`, `items2dict`), list processing (`selectattr`, `map`, `reject`, `unique`, `flatten`), type conversion (`int`, `float`, `bool`, `string`, `list`), and encoding (`b64encode`, `b64decode`, `to_yaml`, `from_json`). Learning the 20 most common filters eliminates most custom Python scripting.

**Lookup plugins** are Ansible's mechanism for reading external data at playbook runtime. They run on the **control node** (not managed hosts) and can read files, environment variables, AWS SSM parameters, HashiCorp Vault secrets, DNS records, and dozens of other sources. The `lookup` function is used inline in playbooks and templates; `query` (or `q`) is the modern equivalent that always returns a list. Understanding the difference matters because some lookups return strings and some return lists, and using the wrong one causes subtle bugs.

The `combine` filter and `dict2items`/`items2dict` pair are particularly valuable for infrastructure work: merging per-environment configuration dicts, transforming inventory data into config file formats, and building dynamic AWS tag structures from Ansible variable hierarchies.

---

## 💡 Real-World Use Cases

- Use `selectattr` and `map` to extract all `ansible_host` values from the `app` group and generate an nginx upstream block dynamically from inventory.
- Use the `aws_ssm` lookup plugin to read database passwords directly from AWS SSM Parameter Store at playbook runtime — no secrets in YAML files.
- Use `combine` to merge base configuration dicts with environment-specific overrides, producing per-environment config files from a single template.

---

## 🔧 Commands & Examples

### Essential String Filters

```yaml
- name: String manipulation filters
  ansible.builtin.debug:
    msg:
      # Case
      upper: "{{ 'hello world' | upper }}"               # HELLO WORLD
      lower: "{{ 'HELLO WORLD' | lower }}"               # hello world
      title: "{{ 'hello world' | title }}"               # Hello World
      capitalize: "{{ 'hELLO' | capitalize }}"           # Hello

      # Trimming and replacement
      strip: "{{ '  hello  ' | trim }}"                  # hello
      replace: "{{ 'hello world' | replace('world', 'ansible') }}"  # hello ansible
      regex_replace: "{{ 'foo123bar' | regex_replace('[0-9]+', 'NUM') }}"  # fooNUMbar

      # Splitting and joining
      split: "{{ 'a,b,c' | split(',') }}"                # ['a', 'b', 'c']
      join: "{{ ['a', 'b', 'c'] | join(' | ') }}"        # a | b | c

      # Testing
      starts: "{{ 'hello' | regex_search('^hel') }}"     # hel
      truncate: "{{ 'hello world' | truncate(8) }}"       # hello...

      # Encoding
      b64encode: "{{ 'secret' | b64encode }}"            # c2VjcmV0
      b64decode: "{{ 'c2VjcmV0' | b64decode }}"          # secret
      hash: "{{ 'password' | password_hash('sha512') }}" # hashed password for /etc/shadow
```

---

### Numeric & Type Filters

```yaml
- name: Numeric and type conversion
  ansible.builtin.debug:
    msg:
      int: "{{ '42' | int }}"                    # 42 (integer)
      float: "{{ '3.14' | float }}"              # 3.14 (float)
      bool: "{{ 'true' | bool }}"                # True
      string: "{{ 42 | string }}"                # '42'
      abs: "{{ -5 | abs }}"                      # 5
      round: "{{ 3.7 | round }}"                 # 4.0
      log: "{{ 1024 | log(2) | round | int }}"   # 10

      # Practical: calculate heap size as 50% of RAM
      heap: "{{ (ansible_memtotal_mb * 0.5) | int }}m"

      # Practical: format disk size
      gb: "{{ (ansible_mounts[0].size_total / 1024 / 1024 / 1024) | round(1) }} GB"
```

---

### List Filters — selectattr, map, reject, flatten

```yaml
# Source data: list of dicts
# servers:
#   - { name: web01, role: web, port: 80, enabled: true }
#   - { name: web02, role: web, port: 80, enabled: false }
#   - { name: app01, role: app, port: 8080, enabled: true }
#   - { name: db01, role: db, port: 5432, enabled: true }

- name: List processing examples
  ansible.builtin.debug:
    msg:
      # selectattr — filter list by attribute value
      enabled_only: "{{ servers | selectattr('enabled', 'eq', true) | list }}"
      web_servers: "{{ servers | selectattr('role', 'eq', 'web') | list }}"
      non_db: "{{ servers | rejectattr('role', 'eq', 'db') | list }}"

      # map — extract a single attribute from each dict in list
      all_names: "{{ servers | map(attribute='name') | list }}"
      # ['web01', 'web02', 'app01', 'db01']

      enabled_names: "{{ servers | selectattr('enabled') | map(attribute='name') | list }}"
      # ['web01', 'app01', 'db01']

      # Combine selectattr + map for targeted extraction
      web_hosts: "{{ servers | selectattr('role', 'eq', 'web') | map(attribute='name') | list }}"

      # map with 'extract' — pull values from a dict using a list of keys
      app_ips: "{{ groups['app'] | map('extract', hostvars, 'ansible_host') | list }}"
      # Extracts ansible_host for every host in the 'app' group

      # sort, unique, reverse
      sorted_ports: "{{ servers | map(attribute='port') | unique | sort | list }}"
      # [80, 5432, 8080]

      # flatten — collapse nested lists
      flat: "{{ [[1, 2], [3, [4, 5]]] | flatten }}"        # [1, 2, 3, 4, 5]
      flat1: "{{ [[1, 2], [3, [4, 5]]] | flatten(levels=1) }}"  # [1, 2, 3, [4, 5]]

      # min/max/sum
      max_port: "{{ servers | map(attribute='port') | max }}"   # 8080
      total: "{{ [10, 20, 30] | sum }}"                          # 60
```

---

### Dict Filters — combine, dict2items, items2dict

```yaml
# combine — deep merge of dicts (rightmost wins)
- name: Merge configuration dicts
  vars:
    base_config:
      log_level: info
      max_connections: 100
      timeout: 30
      features:
        ssl: false
        metrics: true

    prod_overrides:
      log_level: warning
      max_connections: 500
      features:
        ssl: true

    final_config: "{{ base_config | combine(prod_overrides, recursive=True) }}"
    # Result:
    # log_level: warning         (overridden)
    # max_connections: 500       (overridden)
    # timeout: 30                (from base)
    # features:
    #   ssl: true                (overridden, recursive merge)
    #   metrics: true            (from base, preserved)

  ansible.builtin.debug:
    var: final_config

# dict2items / items2dict
- name: Dict to list transformations
  vars:
    env_vars:
      JAVA_HOME: /usr/lib/jvm/java-17
      APP_PORT: "8080"
      DB_HOST: prod-db.example.com

    # dict2items → [{key: JAVA_HOME, value: /usr/...}, ...]
    env_list: "{{ env_vars | dict2items }}"

    # items2dict — rebuild dict from list (useful after filtering)
    filtered_env: "{{ env_vars | dict2items | rejectattr('key', 'search', 'DB_') | items2dict }}"
    # Removes DB_HOST, keeps JAVA_HOME and APP_PORT

  ansible.builtin.template:
    src: environment.j2
    dest: /etc/myapp/environment
  # Template: {% for item in env_vars | dict2items %}
  #           {{ item.key }}={{ item.value }}
  #           {% endfor %}
```

---

### Template File Patterns

```jinja2
{# templates/nginx_upstream.conf.j2 #}
{# Generate nginx upstream from Ansible inventory #}

{% set app_hosts = groups['app'] | map('extract', hostvars) | list %}

upstream {{ app_name }}_backend {
    least_conn;
{% for host in app_hosts %}
    server {{ host.ansible_host }}:{{ host.app_port | default(8080) }} weight={{ host.lb_weight | default(1) }};
{% endfor %}

    keepalive 32;
}

server {
    listen 80;
    server_name {{ nginx_server_names | join(' ') }};

{% if nginx_ssl_enabled | default(false) %}
    listen 443 ssl;
    ssl_certificate     {{ nginx_ssl_cert_path }};
    ssl_certificate_key {{ nginx_ssl_key_path }};
{% endif %}

    location / {
        proxy_pass http://{{ app_name }}_backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_connect_timeout {{ nginx_proxy_timeout | default(60) }}s;
    }

    location /health {
        access_log off;
        return 200 'OK';
        add_header Content-Type text/plain;
    }
}
```

```jinja2
{# templates/prometheus_targets.yml.j2 #}
{# Generates Prometheus static_configs from inventory groups #}

{% macro render_group(group_name, port, labels={}) %}
- targets:
{% for host in groups[group_name] %}
  - "{{ hostvars[host]['ansible_host'] }}:{{ port }}"
{% endfor %}
  labels:
    job: {{ group_name }}
    env: {{ env }}
{% for k, v in labels.items() %}
    {{ k }}: {{ v }}
{% endfor %}
{% endmacro %}

scrape_configs:
  - job_name: node_exporter
    static_configs:
{{ render_group('all', 9100) | indent(6) }}

  - job_name: app_metrics
    static_configs:
{{ render_group('app', 8080, {'app': 'myapp'}) | indent(6) }}
```

---

### Lookup Plugins

```yaml
# file lookup — read local file content
- name: Read SSH public key
  ansible.builtin.authorized_key:
    user: ec2-user
    key: "{{ lookup('file', '~/.ssh/id_rsa.pub') }}"
    state: present

# env lookup — read environment variable on control node
- name: Use AWS credentials from environment
  amazon.aws.ec2_instance:
    region: "{{ lookup('env', 'AWS_DEFAULT_REGION') | default('us-east-1') }}"

# aws_ssm lookup — read from AWS SSM Parameter Store
- name: Get database password from SSM
  ansible.builtin.set_fact:
    db_password: "{{ lookup('amazon.aws.aws_ssm', '/myapp/prod/db_password', decrypt=True, region='us-east-1') }}"

# Multiple SSM params at once
- name: Get multiple SSM parameters
  ansible.builtin.set_fact:
    app_secrets:
      db_password: "{{ lookup('amazon.aws.aws_ssm', '/myapp/prod/db_password', decrypt=True) }}"
      api_key: "{{ lookup('amazon.aws.aws_ssm', '/myapp/prod/api_key', decrypt=True) }}"
      jwt_secret: "{{ lookup('amazon.aws.aws_ssm', '/myapp/prod/jwt_secret', decrypt=True) }}"

# password lookup — generate and cache random passwords
- name: Generate a random password for the app user
  ansible.builtin.debug:
    msg: "{{ lookup('ansible.builtin.password', '/tmp/passwords/appuser length=32 chars=ascii_letters,digits') }}"
  # Password is generated once and cached at the path — idempotent

# pipe lookup — run command on control node and capture output
- name: Get git commit SHA for deployment tagging
  ansible.builtin.set_fact:
    git_sha: "{{ lookup('pipe', 'git rev-parse --short HEAD') }}"

# template lookup — render a template inline without writing a file
- name: Post rendered config to API
  ansible.builtin.uri:
    url: "{{ config_api_endpoint }}"
    method: PUT
    body: "{{ lookup('template', 'templates/app_config.json.j2') }}"
    body_format: json

# query (q) — always returns a list, safer than lookup for multi-value results
- name: Get all instances in a tag group
  ansible.builtin.set_fact:
    all_values: "{{ query('amazon.aws.aws_ssm', '/myapp/prod/', recursive=True) }}"
```

---

### Advanced Inline Jinja2 Patterns

```yaml
# Ternary operator
- name: Set feature flag based on env
  ansible.builtin.set_fact:
    debug_mode: "{{ (env == 'staging') | ternary('true', 'false') }}"

# default filter with boolean check
- name: Use default if variable is undefined or empty
  ansible.builtin.debug:
    msg: "{{ my_var | default('fallback_value') }}"
    # default(omit) — omits the parameter entirely if undefined (useful in module params)

- name: Use omit to conditionally set a module param
  ansible.builtin.user:
    name: appuser
    uid: "{{ myapp_user_uid | default(omit) }}"  # don't set uid if not defined

# to_json / to_yaml / from_json / from_yaml
- name: Pretty-print a complex data structure
  ansible.builtin.debug:
    msg: "{{ my_complex_dict | to_nice_yaml(indent=2) }}"

- name: Parse JSON string from command output
  ansible.builtin.set_fact:
    parsed: "{{ raw_json_string | from_json }}"

# zip and zip_longest for parallel list iteration
- name: Create user accounts with UIDs
  ansible.builtin.user:
    name: "{{ item.0 }}"
    uid: "{{ item.1 }}"
    state: present
  with_together:
    - [alice, bob, carol]
    - [1001, 1002, 1003]
```

---

## ⚠️ Gotchas & Pro Tips

- **`lookup` vs `query`/`q`:** `lookup` returns a string by default (comma-separated for multiple values) and raises an error if the result is empty. `query` always returns a list and never errors on empty. For most use cases in modern Ansible (2.5+), prefer `query`/`q` for multi-value lookups and `lookup` only for single-value string lookups where the result will be used as a string.

- **Lookups run on the control node, not managed hosts:** The `file` lookup reads files from the Ansible controller's filesystem. The `env` lookup reads the controller's environment variables. The `pipe` lookup runs commands on the controller. If you want to read a file from a managed host, use `slurp` module and decode with `b64decode`, not `lookup('file', ...)`.

- **`combine` without `recursive=True` does a shallow merge:** Without `recursive=True`, nested dicts are replaced entirely, not merged. `base | combine(override)` where both have a `features` key will use override's `features` dict wholesale, discarding all base keys. Always use `recursive=True` for deep config merging.

- **Template whitespace control:** Jinja2 block tags (`{% %}`) leave blank lines in the output. Use `{%- -%}` (strip both sides) or `-%}` (strip right side only) to control whitespace in generated config files. Compare `{%- for ... %}` vs `{% for ... %}` when generating nginx or Prometheus configs — one produces clean output, the other has empty lines between each entry.

- **`| list` after filters is often required:** `map()`, `selectattr()`, and `rejectattr()` return **generator objects** in Python 3, not lists. These can't be used directly as lists in Jinja2. Always pipe through `| list` at the end of filter chains: `groups['app'] | map(attribute='ansible_host') | list`. Without it, you'll get unexpected behaviour with `| join`, `| length`, and `loop:`.

---

*Part of the [Ansible Mastery](https://github.com/naveenramasamy11/ansible-mastery) series.*
