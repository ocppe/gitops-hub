# CI Pipelines

Centralized Tekton pipelines for building, testing, and containerizing applications.

## Overview

This setup provides:

- **GitHub webhook integration** via Tekton Triggers
- **Auto-detection** of application type (Quarkus, Angular, Node.js)
- **Build, test, and containerize** workflow
- **Push to quay.io/ocppe** container registry
- **Secrets managed via External Secrets Operator** from AWS Secrets Manager

## Architecture

```
GitHub Webhook
      │
      ▼
┌─────────────────┐
│ EventListener   │  ← Receives push/PR events
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ TriggerBinding  │  ← Extracts params from payload
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ TriggerTemplate │  ← Creates PipelineRun
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ CI Pipeline     │
│  ├─ git-clone   │
│  ├─ detect-type │
│  ├─ test        │
│  ├─ build       │
│  └─ push-image  │
└─────────────────┘
         │
         ▼
    quay.io/ocppe
```

## Prerequisites

### AWS Secrets Manager Setup

Create the CI pipeline secrets in AWS Secrets Manager:

```shell
# Generate a webhook secret
WEBHOOK_SECRET=$(openssl rand -hex 20)

# Create the secret in AWS Secrets Manager
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

To update an existing secret:

```shell
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

### Quay.io Robot Account

1. Log in to [quay.io](https://quay.io)
2. Create organization `ocppe` (or use existing)
3. Go to **Organization Settings** → **Robot Accounts**
4. Create a robot account with write permissions
5. Use the robot account credentials in the secret above

### GitHub Personal Access Token

Create a PAT with these scopes:
- `repo` (for private repositories)
- `read:org` (if using organization repos)

## Supported Application Types

| Type    | Detection              | Build Command                |
| ------- | ---------------------- | ---------------------------- |
| Quarkus | pom.xml with quarkus   | ./mvnw package               |
| Angular | angular.json           | (coming soon)                |
| Node.js | package.json           | (coming soon)                |

## Setup Instructions

### 1. Create AWS Secrets (see above)

### 2. Deploy via ArgoCD

The pipelines are deployed automatically via ArgoCD when using the app-of-apps pattern.

For manual deployment:

```shell
oc apply -k cluster-config/pipelines/
```

### 3. Verify External Secrets

```shell
# Check ExternalSecret status
oc get externalsecrets -n ci-pipelines

# Verify secrets were created
oc get secrets -n ci-pipelines
```

### 4. Get Webhook URL

```shell
oc get route github-webhook -n ci-pipelines -o jsonpath='{.spec.host}'
```

### 5. Configure GitHub Webhook

In your GitHub repository settings:

1. Go to **Settings** → **Webhooks** → **Add webhook**
2. **Payload URL**: `https://<webhook-route>/`
3. **Content type**: `application/json`
4. **Secret**: Use the `WEBHOOK_SECRET` value from AWS setup
5. **Events**: Select "Just the push event" or "Pull requests"

## Directory Structure

```
pipelines/
├── base/
│   ├── namespace.yaml        # Namespace and RBAC
│   ├── external-secrets.yaml # ExternalSecret definitions
│   └── kustomization.yaml
├── tasks/
│   ├── detect-app-type.yaml  # Detects Quarkus/Angular/Node.js
│   ├── build-quarkus.yaml    # Maven build
│   ├── test-quarkus.yaml     # Maven test
│   ├── build-push-image.yaml # Buildah build and push
│   └── kustomization.yaml
├── pipelines/
│   ├── ci-pipeline.yaml      # Main CI pipeline
│   ├── quarkus-build-pipeline.yaml
│   └── kustomization.yaml
├── triggers/
│   ├── trigger-binding.yaml  # GitHub payload parsing
│   ├── trigger-template.yaml # PipelineRun template
│   ├── event-listener.yaml   # Webhook endpoint
│   └── kustomization.yaml
└── kustomization.yaml
```

## Manual Pipeline Run

To manually trigger a pipeline run:

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
    - name: image-registry
      value: quay.io
    - name: image-repository
      value: ocppe
    - name: run-tests
      value: "true"
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

## Viewing Pipeline Runs

```shell
# List pipeline runs
oc get pipelineruns -n ci-pipelines

# View logs
tkn pipelinerun logs <pipelinerun-name> -n ci-pipelines

# View in OpenShift Console
# Navigate to Pipelines → PipelineRuns
```

## Troubleshooting

### External Secrets Not Syncing

```shell
# Check ExternalSecret status
oc describe externalsecret github-webhook-secret -n ci-pipelines

# Check ClusterSecretStore
oc describe clustersecretstore aws-secrets-manager
```

### Webhook Not Triggering

```shell
# Check EventListener logs
oc logs -l eventlistener=github-webhook-listener -n ci-pipelines

# Verify webhook secret matches
oc get secret github-webhook-secret -n ci-pipelines -o jsonpath='{.data.webhook-secret}' | base64 -d
```
