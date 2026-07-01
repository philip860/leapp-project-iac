# RHEL Fleet Upgrade Automation for Ansible Automation Platform

## Overview

This repository bootstraps an existing Ansible Automation Platform (AAP)
Controller with all of the objects required to automate Red Hat
Enterprise Linux major version upgrades using Leapp.

The bootstrap configures:

-   Organizations (or an existing organization)
-   Inventories
-   Credentials
-   Projects
-   Execution Environments (optional)
-   Job Templates
-   Workflow Job Templates

The actual upgrade playbooks are **not** embedded in this repository.
They are pulled from a Git project configured in AAP.

------------------------------------------------------------------------

# Overall Process

``` text
Bootstrap Playbook
        │
        ▼
Connect to AAP Controller
        │
        ▼
Verify Organization Exists
        │
        ▼
Create Inventories
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
Controller Ready
```

After bootstrapping, operators launch one of the workflows from the AAP
UI.

------------------------------------------------------------------------

# Upgrade Workflow

``` text
Analysis
    │
    ▼
Create LVM Snapshot
    │
    ▼
Leapp Preupgrade
    │
    ▼
Remediate Inhibitors
    │
    ▼
Run Preupgrade Again
    │
    ▼
Run Leapp Upgrade
    │
    ▼
Reboot
    │
    ▼
Validate Upgrade
    │
    ├────────────── Success ───────────────► Remove Snapshot
    │
    └────────────── Failure ───────────────► Roll Back Snapshot
```

------------------------------------------------------------------------

# Repository Layout

``` text
playbooks/
    configure-aap-rhel-upgrades.yml

vars/
    secrets.yml            (encrypted)
    inventories.yml
    credentials.yml
    projects.yml
    job_templates.yml
    workflow_job_templates.yml
```

------------------------------------------------------------------------

# Role of vars/secrets.yml

The `vars/secrets.yml` file is intended to be encrypted with Ansible
Vault.

It contains customer-specific values that should not be committed in
plaintext.

Typical contents include:

-   Controller hostname
-   Controller IP
-   Controller username/password
-   SSH private key
-   Default organization
-   Optional execution environment
-   Controller TLS validation setting

Example structure:

``` yaml
controller_hostname:
controller_ip:
controller_username:
controller_password:

controller_organization:

controller_validate_certs:

controller_execution_environment:

aap_ssh_username:
aap_ssh_private_key:
```

Because these values are isolated in one encrypted file, the same
Infrastructure-as-Code repository can be reused across multiple AAP
environments without changing the playbooks.

------------------------------------------------------------------------

# Customizing for Different Controllers

A new customer typically only changes:

-   `vars/secrets.yml`
-   `vars/inventories.yml`
-   `vars/projects.yml`

Everything else can remain unchanged.

Examples include:

-   Different controller hostname
-   Different organization
-   Different SSH credential
-   Different Git branch
-   Different execution environment
-   Different inventory names

------------------------------------------------------------------------

# Bootstrap Sequence

The bootstrap playbook performs the following:

1.  Load encrypted configuration.
2.  Locate the reachable AAP Controller.
3.  Verify connectivity.
4.  Verify the target organization exists.
5.  Create inventories.
6.  Create credentials.
7.  Create the AAP project.
8.  Synchronize the Git repository.
9.  Create job templates.
10. Create workflow job templates.

------------------------------------------------------------------------

# Git Project

The AAP Project references the Leapp automation repository.

During bootstrap the project is synchronized before any job templates
are created so that all referenced playbooks already exist in the
project.

------------------------------------------------------------------------

# Enterprise Design Goals

-   Idempotent
-   Infrastructure as Code
-   Reusable across multiple customers
-   Controller-agnostic
-   Git-driven
-   Secure (Ansible Vault)
-   Minimal customer customization
-   Supports future RHEL upgrade paths

------------------------------------------------------------------------

# Running

``` bash
ansible-playbook playbooks/configure-aap-rhel-upgrades.yml \
  --ask-vault-pass
```

After completion, the controller contains all inventories, credentials,
projects, job templates, and workflows required to perform managed Leapp
upgrades.
