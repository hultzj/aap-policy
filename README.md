# AAP Setup GitOps

[![License](https://img.shields.io/badge/License-Apache%202.0-blue.svg)](https://opensource.org/licenses/Apache-2.0)
[![OpenShift](https://img.shields.io/badge/OpenShift-4.10+-red.svg)](https://www.openshift.com/)
[![AAP](https://img.shields.io/badge/AAP-2.5-orange.svg)](https://www.redhat.com/en/technologies/management/ansible)

Red Hat Ansible Automation Platform (AAP) deployment using GitOps methodology with Red Hat Advanced Cluster Management (ACM) and ArgoCD.

Following the [ACM policies repository pattern](https://github.com/hultzj/acm-policys/tree/main), this repository provides a comprehensive GitOps approach for AAP deployment and management.

## 🎯 Quick Start

```bash
# 1. Deploy Day-1 (Infrastructure)
oc apply -f gitops-apps-day-1.yaml

# 2. Wait for operator installation
oc wait --for=condition=Succeeded csv -l operators.coreos.com/aap-operator.ansible-automation-platform -n ansible-automation-platform --timeout=600s

# 3. Deploy Day-2 (Applications)
oc apply -f gitops-apps-day-2.yaml

# 4. Get access information
oc get configmap aap-controller-access-info -n ansible-automation-platform -o jsonpath='{.data.instructions}'
```

## 📋 Table of Contents

- [Overview](#overview)
- [Prerequisites](#prerequisites)
- [Installation](#installation)
- [Configuration](#configuration)
- [Optional Components](#optional-components)
- [Monitoring & Troubleshooting](#monitoring--troubleshooting)
- [Examples](#examples)
- [Management Operations](#management-operations)
- [ACM Integration](#acm-integration)
- [Support](#support)

## Overview

This GitOps repository automates the installation and configuration of Red Hat Ansible Automation Platform using a day-1/day-2 operational approach:

- **Day-1 Operations**: RBAC setup and AAP operator installation
- **Day-2 Operations**: AAP instance configuration and optional components

### 🧩 Available Components

| Component | Status | Purpose |
|-----------|--------|---------|
| **AutomationController** | 🔴 **Required** | Core AAP controller for job execution and workflow management |
| **Private Automation Hub** | 🟡 **Optional** | Content management for collections and execution environments |
| **Event Driven Controller** | 🟡 **Optional** | Reactive automation based on events and triggers |
| **Service Mesh Integration** | 🟡 **Optional** | Istio/OpenShift Service Mesh routing configuration |
| **External PostgreSQL** | 🟡 **Optional** | Enterprise external database connectivity |

### 🏗️ Repository Structure

```
aap-setup/
├── 📄 gitops-apps-day-1.yaml        # Day-1: RBAC + Operator installation
├── 📄 gitops-apps-day-2.yaml        # Day-2: Instance + Optional components
└── 📁 apps/
    ├── 🔐 aap-rbac/                 # Namespace and RBAC configuration
    ├── ⚙️  aap-operator/             # AAP operator installation
    ├── 🎮 aap-instance/             # AAP controller and optional components
    └── 🌐 aap-servicemesh/          # Service mesh integration
```

## Prerequisites

### Required

- **OpenShift 4.10+** cluster
- **Red Hat GitOps Operator** installed
- **GitOps ServiceAccount** with cluster-admin privileges
- **Red Hat Operators catalog** available

### Critical Setup Steps

Following the [ACM pattern](https://github.com/hultzj/acm-policys/tree/main):

1. **Install Red Hat GitOps operator**
   ```bash
   oc apply -f https://raw.githubusercontent.com/redhat-developer/gitops-operator/master/deploy/olm-catalog/gitops-operator/manifests/gitops-operator.v1.9.0.clusterserviceversion.yaml
   ```

2. **Grant GitOps ServiceAccount cluster-admin permissions**
   ```bash
   oc adm policy add-cluster-role-to-user cluster-admin system:serviceaccount:openshift-gitops:openshift-gitops-argocd-application-controller
   ```

3. **For ACM environments** - Add cluster labels:
   ```bash
   oc label cluster <cluster-name> infra-nodes=3 storage-nodes=3
   ```

### Optional Prerequisites

- **OpenShift Service Mesh Operator** (for OSSM integration)
- **Istio** (for vanilla Istio integration)
- **External PostgreSQL** (for external database connectivity)

## Installation

### Step 1: Deploy Day-1 Applications

Deploy RBAC and operator infrastructure:

```bash
oc apply -f gitops-apps-day-1.yaml
```

**What this deploys:**
- ✅ Namespace with monitoring labels
- ✅ ServiceAccount with cluster permissions
- ✅ AAP operator subscription

### Step 2: Verify Operator Installation

```bash
# Monitor operator installation
oc get csv -n ansible-automation-platform -w

# Wait for successful installation
oc wait --for=condition=Succeeded csv -l operators.coreos.com/aap-operator.ansible-automation-platform -n ansible-automation-platform --timeout=600s

# Verify operator is ready
oc get subscription aap-operator -n ansible-automation-platform
```

### Step 3: Deploy Day-2 Applications

Deploy AAP instances and optional components:

```bash
oc apply -f gitops-apps-day-2.yaml
```

**What this deploys:**
- ✅ AAP Controller (required)
- ⚙️ Private Automation Hub (if enabled)
- ⚙️ Event Driven Controller (if enabled)
- 🌐 Service Mesh routing (if enabled)

### Step 4: Access Your AAP Environment

```bash
# Get comprehensive access information
oc get configmap aap-controller-access-info -n ansible-automation-platform -o jsonpath='{.data.instructions}' | echo -e "$(cat)"

# Get route URLs
oc get route -n ansible-automation-platform

# Get admin passwords
echo "Controller Password:"
oc get secret aap-controller-admin-password -n ansible-automation-platform -o jsonpath='{.data.password}' | base64 -d && echo
```

## Configuration

### Basic Configuration

Edit ArgoCD Application parameters in `gitops-apps-day-2.yaml`:

```yaml
spec:
  source:
    helm:
      parameters:
        - name: aapOperator.version
          value: "2.5"                    # AAP operator version
        - name: aapInstance.admin.username  
          value: "admin"                  # Controller admin username
        - name: aapInstance.admin.email
          value: "admin@mycompany.com"    # Admin email
```

### Repository URL Configuration

**🚨 Important:** Update the repository URL in both GitOps application files:

```yaml
spec:
  source:
    repoURL: https://github.com/your-org/your-aap-setup-repo.git
```

### Advanced Configuration

For complex scenarios, modify values in `apps/aap-instance/values.yaml`:

```yaml
aapInstance:
  resources:
    requests:
      cpu: "2000m"
      memory: "4Gi"
    limits:
      cpu: "4000m" 
      memory: "8Gi"
  storage:
    size: "50Gi"
    storageClass: "gp3-csi"
```

## Optional Components

### 1. 🏪 Private Automation Hub

**Purpose**: Content management for Ansible collections and execution environments

**Enable via GitOps:**
```bash
oc patch application aap-instance-day2 -n openshift-gitops --type='merge' -p='{"spec":{"source":{"helm":{"parameters":[{"name":"automationHub.enabled","value":"true"}]}}}}'
```

**Configure in `gitops-apps-day-2.yaml`:**
```yaml
parameters:
  - name: automationHub.enabled
    value: "true"
  - name: automationHub.admin.username
    value: "hub-admin"
  - name: automationHub.admin.email
    value: "hub-admin@mycompany.com"
```

### 2. ⚡ Event Driven Ansible Controller

**Purpose**: Reactive automation based on events and triggers

**Enable via GitOps:**
```bash
oc patch application aap-instance-day2 -n openshift-gitops --type='merge' -p='{"spec":{"source":{"helm":{"parameters":[{"name":"edaController.enabled","value":"true"}]}}}}'
```

**Configure in `gitops-apps-day-2.yaml`:**
```yaml
parameters:
  - name: edaController.enabled
    value: "true"
  - name: edaController.admin.username
    value: "eda-admin"
  - name: edaController.admin.email
    value: "eda-admin@mycompany.com"
```

### 3. 🗄️ External PostgreSQL

**Purpose**: Use enterprise external PostgreSQL instead of internal databases

**Configure for Controller:**
```yaml
parameters:
  - name: aapInstance.postgres.external.enabled
    value: "true"
  - name: aapInstance.postgres.external.host
    value: "postgres.mycompany.com"
  - name: aapInstance.postgres.external.database
    value: "awx"
  - name: aapInstance.postgres.external.username
    value: "awx"
  - name: aapInstance.postgres.external.password
    value: "secure-password"
```

**Configure for Hub and EDA:**
```yaml
parameters:
  # Private Automation Hub external DB
  - name: automationHub.postgres.external.enabled
    value: "true"
  - name: automationHub.postgres.external.host
    value: "postgres.mycompany.com"
  - name: automationHub.postgres.external.database
    value: "pulp"
  
  # Event Driven Controller external DB
  - name: edaController.postgres.external.enabled
    value: "true"
  - name: edaController.postgres.external.host
    value: "postgres.mycompany.com"
  - name: edaController.postgres.external.database
    value: "eda"
```

### 4. 🌐 Service Mesh Integration

**Purpose**: Integrate with Istio or OpenShift Service Mesh

**Configure in `apps/aap-servicemesh/values.yaml`:**
```yaml
serviceMesh:
  enabled: true
  istio:
    enabled: true        # For vanilla Istio
    virtualService:
      hosts:
        - "aap.mycompany.com"
  ossm:
    enabled: false       # For OpenShift Service Mesh
```

## Examples

### Example 1: Minimal AAP (Controller Only)

```bash
# Basic installation - just controller
oc apply -f gitops-apps-day-1.yaml
oc apply -f gitops-apps-day-2.yaml
```

### Example 2: Full AAP Stack

```bash
# Enable all components
oc patch application aap-instance-day2 -n openshift-gitops --type='merge' -p='{"spec":{"source":{"helm":{"parameters":[{"name":"automationHub.enabled","value":"true"},{"name":"edaController.enabled","value":"true"}]}}}}'
```

### Example 3: Enterprise Setup with External Database

```bash
# Configure external postgres for all components
oc patch application aap-instance-day2 -n openshift-gitops --type='merge' -p='{"spec":{"source":{"helm":{"parameters":[{"name":"aapInstance.postgres.external.enabled","value":"true"},{"name":"aapInstance.postgres.external.host","value":"postgres.mycompany.com"},{"name":"automationHub.enabled","value":"true"},{"name":"automationHub.postgres.external.enabled","value":"true"},{"name":"edaController.enabled","value":"true"},{"name":"edaController.postgres.external.enabled","value":"true"}]}}}}'
```

### Example 4: Service Mesh Integration

```bash
# Enable OpenShift Service Mesh
oc patch application aap-servicemesh-day2 -n openshift-gitops --type='merge' -p='{"spec":{"source":{"helm":{"parameters":[{"name":"serviceMesh.enabled","value":"true"},{"name":"serviceMesh.ossm.enabled","value":"true"}]}}}}'
```

## Monitoring & Troubleshooting

### 📊 Monitoring

```bash
# Check GitOps applications
oc get applications -n openshift-gitops

# Monitor all AAP components
oc get automationcontroller,automationhub,edacontroller -n ansible-automation-platform

# Check component status
oc get configmap aap-controller-access-info -n ansible-automation-platform -o jsonpath='{.data.components-enabled}'

# Watch deployment progress
oc get pods -n ansible-automation-platform -w
```

### 🔧 Troubleshooting

#### GitOps Application Issues
```bash
# Force application sync
argocd app sync aap-operator-day1
argocd app sync aap-instance-day2

# Check application status
oc describe application aap-operator-day1 -n openshift-gitops
```

#### Operator Installation Issues
```bash
# Check operator status
oc get subscription aap-operator -n ansible-automation-platform
oc get csv -n ansible-automation-platform
oc get installplan -n ansible-automation-platform
```

#### Component Issues
```bash
# Check AAP Controller
oc describe automationcontroller aap-controller -n ansible-automation-platform

# Check Private Automation Hub (if enabled)
oc describe automationhub automation-hub -n ansible-automation-platform

# Check Event Driven Controller (if enabled)
oc describe edacontroller eda-controller -n ansible-automation-platform

# Check external postgres connectivity
oc get secrets -n ansible-automation-platform | grep postgres-configuration
```

## Management Operations

### Upgrade AAP Version

```bash
oc patch application aap-instance-day2 -n openshift-gitops --type='merge' -p='{"spec":{"source":{"helm":{"parameters":[{"name":"aapOperator.version","value":"2.6"}]}}}}'
```

### Enable/Disable Components

```bash
# Enable Private Automation Hub
oc patch application aap-instance-day2 -n openshift-gitops --type='merge' -p='{"spec":{"source":{"helm":{"parameters":[{"name":"automationHub.enabled","value":"true"}]}}}}'

# Enable Event Driven Controller
oc patch application aap-instance-day2 -n openshift-gitops --type='merge' -p='{"spec":{"source":{"helm":{"parameters":[{"name":"edaController.enabled","value":"true"}]}}}}'

# Disable Service Mesh
oc patch application aap-servicemesh-day2 -n openshift-gitops --type='merge' -p='{"spec":{"source":{"helm":{"parameters":[{"name":"serviceMesh.enabled","value":"false"}]}}}}'
```

### Repository Updates

```bash
# Refresh applications to pick up repository changes
argocd app refresh aap-operator-day1
argocd app refresh aap-instance-day2
argocd app refresh aap-servicemesh-day2
```

## ACM Integration

This repository is designed for [Red Hat Advanced Cluster Management](https://github.com/hultzj/acm-policys/tree/main) integration:

### Multi-Cluster Deployment

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: aap-setup-clusters
  namespace: openshift-gitops
spec:
  generators:
  - clusters:
      selector:
        matchLabels:
          infra-nodes: "3"
          storage-nodes: "3"
  template:
    metadata:
      name: '{{name}}-aap-setup'
    spec:
      project: default
      source:
        repoURL: https://github.com/your-org/aap-setup.git
        targetRevision: main
        path: .
      destination:
        server: '{{server}}'
        namespace: ansible-automation-platform
```

### Cluster Labels

Apply these labels for targeted AAP deployment:

```bash
# Label clusters for AAP deployment
oc label cluster prod-cluster-1 infra-nodes=3 storage-nodes=3 aap-enabled=true
oc label cluster dev-cluster-1 infra-nodes=1 storage-nodes=1 aap-enabled=true
```

## Verification

### Component Status Check

```bash
# Check enabled components
oc get configmap aap-controller-access-info -n ansible-automation-platform -o jsonpath='{.data.components-enabled}'

# Verify all routes
oc get route -n ansible-automation-platform

# Check external postgres usage
oc get configmap aap-controller-access-info -n ansible-automation-platform -o jsonpath='{.data.external-postgres}'
```

### Access All Components

```bash
# Get comprehensive access information
oc get configmap aap-controller-access-info -n ansible-automation-platform -o jsonpath='{.data.instructions}' | echo -e "$(cat)"

# Get all passwords
echo "=== Controller ===" && oc get secret aap-controller-admin-password -n ansible-automation-platform -o jsonpath='{.data.password}' | base64 -d && echo
echo "=== Hub ===" && oc get secret automation-hub-admin-password -n ansible-automation-platform -o jsonpath='{.data.password}' | base64 -d 2>/dev/null && echo
echo "=== EDA ===" && oc get secret eda-controller-admin-password -n ansible-automation-platform -o jsonpath='{.data.password}' | base64 -d 2>/dev/null && echo
```

## Uninstallation

```bash
# Remove day-2 applications
oc delete -f gitops-apps-day-2.yaml

# Wait for cleanup
sleep 30

# Remove day-1 applications
oc delete -f gitops-apps-day-1.yaml

# Manual cleanup if needed
oc delete namespace ansible-automation-platform
```

## Support

### 📚 Documentation
- [Red Hat AAP Documentation](https://access.redhat.com/documentation/en-us/red_hat_ansible_automation_platform)
- [Red Hat GitOps Documentation](https://docs.openshift.com/container-platform/latest/cicd/gitops/understanding-openshift-gitops.html)
- [Private Automation Hub](https://access.redhat.com/documentation/en-us/red_hat_ansible_automation_platform/2.5/html/managing_content_in_automation_hub/index)
- [Event Driven Ansible](https://access.redhat.com/documentation/en-us/red_hat_ansible_automation_platform/2.5/html/getting_started_with_event-driven_ansible/index)

### 🤝 Community
- Create an issue in this repository
- [ArgoCD Documentation](https://argo-cd.readthedocs.io/)
- [OpenShift Service Mesh](https://docs.openshift.com/container-platform/latest/service_mesh/v2x/installing-ossm.html)

### 🔗 References
- [ACM Policies Repository Pattern](https://github.com/hultzj/acm-policys/tree/main)
- [Red Hat GitOps Operator](https://docs.openshift.com/container-platform/latest/cicd/gitops/installing-openshift-gitops.html)
- [Red Hat AAP Operator Guide](https://access.redhat.com/documentation/en-us/red_hat_ansible_automation_platform/2.4/html/red_hat_ansible_automation_platform_operator_installation_guide/index)

## Security Considerations

- **ServiceAccount Permissions**: Uses cluster-admin for comprehensive AAP management
- **Password Security**: Auto-generated passwords stored in Kubernetes secrets
- **Network Policies**: Consider implementing for production environments
- **External Database**: Use TLS connections for external PostgreSQL

## License

This project is licensed under the Apache License 2.0 - see the [LICENSE](LICENSE) file for details.

---

**🚀 Ready to automate? Start with the [Quick Start](#-quick-start) guide!**