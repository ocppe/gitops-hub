# Red Hat Developer Hub Instance

Backstage-based internal developer portal with OpenShift OIDC authentication.

## Overview

This configuration deploys Red Hat Developer Hub with:

- OpenShift OAuth integration for SSO
- Secrets managed via External Secrets Operator from AWS Secrets Manager
- Software catalog integration
- TechDocs support

## Prerequisites

### 1. Create AWS Secrets

```shell
# Generate secrets
BACKEND_SECRET=$(openssl rand -base64 32)
OAUTH_SECRET=$(openssl rand -base64 32)

# Create in AWS Secrets Manager
aws secretsmanager create-secret \
  --name ocppe/developer-hub \
  --secret-string "{
    \"backend-secret\": \"${BACKEND_SECRET}\",
    \"oauth-client-secret\": \"${OAUTH_SECRET}\"
  }"

# Save OAuth secret - you'll need it for the kustomization patch
echo "OAuth Client Secret: ${OAUTH_SECRET}"
```

### 2. Update kustomization.yaml

Edit `kustomization.yaml` to set your environment values:

```yaml
patches:
  - patch: |-
      - op: replace
        path: /redirectURIs/0
        value: https://backstage-developer-hub-rhdh.apps.hub.ocp.sandbox${SANDBOX_ID}.opentlc.com/api/auth/oidc/handler/frame
      - op: replace
        path: /secret
        value: <OAUTH_SECRET from step 1>
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

## Secrets Reference

The ExternalSecret syncs these values from `ocppe/developer-hub` in AWS Secrets Manager:

| AWS Property         | K8s Secret Key     | Purpose                          |
| -------------------- | ------------------ | -------------------------------- |
| backend-secret       | BACKEND_SECRET     | Backstage backend session secret |
| oauth-client-secret  | OAUTH_CLIENT_SECRET| OpenShift OAuth client secret    |

## Verify Deployment

```shell
# Check ExternalSecret status
oc get externalsecret developer-hub-secrets -n rhdh

# Verify secret was created
oc get secret developer-hub-secrets -n rhdh

# Check Backstage deployment
oc get backstage developer-hub -n rhdh

# Get the route URL
oc get route backstage-developer-hub -n rhdh -o jsonpath='{.spec.host}'
```

## Access Developer Hub

After deployment, access the Developer Hub at:

```
https://backstage-developer-hub-rhdh.apps.hub.ocp.sandbox${SANDBOX_ID}.opentlc.com
```

Login using your OpenShift credentials.

## Troubleshooting

### OAuth Login Fails

1. Verify the OAuthClient secret matches the AWS secret:
   ```shell
   oc get oauthclient developer-hub -o jsonpath='{.secret}'
   ```

2. Check the redirect URI matches your cluster domain

### Backstage Not Starting

```shell
# Check pod logs
oc logs -n rhdh -l app.kubernetes.io/name=backstage

# Verify secrets are synced
oc describe externalsecret developer-hub-secrets -n rhdh
```
