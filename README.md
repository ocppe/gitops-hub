# GitOps Hub

GitOps repository for the OpenShift Platform Engineering hub cluster.

## Components

This repository manages the following components on the hub cluster:

| Component                        | Description                                      |
| -------------------------------- | ------------------------------------------------ |
| OpenShift GitOps                 | ArgoCD-based GitOps operator                     |
| Advanced Cluster Management      | Multi-cluster management and governance          |
| Advanced Cluster Security        | Kubernetes-native security platform              |
| OpenShift Pipelines              | Tekton-based CI/CD pipelines                     |
| Red Hat Developer Hub            | Internal developer portal (Backstage)            |

## Directory Structure

```
gitops-hub/
├── bootstrap/                 # Bootstrap OpenShift GitOps operator
├── argocd-apps/               # ArgoCD Application definitions (app-of-apps)
├── operators/                 # Operator installations
│   ├── acm/                   # Advanced Cluster Management
│   ├── acs/                   # Advanced Cluster Security
│   ├── openshift-pipelines/   # OpenShift Pipelines
│   └── developer-hub/         # Red Hat Developer Hub
├── cluster-config/            # Cluster configuration
└── components/                # Reusable Kustomize components
```

## Bootstrap Instructions

### 1. Install OpenShift GitOps Operator

```shell
oc apply -k bootstrap/
```

Wait for the operator CSV to succeed:

```shell
oc wait --for=jsonpath='{.status.phase}'=Succeeded csv \
  -n openshift-gitops-operator \
  -l operators.coreos.com/openshift-gitops-operator.openshift-gitops-operator \
  --timeout=300s
```

Wait for ArgoCD to be ready:

```shell
oc wait --for=condition=Available deployment/openshift-gitops-server \
  -n openshift-gitops --timeout=300s
```

### 2. Grant ArgoCD Cluster Admin

```shell
oc adm policy add-cluster-role-to-user cluster-admin \
  system:serviceaccount:openshift-gitops:openshift-gitops-argocd-application-controller
```

### 3. Deploy Root Application (App-of-Apps)

Before applying, update the `repoURL` in the ArgoCD Application manifests to point to your Git repository.

```shell
oc apply -f argocd-apps/root-application.yaml
```

This will automatically deploy all operators and configurations.

### 4. Access ArgoCD Console

```shell
# Get the ArgoCD route
oc get route openshift-gitops-server -n openshift-gitops -o jsonpath='{.spec.host}'

# Get the admin password
oc extract secret/openshift-gitops-cluster -n openshift-gitops --to=-
```

## Manual Deployment (Without App-of-Apps)

If you prefer to deploy components individually:

```shell
# Deploy ACM
oc apply -k operators/acm/

# Deploy ACS
oc apply -k operators/acs/

# Deploy OpenShift Pipelines
oc apply -k operators/openshift-pipelines/

# Deploy Developer Hub
oc apply -k operators/developer-hub/
```

## Customization

### Updating Git Repository URL

Update the `repoURL` in all files under `argocd-apps/` to match your Git repository:

```yaml
spec:
  source:
    repoURL: https://github.com/YOUR-ORG/gitops-hub.git
```
