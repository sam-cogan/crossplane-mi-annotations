# Minimal Crossplane Demo - Annotate ServiceAccount with MI ClientId

This is an example deployment to demonstrate Crossplane patching the managed identity's clientId onto a ServiceAccount annotation in a single deployment.

## What This Does

1. Creates an Azure Managed Identity
2. Gets the `clientId` from the MI's status
3. Patches it onto a ServiceAccount annotation `azure.workload.identity/client-id` and creates the service account

## Prerequisites

You need:
- Crossplane installed
- `provider-azure-managedidentity` installed and configured
- `provider-kubernetes` installed and configured
- `function-patch-and-transform` and `function-auto-ready` functions installed

Quick install if you don't have these:

```bash
# Crossplane
helm repo add crossplane-stable https://charts.crossplane.io/stable
helm install crossplane crossplane-stable/crossplane -n crossplane-system --create-namespace

# Providers
kubectl apply -f - <<EOF
apiVersion: pkg.crossplane.io/v1
kind: Provider
metadata:
  name: provider-azure-managedidentity
spec:
  package: xpkg.upbound.io/upbound/provider-azure-managedidentity:v1.3.1
---
apiVersion: pkg.crossplane.io/v1
kind: Provider
metadata:
  name: provider-kubernetes
spec:
  package: xpkg.upbound.io/crossplane-contrib/provider-kubernetes:v0.14.0
---
apiVersion: pkg.crossplane.io/v1beta1
kind: Function
metadata:
  name: function-patch-and-transform
spec:
  package: xpkg.upbound.io/crossplane-contrib/function-patch-and-transform:v0.6.0
---
apiVersion: pkg.crossplane.io/v1beta1
kind: Function
metadata:
  name: function-auto-ready
spec:
  package: xpkg.upbound.io/crossplane-contrib/function-auto-ready:v0.3.0
EOF

# Configure Azure provider (replace with your credentials)
kubectl create secret generic azure-secret -n crossplane-system \
  --from-literal=creds='{"clientId":"...","clientSecret":"...","subscriptionId":"...","tenantId":"..."}'

kubectl apply -f - <<EOF
apiVersion: azure.upbound.io/v1beta1
kind: ProviderConfig
metadata:
  name: default
spec:
  credentials:
    source: Secret
    secretRef:
      namespace: crossplane-system
      name: azure-secret
      key: creds
EOF

# Configure Kubernetes provider
kubectl apply -f - <<EOF
apiVersion: kubernetes.crossplane.io/v1alpha1
kind: ProviderConfig
metadata:
  name: kubernetes-provider
spec:
  credentials:
    source: InjectedIdentity
EOF
```

## Deploy

```bash
# 1. Apply the XRD (defines the API)
kubectl apply -f minimal-xrd.yaml

# 2. Apply the Composition (defines what gets created)
kubectl apply -f minimal-composition.yaml

# 3. Create a claim
kubectl apply -f minimal-claim.yaml
```

## Watch It Work

```bash
# Watch the claim
kubectl get workloadidentity my-test -n default -w

# After 30-90 seconds, you'll see clientId populated:
# NAME      CLIENTID                              AGE
# my-test   12345678-1234-1234-1234-123456789abc  2m

# Check the ServiceAccount got the annotation
kubectl get serviceaccount identity-binding-test-sa -n identity-binding-test -o yaml
```

You should see:
```yaml
metadata:
  annotations:
    azure.workload.identity/client-id: "12345678-1234-1234-1234-123456789abc"
    azure.workload.identity/tenant-id: "bdfd0425-c7d4-44bc-89ea-67232d9e215d"
```

## How It Works

1. Composition creates `UserAssignedIdentity` in Azure
2. Azure provisions the MI and returns `status.atProvider.clientId`
3. Composition patches `clientId` from MI status → XR status (`ToCompositeFieldPath`)
4. Composition patches `clientId` from XR status → ServiceAccount annotation (`FromCompositeFieldPath`)
5. `policy.fromFieldPath: Required` prevents SA creation until `clientId` exists

This eliminates the timing gap - the ServiceAccount won't be created until the MI is ready and has a clientId.

## Clean Up

```bash
kubectl delete workloadidentity my-test -n default
```

Crossplane will delete the managed identity and ServiceAccount automatically.
