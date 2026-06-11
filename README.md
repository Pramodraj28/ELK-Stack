# ELK Stack — AWS Infrastructure

Production-grade Elasticsearch cluster deployed on AWS across 3 Availability Zones with dedicated Master, Data (Hot), and Coordinator node roles. Includes CloudFormation templates for multiple deployment architectures and SSM Automation documents for zero-downtime cluster maintenance.

---

## Table of Contents

- [Architecture Overview](#architecture-overview)
- [Repository Structure](#repository-structure)
- [Node Roles](#node-roles)
- [Prerequisites](#prerequisites)
- [Deployment](#deployment)
- [Operational Automation (SSM Documents)](#operational-automation-ssm-documents)
- [Node Configuration](#node-configuration)
- [Security Considerations](#security-considerations)

---

## Architecture Overview

```
                     ┌──────────────────────────────────────┐
                     │           AWS VPC (3 AZs)             │
                     │                                        │
                     │  ┌──────────┐      ┌──────────────┐  │
                     │  │  Master  │      │ Coordinator  │  │
                     │  │   ASG    │      │    ASG (1x)  │  │
                     │  │  (1-2x)  │      └──────────────┘  │
                     │  └──────────┘                         │
                     │                                        │
                     │  ┌──────────────────────────────────┐ │
                     │  │       Data Node ASG (12x)         │ │
                     │  │  /data1 (nvme1+nvme2 LVM/XFS)    │ │
                     │  │  /data2 (nvme3+nvme4 LVM/XFS)    │ │
                     │  └──────────────────────────────────┘ │
                     │                                        │
                     │  ┌──────────────────────────────────┐ │
                     │  │   Internal Network Load Balancer  │ │
                     │  │    (cross-zone, 3 AZ coverage)    │ │
                     │  └──────────────────────────────────┘ │
                     └──────────────────────────────────────┘
```

### Component Summary

| Role | Instance Type | Count | Purpose |
|------|--------------|-------|---------|
| Master | m5d.16xlarge | 1–2 | Cluster state management |
| Data (Hot) | m5d.16xlarge | 12 | Active indexing and search |
| Coordinator | m5d.16xlarge | 1 | Query routing and aggregation |

---

## Repository Structure

```
ELK-Stack/
├── CloudFormation Templates
│   ├── 1_instance_stack_3zones.yml   # Single master + NLB across 3 AZs
│   ├── 3_instance_elk.yaml           # 3 direct EC2 instances (one per AZ)
│   ├── 3AZ.elk.yaml                  # Multi-AZ master/data/coordinator layout
│   ├── Elk-asg.yaml                  # ASG-based (Master x2, Data x12, Coordinator x1)
│   └── pramodelk.yml                 # Lightweight t3.medium variant for testing
│
├── SSM Automation Documents
│   ├── ES_Rolling_final.yml          # Zero-downtime rolling restart
│   ├── ES_Startup.yaml               # Cluster startup with health validation
│   ├── ES_Stop.yaml                  # Graceful cluster shutdown
│   ├── ES_Stop_Start.yaml            # Full stop/start cycle with reboot
│   └── ES_reboot.yaml                # Intelligent node reboot with uptime check
│
└── Node Configuration
    └── ES_UserData1/
        ├── 30-elasticsearch.conf     # Kernel parameter tuning (sysctl)
        ├── 30-es-elasticsearch.conf  # Process limits (ulimits)
        └── narrative.conf            # Internal secrets management client config
```

---

## Node Roles

| Role | Min | Max | Desired | Ports |
|------|-----|-----|---------|-------|
| Master | 1 | 2 | 1–2 | 9100 (health check), 9210 (discovery) |
| Data | 12 | 12 | 12 | 9200 (REST API), 9300 (cluster comms) |
| Coordinator | 1 | 1 | 1 | 9200 (REST API) |

### Port Reference

| Port | Protocol | Purpose |
|------|----------|---------|
| 9100 | TCP | Master node NLB health check |
| 9200 | HTTP/S | Elasticsearch REST API |
| 9210 | TCP | Inter-node discovery / gossip |
| 9300 | TCP | Node-to-node cluster communication |

---

## Prerequisites

- AWS CLI configured with sufficient IAM permissions
- VPC with private subnets across 3 AZs
- The following IAM policies available:
  - `AISSystemLogsPolicy`
  - `AmazonSSMManagedInstanceCore`
  - `elastic-snapshot-testing-full`
  - Permission boundary: `ais-permissions-boundaries`
- SSM Agent enabled on target accounts
- EC2 key pair created in the target region

---

## Deployment

### Option 1: ASG-based (Recommended for Production)

**Template:** `Elk-asg.yaml`

Deploys Master (x2), Data (x12), and Coordinator (x1) nodes via Auto Scaling Groups with an internal NLB.

```bash
aws cloudformation deploy \
  --template-file Elk-asg.yaml \
  --stack-name elk-prod \
  --parameter-overrides \
    VpcId=<vpc-id> \
    SubnetAZ1=<subnet-az1-id> \
    SubnetAZ2=<subnet-az2-id> \
    SubnetAZ3=<subnet-az3-id> \
    KeyName=<key-pair-name> \
  --capabilities CAPABILITY_IAM
```

### Option 2: Direct EC2 Instances (3 AZs)

**Template:** `3_instance_elk.yaml`

Deploys one EC2 instance per AZ — useful when you need full control over node placement without ASG management.

```bash
aws cloudformation deploy \
  --template-file 3_instance_elk.yaml \
  --stack-name elk-direct \
  --parameter-overrides \
    VpcId=<vpc-id> \
    SubnetAZ1=<subnet-az1-id> \
    SubnetAZ2=<subnet-az2-id> \
    SubnetAZ3=<subnet-az3-id> \
    KeyName=<key-pair-name> \
  --capabilities CAPABILITY_IAM
```

### Option 3: Lightweight (Non-Production / Testing)

**Template:** `pramodelk.yml`

Uses `t3.medium` instances with 2 Master nodes and 1 Data node ASG for dev/test environments.

```bash
aws cloudformation deploy \
  --template-file pramodelk.yml \
  --stack-name elk-dev \
  --parameter-overrides \
    VpcId=<vpc-id> \
    SubnetAZ1=<subnet-az1-id> \
    SubnetAZ2=<subnet-az2-id> \
    SubnetAZ3=<subnet-az3-id> \
    KeyName=<key-pair-name> \
  --capabilities CAPABILITY_IAM
```

### Storage Layout (per Data Node)

Each `m5d.16xlarge` node uses all 4 NVMe instance-store SSDs, striped via LVM into two mount points:

| Mount | Physical Volumes | Volume Group | Filesystem |
|-------|-----------------|--------------|------------|
| `/data1` | nvme1n1 + nvme2n1 | data1vg | XFS |
| `/data2` | nvme3n1 + nvme4n1 | data2vg | XFS |

Both are persisted in `/etc/fstab` (mounted by UUID). The root volume is 30 GB gp2, encrypted, with `DeleteOnTermination: false`.

---

## Operational Automation (SSM Documents)

All cluster operations run through AWS Systems Manager Automation — no direct SSH required. Credentials and cluster state are managed via the Elasticsearch API.

### Rolling Restart (Zero-Downtime)

**Document:** `ES_Rolling_final.yml`

Restarts nodes one at a time while maintaining cluster availability:

1. Start stopped Docker containers on the target node
2. Enable full shard allocation (`cluster.routing.allocation.enable: "all"`)
3. Poll `_cluster/health` in 60-second intervals until status is `green`

```bash
aws ssm start-automation-execution \
  --document-name ES_Rolling_final \
  --parameters InstanceIds=<instance-id>
```

### Startup

**Document:** `ES_Startup.yaml`

Validates cluster health and starts Elasticsearch containers in the correct sequence.

```bash
aws ssm start-automation-execution \
  --document-name ES_Startup \
  --parameters InstanceIds=<instance-ids>
```

### Graceful Stop

**Document:** `ES_Stop.yaml`

Safely shuts down the cluster without data loss:

1. Validate cluster is `green`
2. Set allocation to `primaries` (disable replica movement)
3. Flush all indices to disk
4. Stop Docker containers

```bash
aws ssm start-automation-execution \
  --document-name ES_Stop \
  --parameters InstanceIds=<instance-ids>
```

### Stop / Start Cycle

**Document:** `ES_Stop_Start.yaml`

Full graceful stop → reboot → restart cycle for maintenance windows:

1. Validate `green` health
2. Disable shard allocation, flush indices
3. Stop containers, trigger reboot
4. Post-reboot: restart containers
5. Re-enable full shard allocation
6. Poll until `green`

```bash
aws ssm start-automation-execution \
  --document-name ES_Stop_Start \
  --parameters InstanceIds=<instance-ids>
```

### Intelligent Node Reboot

**Document:** `ES_reboot.yaml`

Checks uptime before rebooting to avoid reboot loops. If uptime is under 1 minute, skips the reboot.

- **Pre-reboot:** Kills Docker containers, reboots instance
- **Post-reboot:** Waits 2 minutes, restarts exited containers, verifies Docker service health

```bash
aws ssm start-automation-execution \
  --document-name ES_reboot \
  --parameters InstanceIds=<instance-id>
```

---

## Node Configuration

OS-level tuning applied to all Elasticsearch nodes via bootstrap UserData:

### Kernel Parameters (`30-elasticsearch.conf`)

```ini
vm.max_map_count = 262144     # Required by Elasticsearch (default OS value is too low)
net.ipv4.tcp_retries2 = 5     # Aggressive retry limit for high-performance cluster
```

### Process Limits (`30-es-elasticsearch.conf`)

```
elasticsearch  soft  nproc   64000
elasticsearch  hard  nproc   64000
elasticsearch  soft  nofile  1048576
elasticsearch  hard  nofile  1048576
elasticsearch  soft  memlock unlimited
elasticsearch  hard  memlock unlimited
```

### Monitoring Integrations

| Tool | Purpose |
|------|---------|
| **osQuery** | Endpoint monitoring and audit logging |
| **Narrative (Mezu)** | Internal secrets and credential management |

### EC2 Metadata

- IMDSv2 enforced (`HttpTokens: required`)
- Hop limit: 2
- Instance metadata tags: enabled

---

## Security Considerations

- **Credentials:** Do not hardcode Elasticsearch passwords in SSM documents or CloudFormation templates. Use [AWS Secrets Manager](https://docs.aws.amazon.com/secretsmanager/latest/userguide/intro.html) and reference secrets via `{{resolve:secretsmanager:...}}` in CloudFormation or `{{ssm-secure:...}}` in SSM documents.
- **TLS Verification:** The `curl -sk` flag in SSM scripts skips TLS certificate validation. Replace self-signed certificates with ACM-issued or CA-signed certs and remove the `-k` flag.
- **Security Groups:** Restrict ingress to known CIDR ranges. Avoid open `0.0.0.0/0` rules in production.
- **IAM Least Privilege:** Apply permission boundaries and scope policies to only the required actions and resources.
- **EBS Encryption:** Verify `Encrypted: true` is set on all EBS volumes (root and data).
- **Audit Logging:** osQuery provides endpoint-level audit trails — ensure logs are shipped to a centralized log store.
