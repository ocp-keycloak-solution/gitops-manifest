# OpenShift Keycloak OAuth Integration

This directory contains Kustomize manifests to integrate OpenShift and ArgoCD with Keycloak for single sign-on authentication.

## Overview

This integration provides:
- **OpenShift OAuth** integration with Keycloak as OIDC provider
- **ArgoCD RBAC** configuration with Keycloak groups
- **Keycloak Realm** with OpenShift-specific configuration
- **Keycloak Clients** for both OpenShift and ArgoCD
- **Group synchronization** between Keycloak and OpenShift

## Components

### 1. OAuth Configuration (`oauth-config.yaml`)
- Configures OpenShift to use Keycloak as OIDC identity provider
- Maps Keycloak user attributes to OpenShift user claims
- Includes groups claim for RBAC

### 2. Keycloak Realm (`keycloak-realm.yaml`)
- Creates "openshift" realm in Keycloak
- Defines roles: `openshift-admin`, `openshift-developer`, `argo-admin`, `argo-readonly`
- Creates groups: `cluster-admins`, `developers`
- Includes default admin user

### 3. Keycloak Clients (`keycloak-clients.yaml`)
- **OpenShift Client**: OAuth client for OpenShift console authentication
- **ArgoCD Client**: OAuth client for ArgoCD authentication
- Both clients include proper redirect URIs and protocol mappers

### 4. ArgoCD RBAC (`argo-rbac-config.yaml`)
- Configures ArgoCD to use Keycloak for authentication
- Maps Keycloak groups to ArgoCD roles
- `cluster-admins` → `role:admin`
- `developers` → `role:readonly`

### 5. Group Sync (`group-sync.yaml`)
- CronJob to synchronize groups from Keycloak to OpenShift
- Ensures proper RBAC mapping in OpenShift
- Runs every 15 minutes

## Deployment

### Prerequisites
1. Keycloak instance deployed and accessible
2. OpenShift GitOps (ArgoCD) installed
3. Proper DNS resolution for Keycloak hostname

### Deploy
```bash
# Apply the ArgoCD application
oc apply -f application.yaml

# Or deploy directly with Kustomize
oc apply -k .
```

### Environment-Specific Configuration

Use overlays for different environments:

```bash
# Development environment
oc apply -k overlays/dev/
```

## Post-Deployment Configuration

### 1. Update Client Secrets
Replace placeholder secrets in:
- `oauth-config.yaml`: `clientSecret`
- `keycloak-clients.yaml`: Client secrets for both clients
- `argo-rbac-config.yaml`: ArgoCD OIDC client secret

### 2. Update Hostnames
Modify hostname patches in overlays to match your cluster domain:
- `oauth-hostname-patch.yaml`
- `keycloak-hostname-patch.yaml`

### 3. Create Users in Keycloak
1. Access Keycloak admin console
2. Navigate to "openshift" realm
3. Create users and assign to appropriate groups:
   - Add cluster administrators to `cluster-admins` group
   - Add developers to `developers` group

## Testing the Integration

### 1. OpenShift Console
1. Navigate to OpenShift console
2. Click "keycloak" login option
3. Authenticate with Keycloak credentials
4. Verify proper role assignment

### 2. ArgoCD
1. Navigate to ArgoCD UI
2. Click "LOG IN VIA KEYCLOAK"
3. Authenticate with Keycloak credentials
4. Verify access based on group membership

## RBAC Mapping

| Keycloak Group | OpenShift Role | ArgoCD Role | Permissions |
|----------------|----------------|-------------|-------------|
| cluster-admins | cluster-admin  | role:admin  | Full cluster and ArgoCD admin |
| developers     | basic-user     | role:readonly | Limited cluster access, ArgoCD read-only |

## Troubleshooting

### Common Issues

1. **Login redirect fails**
   - Check redirect URIs in Keycloak clients
   - Verify hostname configuration

2. **Groups not syncing**
   - Check group-sync CronJob logs
   - Verify Keycloak group configuration

3. **ArgoCD authentication fails**
   - Check OIDC configuration in ArgoCD
   - Verify client secret is correct

### Useful Commands

```bash
# Check OAuth configuration
oc get oauth cluster -o yaml

# Check ArgoCD configuration
oc get configmap argocd-cmd-params-cm -n openshift-gitops -o yaml

# Check group sync logs
oc logs -l job-name=keycloak-group-sync -n openshift-config

# List OpenShift groups
oc get groups
```

## Security Considerations

1. **Client Secrets**: Store in proper secret management system
2. **TLS**: Ensure all communications use HTTPS
3. **Token Expiry**: Configure appropriate token lifetimes in Keycloak
4. **Group Management**: Regularly audit group memberships
5. **Monitoring**: Monitor authentication attempts and failures

## Customization

### Adding New Roles
1. Add role to `keycloak-realm.yaml`
2. Update RBAC policies in `argo-rbac-config.yaml`
3. Create corresponding OpenShift ClusterRole if needed

### Additional OIDC Clients
Create new KeycloakClient resources following the same pattern as existing clients.
