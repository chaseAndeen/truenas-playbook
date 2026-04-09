# truenas-playbook

Ansible-driven Infrastructure as Code for TrueNAS SCALE. Provisions and configures a TrueNAS SCALE node entirely via the TrueNAS v2.0 REST API — no SSH required. Secrets are managed via AWS SSM Parameter Store, accessed through AWS IAM Identity Center SSO.

---

## Overview

This playbook connects to TrueNAS using `ansible_connection: local` and drives all configuration through HTTP calls to the TrueNAS API using a Bearer token. It is idempotent — safe to re-run at any time to converge configuration drift.

### What it configures

| Role | Responsibilities |
|---|---|
| `system_base` | Hostname, domain, admin email, SMTP/SES alerts, ACME/TLS cert via Cloudflare DNS |
| `identity_managment` | Service groups and users with explicit UIDs/GIDs aligned to NFS clients |
| `storage_provisioning` | ZFS pool, datasets, quotas, ACLs, NFS/SMB shares, periodic snapshots, S3 cloud sync |
| `system_hardening` | Lock default admin account, restrict SMB to trusted networks |

---

## Prerequisites

### 1. AWS SSO Login
All secrets are fetched from AWS SSM at runtime using the profile set in `aws_profile` (default: `InfraProvisioner`):

```bash
aws sso login --profile <aws_profile>
```

### 2. Ansible Collections
```bash
ansible-galaxy collection install amazon.aws
ansible-galaxy collection install community.general
```

### 3. Python Dependencies
```bash
pip install boto3 botocore
```

---

## SSM Parameters

The following parameters must exist in AWS SSM before running the playbook:

| SSM Path | Description |
|---|---|
| `/storage/truenas/api_key` | TrueNAS API key (SecureString) |
| `/storage/truenas/zfs_key` | ZFS dataset encryption key (SecureString) |
| `/storage/truenas/backups/s3_access_key` | AWS S3 access key for cloud sync (SecureString) |
| `/storage/truenas/backups/s3_secret_key` | AWS S3 secret key for cloud sync (SecureString) |
| `/storage/truenas/backups/cloudsync_password` | TrueNAS cloud sync encryption password (SecureString) |
| `/storage/truenas/backups/cloudsync_salt` | TrueNAS cloud sync encryption salt (SecureString) |
| `/infra/common/smtp/user` | SMTP username for AWS SES — shared with proxmox-playbook (SecureString) |
| `/infra/common/smtp/pass` | SMTP password for AWS SES — shared with proxmox-playbook (SecureString) |
| `/infra/common/cloudflare/api_token` | Cloudflare API token — shared with proxmox-playbook (SecureString) |
| `/infra/common/cloudflare/zone_id` | Cloudflare zone ID — shared with proxmox-playbook (SecureString) |

Store a parameter:
```bash
aws ssm put-parameter \
  --name "/storage/truenas/api_key" \
  --value "your-value-here" \
  --type SecureString \
  --profile <aws_profile> \
  --region <aws_region>
```

---

## Setup

### 1. TrueNAS Post-Install (UI)
Before running the playbook, complete these steps manually in the TrueNAS UI:

1. Create your admin user (e.g. `sysop`) and assign `builtin_administrators` group
2. Generate an API key for that user
3. Store the API key in SSM at `/storage/truenas/api_key`

### 2. Configure Inventory
```bash
cp inventory/hosts.yml.example inventory/hosts.yml
cp inventory/group_vars/all.yml.example inventory/group_vars/all.yml
```

Edit `hosts.yml` with your TrueNAS IP. Edit `all.yml` to match your environment.

### 3. Run the Playbook
```bash
ansible-playbook playbooks/site.yml
```

---

## Architecture Notes

**API-driven, no SSH** — TrueNAS SCALE does not support standard SSH-based Ansible management. All tasks use `ansible.builtin.uri` against the TrueNAS v2.0 REST API with a Bearer token. `ansible_connection: local` means the playbook runs from your workstation and makes HTTP calls directly to the TrueNAS host.

**Idempotent by design** — every task checks current state before acting. Re-running is safe and will only change what has drifted.

**UID/GID alignment** — service user UIDs and GIDs defined here must match what NFS clients expect. For example, `pbs_user` (uid 2000) / `proxmox_group` (gid 2000) is mirrored in the proxmox-playbook `pbs` role so the PBS VM can write to the NFS share correctly.

**Shared Cloudflare credentials** — Cloudflare token and zone ID live at `/infra/common/cloudflare/` in SSM and are shared with the proxmox-playbook. One set of credentials, no duplication.

**S3 cloud sync** — nightly encrypted sync of the entire ZFS pool to S3 DEEP_ARCHIVE using TrueNAS built-in cloud sync. Encryption password and salt are stored in SSM.

---

## Repository Structure

```
.
├── ansible.cfg
├── README.md
├── inventory/
│   ├── hosts.yml.example
│   └── group_vars/
│       └── all.yml.example
├── playbooks/
│   └── site.yml
└── roles/
    ├── system_base/          # hostname, SMTP, ACME/TLS cert
    ├── identity_managment/   # service groups and users
    ├── storage_provisioning/ # ZFS pool, datasets, shares, snapshots, cloud sync
    └── system_hardening/     # account lockdown, SMB restrictions
```
