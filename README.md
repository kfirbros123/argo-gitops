# Argo GitOps

A complete GitOps infrastructure using ArgoCD for continuous deployment to Kubernetes. This repository demonstrates multi-environment deployments using ApplicationSets, AppProjects, and automated sync policies.

## Overview

This GitOps repository manages application deployments across multiple environments (dev, prod) using ArgoCD. It leverages:

- **ApplicationSets** for templated multi-cluster deployments
- **AppProjects** for access control and repository scoping
- **Helm Charts** for application templating
- **Automated Sync** with self-healing and pruning

## Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                         ArgoCD                                       │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                    ApplicationSet                            │   │
│  │  ┌──────────────────┐              ┌──────────────────┐     │   │
│  │  │ exam-app-dev     │              │ exam-app-prod    │     │   │
│  │  │ (dev cluster)    │              │ (prod cluster)   │     │   │
│  │  └──────────────────┘              └──────────────────┘     │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                              │                                       │
│                              ▼                                       │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                      AppProject                              │   │
│  │  - Controls which repos can be used                         │   │
│  │  - Defines allowed destination namespaces                   │   │
│  │  - Manages RBAC and permissions                             │   │
│  └─────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────────┐
│                      Kubernetes Cluster                              │
│  ┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐ │
│  │ exam-app-dev    │    │ exam-app-prod   │    │ default         │ │
│  │ namespace       │    │ namespace       │    │ namespace       │ │
│  └─────────────────┘    └─────────────────┘    └─────────────────┘ │
└─────────────────────────────────────────────────────────────────────┘
```

## Repository Structure

```
argo-gitops/
├── README.md                    # This file
├── clusters.yml                 # ApplicationSet for multi-environment deployment
└── config/
    └── test-project.yml         # AppProject configuration
```

## Components

### 1. ApplicationSet (`clusters.yml`)

The ApplicationSet generates ArgoCD Applications for multiple environments automatically:


**Key Features:**
- **List Generator**: Creates applications for dev and prod environments
- **Templating**: Uses `{{project}}` and `{{cluster}}` variables
- **Auto-Sync**: Automatically syncs when Git changes are detected
- **Self-Healing**: Reverts manual changes in the cluster
- **Pruning**: Removes resources no longer in Git
- **Namespace Creation**: Automatically creates target namespaces


### 2. AppProject (`config/test-project.yml`)

Defines access control and resource scoping:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: my-project
  namespace: argocd
spec:
  description: Allow access to a specific Git repository and branch
  sourceRepos:
    - https://github.com/kfirbros123/argo-gitops.git
    - https://github.com/kfirbros123/home-app.git
    - https://github.com/kfirbros123/test-app.git
  destinations:
    - namespace: '*'
      server: https://kubernetes.default.svc
  sourceNamespaces:
    - '*'
```

**Key Features:**
- **Repository Whitelisting**: Only allows specific GitHub repositories
- **Destination Control**: Defines allowed Kubernetes namespaces
- **Namespace Access**: Controls which namespaces can contain applications

## Installation

### Prerequisites

1. **Kubernetes Cluster**: v1.16+ with kubectl configured
2. **ArgoCD**: Installed in the cluster
3. **Git Repository**: Access to the source repositories

### Step 1: Install ArgoCD

```bash
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update 


kubectl create namespace argocd

helm upgrade --install argocd argo/argo-cd  --namespace argocd 

#with amazon you can use this patch to get a loadbalancer for argocd
kubectl patch svc  -n argocd argocd-server -p '{"spec": {"type": "LoadBalancer"}}' 

#with google cloudshell
kubectl patch configmap argocd-cmd-params-cm   -n argocd   --type merge   -p '{"data":{"server.insecure":"true"}}'
kubectl -n argocd rollout restart deployment argocd-server

#get admin password
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d ; echo


```

### Step 2: Apply AppProject

```bash
# Apply the project configuration
kubectl apply -f config/test-project.yml
```

### Step 3: Apply ApplicationSet

```bash
# Apply the ApplicationSet
kubectl apply -f clusters.yml

# Verify applications are created
kubectl get applications -n argocd
```

### Step 4: Access ArgoCD

```bash
# Get initial admin password
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d

# Port-forward ArgoCD UI
kubectl port-forward svc/argocd-server -n argocd 8080:80

# Access at cloudshell web preview
# Username: admin
# Password: (from previous command)
```

## Usage

### Adding New Environments

To add a new environment (e.g., staging), update `clusters.yml`:

```yaml
generators:
  - list:
      elements:
        - cluster: dev
          project: exam-app
        - cluster: staging    # New environment
          project: exam-app
        - cluster: prod
          project: exam-app
```

### Adding New Applications

Create a new Application or add to the ApplicationSet:

```yaml
# For ApplicationSet, add new element
- cluster: new-env
  project: new-app
```

### Manual Sync

```bash
# Sync a specific application
argocd app sync exam-app-dev -n argocd

# Or via kubectl
kubectl apply -f clusters.yml -n argocd
```

### Rollback

```bash
# Rollback to previous revision
argocd app rollback exam-app-dev -n argocd

# View revision history
argocd app history exam-app-dev -n argocd
```

## CI/CD Integration

### Github Actions Workflow   
using the app repo workflow a new helm template manifest gets commited into the application branch of this repo 

### Logs and Debugging

```bash
# View ArgoCD logs
kubectl logs -n argocd -l app.kubernetes.io/name=argocd-application-controller

# View application events
kubectl get events -n argocd --sort-by='.lastTimestamp'

# Check resource diff
argocd app diff exam-app-dev -n argocd
```

## Best Practices

### 1. Branch Strategy

- **Main Branch**: Production-ready configurations
- **Application Branch**: Holds application manifest yamls
- **Terraform Branches**: will create argo cluster on aws using terraform

### 2. Security

- Use separate AppProjects for different teams/environments
- Restrict repository access using sourceRepos
- Implement RBAC with ArgoCD roles
- Use sealed secrets or external secret management


## Troubleshooting

### Common Issues

1. **Application Not Syncing**
   ```bash
   # Check application status
   argocd app get exam-app-dev -n argocd
   
   # View logs
   kubectl logs -n argocd -l app.kubernetes.io/name=argocd-repo-server
   ```

2. **Git Authentication Failures**
   - Ensure ArgoCD has access to the repository
   - Add repository credentials: `argocd repo add <url> --username <user> --password <pass>`

3. **Namespace Not Created**
   - Verify `CreateNamespace=true` sync option is set
   - Check RBAC permissions

4. **Resource Conflicts**
   ```bash
   # Check for existing resources
   kubectl get all -n exam-app-dev
   
   # Force refresh
   argocd app refresh exam-app-dev -n argocd
   ```

### Debug Commands

```bash
# List all ArgoCD resources
kubectl get crds | grep argoproj.io

# Check ApplicationSet status
kubectl get applicationset -n argocd

# View ArgoCD configuration
kubectl get cm argocd-cm -n argocd -o yaml

# Test repository access
argocd repo get https://github.com/kfirbros123/argo-gitops.git
```

## Additional Resources

- [ArgoCD Documentation](https://argo-cd.readthedocs.io/)
- [ApplicationSet Documentation](https://argo-cd.readthedocs.io/en/stable/operator-manual/applicationset/)
- [AppProject Documentation](https://argo-cd.readthedocs.io/en/stable/operator-manual/projects/)
- [GitOps Best Practices](https://argo-cd.readthedocs.io/en/stable/operator-manual/declarative-setup/)

## Related Repositories

- **Application Repository**: https://github.com/kfirbros123/test-app.git
- **Main GitOps Repository**: https://github.com/kfirbros123/argo-gitops.git
- **Home App**: https://github.com/kfirbros123/home-app.git
