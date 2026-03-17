# 🔧 AWS EC2 Provisioning End-to-End — Ansible Mastery

> **A complete, production-grade AWS provisioning playbook is the clearest demonstration that you can translate infrastructure architecture into reliable, repeatable automation.**

## 📖 Concept

End-to-end AWS provisioning with Ansible covers the full stack from VPC networking to application readiness: security groups, EC2 instances with IMDSv2 enforced, ALB target group registration, DNS record creation, application bootstrap, and health verification. The result is a single playbook that takes an environment variable and produces a running, load-balanced, DNS-reachable application tier from nothing — or safely converges an existing one.

The key architectural principle is **idempotency at every layer**. Every AWS module in the `amazon.aws` collection is designed to converge to the desired state without duplicating resources. Running the playbook against an already-provisioned environment produces zero changes and confirms everything is in the expected state. This makes the playbook double as a **drift detection tool** — run it in check mode to verify actual state matches desired state.

The **project structure** follows a pattern used by mature Ansible shops: the provisioning playbook (`provision.yml`) creates infrastructure, a separate configuration playbook (`configure.yml`) bootstraps the OS and application, and a deploy playbook (`deploy.yml`) handles rolling application updates. This separation means infrastructure and config changes can be made independently without touching each other's blast radius.

The **GitHub Actions integration** wraps this playbook in a CI/CD pipeline: plan on pull requests (check mode + diff), apply on merge to main, with separate workflows for staging and production. AWS credentials are supplied via OIDC (no static keys), and the vault password is stored in GitHub Secrets.

---

## 💡 Real-World Use Cases

- Provision a new production environment from scratch in under 15 minutes — VPC, security groups, instances, ALB, DNS — with a single `ansible-playbook provision.yml -e env=production` command.
- Use the same playbook in `--check --diff` mode as part of a GitHub Actions PR check to detect infrastructure drift before merging.
- Integrate with Terraform: Terraform manages VPC/subnet/route table (slow-changing shared infra), Ansible manages EC2 instances and application config (fast-changing compute layer).

---

## 🔧 Commands & Examples

### Project Structure

```
aws-app-infra/
├── ansible.cfg
├── provision.yml                    # infrastructure provisioning
├── configure.yml                    # OS + app bootstrap
├── deploy.yml                       # rolling application deployment
├── requirements.yml                 # collection dependencies
├── inventory/
│   └── aws_ec2.yml                  # dynamic inventory
├── group_vars/
│   ├── all/
│   │   ├── main.yml
│   │   └── vault.yml                # encrypted
│   ├── prod/
│   │   ├── main.yml
│   │   └── vault.yml
│   └── staging/
│       └── main.yml
├── roles/
│   ├── base_hardening/
│   ├── cloudwatch_agent/
│   └── myapp/
└── .github/
    └── workflows/
        ├── plan.yml
        └── deploy.yml
```

---

### provision.yml — Full Infrastructure Playbook

```yaml
---
- name: Provision AWS infrastructure for {{ env }} environment
  hosts: localhost
  gather_facts: false
  collections:
    - amazon.aws
    - community.aws

  vars:
    aws_region: "{{ lookup('env', 'AWS_DEFAULT_REGION') | default('us-east-1') }}"
    vpc_cidr: "10.{{ {'production': '0', 'staging': '1', 'dev': '2'}[env] }}.0.0/16"
    instance_count: "{{ {'production': 3, 'staging': 1, 'dev': 1}[env] }}"
    instance_type: "{{ {'production': 't3.large', 'staging': 't3.small', 'dev': 't3.micro'}[env] }}"

  tasks:
    # --- Networking ---
    - name: Create VPC
      amazon.aws.ec2_vpc_net:
        name: "myapp-{{ env }}-vpc"
        cidr_block: "{{ vpc_cidr }}"
        region: "{{ aws_region }}"
        dns_hostnames: true
        dns_support: true
        tags:
          Environment: "{{ env }}"
          ManagedBy: ansible
          Project: myapp
        state: present
      register: vpc

    - name: Create public subnets (multi-AZ)
      amazon.aws.ec2_vpc_subnet:
        vpc_id: "{{ vpc.vpc.id }}"
        cidr: "{{ item.cidr }}"
        az: "{{ aws_region }}{{ item.az }}"
        map_public: true
        region: "{{ aws_region }}"
        tags:
          Name: "myapp-{{ env }}-public-{{ item.az }}"
          Environment: "{{ env }}"
          Tier: public
        state: present
      loop:
        - { az: a, cidr: "{{ vpc_cidr | regex_replace('.0/16', '.10.0/24') }}" }
        - { az: b, cidr: "{{ vpc_cidr | regex_replace('.0/16', '.11.0/24') }}" }
        - { az: c, cidr: "{{ vpc_cidr | regex_replace('.0/16', '.12.0/24') }}" }
      register: public_subnets

    - name: Create private subnets (multi-AZ)
      amazon.aws.ec2_vpc_subnet:
        vpc_id: "{{ vpc.vpc.id }}"
        cidr: "{{ item.cidr }}"
        az: "{{ aws_region }}{{ item.az }}"
        region: "{{ aws_region }}"
        tags:
          Name: "myapp-{{ env }}-private-{{ item.az }}"
          Environment: "{{ env }}"
          Tier: private
        state: present
      loop:
        - { az: a, cidr: "{{ vpc_cidr | regex_replace('.0/16', '.20.0/24') }}" }
        - { az: b, cidr: "{{ vpc_cidr | regex_replace('.0/16', '.21.0/24') }}" }
        - { az: c, cidr: "{{ vpc_cidr | regex_replace('.0/16', '.22.0/24') }}" }
      register: private_subnets

    - name: Create Internet Gateway
      amazon.aws.ec2_vpc_igw:
        vpc_id: "{{ vpc.vpc.id }}"
        region: "{{ aws_region }}"
        tags:
          Name: "myapp-{{ env }}-igw"
          Environment: "{{ env }}"
        state: present
      register: igw

    # --- Security Groups ---
    - name: Create ALB security group
      amazon.aws.ec2_security_group:
        name: "myapp-{{ env }}-alb-sg"
        description: "ALB security group for {{ env }}"
        vpc_id: "{{ vpc.vpc.id }}"
        region: "{{ aws_region }}"
        rules:
          - proto: tcp
            ports: [80, 443]
            cidr_ip: 0.0.0.0/0
            rule_desc: Public HTTP/HTTPS
        tags:
          Environment: "{{ env }}"
          Name: "myapp-{{ env }}-alb-sg"
        state: present
      register: alb_sg

    - name: Create app security group
      amazon.aws.ec2_security_group:
        name: "myapp-{{ env }}-app-sg"
        description: "App servers security group for {{ env }}"
        vpc_id: "{{ vpc.vpc.id }}"
        region: "{{ aws_region }}"
        rules:
          - proto: tcp
            ports: [8080]
            group_id: "{{ alb_sg.group_id }}"
            rule_desc: Allow from ALB only
          - proto: tcp
            ports: [22]
            cidr_ip: "{{ bastion_cidr | default('10.0.0.0/8') }}"
            rule_desc: SSH from bastion/VPN
        tags:
          Environment: "{{ env }}"
          Name: "myapp-{{ env }}-app-sg"
        state: present
      register: app_sg

    # --- IAM ---
    - name: Create EC2 instance profile
      amazon.aws.iam_role:
        name: "myapp-{{ env }}-ec2-role"
        assume_role_policy_document:
          Version: "2012-10-17"
          Statement:
            - Effect: Allow
              Principal:
                Service: ec2.amazonaws.com
              Action: sts:AssumeRole
        managed_policies:
          - arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore
          - arn:aws:iam::aws:policy/CloudWatchAgentServerPolicy
        state: present
      register: ec2_role

    # --- EC2 Instances ---
    - name: Find latest Amazon Linux 2023 AMI
      amazon.aws.ec2_ami_info:
        region: "{{ aws_region }}"
        owners: amazon
        filters:
          name: "al2023-ami-2023*-x86_64"
          architecture: x86_64
          state: available
      register: ami_info

    - name: Launch EC2 instances
      amazon.aws.ec2_instance:
        name: "myapp-{{ env }}-app-{{ '%02d' | format(item | int) }}"
        instance_type: "{{ instance_type }}"
        image_id: "{{ (ami_info.images | sort(attribute='creation_date') | last).image_id }}"
        key_name: "{{ ec2_key_name }}"
        vpc_subnet_id: "{{ private_subnets.results[item | int % 3].subnet.id }}"
        region: "{{ aws_region }}"
        security_groups:
          - "{{ app_sg.group_id }}"
        iam_instance_profile: "myapp-{{ env }}-ec2-role"
        metadata_options:
          http_tokens: required            # Enforce IMDSv2
          http_put_response_hop_limit: 1
        volumes:
          - device_name: /dev/xvda
            ebs:
              volume_size: 30
              volume_type: gp3
              iops: 3000
              throughput: 125
              encrypted: true
              delete_on_termination: true
        user_data: |
          #!/bin/bash
          set -e
          yum update -y
          yum install -y amazon-ssm-agent
          systemctl enable --now amazon-ssm-agent
          # Signal readiness via SSM Parameter Store
          aws ssm put-parameter \
            --name "/myapp/{{ env }}/instance-ready/$(curl -s http://169.254.169.254/latest/meta-data/instance-id)" \
            --value "$(date -u +%Y-%m-%dT%H:%M:%SZ)" \
            --type String --overwrite \
            --region {{ aws_region }}
        state: running
        wait: true
        wait_timeout: 300
        tags:
          Name: "myapp-{{ env }}-app-{{ '%02d' | format(item | int) }}"
          Environment: "{{ env }}"
          Role: app
          ManagedBy: ansible
          AutoScalingGroup: "myapp-{{ env }}-app"
      loop: "{{ range(instance_count | int) | list }}"
      register: ec2_instances

    # --- Load Balancer ---
    - name: Create ALB target group
      community.aws.elb_target_group:
        name: "myapp-{{ env }}-tg"
        protocol: HTTP
        port: 8080
        vpc_id: "{{ vpc.vpc.id }}"
        region: "{{ aws_region }}"
        health_check_protocol: HTTP
        health_check_path: /health
        health_check_port: traffic-port
        healthy_threshold_count: 2
        unhealthy_threshold_count: 3
        health_check_timeout: 5
        health_check_interval: 30
        successful_response_codes: "200"
        state: present
        tags:
          Environment: "{{ env }}"
      register: target_group

    - name: Register instances with target group
      community.aws.elb_target:
        target_group_arn: "{{ target_group.target_group_arn }}"
        target_id: "{{ item.instances[0].instance_id }}"
        target_port: 8080
        region: "{{ aws_region }}"
        state: present
      loop: "{{ ec2_instances.results }}"
      when: item.instances is defined

    # --- DNS ---
    - name: Get ALB DNS name
      community.aws.elb_application_lb_info:
        names: ["myapp-{{ env }}-alb"]
        region: "{{ aws_region }}"
      register: alb_info

    - name: Create Route53 DNS alias record
      amazon.aws.route53:
        state: present
        zone: "{{ base_domain }}"
        record: "{{ env }}.myapp.{{ base_domain }}"
        type: A
        alias: true
        alias_hosted_zone_id: "{{ alb_info.load_balancers[0].canonical_hosted_zone_id }}"
        value: "{{ alb_info.load_balancers[0].dns_name }}"
        region: "{{ aws_region }}"
      when: alb_info.load_balancers | length > 0

    # --- Output Summary ---
    - name: Show provisioned resources
      ansible.builtin.debug:
        msg: |
          Provisioning complete for {{ env }}:
          VPC ID:    {{ vpc.vpc.id }}
          ALB DNS:   {{ alb_info.load_balancers[0].dns_name | default('pending') }}
          App URL:   https://{{ env }}.myapp.{{ base_domain }}
          Instances: {{ ec2_instances.results | map(attribute='instances') | flatten | map(attribute='instance_id') | list | join(', ') }}
```

---

### GitHub Actions CI/CD Pipeline

```yaml
# .github/workflows/deploy.yml
name: Deploy Infrastructure

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

permissions:
  id-token: write    # for OIDC
  contents: read

jobs:
  plan:
    name: Ansible Check (Plan)
    runs-on: ubuntu-latest
    if: github.event_name == 'pull_request'

    steps:
      - uses: actions/checkout@v4

      - name: Configure AWS credentials via OIDC
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::${{ secrets.AWS_ACCOUNT_ID }}:role/GitHubActionsAnsibleRole
          aws-region: us-east-1

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: "3.11"

      - name: Install dependencies
        run: |
          pip install ansible boto3 botocore
          ansible-galaxy collection install -r requirements.yml -p ./collections/

      - name: Write vault password
        run: echo "${{ secrets.ANSIBLE_VAULT_PASSWORD }}" > /tmp/vault_pass.txt && chmod 600 /tmp/vault_pass.txt

      - name: Run in check + diff mode (plan)
        run: |
          ansible-playbook provision.yml \
            --check --diff \
            --vault-password-file /tmp/vault_pass.txt \
            -e env=production \
            -e ec2_key_name=prod-key \
            -e base_domain=example.com
        env:
          ANSIBLE_FORCE_COLOR: true

  deploy:
    name: Ansible Apply (Deploy)
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main' && github.event_name == 'push'
    environment: production

    steps:
      - uses: actions/checkout@v4

      - name: Configure AWS credentials via OIDC
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::${{ secrets.AWS_ACCOUNT_ID }}:role/GitHubActionsAnsibleRole
          aws-region: us-east-1

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: "3.11"

      - name: Install dependencies
        run: |
          pip install ansible boto3 botocore
          ansible-galaxy collection install -r requirements.yml -p ./collections/

      - name: Write vault password
        run: echo "${{ secrets.ANSIBLE_VAULT_PASSWORD }}" > /tmp/vault_pass.txt && chmod 600 /tmp/vault_pass.txt

      - name: Provision infrastructure
        run: |
          ansible-playbook provision.yml \
            --vault-password-file /tmp/vault_pass.txt \
            -e env=production \
            -e ec2_key_name=prod-key \
            -e base_domain=example.com

      - name: Configure and deploy application
        run: |
          ansible-playbook configure.yml deploy.yml \
            --vault-password-file /tmp/vault_pass.txt \
            -i inventory/aws_ec2.yml \
            --limit role_app \
            -e env=production \
            -e app_version=${{ github.sha }}

      - name: Cleanup vault password
        if: always()
        run: rm -f /tmp/vault_pass.txt
```

---

### Rolling Deploy Playbook

```yaml
# deploy.yml — rolling application update
---
- name: Rolling application deployment
  hosts: role_app
  serial: "{{ deploy_serial | default('33%') }}"
  max_fail_percentage: 0
  gather_facts: true

  vars:
    deploy_timeout: 300
    health_check_retries: 20
    health_check_delay: 15

  tasks:
    - name: Deregister from ALB target group
      community.aws.elb_target:
        target_group_arn: "{{ alb_target_group_arn }}"
        target_id: "{{ instance_id }}"
        target_port: 8080
        state: absent
        region: "{{ aws_region }}"
        deregister_unused: false
      delegate_to: localhost
      vars:
        instance_id: "{{ ansible_ec2_instance_id | default(lookup('pipe', 'curl -sf http://169.254.169.254/latest/meta-data/instance-id')) }}"

    - name: Wait for connections to drain
      ansible.builtin.pause:
        seconds: 30

    - name: Deploy new application version
      ansible.builtin.include_role:
        name: myapp
      vars:
        myapp_version: "{{ app_version }}"

    - name: Verify health endpoint
      ansible.builtin.uri:
        url: "http://localhost:8080/health"
        status_code: 200
        timeout: 10
      register: health
      retries: "{{ health_check_retries }}"
      delay: "{{ health_check_delay }}"
      until: health.status == 200

    - name: Re-register with ALB target group
      community.aws.elb_target:
        target_group_arn: "{{ alb_target_group_arn }}"
        target_id: "{{ ansible_ec2_instance_id }}"
        target_port: 8080
        state: present
        region: "{{ aws_region }}"
      delegate_to: localhost

    - name: Wait for target to be healthy in ALB
      community.aws.elb_target_group_info:
        target_group_arns: ["{{ alb_target_group_arn }}"]
        region: "{{ aws_region }}"
      register: tg_info
      delegate_to: localhost
      until: >
        tg_info.target_groups[0].target_health_descriptions |
        selectattr('target.id', 'eq', ansible_ec2_instance_id) |
        selectattr('target_health.state', 'eq', 'healthy') |
        list | length > 0
      retries: 30
      delay: 10
```

---

## ⚠️ Gotchas & Pro Tips

- **IMDSv2 enforcement is non-negotiable for production:** Set `http_tokens: required` in `metadata_options` on all EC2 instances. IMDSv1 is vulnerable to SSRF attacks that can expose instance credentials. AWS now defaults new instances to IMDSv2, but explicitly setting it in Ansible ensures you don't accidentally provision legacy instances.

- **ALB connection draining before deployment:** Always deregister from the target group and wait for the `deregistration_delay.timeout_seconds` (default 300s, usually reduced to 30s for fast deploys) before touching the application. Skipping this step means in-flight requests get terminated mid-response. Set the target group deregistration delay to 30s for most applications.

- **OIDC authentication for GitHub Actions is better than static keys:** Using `role-to-assume` with OIDC means no `AWS_ACCESS_KEY_ID`/`AWS_SECRET_ACCESS_KEY` secrets in GitHub — the credentials are short-lived tokens issued per workflow run. The IAM role's trust policy restricts which GitHub repos and branches can assume it, making it significantly more secure.

- **`serial: "33%"` for three-instance groups means one at a time:** With `serial: "33%"` and three instances, each batch is 1 host (floor(3 * 0.33) = 0, Ansible rounds up to 1). This is the correct default for production — never more than one host updating at a time, maintaining 2/3 capacity throughout. Adjust for larger fleets: `serial: [1, "20%", "50%"]` uses increasing batch sizes.

- **Store provisioning output in SSM for later use:** After provisioning, write VPC IDs, subnet IDs, ALB ARNs, and target group ARNs to SSM Parameter Store. Subsequent playbooks (configure, deploy) read these from SSM rather than requiring them as extra vars or re-querying AWS APIs. This decouples provisioning from configuration and works well across pipeline stages.

---

*Part of the [Ansible Mastery](https://github.com/naveenramasamy11/ansible-mastery) series.*
