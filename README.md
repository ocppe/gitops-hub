# GitOps Hub

GitOps repository for the OpenShift Platform Engineering hub cluster.

## Components

| Component                   | Description                                 |
| --------------------------- | ------------------------------------------- |
| OpenShift GitOps            | ArgoCD-based GitOps operator                |
| External Secrets Operator   | Sync secrets from AWS Secrets Manager       |
| Advanced Cluster Management | Multi-cluster management and governance     |
| Advanced Cluster Security   | Kubernetes-native security platform         |
| OpenShift Pipelines         | Tekton-based CI/CD pipelines                |
| Red Hat Developer Hub       | Internal developer portal (Backstage)       |
| CI Pipelines                | Centralized build pipelines for apps        |

## Prerequisites

All infrastructure setup is automated via Ansible playbooks. See the [ansible README](../ansible/README.md) for complete instructions.

**Required setup before deploying:**
1. Quay.io organization and robot account: `ansible-playbook playbooks/quay-setup.yml`
2. AWS Secrets Manager secrets: `ansible-playbook playbooks/aws-secrets-setup.yml`
3. GitHub webhooks (after deployment): `ansible-playbook playbooks/github-webhooks-setup.yml`

## Directory Structure

```
gitops-hub/
├── bootstrap/                      # Bootstrap OpenShift GitOps operator
├── argocd-apps/                    # ArgoCD Application definitions (app-of-apps)
├── operators/
│   ├── external-secrets-operator/  # External Secrets Operator
│   ├── acm-operator/               # ACM operator subscription
│   ├── acm-hub/                    # MultiClusterHub instance
│   ├── acs-operator/               # ACS operator subscription
│   ├── acs-central/                # Central instance
│   ├── openshift-pipelines/        # OpenShift Pipelines
│   ├── developer-hub-operator/     # RHDH operator subscription
│   └── developer-hub-instance/     # Backstage instance with OIDC
├── cluster-config/
│   ├── external-secrets/           # ClusterSecretStore for AWS
│   └── pipelines/                  # CI pipeline configuration
└── components/                     # Reusable Kustomize components
```

## Bootstrap Instructions

### 1. Install OpenShift GitOps Operator

```shell
oc apply -k bootstrap/
```

Wait for the operator:

```shell
oc wait --for=jsonpath='{.status.phase}'=Succeeded csv \
  -n openshift-gitops-operator \
  -l operators.coreos.com/openshift-gitops-operator.openshift-gitops-operator \
  --timeout=300s

oc wait --for=condition=Available deployment/openshift-gitops-server \
  -n openshift-gitops --timeout=300s
```

### 2. Grant ArgoCD Cluster Admin

```shell
oc adm policy add-cluster-role-to-user cluster-admin \
  system:serviceaccount:openshift-gitops:openshift-gitops-argocd-application-controller
```

### 3. Update Git Repository URL

Update the `repoURL` in all files under `argocd-apps/` to match your Git repository:

```yaml
spec:
  source:
    repoURL: https://github.com/YOUR-ORG/gitops-hub.git
```

### 4. Configure Developer Hub OAuth

Update `operators/developer-hub-instance/kustomization.yaml` with your OAuth secret from the Ansible setup:

```shell
# Get the OAuth secret from ansible/.secrets.env
source ../ansible/.secrets.env
echo $OAUTH_CLIENT_SECRET
```

### 5. Deploy Root Application

```shell
oc apply -f argocd-apps/root-application.yaml
```

### 6. Setup GitHub Webhooks

After pipelines are deployed:

```shell
cd ../ansible
ansible-playbook playbooks/github-webhooks-setup.yml
```

## Access Points

```shell
# ArgoCD
oc get route openshift-gitops-server -n openshift-gitops -o jsonpath='{.spec.host}'
oc extract secret/openshift-gitops-cluster -n openshift-gitops --to=-

# ACM Console
oc get route multicloud-console -n open-cluster-management -o jsonpath='{.spec.host}'

# ACS Central
oc get route central -n stackrox -o jsonpath='{.spec.host}'

# Developer Hub
oc get route backstage-developer-hub -n rhdh -o jsonpath='{.spec.host}'

# CI Webhook
oc get route github-webhook -n ci-pipelines -o jsonpath='{.spec.host}'
```

## Secrets Reference

All secrets are stored in AWS Secrets Manager under the `ocppe/` prefix and synced via External Secrets Operator:

| AWS Secret Path       | Kubernetes Secret          | Namespace       |
| --------------------- | -------------------------- | --------------- |
| ocppe/developer-hub   | developer-hub-secrets      | rhdh            |
| ocppe/ci-pipelines    | github-webhook-secret      | ci-pipelines    |
| ocppe/ci-pipelines    | quay-credentials           | ci-pipelines    |
| ocppe/ci-pipelines    | git-credentials            | ci-pipelines    |

## Manual Deployment

If you prefer to deploy components individually:

```shell
# External Secrets Operator
oc apply -k operators/external-secrets-operator/
oc apply -k cluster-config/external-secrets/

# ACM
oc apply -k operators/acm-operator/
# Wait for CRD, then:
oc apply -k operators/acm-hub/

# ACS
oc apply -k operators/acs-operator/
# Wait for CRD, then:
oc apply -k operators/acs-central/

# OpenShift Pipelines
oc apply -k operators/openshift-pipelines/

# Developer Hub
oc apply -k operators/developer-hub-operator/
# Wait for CRD, then:
oc apply -k operators/developer-hub-instance/

# CI Pipelines
oc apply -k cluster-config/pipelines/
```
