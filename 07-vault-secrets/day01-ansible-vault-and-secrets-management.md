# 🔧 Ansible Vault & Secrets Management — Ansible Mastery

> **Every secret committed to version control is a security incident waiting to happen — ansible-vault and cloud-native secret stores are the two pillars of secure Ansible automation.**

## 📖 Concept

**ansible-vault** is Ansible's built-in encryption tool. It uses AES-256 symmetric encryption to encrypt variable files, entire playbooks, or individual variable values. The encrypted content is stored inline in YAML files with a `$ANSIBLE_VAULT` header and can be committed safely to version control. The vault password (or a script that retrieves it) is provided at playbook runtime via `--vault-password-file`, `--vault-id`, or the `ANSIBLE_VAULT_PASSWORD_FILE` environment variable.

The modern approach to vault is **vault IDs** (introduced in Ansible 2.4), which allow multiple vault passwords in a single project. This is essential for enterprises that need different passwords for different environments (prod vs staging), different teams (ops vs dev), or different secret categories (infra secrets vs app secrets). Each vault ID labels which password to use for which file, and Ansible resolves them automatically.

However, ansible-vault has limitations: it's a static encryption scheme requiring the vault password to be accessible at runtime, rotation is manual (re-encrypt all files with the new password), and it doesn't provide audit trails or dynamic secret generation. For production AWS environments, **AWS Secrets Manager** and **SSM Parameter Store** are the preferred secrets backends — they support automatic rotation, fine-grained IAM access control, versioning, and audit trails via CloudTrail.

**HashiCorp Vault** provides a more complete secrets management platform for multi-cloud and hybrid environments: dynamic credentials (database passwords that expire), PKI certificate generation, and secrets engines for AWS, Kubernetes, SSH, and more. Ansible integrates via the `community.hashi_vault` collection's lookup plugins, making it straightforward to pull any Vault secret into a playbook.

The best practice for any serious Ansible deployment is a **hybrid approach**: use ansible-vault only for bootstrap secrets (the credentials needed to connect to AWS or Vault), and use a cloud-native secret store for everything else.

---

## 💡 Real-World Use Cases

- Encrypt `group_vars/prod/vault.yml` containing RDS passwords and API keys with ansible-vault, commit to Git, and use a vault password stored in AWS SSM to decrypt at pipeline runtime.
- Use vault IDs to maintain separate prod and staging vault passwords — junior engineers have access to the staging vault password but not prod.
- Use the `amazon.aws.aws_ssm` lookup to pull database credentials from SSM Parameter Store at playbook runtime, avoiding vault management entirely for cloud environments.

---

## 🔧 Commands & Examples

### ansible-vault Basics

```bash
# Encrypt a new file interactively
ansible-vault encrypt group_vars/prod/vault.yml

# Encrypt an existing file (prompts for password)
ansible-vault encrypt --vault-id prod@prompt group_vars/prod/vault.yml

# Decrypt a file to stdout (view without decrypting on disk)
ansible-vault view group_vars/prod/vault.yml

# Edit an encrypted file in $EDITOR
ansible-vault edit group_vars/prod/vault.yml

# Decrypt to disk (use with caution — plaintext on disk)
ansible-vault decrypt group_vars/prod/vault.yml

# Rekey — change the encryption password
ansible-vault rekey --new-vault-password-file new_vault_pass.txt group_vars/prod/vault.yml

# Encrypt a single string value (to embed in YAML inline)
ansible-vault encrypt_string 'MyS3cr3tPassword' --name 'db_password'
# Output:
# db_password: !vault |
#           $ANSIBLE_VAULT;1.1;AES256
#           3234...

# Create a new encrypted file from scratch
ansible-vault create group_vars/prod/vault.yml
```

```yaml
# group_vars/prod/vault.yml — contents before encryption
---
vault_db_password: "MyS3cr3tPassword123!"
vault_api_key: "sk-prod-abc123xyz789"
vault_jwt_secret: "super-long-jwt-secret-for-production"
vault_smtp_password: "smtp_pass_here"

# group_vars/prod/main.yml — references vault vars (no encryption needed)
---
db_password: "{{ vault_db_password }}"
api_key: "{{ vault_api_key }}"
```

```bash
# Run playbook with vault password from a file
ansible-playbook site.yml --vault-password-file ~/.vault_pass.txt

# Run with vault password from environment variable
export ANSIBLE_VAULT_PASSWORD_FILE=~/.vault_pass.txt
ansible-playbook site.yml

# Prompt for vault password interactively
ansible-playbook site.yml --ask-vault-pass
```

---

### Vault IDs — Multiple Passwords

```bash
# Encrypt different files with different vault IDs
ansible-vault encrypt --vault-id prod@prompt group_vars/prod/vault.yml
ansible-vault encrypt --vault-id staging@prompt group_vars/staging/vault.yml
ansible-vault encrypt --vault-id dev@~/.vault_dev.txt group_vars/dev/vault.yml

# Run with multiple vault IDs (Ansible tries each until decryption succeeds)
ansible-playbook site.yml \
  --vault-id prod@~/.vault_prod.txt \
  --vault-id staging@~/.vault_staging.txt \
  --vault-id dev@~/.vault_dev.txt

# Use a script to retrieve vault password from AWS SSM
ansible-playbook site.yml --vault-id prod@scripts/get_vault_pass.py
```

```python
#!/usr/bin/env python3
# scripts/get_vault_pass.py — retrieve vault password from AWS SSM
import boto3
import sys

def get_vault_password():
    client = boto3.client('ssm', region_name='us-east-1')
    response = client.get_parameter(
        Name='/ansible/vault/prod_password',
        WithDecryption=True
    )
    print(response['Parameter']['Value'])

if __name__ == '__main__':
    get_vault_password()
```

```ini
# ansible.cfg — set default vault ID configuration
[defaults]
vault_identity_list = prod@scripts/get_vault_pass_prod.py, staging@~/.vault_staging.txt
```

---

### Vault in CI/CD Pipelines

```yaml
# .github/workflows/deploy.yml — GitHub Actions with vault
name: Deploy to Production

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Install Ansible
        run: pip install ansible boto3

      - name: Write vault password from GitHub Secret
        run: echo "${{ secrets.ANSIBLE_VAULT_PASSWORD }}" > /tmp/vault_pass.txt

      - name: Run deployment playbook
        run: |
          ansible-playbook site.yml \
            --vault-password-file /tmp/vault_pass.txt \
            --inventory inventory/prod/ \
            -e env=production
        env:
          AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
          AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}

      - name: Cleanup vault password
        if: always()
        run: rm -f /tmp/vault_pass.txt
```

```yaml
# GitLab CI equivalent
deploy_prod:
  stage: deploy
  image: python:3.11-slim
  before_script:
    - pip install ansible boto3
    - echo "$ANSIBLE_VAULT_PASSWORD" > /tmp/vault_pass.txt
  script:
    - ansible-playbook site.yml
        --vault-password-file /tmp/vault_pass.txt
        --inventory inventory/prod/
  after_script:
    - rm -f /tmp/vault_pass.txt
  environment:
    name: production
  only:
    - main
```

---

### AWS SSM Parameter Store Integration

```bash
# Store a secret in SSM
aws ssm put-parameter \
  --name '/myapp/prod/db_password' \
  --value 'MyS3cr3tPassword123!' \
  --type SecureString \
  --key-id alias/myapp-prod \
  --overwrite

# Store as a regular (unencrypted) parameter
aws ssm put-parameter \
  --name '/myapp/prod/db_host' \
  --value 'prod-db.cluster.example.com' \
  --type String

# List parameters by path
aws ssm get-parameters-by-path \
  --path '/myapp/prod/' \
  --with-decryption \
  --query 'Parameters[*].[Name,Value]' \
  --output table
```

```yaml
# Use SSM in playbooks via amazon.aws collection lookup
- name: Read all app secrets from SSM
  ansible.builtin.set_fact:
    app_config:
      db_host: "{{ lookup('amazon.aws.aws_ssm', '/myapp/prod/db_host', region='us-east-1') }}"
      db_password: "{{ lookup('amazon.aws.aws_ssm', '/myapp/prod/db_password', decrypt=True, region='us-east-1') }}"
      api_key: "{{ lookup('amazon.aws.aws_ssm', '/myapp/prod/api_key', decrypt=True, region='us-east-1') }}"

# Bulk-read all parameters under a path
- name: Get all /myapp/prod/ parameters
  ansible.builtin.set_fact:
    all_params: "{{ query('amazon.aws.aws_ssm', '/myapp/prod/', bypath=True, recursive=True, decrypt=True, region='us-east-1') }}"

# Use SSM in a template directly
- name: Generate app config with SSM-sourced secrets
  ansible.builtin.template:
    src: templates/app.conf.j2
    dest: /etc/myapp/app.conf
  vars:
    db_password: "{{ lookup('amazon.aws.aws_ssm', '/myapp/{{ env }}/db_password', decrypt=True) }}"
```

---

### AWS Secrets Manager Integration

```bash
# Create a secret in Secrets Manager
aws secretsmanager create-secret \
  --name myapp/prod/database \
  --description "Production database credentials" \
  --secret-string '{"username":"myapp","password":"MyS3cr3t!","host":"prod-db.example.com"}'

# Rotate a secret
aws secretsmanager rotate-secret \
  --secret-id myapp/prod/database \
  --rotation-lambda-arn arn:aws:lambda:us-east-1:123456789:function:SecretsManagerRDS
```

```yaml
# Read from Secrets Manager using community.aws collection
- name: Get RDS credentials from Secrets Manager
  community.aws.aws_secret:
    name: myapp/prod/database
    region: us-east-1
  register: db_secret
  delegate_to: localhost

- name: Use the secret
  ansible.builtin.debug:
    msg: "DB host: {{ (db_secret.secret | from_json).host }}"

# Alternative: use aws CLI with pipe lookup
- name: Get secret via aws CLI
  ansible.builtin.set_fact:
    db_creds: "{{ lookup('pipe', 'aws secretsmanager get-secret-value --secret-id myapp/prod/database --query SecretString --output text') | from_json }}"
  delegate_to: localhost
```

---

### HashiCorp Vault Integration

```bash
# Install community.hashi_vault collection
ansible-galaxy collection install community.hashi_vault

# Authenticate Vault via AWS IAM (recommended for EC2/EKS workloads)
vault login -method=aws role=ansible-role

# Write a secret to Vault
vault kv put secret/myapp/prod db_password=MyS3cr3t! api_key=sk-abc123
```

```yaml
# Read from HashiCorp Vault in playbooks
- name: Get secrets from Vault (token auth)
  ansible.builtin.set_fact:
    vault_secrets: "{{ lookup('community.hashi_vault.hashi_vault',
                      'secret/data/myapp/prod',
                      url='https://vault.example.com',
                      token=vault_token) }}"

# AWS IAM auth (no token needed on EC2)
- name: Get secrets from Vault (AWS IAM auth)
  ansible.builtin.set_fact:
    app_secrets: "{{ lookup('community.hashi_vault.hashi_vault',
                    'secret/data/myapp/{{ env }}',
                    url=vault_addr,
                    auth_method='aws_iam',
                    role_id='ansible-role') }}"

- name: Use the Vault secret
  ansible.builtin.template:
    src: app.conf.j2
    dest: /etc/myapp/app.conf
  vars:
    db_password: "{{ app_secrets.data.data.db_password }}"
    api_key: "{{ app_secrets.data.data.api_key }}"

# Get dynamic AWS credentials from Vault
- name: Get temporary AWS credentials from Vault
  ansible.builtin.set_fact:
    dynamic_aws: "{{ lookup('community.hashi_vault.hashi_vault',
                    'aws/creds/ansible-deploy-role',
                    url=vault_addr,
                    auth_method='token',
                    token=vault_token) }}"

- name: Use dynamic credentials for subsequent AWS tasks
  amazon.aws.ec2_instance_info:
    region: us-east-1
  environment:
    AWS_ACCESS_KEY_ID: "{{ dynamic_aws.access_key }}"
    AWS_SECRET_ACCESS_KEY: "{{ dynamic_aws.secret_key }}"
    AWS_SESSION_TOKEN: "{{ dynamic_aws.security_token }}"
```

---

### Best Practices: Variable File Structure

```
group_vars/
├── all/
│   ├── main.yml          # non-sensitive vars (committed as plaintext)
│   └── vault.yml         # ENCRYPTED: vault_* prefixed vars
├── prod/
│   ├── main.yml          # references vault vars: db_password: "{{ vault_db_password }}"
│   └── vault.yml         # ENCRYPTED: vault_db_password, vault_api_key
└── staging/
    ├── main.yml
    └── vault.yml         # ENCRYPTED with different password

# .gitignore — NEVER commit these
.vault_pass.txt
*.vault_pass
vault_password
```

```yaml
# Pattern: prefix all vault variables with 'vault_'
# group_vars/prod/vault.yml (before encryption):
---
vault_db_password: "prod_pass_123!"
vault_api_key: "prod_api_key"
vault_jwt_secret: "prod_jwt_secret"

# group_vars/prod/main.yml (plaintext — shows what secrets exist without values):
---
db_password: "{{ vault_db_password }}"
api_key: "{{ vault_api_key }}"
jwt_secret: "{{ vault_jwt_secret }}"
```

---

## ⚠️ Gotchas & Pro Tips

- **Never encrypt `main.yml` — only encrypt `vault.yml`:** Encrypting the main variable file makes PRs impossible to review (no diff). The pattern of a plaintext `main.yml` that references `vault_*` variables from an encrypted `vault.yml` gives you both security and reviewability. You can see what secrets exist (variable names) without seeing their values.

- **`ansible-vault encrypt_string` output includes a label — use it:** When encrypting individual strings with `encrypt_string`, always provide `--name` to include the variable name in the output. Without it, you get an unlabeled `!vault` block that's confusing to insert into YAML correctly.

- **Vault password files must not be world-readable:** Ansible checks permissions. If `~/.vault_pass.txt` is readable by group or world, Ansible may warn or fail. Use `chmod 600 ~/.vault_pass.txt`. In CI/CD, write the password to a temp file with `umask 177` to ensure 0600 permissions.

- **Rotation with ansible-vault requires re-encrypting all files:** There's no incremental rotation. When a vault password is compromised, you must decrypt all files with the old password and re-encrypt with the new one. This is operationally painful for large repos. This is one of the strongest arguments for using a cloud-native secret store with built-in rotation instead.

- **SSM vs Secrets Manager for Ansible:** Use **SSM Parameter Store** for configuration values that happen to be secret (database hostnames, ports, passwords). Use **Secrets Manager** for credentials that need automatic rotation (RDS, Redshift, API keys). Both support IAM-based access control and CloudTrail auditing, but Secrets Manager adds rotation automation and costs more per secret.

---

*Part of the [Ansible Mastery](https://github.com/naveenramasamy11/ansible-mastery) series.*
