---
title: Setup Production-ready HashiCorp Vault cluster
key: 20241002
tags: howtos
modify_date: 2017-09-09
author: Harry - The DevOps Guy
show_author_profile: true
aside:
  toc: true
---

To set up a production-ready HashiCorp Vault cluster using AWS Auto Scaling Group (ASG) and DynamoDB as the storage backend, you need to ensure scalability, high availability, auto-unsealing, and proper security configurations. Below is a detailed step-by-step guide:

### **Architecture Overview:**

- **Vault Instances**: Managed by an ASG for scalability and availability.
- **DynamoDB**: Used as the highly available storage backend for Vault.
- **ALB (Application Load Balancer)**: For routing requests to Vault instances.
- **Auto-unseal**: Using AWS KMS to automatically unseal Vault instances.
- **IAM Roles**: For secure access to DynamoDB and KMS.

### **Step-by-Step Guide:**

### Step 1: **Set up AWS Infrastructure**

#### 1.1 **Launch Configuration/Launch Template**
- Create a Launch Configuration or Launch Template that will define the EC2 instance details used by the Auto Scaling Group.
- **Example configuration**:
  - **AMI**: Use an Ubuntu AMI or other Linux distributions supported by Vault.
  - **Instance Type**: Select an instance type with enough memory and CPU (e.g., `t3.medium` or `m5.large`).
  - **Security Groups**:
    - Allow traffic on **port 8200** (for Vault API/UI) and **port 8201** (for Vault clustering).
    - Restrict access to trusted IP ranges.
  - **IAM Role**: Attach an IAM role with permissions for DynamoDB, KMS, and SSM (for secrets management and configuration).

#### 1.2 **Create an Auto Scaling Group**
- Create an Auto Scaling Group (ASG) across multiple availability zones (AZs) for high availability.
  - **Min Size**: 3 instances (at least 3 nodes for HA).
  - **Max Size**: Set according to your scalability needs.
- Attach the Launch Configuration or Launch Template created in the previous step.
- Ensure **health checks** are set to **EC2 status checks** and/or **ALB health checks**.

#### 1.3 **Configure an Application Load Balancer (ALB)**
- Create an ALB for routing traffic to the Vault instances.
  - **Target Group**: Configure a target group with HTTP health checks on port **8200**.
  - **Listeners**: Set up listeners on the ALB to forward traffic to port **8200** on the Vault instances.
  - Enable cross-zone load balancing for even traffic distribution.

### Step 2: **Set Up DynamoDB for Vault Backend Storage**

1. **Create a DynamoDB Table**:
   - Create a table (e.g., `vault`) with the primary key being `Key` (type: string).
   - Ensure the table is set up for **on-demand scaling** or provision capacity based on traffic needs.
   - No secondary indexes are required.

### Step 3: **IAM Permissions**

#### 3.1 **DynamoDB Access**
- Create an IAM Role with the following permissions for DynamoDB:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "dynamodb:PutItem",
        "dynamodb:GetItem",
        "dynamodb:DeleteItem",
        "dynamodb:Scan",
        "dynamodb:Query",
        "dynamodb:UpdateItem"
      ],
      "Resource": "arn:aws:dynamodb:<region>:<account-id>:table/vault"
    }
  ]
}
```

#### 3.2 **KMS for Auto-Unsealing**
- Grant the IAM role access to AWS KMS for auto-unsealing:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "kms:Decrypt",
        "kms:Encrypt",
        "kms:GenerateDataKey"
      ],
      "Resource": "arn:aws:kms:<region>:<account-id>:key/<kms-key-id>"
    }
  ]
}
```

### Step 4: **Install and Configure Vault on EC2 Instances**

#### 4.1 **Install Vault**
Use the user data in the ASG or SSH into your EC2 instances to install Vault:

```bash
sudo apt-get update
sudo apt-get install -y unzip
curl -O https://releases.hashicorp.com/vault/<version>/vault_<version>_linux_amd64.zip
unzip vault_<version>_linux_amd64.zip
sudo mv vault /usr/local/bin/
vault -autocomplete-install
complete -C /usr/local/bin/vault vault
```

#### 4.2 **Configure Vault**
Create a Vault configuration file (`/etc/vault.d/vault.hcl`) to configure storage, listener, and clustering options.

**Sample Vault Configuration:**

```hcl
# Storage backend using DynamoDB
storage "dynamodb" {
  table = "vault"
  region = "<aws-region>"
}

# AWS KMS Auto-Unseal
seal "awskms" {
  region     = "<aws-region>"
  kms_key_id = "<kms-key-id>"
}

# Listener
listener "tcp" {
  address     = "0.0.0.0:8200"
  tls_disable = 1
}

# Vault UI
ui = true

# Cluster configuration
api_addr     = "https://<ALB-DNS>:8200"
cluster_addr = "https://<Instance-Private-IP>:8201"
```

- Replace `<aws-region>`, `<kms-key-id>`, and `<ALB-DNS>` with appropriate values.

#### 4.3 **Set Vault as a Service**
Configure Vault to run as a service using `systemd`.

- Create a `vault.service` file at `/etc/systemd/system/vault.service`:

```ini
[Unit]
Description=Vault
Documentation=https://www.vaultproject.io/docs/
After=network-online.target
Wants=network-online.target

[Service]
User=vault
Group=vault
ExecStart=/usr/local/bin/vault server -config=/etc/vault.d/vault.hcl
ExecReload=/bin/kill --signal HUP $MAINPID
KillMode=process
LimitNOFILE=65536
Restart=on-failure
RestartSec=5
TimeoutStartSec=30s
TimeoutStopSec=5s
StartLimitBurst=3
StartLimitInterval=60s

[Install]
WantedBy=multi-user.target
```

- Enable and start Vault:

```bash
sudo systemctl enable vault
sudo systemctl start vault
```

### Step 5: **Initialize and Unseal Vault**

#### 5.1 **Initialize Vault (only once)**
- SSH into one of the Vault instances and initialize it:

```bash
vault operator init
```

- Save the unseal keys and root token securely.

#### 5.2 **Unseal Vault on Each Instance**
- Use the `vault operator unseal` command on each instance to unseal the Vault cluster nodes.

#### 5.3 **Configure Auto-Unseal (Optional)**
- If you've set up KMS for auto-unsealing, Vault will automatically unseal on node restarts without manual intervention.

### Step 6: **Configure ALB and Health Checks**

- Ensure your ALB is configured with health checks on port `8200`.
- Vault’s health check endpoint is `/v1/sys/health`.

**Sample ALB Health Check Configuration:**
- **Protocol**: HTTP
- **Path**: `/v1/sys/health`
- **Port**: `8200`
- **Healthy Threshold**: `3`
- **Unhealthy Threshold**: `3`
- **Timeout**: `5`
- **Interval**: `30`

### Step 7: **Monitoring and Logging**

- Enable monitoring with **CloudWatch** for your EC2 instances, ASG, and DynamoDB.
- Use **Vault audit logs** to track requests and responses for security and troubleshooting.

**Enable Vault Audit Logging:**

```bash
vault audit enable file file_path=/var/log/vault_audit.log
```

### Step 8: **Scale and Test**

- Test the ASG by scaling up/down to ensure new instances join the cluster properly.
- Validate that ALB routing and Vault health checks work as expected.
