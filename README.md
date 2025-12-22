Break-Glass Account Compliance Demo with Red Hat Ansible Automation Platform
=============================================================================

## 1. Introduction

This repository contains a fully automated demo showcasing how **Red Hat Ansible Automation Platform (AAP)** can be used to implement a **compliance control** that verifies the presence of a **break-glass emergency account** across an infrastructure composed of Linux servers.

The demo is built to be:

- **Fully automated** via **Configuration-as-Code (CasC)** using `infra.aap_configuration` and certified AAP collections
- **Clean, structured, and realistic**, suitable for customer demos or internal enablement
- **Extensible** to Windows nodes via WinRM (not included in this demo)

---

## 2. Scenario Overview

Many organizations require a **non-expiring, high-privilege emergency account**, often called _break-glass_, available on all critical hosts.

This demo validates compliance by ensuring:

- The break-glass account **exists**
- Its **UID** is within a valid range
- Its **shell** is correctly configured
- Its **home directory** exists
- It belongs to the required **groups** (e.g., `wheel`)
- Its password **never expires**
- Its **SSH authorized key** is configured properly

Deviations are automatically **reported** (JSON format) and can be optionally **remediated**.

### Integration Possibilities

- **Event-Driven Ansible (EDA)**: Can trigger real-time reactive remediation when compliance drift is detected
- **SIEM/SOAR Integration**: Compliance reports can be forwarded via webhook to security platforms for centralized monitoring

---

## 3. Architecture

### 3.1 Infrastructure Architecture

```
                 +---------------------------+
                 |   Red Hat AAP Controller  |
                 |   (Controller + Hub)      |
                 +------------+--------------+
                              |
                (API, SCM webhook, schedules)
                              |
                    +---------+----------+
                    |                    |
        +-----------+--------+  +--------+------------+
        |  Linux Hosts       |  | (Optional) Windows  |
        | (RHEL, CentOS, ...)|  | Servers via WinRM   |
        +--------------------+  +---------------------+
                    |
                    | SSH
                    |
        +-----------+--------+
        |  break-glass user  |
        |   compliance       |
        +--------------------+
```

### 3.2 Automation Logic Architecture

```
                        +--------------------+
                        |   SCM Repository   |
                        |  (GitHub / GitLab) |
                        +----------+---------+
                                   |
                                   v  (Webhook / Poll SCM)
                    +--------------+--------------+
                    |     AAP Configuration       |
                    |  (infra.aap_configuration)  |
                    +--------------+--------------+
                                   |
                                   v
    +-------------------------------------------------------------------+
    | AAP Controller: Organizations, Projects, Inventories, Credentials |
    |                 Job Templates, Users, Teams, Roles                |
    +------------------------------+------------------------------------+
                                   |
                                   v  (Job Launch)
                     +-------------+--------------+
                     | Break-glass Compliance Job |
                     +-------------+--------------+
                                   |
                                   v SSH
                        +--------------------+
                        |   Linux Targets    |
                        +--------------------+
```

---

## 4. End-to-End Demo Workflow

### 4.1 High-Level AAP Workflow

1. **AAP is configured automatically via CasC**  
   Using `infra.aap_configuration`, the Controller is populated with:
   - Organization (`BreakGlass-Demo`)
   - Project pointing to this Git repository
   - Inventory with target hosts
   - Machine Credentials
   - Job Templates (Compliance Check + Remediation)
   - Users, Teams, and Role assignments

2. **SCM repository hosts the playbooks and configuration**

3. **Job Template runs the compliance check**

4. **Output includes:**
   - AAP standard output with detailed check results
   - JSON compliance report stored on each host
   - Structured data suitable for EDA or SIEM integration

5. **Remediation job** can be launched manually or automated via EDA

---

## 5. Repository Structure

```
aap-demo-breakglass-compliance-clean/
│
├── ansible.cfg                         # Ansible configuration (Galaxy tokens)
├── inventory.yml                       # Inventory for AAP controller
├── vault.yml                           # Vault with sensitive variables (encrypt this!)
│
├── group_vars/
│   └── all/
│       ├── credentials.yml             # AAP credentials definitions
│       ├── demo_variables.yml          # Demo-specific variables
│       ├── hosts.yml                   # Target hosts for inventory
│       ├── inventories.yml             # AAP inventory definitions
│       ├── job_templates.yml           # Job template definitions
│       ├── labels.yml                  # AAP labels
│       ├── organizations.yml           # AAP organizations
│       ├── projects.yml                # AAP projects (SCM)
│       ├── roles.yml                   # AAP role assignments
│       ├── teams.yml                   # AAP teams
│       └── users.yml                   # AAP users
│
├── collections/
│   └── requirements.yml                # Required Ansible collections
│
├── files/
│   ├── README.md                       # Instructions for SSH keys
│   ├── breakglass_id_rsa               # SSH private key (DO NOT COMMIT)
│   └── breakglass_id_rsa.pub           # SSH public key
│
├── playbooks/
│   ├── aap_config.yml                  # CasC playbook to configure AAP
│   ├── check_breakglass_compliance.yml # Compliance check playbook
│   ├── remediate_breakglass.yml        # Remediation playbook
│   │
│   └── roles/
│       └── breakglass_compliance/
│           ├── README.md               # Role documentation
│           ├── defaults/main.yml       # Default variables
│           ├── vars/main.yml           # Internal variables
│           └── tasks/
│               ├── main.yml            # Entry point (orchestrates all checks)
│               ├── init.yml            # Initialize compliance structure
│               ├── check_user.yml      # Check user existence
│               ├── check_uid.yml       # Check UID range
│               ├── check_shell.yml     # Check shell configuration
│               ├── check_home.yml      # Check home directory
│               ├── check_groups.yml    # Check group membership
│               ├── check_password.yml  # Check password expiration
│               ├── check_ssh.yml       # Check SSH authorized keys
│               ├── finalize.yml        # Final compliance evaluation
│               ├── report.yml          # Generate JSON report
│               └── remediate.yml       # Remediation tasks
│
├── .gitignore                          # Git ignore rules
└── README.md                           # This file
```

---

## 6. The `files/` Directory

The `files/` directory contains sensitive credentials used by the demo. **These files are excluded from Git via `.gitignore`.**

### Required Files

| File | Description |
|------|-------------|
| `breakglass_id_rsa` | SSH **private key** for Machine Credential in AAP |
| `breakglass_id_rsa.pub` | SSH **public key** deployed to target hosts |

### How to Generate SSH Keys

```bash
# Generate Ed25519 key pair (recommended)
ssh-keygen -t ed25519 -f files/breakglass_id_ed25519 -C "breakglass@emergency" -N ""

# Or generate RSA key pair (legacy compatibility)
ssh-keygen -t rsa -b 4096 -f files/breakglass_id_rsa -C "breakglass@emergency" -N ""
```

### Security Best Practices

1. **Encrypt with Ansible Vault:**
   ```bash
   ansible-vault encrypt files/breakglass_id_rsa
   ```

2. **Use AAP Credential Store** instead of files when possible

3. **Rotate keys regularly** per your security policy

---

## 7. Prerequisites

Before starting, ensure you have:

- A running instance of **Red Hat AAP 2.5+** (containerized or VM-based)
- A **bastion/control node** with network access to AAP and target hosts
- **SSH access** to your Linux target hosts
- A **GitHub/GitLab account** with access to this repository
- **Red Hat account** for Automation Hub access (for certified collections)

---

## 8. Step-by-Step Deployment Guide

This section provides a complete, detailed guide to deploy the demo from scratch.

---

### Step 1 — Prepare the Bastion Host

The bastion host is where you'll run the CasC playbook to configure AAP.

#### 1.1 Install Ansible Core

```bash
sudo dnf install ansible-core -y
```

#### 1.2 Configure AAP Hostname Resolution

Add your AAP Controller hostname to `/etc/hosts` if DNS is not configured:

```bash
sudo vi /etc/hosts
```

Add a line like:
```
192.168.1.100   aap.example.com
```

Replace with your actual AAP Controller IP and hostname.

#### 1.3 Clone the Repository

```bash
git clone https://github.com/guarnacciaa/aap-demo-breakglass-compliance-clean.git
cd aap-demo-breakglass-compliance-clean
```

---

### Step 2 — Configure Automation Hub and Galaxy Tokens

The `ansible.cfg` file requires tokens to download certified and validated collections from Red Hat Automation Hub.

#### 2.1 Get Your Automation Hub Token

1. Go to [Red Hat Hybrid Cloud Console](https://console.redhat.com)
2. Navigate to **Ansible Automation Platform** → **Automation Hub**
3. Click on **Connect to Hub** → **Load Token**
4. Copy the token

#### 2.2 Get Your Galaxy Token (Optional)

1. Go to [Ansible Galaxy](https://galaxy.ansible.com)
2. Login and go to **Preferences** → **API Token**
3. Copy the token

#### 2.3 Update ansible.cfg

Edit the `ansible.cfg` file and replace the placeholder tokens:

```bash
vi ansible.cfg
```

Replace:
```ini
[galaxy_server.automation_hub_certified]
token=<token_from_automation_hub>

[galaxy_server.automation_hub_validated]
token=<token_from_automation_hub>

[galaxy_server.galaxy]
token=<token_from_galaxy_portal>
```

With your actual tokens.

---

### Step 3 — Install Required Ansible Collections

Install all required collections from Automation Hub and Galaxy:

```bash
ansible-galaxy collection install -r collections/requirements.yml
```

Or install them manually:

```bash
ansible-galaxy collection install \
  infra.aap_configuration \
  infra.ee_utilities:4.1.0 \
  ansible.controller \
  ansible.platform \
  ansible.hub \
  ansible.posix \
  containers.podman \
  community.general
```

> **Note:** Some collections require specific versions. Check `collections/requirements.yml` for exact versions.

---

### Step 4 — Configure Demo Variables

#### 4.1 Update AAP Connection Settings

Edit `group_vars/all/demo_variables.yml`:

```bash
vi group_vars/all/demo_variables.yml
```

Update the following:

```yaml
# Your AAP Controller hostname (without https://)
aap_hostname: aap.example.com

# Keep these as-is unless you need different organization/project names
demo_organization_name: BreakGlass-Demo
demo_project_name: BreakGlass-Automation
```

#### 4.2 Update Vault Credentials

Edit `vault.yml` with your AAP admin credentials:

```bash
vi vault.yml
```

```yaml
controller_username: "admin"
controller_password: "your-aap-admin-password"
vault_breakglass_password: "YourSecureBreakGlassPassword123!"
```

> **Security Recommendation:** Encrypt the vault file:
> ```bash
> ansible-vault encrypt vault.yml
> ```

#### 4.3 Update Target Hosts

Edit `group_vars/all/hosts.yml` to define your target Linux hosts:

```bash
vi group_vars/all/hosts.yml
```

```yaml
controller_hosts:
  - name: rhel-server-01
    inventory: "{{ demo_inventory_name }}"
    variables:
      ansible_host: "192.168.1.50"

  - name: rhel-server-02
    inventory: "{{ demo_inventory_name }}"
    variables:
      ansible_host: "192.168.1.51"
```

#### 4.4 Update Machine Credentials

Edit `group_vars/all/credentials.yml` with the SSH credentials for your target hosts:

```bash
vi group_vars/all/credentials.yml
```

```yaml
- name: RHEL
  description: RHEL Machine Credentials
  organization: "{{ demo_organization_name }}"
  credential_type: Machine
  inputs:
    username: root           # Or your SSH user
    password: your-password  # Or use ssh_key_data for key-based auth
    become_method: sudo
```

---

### Step 5 — Generate Break-Glass SSH Keys

Generate the SSH key pair for the break-glass account:

```bash
cd files/
ssh-keygen -t ed25519 -f breakglass_id_ed25519 -C "breakglass@emergency" -N ""
cd ..
```

Or for RSA (legacy compatibility):

```bash
cd files/
ssh-keygen -t rsa -b 4096 -f breakglass_id_rsa -C "breakglass@emergency" -N ""
cd ..
```

---

### Step 6 — Deploy AAP Configuration (CasC)

Now run the Configuration-as-Code playbook to populate AAP:

```bash
ansible-playbook playbooks/aap_config.yml
```

If you encrypted `vault.yml`:

```bash
ansible-playbook playbooks/aap_config.yml --ask-vault-pass
```

#### What This Creates in AAP:

| Object | Name |
|--------|------|
| Organization | `BreakGlass-Demo` |
| Project | `BreakGlass-Automation` |
| Inventory | `Demo-BreakGlass-Inventory` |
| Credential | `RHEL` (Machine), `github` (SCM) |
| Job Template | `BreakGlass - Compliance Check` |
| Job Template | `BreakGlass - Remediation` |
| Users | `breakglass-admin`, `breakglass-user`, `auditor` |
| Team | `BreakGlass-Team` |
| Label | `BreakGlass` |

---

### Step 7 — Verify AAP Configuration

1. Open your AAP Controller web UI: `https://aap.example.com`
2. Login with your admin credentials
3. Navigate to **Organizations** → Verify `BreakGlass-Demo` exists
4. Navigate to **Projects** → Verify `BreakGlass-Automation` is synced
5. Navigate to **Inventories** → Verify hosts are present
6. Navigate to **Templates** → Verify both job templates exist

---

### Step 8 — Run the Compliance Check

#### From the AAP UI:

1. Navigate to **Templates**
2. Click on **BreakGlass - Compliance Check**
3. Click **Launch**
4. Review the job output

#### From the CLI (using `awx`):

```bash
awx job_templates launch "BreakGlass - Compliance Check" --monitor
```

#### Expected Output:

The job will check each host and report:

```
TASK [breakglass_compliance : Record user existence check] ********************
ok: [rhel-server-01]

TASK [breakglass_compliance : Record UID check] ********************************
ok: [rhel-server-01]

...

TASK [breakglass_compliance : Write compliance report to file] ****************
changed: [rhel-server-01]
```

A JSON report is written to `/tmp/breakglass_compliance_report.json` on each host.

---

### Step 9 — Review the Compliance Report

SSH into a target host and view the report:

```bash
ssh root@192.168.1.50
cat /tmp/breakglass_compliance_report.json | jq .
```

Example output:

```json
{
  "hostname": "rhel-server-01",
  "user": "breakglass",
  "timestamp": "2025-01-15T10:30:00Z",
  "compliant": false,
  "checks": {
    "user_exists": {
      "passed": false,
      "message": "User does not exist"
    }
  },
  "summary": [
    "User does not exist"
  ]
}
```

---

### Step 10 — Run the Remediation Job (Optional)

If the compliance check found issues, run the remediation:

#### From the AAP UI:

1. Navigate to **Templates**
2. Click on **BreakGlass - Remediation**
3. Click **Launch**

#### What Remediation Does:

- Creates the `breakglass` user with UID 1999
- Sets the password (hashed)
- Configures password aging policy (never expires)
- Creates `.ssh` directory with proper permissions
- Deploys the SSH authorized key
- Grants passwordless sudo access
- Unlocks the account if locked

After remediation, run the compliance check again to verify all checks pass.

---

## 9. Key Files Explained

### 9.1 Playbook: `playbooks/check_breakglass_compliance.yml`

The main compliance check playbook:

```yaml
- name: Break-Glass Account Compliance Check
  hosts: all
  gather_facts: true
  become: true

  roles:
    - breakglass_compliance
```

### 9.2 Playbook: `playbooks/remediate_breakglass.yml`

The remediation playbook:

```yaml
- name: Break-Glass Account Remediation
  hosts: all
  gather_facts: true
  become: true
  vars_files:
    - ../vault.yml

  vars:
    breakglass_password: "{{ vault_breakglass_password }}"
    breakglass_ssh_pub_key: >-
      {{ lookup('file', 'files/breakglass_id_rsa.pub', errors='ignore') | default('') }}

  tasks:
    - name: Remediate break-glass account
      ansible.builtin.include_role:
        name: breakglass_compliance
        tasks_from: remediate.yml
```

### 9.3 Role: `breakglass_compliance`

| Task File | Purpose |
|-----------|---------|
| `main.yml` | Orchestrates all compliance checks |
| `init.yml` | Initializes the compliance data structure |
| `check_user.yml` | Verifies break-glass user exists |
| `check_uid.yml` | Validates UID is in allowed range (1000-65534) |
| `check_shell.yml` | Verifies shell is `/bin/bash` |
| `check_home.yml` | Checks home directory exists |
| `check_groups.yml` | Validates group membership (e.g., `wheel`) |
| `check_password.yml` | Ensures password never expires |
| `check_ssh.yml` | Verifies SSH authorized_keys configured |
| `finalize.yml` | Calculates final compliance status |
| `report.yml` | Generates JSON compliance report |
| `remediate.yml` | Creates/fixes the break-glass account |

### 9.4 AAP Configuration Files (CasC)

| File | Purpose |
|------|---------|
| `group_vars/all/organizations.yml` | Defines AAP organizations |
| `group_vars/all/projects.yml` | Defines SCM projects |
| `group_vars/all/inventories.yml` | Defines inventories |
| `group_vars/all/hosts.yml` | Defines target hosts |
| `group_vars/all/credentials.yml` | Configures credentials |
| `group_vars/all/job_templates.yml` | Creates job templates |
| `group_vars/all/users.yml` | Defines AAP users |
| `group_vars/all/teams.yml` | Defines AAP teams |
| `group_vars/all/roles.yml` | Assigns roles and permissions |
| `group_vars/all/labels.yml` | Defines labels for organization |

---

## 10. Troubleshooting

### Problem: CasC playbook fails to connect to AAP

**Solution:** Verify:
1. `aap_hostname` in `demo_variables.yml` is correct
2. AAP is reachable from bastion: `curl -k https://aap.example.com`
3. Credentials in `vault.yml` are correct

### Problem: Collections fail to install

**Solution:** Verify:
1. Tokens in `ansible.cfg` are valid
2. You have internet access to Automation Hub and Galaxy
3. Try installing collections one at a time to identify the failing one

### Problem: Compliance check fails with SSH errors

**Solution:** Verify:
1. Target hosts are reachable from AAP execution environment
2. Machine credential in AAP has correct username/password or SSH key
3. SSH is enabled on target hosts

### Problem: Remediation doesn't set SSH key

**Solution:** Verify:
1. `files/breakglass_id_rsa.pub` exists
2. The file is accessible from the playbook (check path in `remediate_breakglass.yml`)

---

## 11. Demo Presentation Summary

### Title

**"Ensuring Break-Glass Account Compliance with Ansible Automation Platform"**

### Key Talking Points

1. **Problem Statement**
   - Emergency accounts are mandatory for incident response
   - Manual verification is error-prone and inconsistent
   - Compliance drift happens over time

2. **Solution Overview**
   - Automated discovery and validation
   - Centralized reporting (JSON format)
   - Optional automated remediation
   - Integration-ready for EDA and SIEM

3. **Architecture**
   - AAP Controller as central automation hub
   - Configuration-as-Code for repeatable deployments
   - Git-based workflow for change management

4. **Automation Flow**
   - CasC bootstraps AAP environment
   - Compliance playbook validates all hosts
   - Reports generated for audit trails
   - Remediation available on-demand or automated

5. **Benefits**
   - Repeatable and auditable
   - Zero-touch compliance validation
   - Perfect for frameworks: ISO 27001, NIST, CIS
   - Reduces manual effort and human error

---

## 12. License

This project is provided as-is for demonstration purposes.

---

## 13. Contributing

Contributions are welcome! Please submit pull requests or open issues for improvements.



