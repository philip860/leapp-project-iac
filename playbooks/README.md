
# RHEL Fleet Upgrade Automation for Ansible Automation Platform

## Overview

This repository bootstraps an existing Ansible Automation Platform (AAP) Controller with all of the objects required to automate Red Hat Enterprise Linux (RHEL) major version upgrades using Leapp.

The bootstrap playbook configures:

- Inventory
- Inventory Groups
- Credentials
- Projects
- Job Templates
- Workflow Job Templates
- Workflow Surveys
- Optional Execution Environment assignment

The upgrade playbooks themselves are maintained in the Git repository configured as the AAP Project and are synchronized automatically before Job Templates are created.

---

# Overall Bootstrap Process

```text
Bootstrap Playbook
        │
        ▼
Locate Reachable AAP Controller
        │
        ▼
Authenticate
        │
        ▼
Verify Organization Exists
        │
        ▼
Create Inventory
        │
        ▼
Create Inventory Groups
        │
        ▼
Create Credentials
        │
        ▼
Create Project
        │
        ▼
Synchronize Git Repository
        │
        ▼
Create Job Templates
        │
        ▼
Create Workflow Job Templates
        │
        ▼
AAP Ready
```

---

# Inventory Design

A single inventory is created:

```
RHEL Upgrade Automation
```

with inventory groups:

```text
ALL_rhel
├── rhel7
├── rhel8
└── rhel9
```

Operators select the desired inventory group at workflow launch.

---

# Upgrade Workflow

```text
Fleet Analysis
      │
      ▼
Create Snapshot
      │
      ▼
Remediate Inhibitors
      │
      ▼
Post-Remediation Analysis
      │
      ▼
Run Leapp Upgrade
      │
      ▼
Reboot Host
      │
      ▼
Post Upgrade Validation
      │
      ├── Success ──► Remove Snapshot
      └── Failure ──► Roll Back Snapshot
```

---

# Repository Layout

```text
playbooks/
    configure-aap-rhel-upgrades.yml

vars/
    secrets.yml
    credentials.yml
    projects.yml
    groups.yml
    surveys.yml
    job_templates.yml
    workflow_job_templates.yml
```

---

# The Importance of `vars/secrets.yml`

`vars/secrets.yml` is the primary deployment profile for this project and should always be encrypted using Ansible Vault.

```bash
ansible-vault encrypt vars/secrets.yml
```

Almost every customer-specific customization is performed in this single file. By changing only these values, the same Infrastructure-as-Code repository can be reused across development, lab, cloud, and production AAP environments.

## Supported Variables

| Variable | Required | Purpose |
|-----------|----------|---------|
| controller_hostname | Yes | Preferred AAP Gateway DNS name. |
| controller_ip | Yes | Fallback controller IP address if DNS is unavailable. |
| controller_username | Yes | AAP administrator account. |
| controller_password | Yes | AAP administrator password (vault encrypted). |
| controller_organization | Yes | Existing organization that will contain all created objects. |
| controller_validate_certs | No | Enable or disable TLS certificate validation. |
| controller_execution_environment | No | Default EE for bootstrap tasks when applicable. |
| **leapp_execution_environment** | No | **Execution Environment assigned to all Leapp Job Templates. Leave blank (`""`) to use the AAP Controller default Execution Environment.** |
| **snapshot_execution_environment** | No | **Execution Environment assigned to snapshot Job Templates. Defaults to the Leapp snapshot EE if left unchanged.** |
| aap_ssh_username | Yes | Username used by the Machine Credential. |
| aap_ssh_private_key | Yes | Private SSH key used by the Machine Credential (vault encrypted). |

### Red Hat Subscription Manager (RHSM) / Satellite Variables

| Variable | Required | Purpose |
|-----------|----------|---------|
| rhsm_activate | No | Enables optional RHSM/Satellite registration during analysis. |
| rhsm_org | Optional | Red Hat organization ID or Satellite organization. |
| rhsm_activation_key | Optional | Activation key used for RHSM/Satellite registration. |
| rhsm_username | Optional | Red Hat username used if username/password registration is preferred. |
| rhsm_password | Optional | Red Hat password used with `rhsm_username` and should be vault encrypted. |
| rhsm_server_url | Optional | Satellite server hostname. Leave blank for Red Hat hosted RHSM. |
| rhsm_auto_attach | Optional | Controls automatic subscription attachment when applicable. |

### AWS Snapshot Variables (Optional)

These variables are only required when using AWS EBS snapshot automation.

| Variable | Required | Purpose |
|-----------|----------|---------|
| aws_access_key | Optional | AWS access key used by the AAP AWS Credential. Should be vault encrypted. |
| aws_secret_key | Optional | AWS secret access key used by the AAP AWS Credential. Should be vault encrypted. |
| aws_session_token | Optional | Temporary AWS session token when using STS credentials. Should be vault encrypted. |
| snapshot_aws_region | Optional | AWS region used for EC2/EBS snapshot operations. Leave blank to auto-detect from EC2 metadata. |
| snapshot_aws_instance_id | Optional | EC2 instance ID to snapshot. Leave blank to auto-detect from EC2 metadata. |
| snapshot_aws_description | Optional | Description prefix applied to created EBS snapshots. |
| snapshot_aws_tags | Optional | Additional tags applied to created EBS snapshots. |
| snapshot_aws_wait | Optional | Wait for AWS EBS snapshots to reach completion before finishing the job. |
| snapshot_aws_wait_timeout | Optional | Maximum time in seconds to wait for snapshots to complete. |
| snapshot_aws_delete_on_remove | Optional | Delete AWS snapshots when running the remove snapshot action. |
| snapshot_aws_fail_when_no_volumes | Optional | Fail the workflow if no EBS volumes are detected. |
| snapshot_aws_snapshot_all_volumes | Optional | Snapshot all attached EBS volumes automatically. |
| snapshot_aws_volume_ids | Optional | Explicit list of EBS volume IDs to snapshot instead of auto-detected volumes. |

### Git Authentication Variables

| Variable | Required | Purpose |
|-----------|----------|---------|
| git_username | Optional | Username for private Git repositories. |
| git_password | Optional | Password or Personal Access Token for Git. |
| git_ssh_private_key | Optional | SSH private key for Git authentication. |
## Example

```yaml
controller_hostname: aap.example.com
controller_ip: 192.168.1.25

controller_username: admin
controller_password: !vault |
  ...

controller_organization: Default

controller_validate_certs: false

controller_execution_environment: Default execution environment

# Leave blank to use the AAP default EE.
leapp_execution_environment: ""

aap_ssh_username: ec2-user
aap_ssh_private_key: !vault |
  ...

git_username: gituser
git_password: !vault |
  ...
git_ssh_private_key: !vault |
  ...
```

## Deployment Tuning

Typical changes between environments include:

- Controller hostname/IP
- Organization
- SSH username (ec2-user, azureuser, cloud-user, ansible, root, etc.)
- TLS validation
- Default Execution Environment
- Leapp-specific Execution Environment
- Git authentication method

No playbook changes are normally required.

---

# Bootstrap Sequence

1. Load encrypted configuration.
2. Discover the reachable AAP Controller.
3. Authenticate.
4. Verify the target organization.
5. Create the RHEL Upgrade Automation inventory.
6. Create the ALL_rhel, rhel7, rhel8 and rhel9 groups.
7. Create credentials.
8. Create the Git project.
9. Synchronize the project.
10. Create Job Templates.
11. Create Workflow Job Templates.
12. Bootstrap complete.

---

# Enterprise Design Goals

- Infrastructure as Code
- Idempotent
- Git-driven
- Secure using Ansible Vault
- Reusable across customer environments
- Controller agnostic
- Minimal customer customization
- Supports future RHEL major upgrade paths

---

# Running

```bash
ansible-playbook playbooks/configure-aap-rhel-upgrades.yml --ask-vault-pass -vvv
```

After completion, the controller is fully configured for managed RHEL major version upgrade workflows.
