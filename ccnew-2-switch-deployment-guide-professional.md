# Mojaloop Switch Environment Deployment Guide

## Table of Contents

1. [Introduction](#1-introduction)
2. [Prerequisites](#2-prerequisites)
3. [Environment Initialization](#3-environment-initialization)
4. [AWS Credentials Configuration](#4-aws-credentials-configuration)
5. [Deployment Configuration](#5-deployment-configuration)
6. [Deployment Execution](#6-deployment-execution)
7. [Post-Deployment Verification](#7-post-deployment-verification)
8. [Service Access Configuration](#8-service-access-configuration)
9. [Testing and Validation](#9-testing-and-validation)
10. [Integration Configuration](#10-integration-configuration)

## 1. Introduction

This guide provides instructions for deploying a Mojaloop Switch environment using the Control Center's GitOps infrastructure. The Switch environment serves as the central hub for processing financial transactions between Digital Financial Service Providers (DFSPs).

### 1.1 Architecture Overview

The Switch deployment consists of:
- Kubernetes cluster with Mojaloop core services
- Identity and access management with Keycloak
- Monitoring and observability stack
- Testing toolkit for validation
- Management portals for operations

### 1.2 Deployment Approach

The deployment utilizes:
- GitLab CI/CD pipelines for automation
- Terragrunt/Terraform for infrastructure provisioning
- ArgoCD for GitOps-based application deployment
- Vault for secrets management

## 2. Prerequisites

### 2.1 Required Access

Before beginning the deployment, ensure you have:

#### 2.1.1 Control Center Access
- Active user account in Zitadel
- Access to Control Center GitLab
- VPN connection via Netbird
- Appropriate RBAC permissions

#### 2.1.2 AWS Resources
- AWS IAM user with deployment permissions
- Access key ID and secret access key
- Sufficient service quotas in target region

#### 2.1.3 Tools and Software
- kubectl CLI installed
- kubelogin for OIDC authentication
- Web browser for GitLab Web IDE access

### 2.2 Knowledge Requirements

Operators should be familiar with:
- Kubernetes operations
- GitLab CI/CD pipelines
- Mojaloop architecture
- YAML configuration syntax

## 3. Environment Initialization

### 3.1 Bootstrap Environment Repository

Initialize the environment repository using the Control Center's bootstrap project.

#### 3.1.1 Access GitLab Bootstrap Project
Navigate to the bootstrap project in Control Center GitLab

#### 3.1.2 Configure Pipeline Variables
Set the following variables for the `deploy-env-templates` job:

```yaml
ENV_TO_UPDATE: sw004
IAC_MODULES_VERSION_TO_UPDATE: v5.9.0
```

#### 3.1.3 Execute Pipeline
Run the `deploy-env-templates` job to create the environment repository structure

### 3.2 Access Environment Repository

After successful initialization:

#### 3.2.1 Navigate to Repository
Access the newly created environment repository at:
```
https://gitlab.cc004.perf004.mojaperflab.org/iac/sw004
```

#### 3.2.2 Verify Structure
Ensure the repository contains:
- `custom-config/` directory
- `default-config/` directory
- CI/CD pipeline configuration

## 4. AWS Credentials Configuration

### 4.1 Configure GitLab CI/CD Variables

#### 4.1.1 Access Project Settings
1. Navigate to your environment repository in GitLab
2. Go to Settings → CI/CD → Variables

#### 4.1.2 Add AWS Access Key
Create a new variable:
- Key: `AWS_ACCESS_KEY_ID`
- Value: Your AWS access key ID
- Type: Variable
- Protected: Yes
- Masked: Yes

### 4.2 Configure Vault Secret

#### 4.2.1 Access Vault
Connect to Vault using the internal URL (requires VPN):
```
https://vault.int.cc004.perf004.mojaperflab.org/
```

#### 4.2.2 Create Secret
Navigate to the appropriate path and create:
- Path: `secret/data/sw004/cloud_platform_client_secret`
- Key: `value`
- Value: Your AWS secret access key

## 5. Deployment Configuration

### 5.1 Configure Cluster Settings

#### 5.1.1 Edit Cluster Configuration
1. In the GitLab repository, navigate to `custom-config/cluster-config.yaml`
2. Click "Edit" to open the GitLab Web IDE
3. Modify the configuration as follows:

#### 5.1.2 Set Configuration Parameters
```yaml
env: sw004 # update according to your DNS planning
vpc_cidr: "10.107.0.0/23"
managed_vpc_cidr: "10.29.0.0/23"
domain: hub004.mojaperflab.org # update according to DNS planning
managed_svc_enabled: false
k8s_cluster_type: microk8s
currency: EUR
cloud_region: eu-north-1 # update according to your region planning
ansible_collection_tag: v5.5.0-rc3
iac_terraform_modules_tag: v5.9.0
letsencrypt_email: admin@yourdomain.com
tags:
  Origin: Terraform
  mojaloop/cost_center: mlf-perf004-sw
  mojaloop/env: ft-sbox-rw
  mojaloop/owner: Your-Name
```

### 5.2 Configure Mojaloop Version

#### 5.2.1 Edit Mojaloop Variables
1. In the GitLab repository, navigate to `custom-config/mojaloop-vars.yaml`
2. Click "Edit" to open the GitLab Web IDE
3. Update the configuration:

#### 5.2.2 Set Chart Version
```yaml
# Check latest version at https://github.com/mojaloop/helm/releases/
mojaloop_chart_version: 17.0.0
```

### 5.3 Copy Required Configuration Files

#### 5.3.1 Copy RBAC Configuration
1. In the GitLab repository, navigate to `default-config/mojaloop-rbac-api-resources.yaml`
2. Click on the file to view its contents
3. Copy the entire content
4. Navigate to `custom-config/` directory
5. Click "New file" and name it `mojaloop-rbac-api-resources.yaml`
6. Paste the copied content and commit

#### 5.3.2 Copy Stateful Resources Configuration
1. In the GitLab repository, navigate to `default-config/mojaloop-stateful-resources.json`
2. Click on the file to view its contents
3. Copy the entire content
4. Navigate to `custom-config/` directory
5. Click "New file" and name it `mojaloop-stateful-resources.json`
6. Paste the copied content and commit

### 5.4 Commit Configuration

#### 5.4.1 Commit All Changes
After creating and editing all configuration files:
1. Use GitLab's Web IDE or commit interface
2. Add a commit message
3. Commit directly to the `main` branch

## 6. Deployment Execution

### 6.1 Run Initialization Pipeline

#### 6.1.1 Access GitLab Pipelines
Navigate to CI/CD → Pipelines in your environment repository

#### 6.1.2 Execute Init Pipeline
1. Click "Run pipeline"
2. Select `main` branch
3. Choose `init` stage
4. Monitor execution

### 6.2 Deploy Infrastructure

#### 6.2.1 Execute Deployment Pipeline
1. After init completes successfully
2. Run `deploy-infra` pipeline
3. Monitor progress (45-60 minutes)

#### 6.2.2 Download Kubeconfig
After successful deployment:
1. Go to pipeline artifacts
2. Download `kubeconfig` file
3. Save to local system

## 7. Post-Deployment Verification

### 7.1 Configure Kubernetes Access

#### 7.1.1 Install OIDC Plugin
```bash
# For macOS
brew install int128/kubelogin/kubelogin

# For other systems, see: https://github.com/int128/kubelogin
```

#### 7.1.2 Set Kubeconfig
```bash
# Save downloaded kubeconfig
mkdir -p ~/.kube
mv ~/Downloads/kubeconfig ~/.kube/sw004-config

# Export configuration
export KUBECONFIG=~/.kube/sw004-config
```

### 7.2 Verify Deployment

#### 7.2.1 Check ArgoCD Applications
```bash
kubectl get Application -n argocd
```

#### 7.2.2 Verify All Pods Running
```bash
kubectl get pods --all-namespaces | grep -v Running
```

### 7.3 Configure User Permissions

#### 7.3.1 Access Zitadel
Login to Control Center Zitadel to grant permissions

#### 7.3.2 Add User to Environment Group
1. Navigate to the sw004 project/group
2. Add your user with appropriate role
3. Save changes

## 8. Service Access Configuration

### 8.1 ArgoCD Access

#### 8.1.1 Get ArgoCD URL
```bash
kubectl get VirtualService -n argocd
```

#### 8.1.2 Login Credentials
- Username: `admin`
- Authentication: SSO via Zitadel (recommended)

Alternative password access:
```bash
kubectl get secret argocd-initial-admin-secret -n argocd \
  -o jsonpath='{.data.password}' | base64 -d
```

### 8.2 Grafana Monitoring

#### 8.2.1 Get Grafana URL
```bash
kubectl get VirtualService -n monitoring
```

#### 8.2.2 Access Methods
**Option 1: SSO via Zitadel** (recommended)

**Option 2: Local Admin Account**
```bash
# Get username
kubectl get secret grafana-admin-secret -n monitoring \
  -o jsonpath='{.data.admin-user}' | base64 -d

# Get password
kubectl get secret grafana-admin-secret -n monitoring \
  -o jsonpath='{.data.admin-pw}' | base64 -d
```

### 8.3 Finance Portal

#### 8.3.1 Get Portal URL
```bash
kubectl get VirtualService finance-portal-vs -n mojaloop
```

#### 8.3.2 Login Credentials
- Username: `portal_admin`
- Password: Retrieved from Keycloak secret

### 8.4 Keycloak Administration

#### 8.4.1 Get Keycloak URLs
```bash
# Admin console URL
kubectl get VirtualService keycloak-admin-vs -n keycloak

# External URL for OIDC
kubectl get VirtualService keycloak-ext-vs -n keycloak
```

#### 8.4.2 Admin Credentials
```bash
# Get username
kubectl get secret switch-keycloak-initial-admin -n keycloak \
  -o jsonpath='{.data.username}' | base64 -d

# Get password
kubectl get secret switch-keycloak-initial-admin -n keycloak \
  -o jsonpath='{.data.password}' | base64 -d
```

### 8.5 Mojaloop Connection Manager (MCM)

#### 8.5.1 Get MCM URL
```bash
kubectl get VirtualService mcm-vs -n mcm
```

#### 8.5.2 Login Credentials
- Username: `portal_admin`
- Password:
```bash
kubectl get secret portal-admin-secret -n keycloak \
  -o jsonpath='{.data.secret}' | base64 -d
```

## 9. Testing and Validation

### 9.1 Testing Toolkit (TTK) Access

#### 9.1.1 Get TTK URL
```bash
kubectl get VirtualService mojaloop-ttkfront-vs -n mojaloop
```

#### 9.1.2 Automated Testing
Note: The pod `moja-ml-ttk-test-setup` in the mojaloop namespace runs automated tests after deployment.

### 9.2 Run Golden Path Tests

#### 9.2.1 Download Test Collection
1. Visit: https://github.com/mojaloop/testing-toolkit-test-cases/releases/tag/v17.0.15
2. Download the appropriate test collection

#### 9.2.2 Configure TTK
1. Load the golden path provisioning collection
2. Navigate to: Collections → Hub → Golden Path → P2P Money Transfer
3. Set environment to: `examples/environments/hub-k8s-default-environment.json`

#### 9.2.3 Execute Tests
1. Run the test suite
2. Verify all tests pass successfully
3. Review test results and logs

## 10. Collect PM4ML Integration Configuration

### 10.1 Collect Integration URLs

For DFSP/PM4ML integration, collect the following URLs:

#### 10.1.1 OIDC Authentication URL
```bash
# External Switch OIDC URL
kubectl get VirtualService keycloak-ext-vs -n keycloak
```

#### 10.1.2 Interoperability API URL
```bash
# External Switch FQDN
kubectl get VirtualService interop-vs -n mojaloop
```

#### 10.1.3 MCM Public URL
```bash
# External MCM Public FQDN
kubectl get VirtualService mcm-vs -n mcm
```

### 10.2 Get JWT Token

#### 10.2.1 Access Keycloak
1. Login to Keycloak admin console
2. Select `fsps` realm from top-left dropdown

#### 10.2.2 Retrieve JWT Secret
1. Navigate to Clients in left menu
2. Select `dfsp-jwt` from the list
3. Go to Credentials tab
4. Copy the `fsp-jwt` secret

### 10.3 Document Integration Details

Create a document with:
- All collected URLs
- JWT token/secret
- Network connectivity requirements

## 11. Troubleshooting

### 11.1 Pipeline Failures

#### 11.1.1 Init Pipeline Issues
- Verify AWS credentials are correctly set
- Check Vault connectivity
- Review pipeline logs for specific errors

#### 11.1.2 Infrastructure Deployment Failures
- Check AWS service quotas
- Verify domain ownership
- Review Terraform state for conflicts

### 11.2 Service Access Issues

#### 11.2.1 OIDC Authentication Problems
- Verify Zitadel user permissions
- Check kubelogin installation
- Ensure VPN connection is active

#### 11.2.2 Service Unavailable
- Check pod status
- Review ArgoCD sync status
- Verify Istio gateways and virtual services

### 11.3 Testing Failures

#### 11.3.1 TTK Connection Issues
- Verify TTK URL accessibility
- Check network policies
- Review Istio configuration

#### 11.3.2 Golden Path Test Failures
- Ensure all services are healthy
- Check database connections
- Review service logs for errors