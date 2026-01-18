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
| CI Pipelines                | Centralized build pipelines for apps        |

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
│       ├── base/                   # Namespace, RBAC, ExternalSecrets
│       ├── tasks/                  # Tekton tasks
│       ├── pipelines/              # Tekton pipelines
│       └── triggers/               # GitHub webhook triggers
└── components/                     # Reusable Kustomize components
```

## Prerequisites

### AWS Secrets Manager Setup

All secrets are managed via External Secrets Operator syncing from AWS Secrets Manager. Create the following secrets before deploying:

#### 1. Developer Hub Secrets

```shell
# Generate secrets
BACKEND_SECRET=$(openssl rand -base64 32)
OAUTH_SECRET=$(openssl rand -base64 32)

# Create secret in AWS Secrets Manager
aws secretsmanager create-secret \
  --name ocppe/developer-hub \
  --secret-string "{
    \"backend-secret\": \"${BACKEND_SECRET}\",
    \"oauth-client-secret\": \"${OAUTH_SECRET}\"
  }"

# Save the OAuth secret for OAuthClient configuration
echo "OAuth Client Secret: ${OAUTH_SECRET}"
```

#### 2. CI Pipeline Secrets

```shell
# Generate webhook secret
WEBHOOK_SECRET=$(openssl rand -hex 20)

# Create secret in AWS Secrets Manager
aws secretsmanager create-secret \
  --name ocppe/ci-pipelines \
  --secret-string "{
    \"github-webhook-secret\": \"${WEBHOOK_SECRET}\",
    \"quay-username\": \"YOUR_QUAY_USERNAME\",
    \"quay-password\": \"YOUR_QUAY_ROBOT_TOKEN\",
    \"git-username\": \"YOUR_GITHUB_USERNAME\",
    \"git-password\": \"YOUR_GITHUB_PAT\"
  }"

# Save the webhook secret for GitHub configuration
echo "GitHub Webhook Secret: ${WEBHOOK_SECRET}"
```

#### Update Existing Secrets

```shell
# Update Developer Hub secrets
aws secretsmanager put-secret-value \
  --secret-id ocppe/developer-hub \
  --secret-string "{
    \"backend-secret\": \"${BACKEND_SECRET}\",
    \"oauth-client-secret\": \"${OAUTH_SECRET}\"
  }"

# Update CI Pipeline secrets
aws secretsmanager put-secret-value \
  --secret-id ocppe/ci-pipelines \
  --secret-string "{
    \"github-webhook-secret\": \"${WEBHOOK_SECRET}\",
    \"quay-username\": \"YOUR_QUAY_USERNAME\",
    \"quay-password\": \"YOUR_QUAY_ROBOT_TOKEN\",
    \"git-username\": \"YOUR_GITHUB_USERNAME\",
    \"git-password\": \"YOUR_GITHUB_PAT\"
  }"
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

The External Secrets Operator needs IAM permissions to access AWS Secrets Manager.

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

## CI Pipelines Setup

### 1. Verify External Secrets

After deployment, verify the secrets are synced:

```shell
# Check ExternalSecret status
oc get externalsecrets -n ci-pipelines

# Verify secrets were created
oc get secrets -n ci-pipelines
```

### 2. Get the Webhook URL

```shell
oc get route github-webhook -n ci-pipelines -o jsonpath='{.spec.host}'
```

### 3. Configure GitHub Webhooks

For each repository (customer-api, product-api, order-api):

1. Go to **Settings** → **Webhooks** → **Add webhook**
2. **Payload URL**: `https://<webhook-route-from-above>/`
3. **Content type**: `application/json`
4. **Secret**: Same value as `github-webhook-secret` from AWS Secrets Manager
5. **Events**: Select "Just the push event" or "Pull requests"

### 4. Supported Application Types

| Type    | Detection            | Build                    | Image Location              |
| ------- | -------------------- | ------------------------ | --------------------------- |
| Quarkus | pom.xml with quarkus | Maven package            | quay.io/ocppe/<app-name>    |
| Angular | angular.json         | (coming soon)            | quay.io/ocppe/<app-name>    |
| Node.js | package.json         | (coming soon)            | quay.io/ocppe/<app-name>    |

### 5. Manual Pipeline Run

```shell
cat <<EOF | oc create -f -
apiVersion: tekton.dev/v1
kind: PipelineRun
metadata:
  generateName: ci-pipeline-manual-
  namespace: ci-pipelines
spec:
  pipelineRef:
    name: ci-pipeline
  params:
    - name: git-url
      value: https://github.com/ocppe/customer-api.git
    - name: git-revision
      value: main
  workspaces:
    - name: shared-workspace
      volumeClaimTemplate:
        spec:
          accessModes:
            - ReadWriteOnce
          resources:
            requests:
              storage: 1Gi
    - name: dockerconfig
      secret:
        secretName: quay-credentials
  serviceAccountName: pipeline-sa
EOF
```

### 6. View Pipeline Runs

```shell
# List runs
oc get pipelineruns -n ci-pipelines

# View logs (requires tkn CLI)
tkn pipelinerun logs <run-name> -n ci-pipelines
```

## Secrets Reference

All secrets are stored in AWS Secrets Manager under the `ocppe/` prefix:

| AWS Secret Path       | Kubernetes Secret          | Namespace       | Purpose                    |
| --------------------- | -------------------------- | --------------- | -------------------------- |
| ocppe/developer-hub   | developer-hub-secrets      | rhdh            | Developer Hub OIDC/backend |
| ocppe/ci-pipelines    | github-webhook-secret      | ci-pipelines    | GitHub webhook validation  |
| ocppe/ci-pipelines    | quay-credentials           | ci-pipelines    | Container registry auth    |
| ocppe/ci-pipelines    | git-credentials            | ci-pipelines    | Git clone authentication   |

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

# Deploy CI Pipelines
oc apply -k cluster-config/pipelines/
```
