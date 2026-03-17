# 🔧 Collections Structure & amazon.aws Deep Dive — Ansible Mastery

> **Collections replaced standalone roles as the distribution unit for Ansible content — understanding their structure and the amazon.aws collection unlocks the full power of Ansible for AWS automation.**

## 📖 Concept

**Ansible Collections** are a packaging format that bundles modules, plugins, roles, playbooks, and documentation into a single distributable unit with a two-part namespace: `namespace.collection_name` (e.g., `amazon.aws`, `community.kubernetes`, `ansible.posix`). Before collections, Ansible's module library was monolithic — everything shipped with Ansible itself. Collections decouple module development from Ansible core releases, enabling faster iteration, independent versioning, and better separation between Red Hat-maintained certified content and community contributions.

A collection lives at `~/.ansible/collections/` (user) or `/usr/share/ansible/collections/` (system) or a project-local `collections/` directory. Within a playbook or role, collection content is referenced with its FQCN (Fully Qualified Collection Name): `ansible.builtin.copy`, `amazon.aws.ec2_instance`, `community.kubernetes.k8s`. Using FQCNs is the modern best practice — it makes module origin unambiguous and avoids name collision between collections.

The **amazon.aws collection** is the primary interface between Ansible and AWS. It covers EC2, S3, Route53, IAM, VPC, ELB, RDS, ECS, and more. Authentication uses the standard AWS credential chain: environment variables, `~/.aws/credentials`, instance profile, or EKS pod IAM — no Ansible-specific credential management needed. The collection's modules are idempotent by design: running `ec2_instance` with the same parameters twice creates one instance, not two.

The **community.kubernetes (now kubernetes.core)** collection handles Kubernetes resource management: `k8s` for applying manifests, `k8s_info` for querying resources, `helm` for chart management, and `k8s_exec` for pod exec. It's the Ansible-native way to deploy to EKS, OpenShift, and vanilla Kubernetes clusters without shelling out to `kubectl`.

---

## 💡 Real-World Use Cases

- Use `amazon.aws.ec2_instance` to provision a fleet of EC2 instances with specific tags, security groups, and user data — idempotently, so re-running the playbook never creates duplicates.
- Use `kubernetes.core.helm` to deploy application Helm charts to EKS with values from Ansible variables — combining Ansible's variable system and AWS integration with Helm's package management.
- Build an internal collection that wraps your organisation's standard EC2 provisioning patterns, distribution it via a private Automation Hub, and enforce consistent tagging and security group policies across all teams.

---

## 🔧 Commands & Examples

### Installing & Managing Collections

```bash
# Install from Galaxy (public)
ansible-galaxy collection install amazon.aws
ansible-galaxy collection install community.kubernetes
ansible-galaxy collection install ansible.posix

# Install specific version
ansible-galaxy collection install amazon.aws:7.6.0

# Install from requirements.yml (recommended for projects)
ansible-galaxy collection install -r requirements.yml

# Install to project-local directory
ansible-galaxy collection install -r requirements.yml -p ./collections/

# List installed collections and versions
ansible-galaxy collection list

# Show collection details
ansible-galaxy collection list amazon.aws

# Install from Automation Hub (private registry)
ansible-galaxy collection install myorg.internal_collection \
  --server https://automationhub.example.com \
  --token my-ah-token

# Download for offline install
ansible-galaxy collection download amazon.aws -p /tmp/collections/
# Produces: /tmp/collections/amazon-aws-7.6.0.tar.gz

# Install from local tarball (air-gapped environments)
ansible-galaxy collection install /tmp/collections/amazon-aws-7.6.0.tar.gz

# Offline install from directory of tarballs
ansible-galaxy collection install /tmp/collections/ --ignore-errors
```

```yaml
# requirements.yml — pin all dependencies
---
collections:
  - name: amazon.aws
    version: ">=7.0.0,<8.0.0"
  - name: community.aws
    version: "6.5.0"
  - name: kubernetes.core
    version: ">=3.0.0"
  - name: community.vmware
    version: "4.2.0"
  - name: ansible.posix
    version: ">=1.5.4"
  - name: community.general
    version: ">=8.0.0"
  # Private collection from Automation Hub
  - name: myorg.platform
    source: https://automationhub.example.com
    version: "1.2.0"
```

---

### ansible.cfg for Collection Management

```ini
# ansible.cfg
[defaults]
collections_path = ./collections:~/.ansible/collections

# Point to a private Automation Hub
[galaxy]
server_list = automation_hub, release_galaxy

[galaxy_server.automation_hub]
url = https://automationhub.example.com
auth_url = https://sso.example.com/auth/token
token = my-ah-token

[galaxy_server.release_galaxy]
url = https://galaxy.ansible.com
```

---

### amazon.aws Collection — EC2 Operations

```yaml
---
- name: AWS EC2 provisioning with amazon.aws
  hosts: localhost
  gather_facts: false
  collections:
    - amazon.aws

  vars:
    aws_region: us-east-1
    instance_type: t3.medium
    ami_id: ami-0abcdef1234567890
    key_name: prod-key
    vpc_id: vpc-0123456789abcdef0
    subnet_id: subnet-0123456789abcdef0

  tasks:
    # Lookup AMI dynamically
    - name: Find latest Amazon Linux 2023 AMI
      amazon.aws.ec2_ami_info:
        region: "{{ aws_region }}"
        owners: amazon
        filters:
          name: "al2023-ami-2023*-x86_64"
          architecture: x86_64
          state: available
      register: ami_info

    - name: Set latest AMI ID
      ansible.builtin.set_fact:
        latest_ami: "{{ ami_info.images | sort(attribute='creation_date') | last }}"

    # Create security group
    - name: Create web server security group
      amazon.aws.ec2_security_group:
        name: myapp-web-sg
        description: Security group for web servers
        vpc_id: "{{ vpc_id }}"
        region: "{{ aws_region }}"
        rules:
          - proto: tcp
            ports: [80, 443]
            cidr_ip: 0.0.0.0/0
            rule_desc: Allow HTTP/HTTPS
          - proto: tcp
            ports: [22]
            cidr_ip: "{{ bastion_cidr }}"
            rule_desc: SSH from bastion only
        tags:
          Name: myapp-web-sg
          Environment: "{{ env }}"
          ManagedBy: ansible
      register: web_sg

    # Launch instances with idempotency
    - name: Launch EC2 instances
      amazon.aws.ec2_instance:
        name: "myapp-web-{{ item }}"
        instance_type: "{{ instance_type }}"
        image_id: "{{ latest_ami.image_id }}"
        key_name: "{{ key_name }}"
        vpc_subnet_id: "{{ subnet_id }}"
        region: "{{ aws_region }}"
        security_groups:
          - "{{ web_sg.group_id }}"
        iam_instance_profile: myapp-ec2-role
        user_data: |
          #!/bin/bash
          yum update -y
          aws s3 cp s3://myapp-bootstrap/install.sh /tmp/
          bash /tmp/install.sh
        metadata_options:
          http_tokens: required        # Require IMDSv2
          http_put_response_hop_limit: 1
        state: running
        wait: true
        wait_timeout: 300
        tags:
          Name: "myapp-web-{{ item }}"
          Environment: "{{ env }}"
          Role: web
          Version: "{{ app_version }}"
          ManagedBy: ansible
      loop: ["01", "02", "03"]
      register: ec2_instances

    # Gather info
    - name: Get instance IDs and IPs
      ansible.builtin.set_fact:
        instance_details: "{{ ec2_instances.results | map(attribute='instances') | flatten | map(attribute='instance_id') | list }}"

    # Add to dynamic in-memory inventory
    - name: Add new instances to inventory
      ansible.builtin.add_host:
        hostname: "{{ item.instances[0].private_ip_address }}"
        groupname: new_web_instances
        ansible_user: ec2-user
        ansible_ssh_private_key_file: "{{ key_path }}"
      loop: "{{ ec2_instances.results }}"
```

---

### amazon.aws — S3, IAM, Route53

```yaml
# S3 bucket management
- name: Create S3 bucket with versioning and encryption
  amazon.aws.s3_bucket:
    name: myapp-prod-artifacts
    region: us-east-1
    versioning: true
    encryption: AES256
    public_access:
      block_public_acls: true
      block_public_policy: true
      ignore_public_acls: true
      restrict_public_buckets: true
    tags:
      Environment: production
      ManagedBy: ansible

# Upload to S3
- name: Sync deployment artifacts to S3
  amazon.aws.s3_object:
    bucket: myapp-prod-artifacts
    object: "releases/{{ app_version }}/myapp.tar.gz"
    src: dist/myapp-{{ app_version }}.tar.gz
    mode: put
    region: us-east-1

# Route53 DNS record management
- name: Create DNS record for new instances
  amazon.aws.route53:
    state: present
    zone: example.com
    record: "{{ env }}.myapp.example.com"
    type: A
    ttl: 300
    value: "{{ ec2_instances.results | map(attribute='instances') | flatten | map(attribute='public_ip_address') | list }}"
    region: us-east-1

# IAM role and policy
- name: Create EC2 instance role
  amazon.aws.iam_role:
    name: myapp-ec2-role
    assume_role_policy_document: |
      {
        "Version": "2012-10-17",
        "Statement": [{
          "Effect": "Allow",
          "Principal": {"Service": "ec2.amazonaws.com"},
          "Action": "sts:AssumeRole"
        }]
      }
    managed_policies:
      - arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore
      - arn:aws:iam::aws:policy/CloudWatchAgentServerPolicy
    tags:
      ManagedBy: ansible
```

---

### kubernetes.core Collection — EKS/K8s Management

```bash
# Install
ansible-galaxy collection install kubernetes.core
pip install kubernetes
```

```yaml
---
- name: Deploy application to EKS
  hosts: localhost
  gather_facts: false

  vars:
    kubeconfig: "{{ lookup('env', 'KUBECONFIG') | default('~/.kube/config') }}"
    namespace: myapp-production
    app_version: "2.4.1"
    image: "123456789.dkr.ecr.us-east-1.amazonaws.com/myapp:{{ app_version }}"

  tasks:
    - name: Create namespace
      kubernetes.core.k8s:
        api_version: v1
        kind: Namespace
        name: "{{ namespace }}"
        state: present
        kubeconfig: "{{ kubeconfig }}"

    - name: Deploy application
      kubernetes.core.k8s:
        state: present
        kubeconfig: "{{ kubeconfig }}"
        definition:
          apiVersion: apps/v1
          kind: Deployment
          metadata:
            name: myapp
            namespace: "{{ namespace }}"
            labels:
              app: myapp
              version: "{{ app_version }}"
          spec:
            replicas: 3
            selector:
              matchLabels:
                app: myapp
            strategy:
              type: RollingUpdate
              rollingUpdate:
                maxSurge: 1
                maxUnavailable: 0
            template:
              metadata:
                labels:
                  app: myapp
              spec:
                containers:
                  - name: myapp
                    image: "{{ image }}"
                    ports:
                      - containerPort: 8080
                    resources:
                      requests:
                        cpu: 250m
                        memory: 512Mi
                      limits:
                        cpu: 1000m
                        memory: 1Gi
                    livenessProbe:
                      httpGet:
                        path: /health
                        port: 8080
                      initialDelaySeconds: 30
                    readinessProbe:
                      httpGet:
                        path: /ready
                        port: 8080

    - name: Wait for rollout to complete
      kubernetes.core.k8s_info:
        api_version: apps/v1
        kind: Deployment
        name: myapp
        namespace: "{{ namespace }}"
        kubeconfig: "{{ kubeconfig }}"
      register: deployment_info
      until: >
        deployment_info.resources[0].status.readyReplicas is defined and
        deployment_info.resources[0].status.readyReplicas == deployment_info.resources[0].spec.replicas
      retries: 30
      delay: 10

    # Deploy with Helm
    - name: Install/upgrade with Helm
      kubernetes.core.helm:
        name: myapp
        chart_ref: myorg/myapp
        chart_version: "{{ app_version }}"
        release_namespace: "{{ namespace }}"
        create_namespace: true
        kubeconfig: "{{ kubeconfig }}"
        values:
          image:
            tag: "{{ app_version }}"
          replicaCount: 3
          resources:
            requests:
              cpu: 250m
              memory: 512Mi
```

---

## ⚠️ Gotchas & Pro Tips

- **Always use FQCNs in production playbooks:** Writing `ec2_instance:` instead of `amazon.aws.ec2_instance:` relies on Ansible's module search path, which can silently use the wrong module if multiple collections define the same short name. FQCNs are unambiguous, version-control friendly, and required by `ansible-lint` in strict mode.

- **Project-local `collections/` directory takes precedence:** When you install collections with `-p ./collections/`, this directory is searched before the user/system paths. This means your CI/CD pipeline can pin exact versions in `requirements.yml`, install to `./collections/`, and guarantee reproducible runs regardless of what's installed globally.

- **`amazon.aws` authentication uses the standard AWS credential chain:** You don't need to pass `aws_access_key` to every module. Set environment variables (`AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `AWS_DEFAULT_REGION`) or rely on instance profiles/EKS IRSA. Using `ansible_aws_assume_role_arn` in inventory or vars enables cross-account deployments.

- **`kubernetes.core.k8s` module is blocking but not always convergent:** Unlike Ansible's own `until` retry loops, applying a Deployment manifest returns immediately after the API accepts it — the pods may still be starting. Always add a `k8s_info` + `until` loop after deploying to verify rollout completion before proceeding. The `wait: true` parameter handles this for simple resources but not complex rollout scenarios.

- **Offline collection installs for air-gapped environments:** Use `ansible-galaxy collection download -r requirements.yml -p /tmp/collections/` on an internet-connected machine, transfer the tarballs, then `ansible-galaxy collection install /tmp/collections/*.tar.gz`. Include `boto3` and `botocore` wheel files too, as `amazon.aws` won't work without them.

---

*Part of the [Ansible Mastery](https://github.com/naveenramasamy11/ansible-mastery) series.*
