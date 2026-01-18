# Red Hat Developer Hub Instance

Backstage-based internal developer portal with OpenShift OIDC authentication.

## Overview

- OpenShift OAuth integration for SSO
- Secrets managed via External Secrets Operator from AWS Secrets Manager
- Software catalog integration
- TechDocs support

## Setup

Secrets are created automatically via Ansible:

```shell
cd ansible
ansible-playbook playbooks/aws-secrets-setup.yml \
  -e quay_username=... \
  -e quay_password=... \
  -e git_username=... \
  -e git_password=...
```

After running, update `kustomization.yaml` with the OAuth secret:

```shell
source ../ansible/.secrets.env
echo "OAuth Secret: $OAUTH_CLIENT_SECRET"
# Update kustomization.yaml with this value
```

See the [ansible README](../../../../ansible/README.md) for complete instructions.

## Configuration

Edit `kustomization.yaml` to set your environment values:

```yaml
patches:
  - patch: |-
      - op: replace
        path: /redirectURIs/0
        value: https://backstage-developer-hub-rhdh.apps.hub.ocp.sandbox${SANDBOX_ID}.opentlc.com/api/auth/oidc/handler/frame
      - op: replace
        path: /secret
        value: <OAUTH_CLIENT_SECRET from .secrets.env>
    target:
      kind: OAuthClient
      name: developer-hub
```

## Components

| File                 | Description                                         |
| -------------------- | --------------------------------------------------- |
| oauth-client.yaml    | OpenShift OAuthClient for OIDC authentication       |
| external-secret.yaml | ExternalSecret for syncing from AWS Secrets Manager |
| app-config.yaml      | Backstage configuration ConfigMap                   |
| backstage.yaml       | Backstage CR for deploying Developer Hub            |

## Verify Deployment

```shell
# Check ExternalSecret
oc get externalsecret developer-hub-secrets -n rhdh

# Check Backstage deployment
oc get backstage developer-hub -n rhdh

# Get route URL
oc get route backstage-developer-hub -n rhdh -o jsonpath='{.spec.host}'
```

## Troubleshooting

### OAuth Login Fails

```shell
# Verify OAuthClient secret
oc get oauthclient developer-hub -o jsonpath='{.secret}'

# Check redirect URI matches cluster domain
```

### Backstage Not Starting

```shell
oc logs -n rhdh -l app.kubernetes.io/name=backstage
oc describe externalsecret developer-hub-secrets -n rhdh
```
