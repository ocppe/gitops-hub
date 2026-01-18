# CI Pipelines

Centralized Tekton pipelines for building, testing, and containerizing applications.

## Overview

This setup provides:

- **GitHub webhook integration** via Tekton Triggers
- **Auto-detection** of application type (Quarkus, Angular, Node.js)
- **Build, test, and containerize** workflow
- **Push to quay.io/ocppe** container registry

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

## Supported Application Types

| Type    | Detection              | Build Command                |
| ------- | ---------------------- | ---------------------------- |
| Quarkus | pom.xml with quarkus   | ./mvnw package               |
| Angular | angular.json           | (coming soon)                |
| Node.js | package.json           | (coming soon)                |

## Setup Instructions

### 1. Configure Secrets

Edit `base/secrets.yaml` with your credentials:

```yaml
# GitHub webhook secret
webhook-secret: "<generate with: openssl rand -hex 20>"

# Quay.io credentials (base64 encoded user:token)
auth: "<echo -n 'username:token' | base64>"

# GitHub PAT for cloning private repos
username: "<github-username>"
password: "<github-pat>"
```

### 2. Deploy via ArgoCD

The pipelines are deployed automatically via ArgoCD when using the app-of-apps pattern.

For manual deployment:

```shell
oc apply -k cluster-config/pipelines/
```

### 3. Get Webhook URL

```shell
oc get route github-webhook -n ci-pipelines -o jsonpath='{.spec.host}'
```

### 4. Configure GitHub Webhook

In your GitHub repository settings:

1. Go to **Settings** → **Webhooks** → **Add webhook**
2. **Payload URL**: `https://<webhook-route>/`
3. **Content type**: `application/json`
4. **Secret**: Use the same value as `webhook-secret`
5. **Events**: Select "Just the push event" or "Pull requests"

## Directory Structure

```
pipelines/
├── base/
│   ├── namespace.yaml       # Namespace and RBAC
│   ├── secrets.yaml         # Credentials (update before deploy)
│   └── kustomization.yaml
├── tasks/
│   ├── detect-app-type.yaml # Detects Quarkus/Angular/Node.js
│   ├── build-quarkus.yaml   # Maven build
│   ├── test-quarkus.yaml    # Maven test
│   ├── build-push-image.yaml# Buildah build and push
│   └── kustomization.yaml
├── pipelines/
│   ├── ci-pipeline.yaml     # Main CI pipeline
│   ├── quarkus-build-pipeline.yaml
│   └── kustomization.yaml
├── triggers/
│   ├── trigger-binding.yaml # GitHub payload parsing
│   ├── trigger-template.yaml# PipelineRun template
│   ├── event-listener.yaml  # Webhook endpoint
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
