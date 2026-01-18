# External Secrets Configuration

ClusterSecretStore configuration for AWS Secrets Manager integration.

## Overview

This configuration creates a `ClusterSecretStore` named `aws-secrets-manager` that allows ExternalSecret resources in any namespace to fetch secrets from AWS Secrets Manager.

## Setup

IAM resources and secrets are created automatically via Ansible:

```shell
cd ansible
ansible-playbook playbooks/aws-secrets-setup.yml \
  -e quay_username=... \
  -e quay_password=... \
  -e git_username=... \
  -e git_password=...
```

See the [ansible README](../../../../ansible/README.md) for complete instructions.

## AWS Secrets Structure

| Secret Path           | Keys                                              | Used By           |
| --------------------- | ------------------------------------------------- | ----------------- |
| ocppe/developer-hub   | backend-secret, oauth-client-secret               | Developer Hub     |
| ocppe/ci-pipelines    | github-webhook-secret, quay-username, quay-password, git-username, git-password | CI Pipelines |

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
# Check operator logs
oc logs -n external-secrets-operator -l app.kubernetes.io/name=external-secrets

# Verify service account has IAM annotation
oc get sa external-secrets-sa -n external-secrets-operator -o yaml | grep eks.amazonaws.com
```

### ExternalSecret shows "SecretSyncedError"

```shell
# Check if AWS secret exists
aws secretsmanager describe-secret --secret-id ocppe/developer-hub

# Verify IAM permissions
aws secretsmanager get-secret-value --secret-id ocppe/developer-hub
```
