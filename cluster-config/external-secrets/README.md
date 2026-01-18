# External Secrets Configuration

ClusterSecretStore configuration for AWS Secrets Manager integration.

## Overview

This configuration creates a `ClusterSecretStore` named `aws-secrets-manager` that allows ExternalSecret resources in any namespace to fetch secrets from AWS Secrets Manager.

## AWS IAM Setup

The External Secrets Operator service account needs IAM permissions via OIDC federation.

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

## AWS Secrets Structure

All secrets are stored under the `ocppe/` prefix:

| Secret Path           | Keys                                              | Used By           |
| --------------------- | ------------------------------------------------- | ----------------- |
| ocppe/developer-hub   | backend-secret, oauth-client-secret               | Developer Hub     |
| ocppe/ci-pipelines    | github-webhook-secret, quay-username, quay-password, git-username, git-password | CI Pipelines |

## Create Secrets in AWS

```shell
# Developer Hub
BACKEND_SECRET=$(openssl rand -base64 32)
OAUTH_SECRET=$(openssl rand -base64 32)
aws secretsmanager create-secret \
  --name ocppe/developer-hub \
  --secret-string "{
    \"backend-secret\": \"${BACKEND_SECRET}\",
    \"oauth-client-secret\": \"${OAUTH_SECRET}\"
  }"

# CI Pipelines
WEBHOOK_SECRET=$(openssl rand -hex 20)
aws secretsmanager create-secret \
  --name ocppe/ci-pipelines \
  --secret-string "{
    \"github-webhook-secret\": \"${WEBHOOK_SECRET}\",
    \"quay-username\": \"YOUR_QUAY_USERNAME\",
    \"quay-password\": \"YOUR_QUAY_ROBOT_TOKEN\",
    \"git-username\": \"YOUR_GITHUB_USERNAME\",
    \"git-password\": \"YOUR_GITHUB_PAT\"
  }"
```

## Verify Setup

```shell
# Check ClusterSecretStore status
oc get clustersecretstore aws-secrets-manager

# Check ExternalSecrets in all namespaces
oc get externalsecrets -A

# Describe a specific ExternalSecret
oc describe externalsecret developer-hub-secrets -n rhdh
```

## Troubleshooting

### ClusterSecretStore shows "SecretStoreNotReady"

```shell
# Check the operator logs
oc logs -n external-secrets-operator -l app.kubernetes.io/name=external-secrets

# Verify the service account has the IAM annotation
oc get sa external-secrets-sa -n external-secrets-operator -o yaml
```

### ExternalSecret shows "SecretSyncedError"

```shell
# Check if the AWS secret exists
aws secretsmanager describe-secret --secret-id ocppe/developer-hub

# Verify IAM permissions
aws secretsmanager get-secret-value --secret-id ocppe/developer-hub
```
