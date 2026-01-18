# GitOps Hub

GitOps repository for the OpenShift Platform Engineering hub cluster.

## Components

This repository manages the following components on the hub cluster:

| Component                   | Description                                 |
| --------------------------- | ------------------------------------------- |
| OpenShift GitOps            | ArgoCD-based GitOps operator                |
| External Secrets Operator   | Sync secrets from AWS Secrets Manager       |
| Advanced Cluster Management | Multi-cluster management and governance     |
| Advanced Cluster Security   | Kubernetes-native security platform         |
| OpenShift Pipelines         | Tekton-based CI/CD pipelines                |
| Red Hat Developer Hub       | Internal developer portal (Backstage)       |

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
│   └── external-secrets/           # ClusterSecretStore for AWS
└── components/                     # Reusable Kustomize components
```

## Prerequisites

### AWS Secrets Manager Setup

Create the secret for Developer Hub in AWS Secrets Manager:

```shell
# Generate secrets
BACKEND_SECRET=$(openssl rand -base64 32)
OAUTH_SECRET=$(openssl rand -base64 32)

# Create secret in AWS Secrets Manager
aws secretsmanager create-secret \
  --name ocppe/developer-hub \
  --secret-string "{\"backend-secret\":\"${BACKEND_SECRET}\",\"oauth-client-secret\":\"${OAUTH_SECRET}\"}"

# Save the OAuth secret for the OAuthClient configuration
echo "OAuth Client Secret: ${OAUTH_SECRET}"
```

### Configure Developer Hub

Update `operators/developer-hub-instance/kustomization.yaml` with your environment values:

```yaml
patches:
  - patch: |-
      - op: replace
        path: /redirectURIs/0
        value: https://backstage-developer-hub-rhdh.apps.hub.ocp.sandbox${SANDBOX_ID}.opentlc.com/api/auth/oidc/handler/frame
      - op: replace
        path: /secret
        value: <OAUTH_SECRET from above>
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

### 3. Update Git Repository URL

Update the `repoURL` in all files under `argocd-apps/` to match your Git repository:

```yaml
spec:
  source:
    repoURL: https://github.com/YOUR-ORG/gitops-hub.git
```

### 4. Deploy Root Application (App-of-Apps)

```shell
oc apply -f argocd-apps/root-application.yaml
```

This will automatically deploy all operators and configurations.

### 5. Access ArgoCD Console

```shell
# Get the ArgoCD route
oc get route openshift-gitops-server -n openshift-gitops -o jsonpath='{.spec.host}'

# Get the admin password
oc extract secret/openshift-gitops-cluster -n openshift-gitops --to=-
```

## External Secrets IAM Configuration

The External Secrets Operator needs IAM permissions to access AWS Secrets Manager. Configure using IRSA (IAM Roles for Service Accounts):

### 1. Create IAM Policy

```shell
cat > /tmp/external-secrets-policy.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "secretsmanager:GetSecretValue",
        "secretsmanager:DescribeSecret"
      ],
      "Resource": "arn:aws:secretsmanager:*:*:secret:ocppe/*"
    }
  ]
}
EOF

aws iam create-policy \
  --policy-name ExternalSecretsPolicy \
  --policy-document file:///tmp/external-secrets-policy.json
```

### 2. Create IAM Role with OIDC Trust

```shell
# Get OIDC provider URL from your cluster
OIDC_PROVIDER=$(oc get authentication cluster -o jsonpath='{.spec.serviceAccountIssuer}' | sed 's|https://||')
AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)

cat > /tmp/trust-policy.json << EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::${AWS_ACCOUNT_ID}:oidc-provider/${OIDC_PROVIDER}"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "${OIDC_PROVIDER}:sub": "system:serviceaccount:external-secrets-operator:external-secrets-sa"
        }
      }
    }
  ]
}
EOF

aws iam create-role \
  --role-name ExternalSecretsRole \
  --assume-role-policy-document file:///tmp/trust-policy.json

aws iam attach-role-policy \
  --role-name ExternalSecretsRole \
  --policy-arn arn:aws:iam::${AWS_ACCOUNT_ID}:policy/ExternalSecretsPolicy
```

### 3. Annotate Service Account

```shell
AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)

oc annotate serviceaccount external-secrets-sa \
  -n external-secrets-operator \
  eks.amazonaws.com/role-arn=arn:aws:iam::${AWS_ACCOUNT_ID}:role/ExternalSecretsRole
```

## Manual Deployment (Without App-of-Apps)

If you prefer to deploy components individually:

```shell
# Deploy External Secrets Operator
oc apply -k operators/external-secrets-operator/

# Deploy ClusterSecretStore (after operator is ready)
oc apply -k cluster-config/external-secrets/

# Deploy ACM
oc apply -k operators/acm-operator/
# Wait for CRD, then:
oc apply -k operators/acm-hub/

# Deploy ACS
oc apply -k operators/acs-operator/
# Wait for CRD, then:
oc apply -k operators/acs-central/

# Deploy OpenShift Pipelines
oc apply -k operators/openshift-pipelines/

# Deploy Developer Hub
oc apply -k operators/developer-hub-operator/
# Wait for CRD, then:
oc apply -k operators/developer-hub-instance/
```
