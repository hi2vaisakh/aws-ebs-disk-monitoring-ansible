# AWS Multi-Account Disk Utilization Monitoring with Ansible

This repository contains an Ansible playbook that assumes a standardized role in multiple AWS accounts (organized under AWS Organizations), discovers EC2 instances, gathers disk utilization via AWS Systems Manager (SSM), aggregates the results into a JSON file, and optionally uploads that file to both **Amazon S3** and **Datadog** for further monitoring.

## Table of Contents

1. [Architecture Overview](#architecture-overview)  
2. [Requirements](#requirements)  
3. [Configuration](#configuration)  
4. [Usage](#usage)  
5. [Datadog Integration](#datadog-integration)  

---

## Architecture Overview
![Editor _ Mermaid Chart-2025-03-28-110021](https://github.com/user-attachments/assets/41c3c2f1-f6fe-4b3d-9eb3-d92ac5e4937d)


## Overview

- **AWS Organizations**: A Management Account oversees multiple Member Accounts.
- **Management Account**: Hosts the Ansible Control Node.
- **Cross-Account IAM Roles**: Each Member Account has a role allowing the Management Account to assume and query EC2.
- **Ansible Playbook**:
  - Discovers EC2 instances (tagged with `ansible: yes`) via cross-account role assumption.
  - Gathers disk usage metrics via AWS SSM.
  - Aggregates the results into a JSON file.
  - Uploads the file to an S3 bucket.
  - Optionally sends the metrics to Datadog.

---

## Requirements

- **Ansible 2.10+**
- **AWS Collections** (install via Galaxy):
  ```bash
  ansible-galaxy collection install amazon.aws community.aws

- **AWS Credentials** with permission to assume roles in each Member Account.
- **AWS SSM Agent** installed on your EC2 instances.
- **IAM Roles**:  
  Each Member Account must have a cross-account role (e.g. `CrossAccountEC2ReadRole`) that trusts the Management Account.
- **S3 Bucket** for uploading the aggregated report.
- **Datadog** (optional) for advanced monitoring and dashboards.

---

## Configuration

### Group Variables (group_vars/all.yml)

```yaml
# List of AWS accounts
aws_accounts:
  - account_id: "20756xxxxx467"
    role_arn: "arn:aws:iam::20756xxxxx467:role/CrossAccountEC2ReadRole"
  # Add more accounts here if needed

# Single region to consider
aws_region: "ap-northeast-1"

# SSM file-transfer bucket name (bucket must exist in the target region)
ssm_bucket: "ansible-ssm-ebs-disk-utilization"

# Datadog (optional)
datadog_api_key: "YOUR_DATADOG_API_KEY"
datadog_endpoint: "https://api.datadoghq.com/api/v1/series"
```

## Ansible Configuration (`ansible.cfg`)

```ini
[defaults]
transport = aws_ssm
remote_tmp = /tmp/.ansible/tmp
host_key_checking = False
```
# Playbook: multi_region_disk_utilization.yml

This playbook contains tasks to:
- Assume cross-account roles.
- Gather EC2 disk usage.
- Write the aggregated report to `aggregated_disk_usage.json`.
- Upload the report to an S3 bucket.

## Usage

### 1. Install Requirements

```bash
ansible-galaxy collection install amazon.aws community.aws

Ensure your AWS credentials can assume the specified cross-account roles.
```
## 2. Run the Playbook

```bash
ansible-playbook multi_region_disk_utilization.yml
```
## Check Output

- The aggregated JSON file (`aggregated_disk_usage.json`) is written locally.
- The same file is uploaded to the specified S3 bucket.
- If Datadog integration is enabled, the metrics are sent to Datadog.
