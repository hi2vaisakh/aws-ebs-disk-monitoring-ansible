# AWS Multi-Account Disk Utilization Monitoring with Ansible

This repository contains an Ansible playbook that assumes a standardized role in multiple AWS accounts (organized under AWS Organizations), discovers EC2 instances, gathers disk utilization via AWS Systems Manager (SSM), aggregates the results into a JSON file, and optionally uploads that file to both **Amazon S3** and **Datadog** for further monitoring.

## Table of Contents

1. [Architecture Overview](#architecture-overview)  
2. [Requirements](#requirements)  
3. [Configuration](#configuration) 
4. [Security](#security)  
5. [Usage](#usage)  
6. [Datadog Integration](#datadog-integration)  

---

## Architecture Overview
<img width="407" alt="image" src="https://github.com/user-attachments/assets/52998ddc-388c-40f9-bffe-547d6775b0ab" />

<img width="556" alt="image" src="https://github.com/user-attachments/assets/61ac5216-b97a-4684-8a85-f6f6beb94586" />


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
  - account_id: "207xxxxxxx467"
    role_arn: "arn:aws:iam::207xxxxxxx467:role/CrossAccountEC2ReadRole"
  # Add more accounts here if needed

# Single region to consider
aws_region: "ap-northeast-1"

# SSM file-transfer bucket name (bucket must exist in the target region)
ssm_bucket: "ansible-ssm-ebs-sample-poc-disk-utilization-bucket"

# Datadog (optional)
datadog_api_key: "YOUR_DATADOG_API_KEY"
datadog_endpoint: "https://api.datadoghq.com/api/v1/series"
```

### Ansible Configuration (`ansible.cfg`)

```ini
[defaults]
transport = aws_ssm
remote_tmp = /tmp/.ansible/tmp
host_key_checking = False
```
### Playbook: multi_account_ebs_disk_utilization.yml

This playbook contains tasks to:
- Assume cross-account roles.
- Gather EC2 disk usage.
- Write the aggregated report to `aggregated_disk_usage.json`.
- Upload the report to an S3 bucket.

## Security

---

### 1. Permissions for Assuming Cross-Account Roles

**Management Account (or the principal executing the playbook):**

- **Must have permission** to call `sts:AssumeRole` on the target accounts’ roles.
- This is typically granted in the management account’s IAM policy so that it can assume the cross-account role in each Member Account.

---

### 2. Permissions in the Target (Member) Accounts

The cross-account role (e.g. `CrossAccountEC2ReadRole`) in each target account must include policies that allow the following actions:

### **EC2 Instance Discovery:**

- `ec2:DescribeInstances`
- `ec2:DescribeTags`
- *(Optional)* Other read-only EC2 permissions if needed for additional metadata.

### **AWS Systems Manager (SSM) Connectivity:**

- The EC2 instances must have an IAM instance profile that includes the **AmazonSSMManagedInstanceCore** policy (or equivalent) so that the SSM agent can register and receive commands.
- Although not directly used by the playbook, SSM requires that the instance can access SSM endpoints. Ensure that the instance role trusts SSM.

### **S3 Upload (for the Aggregated Report):**

- The role (or the IAM user/role assuming these actions) that uploads the aggregated JSON file must have permissions similar to:
  - `s3:PutObject`
  - `s3:GetObject`
  - `s3:ListBucket`
- These permissions must be granted on the specified S3 bucket (e.g. `ansible-ssm-ebs-disk-utilization`).

---

### 3. (Optional) Permissions for Datadog Integration

If you choose to send metrics to Datadog:

- The role or principal executing the playbook must have network access to the Datadog API endpoints.
- No specific AWS permissions are needed for Datadog, but ensure that any API keys are secured and that you comply with Datadog’s requirements (e.g. having a valid API key with proper scopes).

---

### Summary

For the Ansible playbook to run successfully:

- **Management Account / Execution Role:**
  - Must have the `sts:AssumeRole` permission to assume the cross-account roles in the target accounts.

- **Target Account Cross-Account Role (e.g. `CrossAccountEC2ReadRole`):**
  - Must include permissions for:
    - `ec2:DescribeInstances` and `ec2:DescribeTags` (for instance discovery)
    - SSM-related actions are usually managed by the EC2 instance profile (e.g. **AmazonSSMManagedInstanceCore**).

- **S3 Bucket Access:**
  - The role or user uploading the JSON report must have S3 permissions (`PutObject`, `GetObject`, `ListBucket`) on the designated bucket.

- **(Optional) Datadog Integration:**
  - Ensure network access and secure API key usage.

By ensuring these permissions are in place, the playbook will be able to:
- Assume the cross-account roles.
- Query EC2 instances and retrieve necessary data via SSM.
- Aggregate and upload the data to S3.
- Optionally, send metrics to Datadog.

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
