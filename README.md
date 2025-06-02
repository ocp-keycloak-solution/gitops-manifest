# OpenShift Keycloak Manifest

## Bootstrap ArgoCD
```
oc apply -k bootstrap/operator/
```
```
oc apply -k bootstrap/instance/
```

## Deploy Manifest
```
oc apply -f application.yaml -n openshift-gitops
```