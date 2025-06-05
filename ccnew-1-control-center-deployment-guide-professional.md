# Mojaloop Control Center Deployment Guide

## Table of Contents

1. [Introduction](#1-introduction)
2. [Prerequisites](#2-prerequisites)
3. [Infrastructure Preparation](#3-infrastructure-preparation)
4. [Control Center Host Deployment](#4-control-center-host-deployment)
5. [Control Center Utility Container Setup](#5-control-center-utility-container-setup)
6. [Control Center Deployment](#6-control-center-deployment)
7. [Post-Deployment Configuration](#7-post-deployment-configuration)
8. [Verification and Access](#8-verification-and-access)
9. [Troubleshooting](#9-troubleshooting)
10. [Maintenance Operations](#10-maintenance-operations)

## 1. Introduction

This guide provides comprehensive instructions for deploying the Mojaloop Control Center, a centralized management platform that orchestrates multiple Mojaloop environments. The Control Center provides integrated CI/CD pipelines, monitoring, security, and infrastructure management capabilities through a GitOps-driven architecture.

### 1.1 Architecture Overview

The Control Center deployment consists of:
- A Kubernetes cluster running on AWS infrastructure
- Integrated services including GitLab, ArgoCD, Vault, and monitoring tools
- Multi-tenant environment support with isolated namespaces
- Automated certificate management and DNS configuration

### 1.2 Deployment Approach

The deployment utilizes Infrastructure as Code (IaC) principles with:
- Terragrunt for infrastructure provisioning
- Ansible for configuration management
- ArgoCD for GitOps-based application deployment

## 2. Prerequisites

### 2.1 Required Tools and Access

Before beginning the deployment, ensure you have:

#### 2.1.1 AWS Account Access
- Administrative privileges or IAM user with sufficient permissions
- AWS CLI configured with appropriate credentials
- Minimum service quotas:
  - vCPU limit: 64 (for m5.4xlarge instances)
  - Elastic IPs: 5
  - VPCs: 1 additional

#### 2.1.2 Local Development Environment
- SSH client with key management capabilities
- Terminal with bash shell support
- Text editor for configuration files

#### 2.1.3 Network Requirements
- Available domain name for the Control Center
- Access to DNS management (Route53 or external provider)
- Firewall rules allowing SSH access to bastion hosts

### 2.2 Knowledge Requirements

Operators should be familiar with:
- Kubernetes concepts and kubectl operations
- AWS services (EC2, VPC, IAM)
- Docker container management
- Basic networking and DNS concepts

## 3. Infrastructure Preparation

### 3.1 Create SSH Key Pair

Generate an SSH key pair for secure access to the Control Center infrastructure.

#### 3.1.1 Generate Key Pair
Create the key pair through AWS EC2 console or CLI

#### 3.1.2 Store Private Key
Save the private key securely on your local machine:

```bash
# Store the private key
mkdir -p ~/.ssh
vi ~/.ssh/ml-perf-ccu-host-private-key

# Paste your private key content (ensure proper formatting)
# The key should begin with -----BEGIN RSA PRIVATE KEY-----
# and end with -----END RSA PRIVATE KEY-----

# Set appropriate permissions
chmod 400 ~/.ssh/ml-perf-ccu-host-private-key
```

### 3.2 Configure AWS IAM

Create the required IAM group for Control Center operations:

#### 3.2.1 Create IAM Group
```bash
# Create the IAM group
aws iam create-group \
  --group-name iac_admin \
  --profile mojaiac
```

#### 3.2.2 Attach Policies
```bash
# Attach administrator access policy
aws iam attach-group-policy \
  --group-name iac_admin \
  --policy-arn arn:aws:iam::aws:policy/AdministratorAccess \
  --profile mojaiac
```

### 3.3 Provision Control Center Host VM

Deploy a dedicated VM to host the Control Center utility container:

#### 3.3.1 VM Specifications
- **Instance Type**: t3.small
- **Operating System**: Ubuntu 24.04 LTS
- **Storage**: 20GB root volume (expandable as needed)
- **Security Group**: Allow SSH (port 22) from your IP
- **Network**: Public subnet with Elastic IP
- **Authentication**: SSH key created in Step 3.1

#### 3.3.2 Record Configuration
Record the public IP address for SSH access.

## 4. Control Center Host Deployment

### 4.1 Initial System Configuration

Connect to the Control Center host and perform initial setup:

#### 4.1.1 SSH Connection
```bash
# Connect via SSH
ssh -i ~/.ssh/ml-perf-ccu-host-private-key ubuntu@<PUBLIC_IP_ADDRESS>
```

#### 4.1.2 System Updates
```bash
# Switch to root user
sudo su

# Update system packages
apt-get update && apt-get upgrade -y
```

### 4.2 Install Terminal Multiplexer

Install tmux to ensure long-running processes continue if SSH connection drops:

```bash
apt install tmux
tmux -V  # Verify installation
```

### 4.3 Install Docker Engine

Install Docker following the official Ubuntu installation procedure:

#### 4.3.1 Remove Conflicting Packages
```bash
for pkg in docker.io docker-doc docker-compose docker-compose-v2 podman-docker containerd runc; do 
  sudo apt-get remove $pkg
done
```

#### 4.3.2 Add Docker Repository
```bash
# Add Docker's official GPG key
sudo apt-get update
sudo apt-get install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# Add Docker repository
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

#### 4.3.3 Install Docker Packages
```bash
sudo apt-get update
apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

### 4.4 Configure AWS Credentials

Set up AWS credentials for the Control Center utility:

#### 4.4.1 Create Configuration Directory
```bash
mkdir -p ~/.aws/
```

#### 4.4.2 Create Credentials File
```bash
cat <<EOF > ~/.aws/credentials
[oss]
aws_access_key_id = <YOUR_ACCESS_KEY_ID>
aws_secret_access_key = <YOUR_SECRET_ACCESS_KEY>
EOF

# Replace <YOUR_ACCESS_KEY_ID> and <YOUR_SECRET_ACCESS_KEY> with actual values
```

## 5. Control Center Utility Container Setup

### 5.1 Launch Container

Start a new tmux session and launch the Control Center utility container:

#### 5.1.1 Create Tmux Session
```bash
tmux new -s ml-perf-ccu-4
```

#### 5.1.2 Run Control Center Container
```bash
docker run -it -d \
    -v ~/.aws:/root/.aws \
    --name ml-perf-ccu-4 \
    --hostname ml-perf-ccu-4 \
    --cap-add SYS_ADMIN \
    --cap-add NET_ADMIN \
    ghcr.io/mojaloop/control-center-util:6.1.2
```

#### 5.1.3 Access Container
```bash
docker exec -it ml-perf-ccu-4 bash
```

### 5.2 Configure Environment

Navigate to the IaC directory and configure environment variables:

#### 5.2.1 Navigate to Working Directory
```bash
cd /iac-run-dir
```

#### 5.2.2 Edit Environment File
```bash
vi setenv
```

#### 5.2.3 Update Configuration
Update the `setenv` file with your deployment parameters:

```bash
export AWS_PROFILE=oss
export PRIVATE_REPO_TOKEN=nullvalue
export PRIVATE_REPO_USER=nullvalue
export ANSIBLE_BASE_OUTPUT_DIR=$PWD/output
export IAC_TERRAFORM_MODULES_TAG=v5.9.0  # Specify desired version
export PRIVATE_REPO=example.com
```

### 5.3 Initialize Environment

Source the configuration and initialize the deployment environment:

#### 5.3.1 Load Configuration
```bash
source setenv
```

#### 5.3.2 Run Initialization
```bash
./init.sh
```

#### 5.3.3 Navigate to Deployment Directory
```bash
cd /iac-run-dir/iac-modules/terraform/ccnew/
```

## 6. Control Center Deployment

### 6.1 Configure Cluster Settings

Edit the cluster configuration file to define your Control Center parameters:

#### 6.1.1 Open Configuration File
```bash
vi /iac-run-dir/iac-modules/terraform/ccnew/custom-config/cluster-config.yaml
```

#### 6.1.2 Set Cluster Parameters
Configure the following parameters according to your requirements:

```yaml
cluster_name: cc004                          # Unique identifier for your Control Center
domain: perf004.mojaperflab.org             # Your domain name
cloud_region: eu-north-1                    # AWS region for deployment
ansible_collection_tag: v5.5.0-rc3          # Ansible collection version
iac_terraform_modules_tag: v5.9.0           # IaC modules version
letsencrypt_email: admin@yourdomain.com     # Email for Let's Encrypt certificates
tags:                                       # AWS resource tags
  Origin: Terraform
  mojaloop/cost_center: mlf-perf004-cc
  mojaloop/env: ft-sbox-rw
  mojaloop/owner: Your-Name
```

### 6.2 Define Environments

Configure the environments that will be managed by this Control Center:

#### 6.2.1 Edit Environment Configuration

Edit /iac-run-dir/iac-modules/terraform/ccnew/custom-config/environment.yaml to declare the names of the switch and payment manager projects to be created at the control center deployment. 

The environments can be created after the control center deployment as well.

#### 6.2.2 List Environments
```yaml
environments:
  - sw004    # switch environment
  - pm004    # payment manager environment
```

### 6.3 Deploy Control Center

Execute the deployment wrapper script:

#### 6.3.1 Run Deployment
```bash
./wrapper.sh
```

#### 6.3.2 Deployment Process
This process will:
1. Validate configuration files
2. Create AWS infrastructure (VPC, subnets, instances)
3. Deploy Kubernetes cluster
4. Install and configure all Control Center services
5. Set up GitOps with ArgoCD

#### 6.3.3 Monitor Progress
- **Expected Duration**: 45-60 minutes
- Monitor the deployment progress through the terminal output
- The script will display status updates for each component

### 6.4 Migrate Terraform State

After successful deployment, migrate the Terraform state to the Kubernetes backend:

```bash
./movestatetok8s.sh
```

This enables team collaboration and state persistence within the Control Center.

## 7. Post-Deployment Configuration

### 7.1 Access Zitadel Identity Provider

#### 7.1.1 Navigate to Zitadel
Access URL: `https://zitadel.cc004.perf004.mojaperflab.org/`

#### 7.1.2 Login with Default Credentials
- Username: `rootauto@zitadel.zitadel.cc004.perf004.mojaperflab.org`
- Password: `#Password1!`

### 7.2 Create User Account

#### 7.2.1 Create New User
1. Create a new user account for administration
2. Set a strong password following security best practices

#### 7.2.2 Configure Security
1. Enable two-factor authentication
2. Grant appropriate permissions through the root user account

**Important**: All Control Center services (GitLab, ArgoCD, Grafana, Vault, etc.) use Zitadel for Single Sign-On (SSO). Once you create your user account in Zitadel, you will use the same credentials to access all portals.

### 7.3 Configure Service Access

Access each service using your new user credentials:

#### 7.3.1 Netbird VPN Setup
- URL: `https://netbird-dashboard.cc004.perf004.mojaperflab.org`
- Retrieve VPN client configuration
- Note management and admin URLs

#### 7.3.2 Install Netbird Client
1. Download client for your operating system
2. Configure with management URL from dashboard
3. Establish VPN connection

#### 7.3.3 GitLab Configuration
- URL: `https://gitlab.cc004.perf004.mojaperflab.org`
- Enable two-factor authentication for enhanced security

### 7.4 Access Internal Services

Once connected via VPN, access internal services:

#### 7.4.1 ArgoCD
- URL: `https://argocd.int.cc004.perf004.mojaperflab.org`
- Sync the `netbird-post-config` application
- Verify all applications are healthy

#### 7.4.2 Vault
- URL: `https://vault.int.cc004.perf004.mojaperflab.org/`
- Login with OIDC authentication
- Verify secret paths are accessible

#### 7.4.3 Grafana
- URL: `https://grafana.int.cc004.perf004.mojaperflab.org/`
- Review pre-configured dashboards
- Set up alert channels if required

## 8. Verification and Access

### 8.1 Service Health Checks

Verify all services are operational:

#### 8.1.1 Check ArgoCD Applications
```bash
kubectl get applications -n argocd
```

#### 8.1.2 Verify Pod Status
```bash
kubectl get pods --all-namespaces | grep -v Running
```

#### 8.1.3 Check Istio Configuration
```bash
kubectl get gateway -n istio-system
```

### 8.2 DNS Verification

Confirm DNS records are properly configured:

#### 8.2.1 Test External Services
```bash
nslookup gitlab.cc004.perf004.mojaperflab.org
nslookup zitadel.cc004.perf004.mojaperflab.org
```

#### 8.2.2 Test Internal Services
From within VPN connection:
```bash
nslookup argocd.int.cc004.perf004.mojaperflab.org
```

## 9. Troubleshooting

### 9.1 AWS Quota Exceeded

#### 9.1.1 Error Message
"You have requested more vCPU capacity than your current vCPU limit"

#### 9.1.2 Resolution
1. Access AWS Service Quotas console
2. Request increase for EC2 instance vCPU limit
3. Wait for approval before retrying deployment

### 9.2 IAM Group Not Found

#### 9.2.1 Error Message
"The group with name iac_admin cannot be found"

#### 9.2.2 Resolution
Execute the IAM group creation commands from Section 3.2

### 9.3 Terraform State Lock

#### 9.3.1 Error Message
"Error acquiring the state lock"

#### 9.3.2 Resolution
```bash
# Force unlock with the lock ID from error message
terragrunt force-unlock <LOCK_ID>
```

### 9.4 Certificate Generation Failures

#### 9.4.1 Issue
Let's Encrypt certificate requests failing

#### 9.4.2 Resolution
1. Verify DNS propagation has completed
2. Check Let's Encrypt rate limits
3. Ensure domain ownership verification

## 10. Maintenance Operations

### 10.1 Adding New Environments

#### 10.1.1 Update Configuration
```bash
vi custom-config/environment.yaml
# Add new environment to the list
```

#### 10.1.2 Refresh Templates
```bash
./refresh-env-templates.sh
```

#### 10.1.3 Apply Changes
Sync changes through ArgoCD

### 10.2 Storage Expansion

If additional storage is required on the Control Center host:

#### 10.2.1 Check Current Usage
```bash
sudo lsblk
```

#### 10.2.2 Expand Partition
```bash
# Adjust device name as needed
sudo growpart /dev/nvme0n1 1
```

#### 10.2.3 Resize Filesystem
```bash
sudo resize2fs /dev/nvme0n1p1
```

### 10.3 Destroying Control Center

Make sure all switch and PM4ML environments are successfully destroyed before to destroy the control center.

To completely remove the Control Center:

#### 10.3.1 Navigate to Directory
```bash
cd /iac-run-dir/iac-modules/terraform/ccnew/
```

#### 10.3.2 Load Configuration
```bash
source externalrunner.sh
source scripts/setlocalvars.sh
```

#### 10.3.3 Migrate State
```bash
./movestatefromk8s.sh
```

#### 10.3.4 Destroy Resources
```bash
terragrunt run-all destroy --terragrunt-non-interactive
```
