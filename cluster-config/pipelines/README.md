# CI Pipelines

Centralized Tekton pipelines for building, testing, and containerizing applications.

## Overview

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

## Setup

All setup is automated via Ansible. See [ansible/README.md](../../../../ansible/README.md).

```shell
cd ansible

# 1. Setup Quay.io (creates org, robot account, repos)
export QUAY_TOKEN=your_quay_oauth_token
ansible-playbook playbooks/quay-setup.yml

# 2. Setup AWS Secrets Manager
source .quay-credentials.env
ansible-playbook playbooks/aws-secrets-setup.yml \
  -e quay_username="${QUAY_USERNAME}" \
  -e quay_password="${QUAY_PASSWORD}" \
  -e git_username=YOUR_GITHUB_USERNAME \
  -e git_password=YOUR_GITHUB_PAT

# 3. Deploy pipelines via ArgoCD (automatic with app-of-apps)

# 4. Setup GitHub webhooks
ansible-playbook playbooks/github-webhooks-setup.yml
```

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
│   └── build-push-image.yaml # Buildah build and push
├── pipelines/
│   ├── ci-pipeline.yaml      # Main CI pipeline
│   └── quarkus-build-pipeline.yaml
└── triggers/
    ├── trigger-binding.yaml  # GitHub payload parsing
    ├── trigger-template.yaml # PipelineRun template
    └── event-listener.yaml   # Webhook endpoint
```

## Supported Application Types

| Type    | Detection              | Build Command    |
| ------- | ---------------------- | ---------------- |
| Quarkus | pom.xml with quarkus   | ./mvnw package   |
| Angular | angular.json           | (coming soon)    |
| Node.js | package.json           | (coming soon)    |

## Manual Pipeline Run

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
          accessModes: [ReadWriteOnce]
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
# List runs
oc get pipelineruns -n ci-pipelines

# View logs
tkn pipelinerun logs <run-name> -n ci-pipelines

# OpenShift Console: Pipelines → PipelineRuns
```

## Troubleshooting

### External Secrets Not Syncing

```shell
oc describe externalsecret github-webhook-secret -n ci-pipelines
oc describe clustersecretstore aws-secrets-manager
```

### Webhook Not Triggering

```shell
oc logs -l eventlistener=github-webhook-listener -n ci-pipelines
```

### "Invalid character 'p'" Error

Content-Type is wrong. Re-run the webhook setup:
```shell
ansible-playbook playbooks/github-webhooks-setup.yml
```

### 403 Forbidden on Webhook

Webhook secret mismatch:
```shell
oc get secret github-webhook-secret -n ci-pipelines -o jsonpath='{.data.webhook-secret}' | base64 -d
```
