# Python for DevOps Automation

***

## 1. Introduction to Boto3 AWS SDK

**Boto3** is Amazon’s official Python SDK for AWS — enabling Python scripts to:
- **Create, configure, and manage** AWS services (EC2, S3, VPC, IAM, EKS, Lambda, etc.)
- Implement **automation scripts** for:
  - Regular backups & cleanups
  - Configuration changes at scale
  - Health checks & monitoring scripts
  - Bulk tagging for governance and cost tracking

**Alternatives for other clouds**:
- Azure: `azure-mgmt-*` Python SDKs
- GCP: `google-cloud-*` Python clients

***

## 2. Installation & AWS Authentication

Install Boto3:
```bash
pip3 install boto3
```

In Python:
```python
import boto3
```

**Authentication**:
- Automatically uses AWS CLI creds in:
  ```
  ~/.aws/config
  ~/.aws/credentials
  ```
  (Typically set with `aws configure`)

**Best practice**: Use AWS IAM users/roles with **least privilege** policy.

***

## 3. Boto3 Basics – Example: Working with VPCs

To list all VPCs:
```python
import boto3

ec2_client = boto3.client('ec2')
all_vpcs = ec2_client.describe_vpcs()
for vpc in all_vpcs['Vpcs']:
    print(vpc['VpcId'], vpc['CidrBlock'])
```

Specify a region different from default:
```python
ec2_client = boto3.client('ec2', region_name='eu-west-3')
```

### Create a VPC & Subnet
```python
vpc_resp = ec2_client.create_vpc(CidrBlock='10.0.0.0/16')
vpc_id = vpc_resp['Vpc']['VpcId']

# Create subnet via client
ec2_client.create_subnet(VpcId=vpc_id, CidrBlock='10.0.1.0/24')

# Or using resource interface
ec2_resource = boto3.resource('ec2')
vpc_res = ec2_resource.Vpc(vpc_id)
vpc_res.create_subnet(CidrBlock='10.0.2.0/24')

# Add Name tag
ec2_client.create_tags(Resources=[vpc_id], Tags=[{'Key': 'Name', 'Value': 'my-vpc'}])
```

***

## 4. Terraform vs Python – Choosing the Right Tool

| **Terraform**                                | **Python (Boto3)**                               |
|----------------------------------------------|--------------------------------------------------|
| Declarative; idempotent; tracks infra state. | Imperative; no state tracking; flexible logic.   |
| Great for provisioning and infra mgmt.       | Great for scripts, jobs, workflows, monitoring. |
| Maintains current vs desired state diff.     | Executes exactly what you code, every time.     |

**Rule of thumb**:
- Use **Terraform** to provision & manage the infrastructure lifecycle.
- Use **Python/Boto3** for **operational automation** and business logic.

***

## 5. Common DevOps Use Cases with Boto3

### Health Checks (EC2)
- Periodically check instance states (running, stopped, pending, terminated)

***

### Scheduled Tasks with `schedule` Library
Install:
```bash
pip3 install schedule
```
Usage:
```python
import schedule, time

def job_with_arg(name):
    print(f"Running job for {name}")

schedule.every(10).seconds.do(job_with_arg, name="EC2 Checker")

while True:
    schedule.run_pending()
    time.sleep(1)
```
Integrate with EC2 status checks:
```python
schedule.every(5).seconds.do(check_instance_status)
```

***

### Bulk EC2 Tagging
Tag resources across regions for environment classification:
- Paris region → `production`
- Frankfurt region → `development`

***

### EKS Cluster Info
Summarise for multiple clusters:
- Status
- Kubernetes version
- API endpoint

***

### EC2 Volume Backups
Automate AMI/snapshot creation daily, for disaster recovery.

### EC2 Snapshot Cleanup
Keep last N snapshots, delete older ones.

### Restore Volumes from Snapshots
Automated creation of a new volume from latest backup and attach to instance.

***

## 6. Error Handling & Rollback Logic

Since Python doesn’t track infra state:
- Always wrap remote API changes in `try/except`.
- Implement cleanup/rollback logic.
- Log every step for audit.

Example:
```python
try:
    snapshot_id = ec2_client.create_snapshot(VolumeId=vol_id)['SnapshotId']
except Exception as e:
    logging.error(f"Snapshot failed: {e}")
```

***

## 7. Website & Service Monitoring

### Health Check
- Ping HTTP endpoint periodically.

### Email Alerts
- Send email if service is unreachable (via `smtplib` or SES).

### Auto-Recovery
1. Restart container (`docker restart` remote call via SSH).
2. If fails, reboot cloud server (e.g., Linode, AWS EC2).

***

## 8. AWS DevOps Automation Best Practices

- **Use IAM roles** for scripts running from EC2 or Lambda to avoid static creds.
- Implement **rate limiting** & `retry` logic to handle AWS throttling.
- Prefer `resource` interface for object-oriented access; `client` interface for fine-grained control.
- Use **pagination** for AWS list APIs:
```python
paginator = ec2_client.get_paginator('describe_instances')
for page in paginator.paginate():
    ...
```
- For scheduled tasks, consider:
  - **Lambda + EventBridge** for serverless executions.
  - **schedule** library or cronjobs for server-based scripts.
